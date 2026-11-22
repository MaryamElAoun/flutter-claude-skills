---
name: flutter-firebase
description: Set up and use Firebase in Flutter apps — FlutterFire configuration, Auth, Cloud Firestore (models, queries, pagination, converters), Cloud Messaging (FCM) push notifications, Storage, Crashlytics, security rules, and the local Emulator Suite. Use when adding Firebase to a project, writing Firestore data layers, debugging FCM, or writing/reviewing security rules.
---

# Firebase for Flutter

Wire Firebase into Flutter apps cleanly: Firebase calls stay inside data sources/repositories, exposed to the UI through providers (see `flutter-riverpod`).

## 1. Setup checklist

1. Install tools: `npm i -g firebase-tools`, `dart pub global activate flutterfire_cli`, then `firebase login`.
2. From the project root: `flutterfire configure` → generates `lib/firebase_options.dart` and platform config files.
   - Multiple flavors (dev/prod): run it once per flavor with `--project`, `--out=lib/firebase_options_dev.dart`, `--ios-bundle-id`, `--android-package-name`.
3. Add packages with `flutter pub add firebase_core` plus the ones needed (`firebase_auth`, `cloud_firestore`, `firebase_messaging`, `firebase_storage`, `firebase_crashlytics`, `firebase_analytics`).
4. Initialize before `runApp`:

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  runApp(const ProviderScope(child: App()));
}
```

5. Android: `minSdkVersion` at least what the plugins require (check the error message; usually 21–23). iOS: run `pod install` in `ios/` after adding plugins; set the iOS deployment target the plugins require.
6. **Never** commit service account keys. `firebase_options.dart` and `google-services.json` contain public client config and are OK to commit, but restrict API keys in Google Cloud console and rely on security rules + App Check for protection.

## 2. Architecture rules

- Expose Firebase singletons via providers (`firebaseAuthProvider`, `firestoreProvider`) so tests can override them with fakes.
- UI never imports `cloud_firestore` directly.
- Repositories convert `FirebaseException` / `FirebaseAuthException` into app errors with human-readable messages (map the `code`, e.g. `wrong-password`, `user-not-found`, `permission-denied`, `unavailable`).

## 3. Firestore

See `references/firestore.md` for code. Key rules:

- Use `withConverter` for typed collections; keep `fromJson/toJson` on the model (freezed + json_serializable). Store `id` from the document id, not inside the data.
- Timestamps: convert `Timestamp` ↔ `DateTime` in a `JsonConverter`; write `FieldValue.serverTimestamp()` for `createdAt/updatedAt`.
- Queries with `where` on one field + `orderBy` on another need a **composite index**. The error message contains a link to create it — tell the user to open it, and add the index to `firestore.indexes.json`.
- Paginate with `limit` + `startAfterDocument`, never by loading all docs.
- Use `snapshots()` for live UI, `get()` for one-off reads. Cancel listeners by letting the stream provider auto-dispose.
- Batch writes (`WriteBatch`, up to 500 ops) for multi-doc updates; transactions when a write depends on a read (counters, stock).
- Denormalize for reads; avoid client-side joins in loops (N+1 reads cost money).
- Offline persistence is on by default on mobile; writes resolve locally first — don't block UI waiting for server ack unless needed.

## 4. Auth

- Single source of truth: a keep-alive `StreamProvider` on `authStateChanges()` (or `userChanges()` if you need profile updates).
- Router redirects based on that provider (go_router `redirect` + `refreshListenable`).
- On sign-up, create the `users/{uid}` profile doc in the same flow (or via a Cloud Function `onCreate` trigger).
- Phone auth: handle `verificationCompleted` (Android auto-retrieval), `codeSent`, `verificationFailed`, `codeAutoRetrievalTimeout`. Add SHA-1 and SHA-256 fingerprints in the Firebase console for Android.
- Google Sign-In: needs SHA fingerprints (Android) and the reversed client id URL scheme (iOS).

## 5. Cloud Messaging (FCM)

See `references/fcm.md`. Checklist:

- Request permission (`requestPermission`) — required on iOS and Android 13+.
- iOS: upload APNs auth key in the Firebase console, enable Push Notifications + Background Modes (remote notifications) capabilities in Xcode.
- Save the token to Firestore under the user, and listen to `onTokenRefresh`.
- Background handler must be a **top-level** function annotated `@pragma('vm:entry-point')`, registered before `runApp`, and must call `Firebase.initializeApp` itself.
- Foreground messages are not shown automatically on Android — display them with `flutter_local_notifications` (create a high-importance channel).
- Handle taps: `getInitialMessage()` (app terminated) + `onMessageOpenedApp` (app in background) → navigate.

## 6. Security rules

Always write rules alongside new collections. Default to deny; see `references/rules.md` for templates. Test rules with the emulator (`@firebase/rules-unit-testing`) before deploying: `firebase deploy --only firestore:rules`.

## 7. Emulators for development & tests

```bash
firebase init emulators   # auth, firestore, storage, functions
firebase emulators:start
```

```dart
if (useEmulator) {
  final host = Platform.isAndroid ? '10.0.2.2' : 'localhost';
  FirebaseFirestore.instance.useFirestoreEmulator(host, 8080);
  await FirebaseAuth.instance.useAuthEmulator(host, 9099);
  await FirebaseStorage.instance.useStorageEmulator(host, 9199);
}
```

For unit tests without emulators, use `fake_cloud_firestore` and `firebase_auth_mocks`.

## 8. Crashlytics

```dart
FlutterError.onError = FirebaseCrashlytics.instance.recordFlutterFatalError;
PlatformDispatcher.instance.onError = (error, stack) {
  FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
  return true;
};
```

Disable collection in debug: `setCrashlyticsCollectionEnabled(!kDebugMode)`.

## 9. Common errors → fixes

- `[core/no-app] No Firebase App '[DEFAULT]' has been created` → `initializeApp` missing or not awaited (also in the FCM background handler).
- `PERMISSION_DENIED` → security rules; check the rule path and `request.auth`.
- `FAILED_PRECONDITION: The query requires an index` → create the composite index from the link.
- Google sign-in `ApiException: 10` → missing/wrong SHA-1 in console; re-download `google-services.json`.
- iOS build fails after adding a plugin → `cd ios && pod repo update && pod install`; check deployment target.
- No push on iOS → APNs key missing, capability missing, or testing on simulator without proper support.
