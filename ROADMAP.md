# cn-variants Roadmap

Possible additions and changes under consideration. Nothing here is committed to.

## `.key` property on variants

Add a `.key` property that maps each variant key to itself as a string literal. This gives type-safe key references for use in compound variant conditionals.

```ts
const buttonVariant = variants({
  primary: "bg-indigo-600 text-white",
  danger: "bg-red-600 text-white",
});

// buttonVariant.key.primary === "primary" (type: "primary", not string)
cn(
  buttonVariant(variant),
  buttonSize(size),
  variant === buttonVariant.key.primary && size === buttonSize.key.lg && "uppercase tracking-wide",
);
```

Without this, renaming a variant key only breaks the `variants()` call site. The string `"primary"` in compound conditionals silently goes stale. With `.key`, TypeScript catches every reference.

Implementation:

```ts
export function variants<T extends Record<string, string>>(map: T) {
  const fn = (key: keyof T): string => map[key] ?? "";
  fn.options = Object.freeze(map);
  fn.key = Object.freeze(
    Object.keys(map).reduce((acc, k) => ({ ...acc, [k]: k }), {} as { [K in keyof T]: K }),
  );
  return fn;
}
```

### Counterpoint: existing types already cover this

TypeScript already guards the `variants()` call site, and the string `"primary"` in a conditional is already type-checked as long as the prop is typed against the variant keys.

```ts
type ButtonVariant = keyof typeof buttonVariant.options;

function Button({ variant = "primary" }: { variant?: ButtonVariant }) {
  // "primary" here is already type-checked against ButtonVariant.
  // If you rename the key, this string literal errors too.
  variant === "primary" && ...
}
```

The raw string `"primary"` is safe because `variant` is typed as `ButtonVariant`. If you rename the key in the `variants()` call, TypeScript flags every `"primary"` comparison that no longer matches the union. The `.key` property would only help in untyped contexts or when the comparison variable isn't typed against the variant keys.

This may mean `.key` adds API surface without meaningful safety gains for the typical use case.

### Naming

**`.option`** (singular) is another candidate. It pairs naturally with the existing `.options` (plural): `.options` gives you the full map, `.option` gives you a single key back.

```ts
variant === buttonVariant.option.primary &&
  size === buttonSize.option.lg &&
  "uppercase tracking-wide";
```

This reads as "the primary option of buttonVariant" and avoids introducing a new concept. `.key` works but is generic. `.option` ties into the vocabulary already established by the API.

Other candidates considered: `.name`, `.keys`, `.variant`. `.name` collides with `Function.name`. `.keys` suggests an array (like `Object.keys`). `.variant` is redundant with the variable name.

### Performance

The `.key`/`.option` object is built once per `variants()` call at module initialization, not per render. For a typical variant map with 2-5 keys, the cost is a single `Object.keys` + `reduce` + `Object.freeze`. This is negligible.

However, it does add a second frozen object allocation per `variants()` call (on top of `.options`). For a library that prides itself on being tiny, every allocation is worth questioning. The runtime comparison `variant === "primary"` is a string equality check either way, whether the right-hand side is a literal or a property access. There is no performance benefit at the call site.

The real cost is bundle size. The `.key`/`.option` implementation adds ~100 bytes of unminified code to the `variants` function. After minification and gzip this is close to zero, but it's not zero. For a utility whose selling point is minimalism, even small additions shift the character of the library.

If added, the object should be lazily created (via getter) to avoid the allocation cost when the feature isn't used:

```ts
let _key: { [K in keyof T]: K };
Object.defineProperty(fn, "key", {
  get() {
    return (_key ??= Object.freeze(
      Object.keys(map).reduce((acc, k) => ({ ...acc, [k]: k }), {} as { [K in keyof T]: K }),
    ));
  },
});
```

This way the mirror object is never built unless someone accesses `.key`. The tradeoff is slightly more complex implementation for a feature that may not be needed at all.

**Open question:** Does this feature carry its weight given existing type coverage, and if so, is `.option` or `.key` the better name?

---

## Custom tailwind-merge configuration

tailwind-merge supports custom configs for projects with Tailwind plugins that add new utilities (e.g. `animate-*` from a custom plugin). Currently `cn` hardcodes the default `twMerge`. A factory function would let users pass their own config:

