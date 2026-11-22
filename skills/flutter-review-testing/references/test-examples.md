# Test templates

## Notifier test (Riverpod + mocktail)

```dart
class MockCartRepository extends Mock implements CartRepository {}

void main() {
  late MockCartRepository repo;
  late ProviderContainer container;

  setUpAll(() {
    registerFallbackValue(FakeProduct());
  });

  setUp(() {
    repo = MockCartRepository();
    container = ProviderContainer(
      overrides: [cartRepositoryProvider.overrideWithValue(repo)],
    );
    addTearDown(container.dispose);
    // Riverpod 3: ProviderContainer.test(overrides: [...]) disposes automatically
  });

  test('build() loads items from repository', () async {
    when(() => repo.getItems()).thenAnswer((_) async => [item1]);

    final result = await container.read(cartProvider.future);

    expect(result, [item1]);
    verify(() => repo.getItems()).called(1);
  });

  test('add() ends in AsyncError when repository throws', () async {
    when(() => repo.getItems()).thenAnswer((_) async => []);
    when(() => repo.add(any())).thenThrow(Exception('network'));

    // keep the auto-dispose provider alive during the test
    final sub = container.listen(cartProvider, (_, __) {});
    await container.read(cartProvider.future);

    await container.read(cartProvider.notifier).add(product);

    expect(sub.read(), isA<AsyncError>());
  });
}
```

## Recording state transitions

```dart
class Listener<T> extends Mock {
  void call(T? previous, T next);
}

test('emits loading then data', () async {
  final listener = Listener<AsyncValue<List<CartItem>>>();
  container.listen(cartProvider, listener.call, fireImmediately: true);
  await container.read(cartProvider.future);

  verifyInOrder([
    () => listener(null, const AsyncLoading<List<CartItem>>()),
    () => listener(any(), any(that: isA<AsyncData<List<CartItem>>>())),
  ]);
});
```

## Repository test (Dio)

```dart
class MockDio extends Mock implements Dio {}

test('maps 404 to NotFoundFailure', () async {
  final dio = MockDio();
  when(() => dio.get(any())).thenThrow(DioException(
    requestOptions: RequestOptions(path: '/p/1'),
    response: Response(requestOptions: RequestOptions(path: '/p/1'), statusCode: 404),
  ));

  final repo = ProductsRepositoryImpl(dio);

  expect(() => repo.getById('1'), throwsA(isA<NotFoundFailure>()));
});
```

## Firestore data source with fake_cloud_firestore

```dart
test('watchUserOrders returns only the user orders', () async {
  final db = FakeFirebaseFirestore();
  await db.collection('orders').add({'userId': 'u1', 'total': 10, 'status': 'pending'});
  await db.collection('orders').add({'userId': 'u2', 'total': 5, 'status': 'pending'});

  final ds = OrdersRemoteDataSource(db);
  final orders = await ds.watchUserOrders('u1').first;

  expect(orders, hasLength(1));
  expect(orders.single.userId, 'u1');
});
```

## Widget test with overrides

```dart
Widget wrap(Widget child, {List<Override> overrides = const []}) =>
    ProviderScope(
      overrides: overrides,
      child: MaterialApp(home: child),
    );

testWidgets('shows empty message when cart is empty', (tester) async {
  await tester.pumpWidget(wrap(
    const CartScreen(),
    overrides: [cartProvider.overrideWith(() => FakeCartNotifier([]))],
  ));
  await tester.pump(); // let the future resolve

  expect(find.text('Your cart is empty'), findsOneWidget);
});

testWidgets('retry button reloads after error', (tester) async {
  var calls = 0;
  when(() => repo.getItems()).thenAnswer((_) async {
    calls++;
    if (calls == 1) throw Exception('boom');
    return [item1];
  });

  await tester.pumpWidget(wrap(const CartScreen(),
      overrides: [cartRepositoryProvider.overrideWithValue(repo)]));
  await tester.pump();

  await tester.tap(find.text('Retry'));
  await tester.pump();
  await tester.pump();

  expect(find.byType(CartTile), findsOneWidget);
});
```

A fake notifier for overrides:

```dart
class FakeCartNotifier extends Cart { // or extends CartNotifier
  FakeCartNotifier(this.items);
  final List<CartItem> items;
  @override
  Future<List<CartItem>> build() async => items;
}
```

## Golden test

```dart
testWidgets('ProductCard golden', (tester) async {
  await tester.binding.setSurfaceSize(const Size(400, 200));
  await tester.pumpWidget(MaterialApp(
    theme: AppTheme.light,
    home: Scaffold(body: ProductCard(product: sampleProduct)),
  ));
  await expectLater(
    find.byType(ProductCard),
    matchesGoldenFile('goldens/product_card.png'),
  );
});
```

## Integration test

`integration_test/login_flow_test.dart`:

```dart
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('user can log in and see home', (tester) async {
    app.main();
    await tester.pumpAndSettle();

    await tester.enterText(find.byKey(const Key('email')), 'test@test.com');
    await tester.enterText(find.byKey(const Key('password')), 'secret123');
    await tester.tap(find.byKey(const Key('login')));
    await tester.pumpAndSettle(const Duration(seconds: 3));

    expect(find.byKey(const Key('home_screen')), findsOneWidget);
  });
}
```

Run: `flutter test integration_test` (on a device/emulator). Point the app at Firebase emulators or a test backend.
