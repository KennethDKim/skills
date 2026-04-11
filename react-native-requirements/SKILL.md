---
name: react-native-stack
description: >
  Reference stack for building React Native apps with Expo. Use this skill whenever
  the user is working on a React Native or Expo project and needs to implement any
  feature pillar — state management, server requests, persistence, navigation, forms,
  animations, auth, or lists/images. Trigger even if the user doesn't explicitly ask
  for library recommendations; if they're building a feature that maps to a pillar below,
  default to the standard listed here unless the project already uses something else.
  Not every project needs every pillar — only apply what's relevant to the task at hand.
---

# React Native / Expo stack

These are the preferred defaults for Expo/RN projects. They are not prescriptive for every project — some apps won't need all pillars. But **if a pillar needs to be built and no existing choice is in place, default to the library listed here** without suggesting alternatives unless asked.

## Version resolution — always use Context7

Before writing any install command or import, **resolve the current version via Context7**. Training data goes stale; APIs change between majors. Do not assume a version is current.

```
// Pattern for every lib before use:
1. use mcp__context7__resolve-library-id to find the library
2. use mcp__context7__get-library-docs to pull current docs/version
3. Use the version and API surface from those docs — not from memory
```

Apply this to every pillar library when implementing: `zustand`, `@tanstack/react-query`, `pocketbase`, `react-native-mmkv`, `expo-sqlite`, `drizzle-orm`, `expo-router`, `react-hook-form`, `zod`, `react-native-reanimated`, `@shopify/flash-list`, `expo-image`.

If Context7 returns a breaking API change from what's shown in the code snippets below, **use the Context7 version** — the snippets are patterns, not pinned APIs.

**If Context7 MCP is not available**, stop and prompt the user:
> "Context7 isn't connected — I recommend adding it so I can verify the latest library versions before writing code. You can install it at https://context7.com or via your MCP config. Want to proceed with potentially stale versions in the meantime?"

---

## Stack reference

### State — `zustand`
Local/global UI state. Use `zustand/middleware` `persist` with the MMKV storage adapter for any state that needs to survive app restarts. Do not reach for Context + useReducer for anything non-trivial.

```ts
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'
import { storage } from '@/lib/storage' // MMKV instance

export const useAuthStore = create(
  persist(
    (set) => ({ user: null, setUser: (user) => set({ user }) }),
    { name: 'auth', storage: createJSONStorage(() => storage) }
  )
)
```

---

### Server state / cache — `@tanstack/react-query` + `pocketbase` SDK
React Query handles caching, loading/error states, background refetch, and optimistic updates. The PocketBase JS SDK is the fetcher — no axios/ky needed.

```ts
const { data, isLoading } = useQuery({
  queryKey: ['posts'],
  queryFn: () => pb.collection('posts').getList(1, 20),
})
```

For mutations with optimistic updates, use `useMutation` + `queryClient.invalidateQueries`.

---

### Auth — PocketBase SDK (`pb.authStore`)
PocketBase handles email, OAuth2, and OTP out of the box. Persist the auth token in MMKV. No need for Clerk, expo-auth-session, or Firebase auth.

```ts
await pb.collection('users').authWithPassword(email, password)
// token auto-saved in pb.authStore — persist it:
storage.set('pb_auth', JSON.stringify(pb.authStore.exportToCookie()))
```

On app start, restore: `pb.authStore.loadFromCookie(storage.getString('pb_auth') ?? '')`

---

### Realtime — PocketBase SSE
PocketBase subscriptions replace any need for a separate websocket lib.

```ts
pb.collection('messages').subscribe('*', ({ action, record }) => {
  // handle create/update/delete
})
// cleanup:
pb.collection('messages').unsubscribe()
```

---

### KV persistence — `react-native-mmkv`
Fast synchronous key-value store. Use for tokens, user preferences, cached UI state. ~30× faster than AsyncStorage.

```ts
import { MMKV } from 'react-native-mmkv'
export const storage = new MMKV()
storage.set('key', 'value')
storage.getString('key')
```

---

### SQL persistence — `expo-sqlite` + `drizzle-orm`
Only needed for true offline-first apps with relational data. Skip if PocketBase + React Query covers the use case.

```ts
import { drizzle } from 'drizzle-orm/expo-sqlite'
import * as SQLite from 'expo-sqlite'
const expo = SQLite.openDatabaseSync('app.db')
export const db = drizzle(expo)
```

---

### Navigation — `expo-router`
File-based routing built on react-navigation. Use for all new Expo projects. Layouts, tabs, modals, and deep linking are handled via the file system convention.

```
app/
  (tabs)/
    index.tsx
    profile.tsx
  modal.tsx
  _layout.tsx
```

---

### Forms / validation — `react-hook-form` + `zod`
RHF avoids re-renders on every keystroke. Zod provides schema-first validation that doubles as runtime type safety across the stack.

```ts
const schema = z.object({ email: z.string().email(), password: z.string().min(8) })
const { control, handleSubmit } = useForm({ resolver: zodResolver(schema) })
```

---

### Animations — `react-native-reanimated` (v3)
All animations run on the native thread via worklets. Use for gestures, transitions, and any performance-sensitive motion. `moti` is an optional declarative wrapper if the API feels verbose.

```ts
const opacity = useSharedValue(0)
const style = useAnimatedStyle(() => ({ opacity: opacity.value }))
useEffect(() => { opacity.value = withTiming(1) }, [])
```

---

### Lists — `@shopify/flash-list`
Drop-in `FlatList` replacement with significantly better performance. Always prefer over `FlatList` or `ScrollView` + map for long lists.

```tsx
<FlashList data={items} renderItem={({ item }) => <Row item={item} />} estimatedItemSize={72} />
```

---

### Images — `expo-image`
Use instead of React Native's `Image`. Supports blurhash placeholders, proper disk/memory caching, and smooth transitions.

```tsx
<Image source={{ uri }} placeholder={blurhash} contentFit="cover" transition={200} />
```

---

## Decision guide

| Situation | Action |
|---|---|
| Project already uses a different lib for a pillar | Keep it, don't migrate unless asked |
| Simple app with no server | Skip react-query; zustand + MMKV is enough |
| No offline requirement | Skip expo-sqlite + drizzle |
| Non-PocketBase backend | Replace PB SDK with fetch/ky; keep react-query |
| Auth via third-party SSO only | expo-auth-session is acceptable |

---

## Notes
- Zod is shared between forms (RHF resolver) and API response validation — define schemas once, use everywhere.
- MMKV is the zustand persist adapter AND the raw storage util — one instance, two uses.
- PocketBase SSE subscriptions must be cleaned up on component unmount to avoid memory leaks.