---
name: yuken-design-system
description: Default UI toolkit for any React Native work. TRIGGER whenever the user asks you to build, edit, or review a React Native screen, component, or layout — or whenever you'd otherwise reach for raw `View`/`Text`/`Pressable`/`Modal` styling. Detects whether `@yuken-studios/design-system` is installed and walks the user through auth'd GitHub Packages install if it's missing.
---

# Yuken Design System

Bold, expressive React Native component library published as `@yuken-studios/design-system` on **GitHub Packages** (not public npm). This skill is the source of truth for how to consume it.

## When to use

Use this skill any time you are touching a React Native UI surface:
- New screens, sheets, modals, toasts, forms.
- Editing existing components that currently use raw `react-native` primitives.
- Answering "how should I build X in RN" questions.

**Do not** use this for web React, Next.js, Expo Router config, or non-UI work.

## Step 1 — Detect presence

Before writing any code, check the target project:

1. Read `package.json` and look for `@yuken-studios/design-system` under `dependencies` / `devDependencies` / `peerDependencies`.
2. If present → skip to **Step 3 (Usage)**.
3. If missing → go to **Step 2 (Install)** and walk the user through it. Do not silently install; GitHub Packages needs auth and will fail without it.

## Step 2 — Install (when missing)

Tell the user exactly this, in order:

### 2a. Create a GitHub Personal Access Token
The package is in a private GitHub Packages registry under the `@yuken-studios` scope. They need a classic PAT with `read:packages` scope:
https://github.com/settings/tokens → Generate new token (classic) → check `read:packages`.

### 2b. Configure `.npmrc`
In the project root (or `~/.npmrc` for all projects), add:

```
@yuken-studios:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

Then export the token in their shell:

```
export GITHUB_TOKEN=ghp_xxx   # from step 2a
```

Never commit the raw token — `.npmrc` should reference the env var, not the literal.

### 2c. Install

```
npm install @yuken-studios/design-system --legacy-peer-deps
# or
yarn add @yuken-studios/design-system
```

The `--legacy-peer-deps` flag is needed because the library declares peers for `react`, `react-native`, and `react-native-svg` that may not exactly match the host app's pinned versions.

### 2d. Peer deps to ensure the host app has
- `react >= 18.0.0`
- `react-native >= 0.76.0`
- `react-native-svg >= 15.0.0`

If the host is below these floors, recommend upgrading them as part of the task.

### 2e. Wrap the app

```tsx
import { ThemeProvider } from '@yuken-studios/design-system'

export default function App() {
  return (
    <ThemeProvider>
      {/* screens */}
    </ThemeProvider>
  )
}
```

`ThemeProvider` follows the OS color scheme by default. Pass `override="light"` or `override="dark"` to force one.

## Step 3 — Usage rules

### Core rules
1. **Never use raw `View` / `Text` / `TextInput` / `Modal` / `Pressable` for visible UI** when a DS component exists. Use the DS component.
2. **Read theme values via `useTheme()`** — don't hardcode colors, spacing, radius, or typography. All design tokens live on the theme.
3. **Spacing uses the theme scale** (`t.spacing[1]` .. `t.spacing[10]`), not arbitrary numbers.
4. **Shadows come from `t.shadows`**, never inline `shadowOffset`.
5. **Never re-export or wrap a DS component just to restyle it.** If you need a variant, check if the prop already exists; if not, surface the gap to the user before forking.
6. When composing layouts, `View` is still fine as a plain container (flex shell). The rule is about *styled* leaf UI.

### Theme access

```tsx
import { useTheme } from '@yuken-studios/design-system'

