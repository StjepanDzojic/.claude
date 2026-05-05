# Plan: Secondary Account "Secure" Screen

## Context
The goal is to add a minimal "Secure" screen that runs a second Matrix account simultaneously alongside the primary account — for testing purposes. The user navigates to this screen from Home, logs in once with a second account (credentials persist), and sees that account's room list. No push notifications or full feature parity needed.

**Key finding:** Two `ClientBuilder` instances can coexist in the same process as long as they use **different file system paths**. The existing file-lock concern in the code is path-specific, not process-specific.

---

## Implementation Plan

### 1. Secondary credential storage
**New file:** `src/modules/matrix-client/const/secondary-matrix-storage.ts`

Mirror `matrix-storage.ts` but:
- Use a separate MMKV instance: `id: "matrix-secondary-storage"`
- Use keys prefixed with `secondary_` (e.g. `secondary_access_token`, `secondary_user_id`, etc.)
- Expose: `getSecondaryCredentials()`, `saveSecondaryCredentials()`, `clearSecondaryCredentials()`, `hasSecondaryCredentials()`

Reference: [src/modules/matrix-client/const/matrix-storage.ts](src/modules/matrix-client/const/matrix-storage.ts)

---

### 2. Secondary Matrix service
**New file:** `src/modules/matrix-client/services/SecondaryMatrixService.ts`

Manages the secondary client lifecycle:
- `secondaryClient: ClientInterface | null` (module-level, not static singleton)
- `secondarySyncService: SyncServiceInterface | null`
- `initialize(serverUrl)` — builds client with paths:
  - dataPath: `${RNFS.DocumentDirectoryPath}/matrix/secondary/data`
  - cachePath: `${RNFS.CachesDirectoryPath}/matrix/secondary/cache`
- `login(username, password)` — login + save credentials
- `restoreSession()` — restore from saved secondary credentials
- `startSync(onRoomsUpdate: (rooms: RoomInterface[]) => void)` — start sync service, subscribe to room list updates, call callback on each update
- `logout()` — stop sync, clear credentials, nullify instances
- `getClient()` — returns `secondaryClient` (throws if null)

Reference for patterns: [src/modules/auth/service/AuthService.ts](src/modules/auth/service/AuthService.ts), [src/modules/matrix-client/services/MatrixCore.ts](src/modules/matrix-client/services/MatrixCore.ts)

---

### 3. Secure screen
**New file:** `src/screens/secure/SecureScreen.tsx`

Logic:
- On mount: check `hasSecondaryCredentials()`
  - If yes → call `SecondaryMatrixService.restoreSession()` then `startSync(setRooms)`
  - If no → show login form
- Login form: server URL (default same as primary), username, password → call `SecondaryMatrixService.login()` → on success, `startSync(setRooms)`
- Room list: local `useState<RoomInterface[]>([])` populated via the sync callback
- Render: simple `FlatList` of room display names + unread counts (fetch via `room.roomInfo()` per item)
- Header: "Secure" title + logout icon button (calls `SecondaryMatrixService.logout()`, clears `rooms` state, shows login form)

No need to reuse `RoomsListItem` — a simpler per-item component avoids the Jotai dependency.

**New file:** `src/screens/secure/fragments/SecureRoomItem.tsx`

Simple item component:
- Props: `{ room: RoomInterface }`
- Local state: `roomInfo` fetched from `room.roomInfo()`
- Displays: room display name + unread badge (if `numUnreadNotifications > 0`)

---

### 4. Navigation wiring

**Modify:** `src/@types/navigation.d.ts`
- Add `secure: undefined` to the `Routes` type

**Modify:** `src/navigation/RootNavigator.tsx`
- Import `SecureScreen`
- Add `<Stack.Screen name="secure" component={SecureScreen} />`

**Modify:** `src/screens/home/Home.tsx`
- Add a button in the header (next to Settings) that navigates to `"secure"`
- Use existing header button pattern from the Settings link

---

## Critical Files
| File | Role |
|------|------|
| [src/modules/matrix-client/const/matrix-storage.ts](src/modules/matrix-client/const/matrix-storage.ts) | Template for secondary storage |
| [src/modules/auth/service/AuthService.ts](src/modules/auth/service/AuthService.ts) | Template for client initialization |
| [src/modules/matrix-client/services/MatrixCore.ts](src/modules/matrix-client/services/MatrixCore.ts) | Pattern for client/sync instance holding |
| [src/navigation/RootNavigator.tsx](src/navigation/RootNavigator.tsx) | Add `secure` screen |
| [src/@types/navigation.d.ts](src/@types/navigation.d.ts) | Add `secure` route type |
| [src/screens/home/Home.tsx](src/screens/home/Home.tsx) | Add navigation trigger |

---

## Verification
1. Build and run the app on a simulator
2. From Home, tap the "Secure" button in the header → SecureScreen opens
3. Enter second account credentials → login succeeds, room list loads
4. Kill and reopen the app → navigate to Secure again → session restores automatically, rooms appear
5. Tap logout on SecureScreen → returns to login form
6. Primary account continues working normally throughout — no interference
