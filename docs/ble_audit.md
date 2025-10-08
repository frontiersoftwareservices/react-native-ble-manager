# React Native BLE Manager Audit

## Android

- [Connections][AutoConnect] Finding: **Yes (with caveats)**. Proof: `android/src/main/java/it/innove/Peripheral.java` lines 120-135 show `device.connectGatt(... autoconnect ...)` where the JS option maps directly to the native flag. Snippet:
  ```java
  boolean autoconnect = false;
  if (options.hasKey("autoconnect")) {
      autoconnect = options.getBoolean("autoconnect");
  }
  gatt = device.connectGatt(activity, autoconnect, this, BluetoothDevice.TRANSPORT_LE);
  ```
  Risk: **Medium** – foreground callers can accidentally pass `true` and trigger flaky background reconnection semantics. Fix: gate `autoconnect=true` behind background-only flows with documented retry/backoff.
- [Connections][AutoConnect for resume] Finding: **No**. Proof: Same block (`Peripheral.java` L120-135) lacks retry/backoff logic or resume-specific guard. Snippet same as above. Risk: **High** – misusing `autoConnect` without retries can stall connections on many chipsets. Fix: reserve `autoConnect=true` for background restoration with a reconnection scheduler that enforces backoff.
- [Connections][Disconnect close] Finding: **Yes**. Proof: `Peripheral.java` lines 168-185 close GATT when `force` true and in `onConnectionStateChange` lines 398-407. Snippet:
  ```java
  if (force) {
      gatt.close();
      gatt = null;
  }
  ...
  if (gatt != null) {
      gatt.disconnect();
      gatt.close();
  }
  ```
  Risk: **Low**. Fix: Ensure all reconnect paths call `disconnect(force=true)` before retrying.
- [Connections][Backoff] Finding: **No**. Proof: No retry/backoff handling in `BleManager` or `Peripheral`—`onConnectionStateChange` (L374-410) simply emits disconnect. Risk: **High** – immediate retries on statuses 133/8 will loop and drain power. Fix: add capped exponential backoff with jitter before reissuing `connectGatt` after transient errors.
- [Connections][Bonding] Finding: **Partial**. Proof: `Peripheral.java` write path (L1046-1105) only logs auth failures; bonding flows handled in `BleManager.java` (L430-517) but not retried automatically. Risk: **Medium** – encrypted writes fail until caller manually bonds. Fix: surface specific error to JS and optionally trigger `createBond` with bond-state listener before retrying.
- [Connections][Callback resilience] Finding: **Partial**. Proof: `onConnectionStateChange` (L374-410) closes GATT on any non-success but does not guard against double-callbacks or racing `disconnect()`. Risk: **Medium** – rapid connect/disconnect can race `gatt` nulling and clear queues unexpectedly. Fix: synchronize around `connecting/connected` flags and ignore stale callbacks.
- [Threading][Callback thread] Finding: **Yes** (marshaled to main). Proof: `Peripheral.java` `mainHandler.post(...)` in `connect` (L118-158) and `onConnectionStateChange` (L379-411) move work to main thread. Risk: **Low** – avoids binder blocking but main thread contention possible. Fix: consider dedicated serial handler thread to keep UI responsive.
- [Threading][Operation queue] Finding: **Partial**. Proof: Command queue at `Peripheral.java` L911-960 serializes operations but lacks timeouts/cancellation. Risk: **Medium** – stalled GATT callbacks block later commands indefinitely. Fix: add per-operation timeout and cancel/flush when exceeding threshold.
- [Threading][JS dispatch] Finding: **Yes**. Proof: Events emitted via `bleManager.emit...` from main-thread `Handler` (e.g., `onCharacteristicChanged` L433-461). Risk: **Low**. Fix: none.
- [Scanning][ScanSettings] Finding: **Caller-driven only**. Proof: `DefaultScanManager.java` L59-154 builds settings solely from options without defaulting to `LOW_LATENCY` for foreground. Risk: **Medium** – defaults rely on platform (balanced) and may miss foreground devices. Fix: set default scan mode based on foreground/background state.
- [Scanning][Permissions] Finding: **No** runtime gating. Proof: `DefaultScanManager.scan` lacks permission checks; only descriptor name fetch checks `BLUETOOTH_CONNECT` (L206-208). Risk: **High** – calling scan without runtime permissions throws. Fix: validate `BLUETOOTH_SCAN/CONNECT` (and location pre-12) before starting, reject via callback.
- [Scanning][Foreground service] Finding: **No**. Proof: No reference to foreground services or Android 10+ restrictions in Android code. Risk: **Medium** – long scans may be throttled in background. Fix: provide optional foreground service integration for persistent scans.
- [Notifications][CCCD writes] Finding: **Yes**. Proof: `setNotify` (L656-706) selects `ENABLE_NOTIFICATION_VALUE` vs `ENABLE_INDICATION_VALUE` based on properties then writes descriptor. Risk: **Low**. Fix: reapply after reconnect.
- [Notifications][Re-enable after reconnect] Finding: **No automatic retry**. Proof: No logic in `onConnectionStateChange` or elsewhere to resubscribe; buffers cleared on disconnect (L366-372). Risk: **Medium** – notifications stop after reconnect. Fix: track active subscriptions and restore post-`discoverServices`.
- [MTU/PHY][Negotiation] Finding: **Manual only**. Proof: `requestMTU` (L1188-1221) requires caller input, no default `517` request; writes chunk by provided `maxByteSize` (L1040-1120) without referencing negotiated MTU. Risk: **Medium** – callers may overshoot `WRITE_NO_RESPONSE` length. Fix: automatically request 517 on connect and expose negotiated MTU to JS to drive chunk size.
- [MTU/PHY][setPreferredPhy] Finding: **No**. Proof: No usage in repo. Risk: **Low**. Fix: optionally expose API guarded by API level.
- [Writes][Long write retry] Finding: **Partial**. Proof: Chunking loops (L1058-1116) sleep and send but do not check `GATT_WRITE_NOT_PERMITTED` or congestion statuses. Risk: **Medium** – large writes can silently fail. Fix: detect error codes in `onCharacteristicWrite` and retry/backoff.
- [Service Changed] Finding: **No automatic rediscovery**. Proof: On disconnect they clear state but no handling for `SERVICE_CHANGED` or cache invalidation beyond manual `refreshCache`. Risk: **Medium** – stale services persist. Fix: listen for `SERVICE_CHANGED` and trigger `refreshCache` + `discoverServices`.
- [Power niceties] Finding: **Minimal**. Proof: Exposes `requestConnectionPriority` API (L1168-1182) but no automatic use; no supervision/PHY tuning. Risk: **Low**. Fix: document usage; consider auto high-priority during transfers.

