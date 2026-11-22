# FCM push notifications

```dart
@pragma('vm:entry-point')
Future<void> firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  // keep this light: no UI, no providers
}

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  FirebaseMessaging.onBackgroundMessage(firebaseMessagingBackgroundHandler);
  runApp(const ProviderScope(child: App()));
}
```

## Service

```dart
class PushService {
  PushService(this._fm, this._local, this._saveToken);
  final FirebaseMessaging _fm;
  final FlutterLocalNotificationsPlugin _local;
  final Future<void> Function(String token) _saveToken;

  static const channel = AndroidNotificationChannel(
    'high_importance', 'Important notifications',
    importance: Importance.high,
  );

  Future<void> init({required void Function(RemoteMessage) onTap}) async {
    await _fm.requestPermission(alert: true, badge: true, sound: true);

    await _local
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>()
        ?.createNotificationChannel(channel);

    // iOS: show notifications while in foreground
    await _fm.setForegroundNotificationPresentationOptions(
        alert: true, badge: true, sound: true);

    final token = await _fm.getToken();
    if (token != null) await _saveToken(token);
    _fm.onTokenRefresh.listen(_saveToken);

    // Android foreground: show manually
    FirebaseMessaging.onMessage.listen((m) {
      final n = m.notification;
      if (n == null) return;
      _local.show(
        n.hashCode, n.title, n.body,
        NotificationDetails(
          android: AndroidNotificationDetails(channel.id, channel.name,
              importance: Importance.high, priority: Priority.high),
        ),
        payload: jsonEncode(m.data),
      );
    });

    final initial = await _fm.getInitialMessage();
    if (initial != null) onTap(initial);
    FirebaseMessaging.onMessageOpenedApp.listen(onTap);
  }
}
```

Also wire `flutter_local_notifications` `onDidReceiveNotificationResponse` to parse the `payload` and navigate, so taps on foreground notifications work too.

## Store tokens

Save under `users/{uid}/tokens/{token}` with `createdAt` and `platform`, so one user can have several devices, and delete the token on logout (`_fm.deleteToken()`).

## Sending

Send from a trusted server (Cloud Functions / backend) via the FCM HTTP v1 API — never put server credentials in the app. Use `data` payload for navigation (`{"type": "order", "id": "123"}`).
