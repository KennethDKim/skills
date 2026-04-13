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

### Maps — `react-native-maps`
Recommended for all map rendering needs. Wraps Apple Maps (iOS) and Google Maps (Android) with a unified React Native API.

```tsx
import MapView, { Marker } from 'react-native-maps'

<MapView
  style={{ flex: 1 }}
  initialRegion={{ latitude: 37.78, longitude: -122.43, latitudeDelta: 0.05, longitudeDelta: 0.05 }}
>
  <Marker coordinate={{ latitude: 37.78, longitude: -122.43 }} title="Pin" />
</MapView>
```

---

### Location — `expo-location`
Recommended for all device location services. Handles permissions, foreground/background location, geocoding, and heading.

```ts
import * as Location from 'expo-location'

const { status } = await Location.requestForegroundPermissionsAsync()
if (status !== 'granted') return
const { coords } = await Location.getCurrentPositionAsync({})
```

For background tracking, use `Location.startLocationUpdatesAsync` with a defined task (requires `expo-task-manager`).

---

### In-App Purchases (iOS) — `expo-iap`

Expo-compatible wrapper around StoreKit 2. Handles subscriptions, non-consumables, and consumables with a hook-based API. Do not use `react-native-iap` directly in Expo managed projects.

#### Setup

1. Configure products in App Store Connect first (subscriptions, non-consumables, etc.)
2. Install: `npx expo install expo-iap`
3. Add the plugin to `app.json`:
```json
{ "expo": { "plugins": ["expo-iap"] } }
```
4. IAP only works on **real devices** — it will not function in Simulator.

#### Architecture pattern

Split IAP into four modules:

```
src/iap/
  products.ts   — SKU constants, type exports, productId→entitlement mapping
  store.ts      — Zustand store for entitlement state + hydration
  service.ts    — useIAPService() hook wrapping useIAP() from expo-iap
  api.ts        — fetch helpers for your verification backend
```

#### Products module

```ts
export const IAP_PRODUCTS = {
  UNLOCK: 'com.example.unlock',
} as const;

export type IAPProductId = (typeof IAP_PRODUCTS)[keyof typeof IAP_PRODUCTS];
export const NON_CONSUMABLE_SKUS: string[] = [IAP_PRODUCTS.UNLOCK];

export function productToEntitlement(productId: string): string | null {
  if (productId === IAP_PRODUCTS.UNLOCK) return 'premium';
  return null;
}
```

#### Service hook — critical pitfalls

```ts
import { ErrorCode, useIAP, getAvailablePurchases } from 'expo-iap';

export function useIAPService() {
  const { connected, products, finishTransaction, fetchProducts, requestPurchase } = useIAP({
    onPurchaseSuccess: async (purchase) => {
      try {
        // Verify with your backend
        const result = await verifyTransaction({ ... });
        if (result.valid) await store.setEntitlement(result.entitlement);
      } finally {
        // CRITICAL: Always finish — unfinished transactions block ALL future purchases
        await finishTransaction({ purchase, isConsumable: false });
      }
    },
    onPurchaseError: (error) => {
      // Silence user cancellations — they're not errors
      if (error.code !== ErrorCode.UserCancelled) {
        console.warn('[IAP] purchase error', error);
      }
    },
  });

  // Fetch products by type — must be separate calls
  useEffect(() => {
    if (!connected) return;
    fetchProducts({ skus: SUBSCRIPTION_SKUS, type: 'subs' }).catch(console.warn);
    fetchProducts({ skus: NON_CONSUMABLE_SKUS, type: 'in-app' }).catch(console.warn);
  }, [connected, fetchProducts]);

  const purchase = async (sku: string) => {
    try {
      await requestPurchase({
        request: { apple: { sku, appAccountToken: deviceId } },
        type: 'in-app', // or 'subs' for subscriptions
      });
    } catch (e: any) {
      // PITFALL: requestPurchase ALSO throws on cancellation, independently of onPurchaseError
      if (e?.code === ErrorCode.UserCancelled || e?.message?.includes('User cancelled')) return;
      throw e;
    }
  };
}
```

**Key pitfalls:**