## iOS

- [Connections][Queue choice] Finding: **No dedicated BLE queue by default**. Proof: `SwiftBleManager.start` defaults to `DispatchQueue.main` when `queueIdentifierKey` missing (`ios/SwiftBleManager.swift` L200-218). Risk: **Medium** – CoreBluetooth delegate callbacks hit main thread causing UI contention. Fix: default to a serial background queue and only use main when explicitly requested.
- [Connections][connect options] Finding: **No notification/disconnect options**. Proof: `manager?.connect(peripheral.instance)` without options (L300-335). Risk: **Low** – missing background alerts for sleeping devices. Fix: surface `CBConnectPeripheralOptionNotify*` toggles via JS options.
- [Connections][State restoration] Finding: **Partial**. Proof: Supports `restoreIdentifierKey` and implements `willRestoreState` (L200-223, L724-737) but no handling for `registerForConnectionEvents`. Risk: **Medium** – reconnection limited to CoreBluetooth restore. Fix: add `registerForConnectionEvents` on iOS 13+ and surface events to JS.
- [Connections][Retrieve peripherals] Finding: **Yes**. Proof: `connect` tries `retrievePeripherals(withIdentifiers:)` before scanning (L308-324). Risk: **Low**.
- [Connections][Backoff] Finding: **No**. Proof: `didDisconnectPeripheral` (L752-805) only clears callbacks; no retry scheduling. Risk: **High** – immediate manual loops needed. Fix: add retry manager with capped backoff and user override.
- [Threading][Serial queue] Finding: **Partial**. Proof: Many dictionaries accessed under `serialQueue.sync`, but delegate callbacks (e.g., `didConnect`, `didUpdateValue`) run on CoreBluetooth queue (main by default). No explicit serialization of all BLE ops. Risk: **Medium** – potential race between delegate callbacks and operations. Fix: initialize central on dedicated serial queue and ensure bridging to JS occurs via `DispatchQueue.main`/`RCT` helpers.
- [Threading][JS dispatch] Finding: **Yes**. Proof: JS callbacks invoked via stored closures executed on whichever queue `serialQueue.sync` or main uses; some calls (e.g., `didConnect` uses `DispatchQueue.main.async`). Risk: **Low** if BLE queue not main.
- [Notifications] Finding: **Yes but no re-subscribe**. Proof: `startNotification` writes `setNotifyValue(true, ...)` (L683-700) but `didUpdateNotificationState` only emits event; no restore on reconnect. Risk: **Medium** – notifications lost after reconnect. Fix: track subscription intents and re-enable during `didConnect`/`discoverServices`.
- [MTU/Write sizing] Finding: **Partial**. Proof: Provides `getMaximumWriteValueLength` APIs (L730-759) but `writeWithoutResponse` chunks using caller-provided `maxByteSize` and sleeps, ignoring congestion callbacks (L570-609). `peripheralIsReady(toSendWriteWithoutResponse:)` not implemented. Risk: **High** – writes can fail on buffer full. Fix: default chunk size to `maximumWriteValueLength` and implement `peripheralIsReady` to resume streaming.
- [Discovery][Service changed] Finding: **No**. Proof: No `peripheral(_:didModifyServices:)` implementation. Risk: **Medium** – stale handles after Service Changed. Fix: implement delegate to rediscover and notify JS.
- [Background/Permissions] Finding: **Partial**. Proof: Checks `NSBluetoothAlwaysUsageDescription` (L182-189) but no Info.plist enforcement for background modes. Risk: **Low**. Fix: document required `bluetooth-central` background mode when needed.

