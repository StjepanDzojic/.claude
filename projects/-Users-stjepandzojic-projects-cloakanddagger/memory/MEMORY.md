# Cloak & Dagger Project Memory

## Project Overview
React Native 0.80.1 app ("Cloak") — a Matrix/Element-based encrypted chat/call app.
- Bundle ID: `com.prototyp.cloak`
- iOS deployment target: 15.1–15.6
- Dev team: `D277YNPF5V`
- New Architecture enabled (RCTNewArchEnabled)
- Uses Hermes JS engine

## Key Architecture
- Matrix SDK: `@unomed/react-native-matrix-sdk`
- Push notifications: Firebase FCM (`@react-native-firebase/messaging`) + `@notifee/react-native`
- Calls: `react-native-callkeep` (4.3.16) + Element Call WebView
- Push gateway: Sygnal at `https://sygnal.matrix.sedmiodjel.com`
- App ID for FCM pusher: `com.prototyp.cloak`
- App ID for VoIP pusher: `com.prototyp.cloak.ios.voip`

## iOS Native Files
- `ios/cloakanddagger/AppDelegate.swift` — Firebase + CallKit + PKPushRegistryDelegate
- `ios/cloakanddagger/ElementCallPiPModule.swift` + Bridge — PiP for Element Call WebView
- `ios/cloakanddagger/VoIPPushModule.swift` + Bridge — Exposes VoIP push token to JS
- `ios/CloakNotificationService/NotificationService.swift` — NSE: intercepts FCM pushes, routes call events to CallKit via `CXProvider.reportNewIncomingVoIPPushPayload()`

## Call Flow (iOS, including killed app)
```
FCM push (mutable-content:1) → NSE intercepts → type=m.call.invite|m.call.member
  → CXProvider.reportNewIncomingVoIPPushPayload(userInfo)
  → AppDelegate.PKPushRegistryDelegate.didReceiveIncomingPushWith
  → RNCallKeep.reportNewIncomingCall() [native, no JS needed]
  → CallKit UI shown
  → User accepts → answerCall JS event → navigate to elementCall
```

## Key JS Files
- `src/modules/call/service/CallKitService.ts` — RNCallKeep wrapper, answerCall → navigate to elementCall
- `src/modules/call/provider/CallKitProvider.tsx` — sync-based foreground call detection
- `src/modules/push-notifications/service/PushNotificationsService.ts` — FCM handler
- `src/modules/push-notifications/service/VoIPPushService.ts` — VoIP token → Matrix pusher registration
- `src/modules/push-notifications/service/MatrixNotificationService.ts` — pusher API calls

## Important Notes
- NSE bundle ID: `com.prototyp.cloak.CloakNotificationService`
- For full killed-app CallKit: FCM pushes must include `mutable-content: 1` (Sygnal does this by default)
- Encrypted events (m.room.encrypted) NOT routed through NSE (can't decrypt there) — fallback to JS
- VoIP APNs token flow: PKPushRegistry → VoIPPushModule → JS → MatrixNotificationService.setVoIPPusher()
- Server-side: Sygnal needs VoIP APNs cert configured for app_id `com.prototyp.cloak.ios.voip`