```ts
import { createCn } from "cn-variants";
import { extendTailwindMerge } from "tailwind-merge";

const twMerge = extendTailwindMerge({
  extend: {
    classGroups: {
      animate: [{ animate: ["fade-in", "slide-up"] }],
    },
  },
});

export const cn = createCn(twMerge);
```

### Arguments for

Projects using Tailwind plugins (daisyUI, custom design systems) need custom merge behavior. Without a factory, users have to reimplement `cn` themselves, which defeats the purpose of the library.

### Arguments against

This adds a second export and a second concept. The current API is two functions. A factory makes it three, and introduces a setup step. Most projects use default Tailwind and never need this. Users who do need custom merging are advanced enough to write their own one-liner.

If added, the default `cn` should remain unchanged (zero-config). `createCn` would be an opt-in escape hatch.

**Open question:** Is the factory worth the added surface, or should custom merging stay out of scope?

---

## Re-exporting `ClassValue` from clsx

Users who write wrapper functions around `cn` need the `ClassValue` type:

```ts
import { type ClassValue, cn } from "cn-variants";

function card(...classes: ClassValue[]) {
  return cn("rounded-lg shadow-md", ...classes);
}
```

Currently they have to install clsx separately just for the type import, even though cn-variants already depends on it. Re-exporting `ClassValue` from the package entry point would fix this.

### Arguments for

It's a type-only re-export. Zero runtime cost, zero bundle impact. Saves users from adding clsx as a direct dependency for a single type.

### Arguments against

It couples cn-variants's public API to clsx's type names. If clsx renames or restructures `ClassValue` in a future major, cn-variants would need to absorb that as a breaking change or shim it. That said, `ClassValue` has been stable across every clsx major so far.

**Recommendation:** Likely worth doing. The cost is near zero and the ergonomic benefit is clear.

---

## Peer dependencies vs direct dependencies

clsx and tailwind-merge are currently direct dependencies. This means cn-variants controls the exact versions installed. An alternative is to make them peer dependencies.

### Direct deps (current)

- Users install `cn-variants` and it just works. No version management needed.
- Risk of duplicate copies if the consumer also depends on clsx or tailwind-merge directly. Bundlers typically deduplicate, but not always (especially with differing version ranges).

### Peer deps

- Consumer controls the versions. Only one copy in the tree.
- Requires users to install three packages instead of one. The "just `npm install cn-variants`" story breaks.
- Version mismatches between cn-variants's expectations and the consumer's installed version can cause subtle bugs.

**Recommendation:** Keep direct dependencies. cn-variants is a convenience wrapper. Asking users to manually install and version-match two sub-dependencies undermines the "tiny utility" pitch. Deduplication is the bundler's job.

---

## Tree-shaking

`cn` and `variants` are independent. `cn` depends on clsx + tailwind-merge. `variants` has zero dependencies. A user who only wants `variants` should not pay the cost of bundling clsx and tailwind-merge.

Currently both are in separate files (`cn.ts`, `variants.ts`) re-exported through `index.ts`. Modern bundlers (Rolldown, esbuild, Rollup) tree-shake named exports from barrel files, so `import { variants } from "cn-variants"` should already drop `cn.ts` and its dependencies from the bundle.

**Action item:** Verify this works in practice by checking the bundle output of a project that only imports `variants`. If barrel-file tree-shaking fails in any common bundler, consider adding a `cn-variants/variants` subpath export as a fallback.

---

## Upstream versioning policy

cn-variants depends on clsx and tailwind-merge. When either ships a breaking change, cn-variants needs a policy.

### Patch/minor releases upstream

Absorbed automatically via `^` ranges in `dependencies`. No action needed.

### Major releases upstream

A new major of tailwind-merge could change merge behavior (e.g. how conflicting utilities are resolved). A new major of clsx could change how inputs are processed. Either would change cn-variants's observable behavior without any code change in cn-variants itself.

**Proposed policy:**

- Pin to the current major of each dependency (`"clsx": "^2"`, `"tailwind-merge": "^3"`).
- When an upstream major ships, test cn-variants against it. If behavior changes, release a new cn-variants major that bumps the dependency range.
- Document supported upstream version ranges in the README.

This keeps cn-variants's semver honest: if `cn("px-2", "px-4")` returns a different result because of a tailwind-merge update, that's a breaking change from the consumer's perspective and should be a cn-variants major.
