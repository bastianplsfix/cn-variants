# cn-variants

Tiny utilities for working with Tailwind CSS class names. Combines [clsx](https://github.com/lukeed/clsx) + [tailwind-merge](https://github.com/dcastil/tailwind-merge) with a typed `variants` helper.

## Install

```bash
npm install cn-variants
```

## `cn(...inputs)`

Merges class names using clsx and tailwind-merge. Handles conditionals, duplicates, and Tailwind conflicts.

```ts
import { cn } from "cn-variants";

// Tailwind conflict resolution — last value wins
cn("px-4 py-2", "px-6");
// → "py-2 px-6"

// Conditionals
cn("text-red-500", isActive && "text-blue-500");
// → "text-blue-500" (when isActive is true)

// Object syntax
cn("flex", { "gap-4": hasGap, "items-center": centered });
// → "flex gap-4 items-center" (when both are true)

// Arrays
cn(["rounded-lg", "shadow-md"], "p-4");
// → "rounded-lg shadow-md p-4"

// Mixed — all clsx input types work
cn("base", ["flex", "gap-2"], { "font-bold": isActive }, undefined, null, false);
// → "base flex gap-2 font-bold" (when isActive is true)
```

## `variants(map)`

Creates a typed lookup function for Tailwind class variants. Returns `""` for unknown keys at runtime, relying on TypeScript for compile-time safety.

```ts
import { cn, variants } from "cn-variants";

const buttonVariant = variants({
  primary: "bg-indigo-600 text-white border-none",
  secondary: "bg-transparent text-indigo-600 border border-indigo-600",
  danger: "bg-red-600 text-white border-none",
});

const buttonSize = variants({
  sm: "px-3 py-1 text-xs",
  md: "px-4 py-2 text-sm",
  lg: "px-6 py-3 text-base",
});
```

The returned function exposes a frozen `.options` object with the original map, useful for deriving union types:

```ts
type ButtonVariant = keyof typeof buttonVariant.options;
// → "primary" | "secondary" | "danger"

type ButtonSize = keyof typeof buttonSize.options;
// → "sm" | "md" | "lg"
```

### Using variants with `cn` in components

```tsx
interface ButtonProps {
  variant?: ButtonVariant;
  size?: ButtonSize;
  className?: string;
  children: React.ReactNode;
}

export function Button({ variant = "primary", size = "md", className, children }: ButtonProps) {
  return (
    <button
      className={cn("rounded-md font-medium", buttonVariant(variant), buttonSize(size), className)}
    >
      {children}
    </button>
  );
}
```

Callers can override any style through `className` — tailwind-merge ensures the caller's classes win:

```tsx
<Button variant="primary" className="bg-purple-600">
  {/* bg-purple-600 overrides the primary variant's bg-indigo-600 */}
</Button>
```

### Compound variants

For styles that depend on a combination of variants, use conditionals in `cn`:

```tsx
cn(
  "rounded-md font-medium",
  buttonVariant(variant),
  buttonSize(size),
  variant === "primary" && size === "lg" && "uppercase tracking-wide",
  variant === "danger" && "ring-2 ring-red-300",
  className,
);
```

### `ClassValue` type

If you write wrapper functions around `cn`, you can import the `ClassValue` type directly:

```ts
import { type ClassValue, cn } from "cn-variants";

function card(...classes: ClassValue[]) {
  return cn("rounded-lg border bg-white shadow-sm", ...classes);
}
```

## Tailwind IntelliSense

To get autocomplete for class strings inside `variants()` and `cn()`, add them to the `classFunctions` setting in your editor's Tailwind CSS configuration.

### VS Code

Install the [Tailwind CSS IntelliSense](https://marketplace.visualstudio.com/items?itemName=bradlc.vscode-tailwindcss) extension, then add to your `.vscode/settings.json`:

```json
{
  "tailwindCSS.classFunctions": ["cn", "variants"]
}
```

### Zed

Add to your project's `.zed/settings.json`:

```json
{
  "lsp": {
    "tailwindcss-language-server": {
      "settings": {
        "classFunctions": ["cn", "variants"]
      }
    }
  }
}
```

### IntelliJ IDEA / WebStorm

Install the [Tailwind CSS](https://plugins.jetbrains.com/plugin/15321-tailwind-css) plugin, then add to your `tailwind.config.js`:

```js
module.exports = {
  classFunctions: ["cn", "variants"],
};
```

## Tree-shaking

`cn` and `variants` are independent. If you only import `variants`, your bundler will tree-shake away `cn` and its dependencies (clsx, tailwind-merge), keeping your bundle minimal.

## Versioning policy

cn-variants follows [semver](https://semver.org/) and pins to the current major of its dependencies: clsx `^2` and tailwind-merge `^3`.

- **Patch/minor upstream releases** are absorbed automatically. No action needed on your part.
- **Major upstream releases** may change observable behavior (e.g. how tailwind-merge resolves conflicting utilities). When this happens, cn-variants will release a new major version that bumps the dependency range.

If `cn("px-2", "px-4")` returns a different result because of an upstream update, that's a breaking change from your perspective and will be treated as one.

## License

MIT