| Pitfall | Detail |
|---|---|
| `requestPurchase` throws on cancel | `onPurchaseError` catches some errors, but `requestPurchase` itself **also throws** when the user cancels. You need try/catch in both places or you get an unhandled promise rejection. |
| `finishTransaction` is mandatory | If you skip it (even on verify failure), the App Store considers the transaction pending. Future `requestPurchase` calls will hang or fail silently. Always call in `finally`. |
| `fetchProducts` per type | You must call `fetchProducts` separately for `'subs'` and `'in-app'` — a single call with mixed SKUs will only return one type. |
| `appAccountToken` | Pass a stable device/user ID as `appAccountToken` in the purchase request. This is how you link Apple transactions to your backend user. UUID format recommended. |
| Products array uses `id` not `productId` | `useIAP().products` returns objects where the SKU is in the `id` field, not `productId`. Use `.find(p => p.id === sku)`. |
| Sandbox vs Production | StoreKit 2 sandbox transactions look identical to production. Your backend must handle both `environment` values (`Sandbox` / `Production`) and hit the correct Apple API endpoint. |

#### Server-side verification (Cloudflare Worker pattern)

A minimal verification backend needs:
1. **Decode the JWS** — Apple StoreKit 2 transactions are signed JWS tokens. Decode the payload (base64url) to extract `transactionId`, `productId`, `bundleId`, `environment`, `expiresDate`, `revocationDate`.
2. **Validate `bundleId`** — always check `tx.bundleId === YOUR_BUNDLE_ID` to prevent cross-app replay.
3. **Check `appAccountToken`** — match against the requesting device ID.
4. **Optional: cross-check with App Store Server API v2** — call `/inApps/v2/history/{originalTransactionId}` for server-side confirmation. Requires an Apple JWT signed with your App Store Connect API key.
5. **Store entitlements** — upsert into your database keyed by `(device_id)` for single-app or `(device_id, bundle_id)` for multi-app.
6. **Handle webhooks** — Apple sends App Store Server Notifications v2 to your endpoint for subscription renewals, refunds, and revocations.

**Multi-app note:** If you want to share a verification worker across multiple apps, you need: composite key `(device_id, bundle_id)` in your entitlements table, a `bundleId` allow-list instead of a single env var, and a data-driven `productToEntitlement` mapping. It's often simpler to deploy one worker per app sharing the same code.

#### Restore flow

```ts
const restore = async (): Promise<boolean> => {
  const purchases = await getAvailablePurchases();
  for (const p of purchases) {
    const result = await verifyTransaction({ ... });
    if (result.valid) { await store.setEntitlement(result.entitlement); return true; }
  }
  await store.setEntitlement('none');
  return false;
};
```

Apple requires a visible "Restore Purchases" button — this is an App Store Review guideline. Debounce it to prevent rapid-fire calls.

#### Non-consumable vs Subscription differences

| Aspect | Non-consumable | Subscription |
|---|---|---|
| `type` param | `'in-app'` | `'subs'` |
| `expiresDate` | `null` (permanent) | epoch ms |
| Grace period | Not needed | Implement offline grace (e.g. 3 days past expiry) |
| Server re-verify | Once at purchase + restore | Periodic (e.g. daily) to catch cancellations |
| Renewal webhooks | N/A | Must handle `DID_RENEW`, `EXPIRED`, `DID_CHANGE_RENEWAL_STATUS` |

---

## Decision guide

| Situation | Action |
|---|---|
| Project already uses a different lib for a pillar | Keep it, don't migrate unless asked |
| Simple app with no server | Skip react-query; zustand + MMKV is enough |
| No offline requirement | Skip expo-sqlite + drizzle |
| Non-PocketBase backend | Replace PB SDK with fetch/ky; keep react-query |
| Auth via third-party SSO only | expo-auth-session is acceptable |
| App needs IAP (iOS) | Use expo-iap; split into products/store/service/api modules |
| IAP non-consumable only | Skip expiresAt, grace period, and renewal webhook handling |
| Multiple apps sharing IAP backend | Deploy separate workers or add bundle_id composite keys |

---

## Notes
- Zod is shared between forms (RHF resolver) and API response validation — define schemas once, use everywhere.
- MMKV is the zustand persist adapter AND the raw storage util — one instance, two uses.
- PocketBase SSE subscriptions must be cleaned up on component unmount to avoid memory leaks.