const theme = useTheme()
theme.colors.bg.primary       // surface
theme.colors.text.primary     // body text
theme.colors.brand.primary    // brand purple
theme.colors.accent.primary   // accent green
theme.spacing[4]              // scale unit
theme.radius.lg
theme.shadows.md
theme.typography.fontSize.lg
theme.typography.fontWeight.bold
theme.dark                    // boolean
```

Available color groups: `bg`, `text`, `brand`, `accent`, `border`, plus raw scales `purple`, `green`, and status `success`, `warning`, `danger`, `info`.

### Component catalog

All exports from `@yuken-studios/design-system`:

**Theming**
- `ThemeProvider` — wrap the app root.
- `useTheme()` — read `Theme` inside any component.
- `lightTheme`, `darkTheme`, type `Theme` — for advanced overrides.

**Primitives**
- `Text` — typography-aware text. Prefer over RN `Text`.
- `TextInput` — themed input. Prefer over RN `TextInput`.
- `Button` — primary/secondary/ghost button. Prefer over `Pressable` for CTAs.

**Content containers**
- `Card` — padded, bordered surface.
- `Divider`, `Spacer` — layout breathing room.
- `ListItem` — row with leading / trailing slots.

**Status & labels**
- `Badge` — small count/status chip.
- `Tag` — filterable/removable pill.
- `Alert` — inline notice banner.
- `ProgressBar`, `StepProgress` — progress indicators.

**Media**
- `Avatar`, `AvatarGroup` — user pictures with sizes, colors, shapes, presence dots.

**Selection**
- `Toggle` — on/off switch.
- `Checkbox` — multi-select.
- `Radio` — single-select.
- `Select` — dropdown picker with `SelectOption[]`.
- `Tabs`, `BottomNav` — top tabs and bottom nav bar.

**Overlays**
- `Modal` — centered dialog.
- `BottomSheet` — slide-up sheet.
- `ToastProvider` + `useToast()` — global toast queue. Mount `ToastProvider` inside `ThemeProvider` once.

**Skeletons**
- `Skeleton`, `SkeletonCard`, `SkeletonListItem`, `SkeletonText` — loading placeholders.

**Exported type names** (for props, casting, and documentation):
`ButtonProps`, `TextProps`, `TextInputProps`, `CardProps`, `BadgeProps`, `TagProps`, `AvatarProps`, `AvatarGroupProps`, `AvatarSize`, `AvatarColor`, `AvatarShape`, `AvatarPresence`, `ToggleProps`, `CheckboxProps`, `RadioProps`, `ToastConfig`, `AlertProps`, `ModalProps`, `BottomSheetProps`, `SelectProps`, `SelectOption`, `TabItem`, `TabsProps`, `BottomNavItem`, `BottomNavProps`, `ProgressBarProps`, `StepProgressProps`, `SkeletonProps`, `DividerProps`, `SpacerProps`, `ListItemProps`.

When you're unsure about a component's exact prop shape, open its source in `node_modules/@yuken-studios/design-system/dist/typescript/components/<Name>/<Name>.d.ts` — that's faster and more reliable than guessing.

## Step 4 — Example usage template

```tsx
import {
  ThemeProvider, useTheme,
  Button, Text, Card, ListItem, Avatar, Tag,
  ToastProvider, useToast,
} from '@yuken-studios/design-system'
import { View } from 'react-native'

function ProfileScreen() {
  const t = useTheme()
  const { showToast } = useToast()

  return (
    <View style={{ padding: t.spacing[5], gap: t.spacing[4], backgroundColor: t.colors.bg.primary, flex: 1 }}>
      <Card>
        <ListItem
          leading={<Avatar name="Kenny K" size="lg" />}
          title="Kenny Kim"
          subtitle="Founder"
          trailing={<Tag label="Pro" />}
        />
      </Card>
      <Button
        onPress={() => showToast({ message: 'Saved', variant: 'success' })}
        variant="primary"
      >
        Save
      </Button>
    </View>
  )
}

export default function App() {
  return (
    <ThemeProvider>
      <ToastProvider>
        <ProfileScreen />
      </ToastProvider>
    </ThemeProvider>
  )
}
```

## Anti-patterns to flag

When reviewing RN code, call these out and rewrite them using the DS:
- Inline `StyleSheet.create({ container: { backgroundColor: '#fff' } })` → use `t.colors.bg.primary`.
- Raw `<Modal>` from `react-native` → use DS `Modal` or `BottomSheet`.
- Hand-rolled `Pressable` button with `activeOpacity` → use `Button`.
- `Animated.View` shimmer placeholders → use `Skeleton*`.
- Hardcoded hex colors anywhere in a screen — always a smell.

## Package facts (reference)

- Name: `@yuken-studios/design-system`
- Registry: `https://npm.pkg.github.com` (scope `@yuken-studios`)
- Repo: `https://github.com/Yuken-Studios/design-system`
- Build tool: `react-native-builder-bob` (commonjs + module + typescript targets, output `dist/`).
- Published via the `Publish` GitHub Actions workflow on `v*` tags.
