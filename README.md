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

cn("px-4 py-2", "px-6");
// → "py-2 px-6"

cn("text-red-500", isActive && "text-blue-500");
// → "text-blue-500" (when isActive is true)
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
type ButtonVariant = keyof typeof buttonVariant.options;

const buttonSize = variants({
  sm: "px-3 py-1 text-xs",
  md: "px-4 py-2 text-sm",
  lg: "px-6 py-3 text-base",
});
type ButtonSize = keyof typeof buttonSize.options;
```

Use it with `cn` in your components:

```tsx
export function Button({ variant = "primary", size = "md", className }: ButtonProps) {
  return (
    <button
      className={cn("rounded-md font-medium", buttonVariant(variant), buttonSize(size), className)}
    >
      ...
    </button>
  );
}
```

### Compound variants

For styles that depend on a combination of variants, use conditionals in `cn`:

```tsx
cn(
  "rounded-md font-medium",
  buttonVariant(variant),
  buttonSize(size),
  variant === "primary" && size === "lg" && "uppercase tracking-wide",
  className,
);
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