## Cross-platform React Native Layer

- [Error mapping] Finding: **No structured codes**. Proof: Callbacks reject with string messages throughout (`src/index.ts` e.g., L18-74). Risk: **Medium** – JS cannot distinguish timeout vs permissions. Fix: map native errors to standardized code enum.
- [Connect API features] Finding: **Minimal**. Proof: `connect` accepts only `{ autoconnect, phy }` (Android) (L258-273). No timeout/backoff hooks, lifecycle events limited to connect/disconnect notifications (L804-861). Risk: **High** – app must reimplement robust reconnection logic. Fix: extend JS API with timeout/backoff options and explicit lifecycle events (connecting/servicesReady/mtuChanged/etc.).
- [Operation queue exposure] Finding: **No shared abstraction**. Proof: JS layer issues commands directly; Android native serializes, iOS lacks full queue. Risk: **High** – cross-platform behavior inconsistent; callers may trigger concurrent ops on iOS. Fix: add shared JS-side promise queue or align native implementations.
- [Notification metadata] Finding: **No**. Proof: Types do not expose notification vs indication semantics beyond property maps. Risk: **Low**. Fix: expose descriptor/notify type info in `PeripheralInfo`.
- [MTU exposure] Finding: **Partial**. Proof: Android `requestMTU` returns value but JS does not cache/expose event; write helpers default to 20 bytes (L118-207). Risk: **Medium** – callers unaware of negotiated MTU. Fix: emit `mtuChanged` event and default chunk size accordingly.

## Direct Questions

1. **Android autoConnect usage?** Yes; `Peripheral.connect` passes JS `autoconnect` directly into `connectGatt` (L126-135).
2. **Android serial GATT queue with timeouts?** Serial queue exists (`commandQueue` L911-960) but lacks timeouts/cancellation.
3. **Android callback thread?** GATT callbacks post to `mainHandler` (L118-411), so processing occurs on main thread.
4. **Android scans foreground vs background?** No internal distinction; scan settings come entirely from caller options with platform defaults (L59-154) and no foreground service support.
5. **Android notifications vs indications?** `setNotify` chooses CCCD values based on characteristic properties and writes descriptor (L673-704).
6. **Android MTU negotiation/chunking?** `requestMTU` only on demand (L1188-1221); write chunking uses caller `maxByteSize` (L1046-1116) without referencing negotiated MTU.
7. **iOS CBCentralManager queue?** Defaults to `DispatchQueue.main` unless caller supplies `queueIdentifierKey` (L200-218).
8. **iOS connection options/restoration?** No `connect` options (L300-335); restoration available via `restoreIdentifierKey` and `willRestoreState` (L200-223, L724-737); no `registerForConnectionEvents`.
9. **iOS reconnect/backoff?** None; `didDisconnectPeripheral` just clears callbacks (L752-805).
10. **iOS write sizing/back-pressure?** Chunking relies on caller `maxByteSize` (L568-605) and no `peripheralIsReady` handler; `maximumWriteValueLength` helper available (L730-758).
11. **iOS notification resubscribe?** No automatic resubscribe after reconnect; only immediate `setNotifyValue(true, …)` when requested (L680-700).
12. **JS reconnect/state hooks?** JS exposes manual `connect`/`disconnect` promises and events for connect/disconnect/stopScan (L802-861) but no retry/backoff or lifecycle events beyond basics.
13. **JS single op queue?** None; JS issues commands directly and relies on native implementations, leading to inconsistent concurrency control.
