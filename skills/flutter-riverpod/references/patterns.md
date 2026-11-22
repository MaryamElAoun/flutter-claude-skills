# Riverpod patterns (codegen first, manual below)

## Repository provider

```dart
// codegen
@Riverpod(keepAlive: true)
ProductsRepository productsRepository(Ref ref) {
  return ProductsRepositoryImpl(ref.watch(dioProvider));
}

// manual
final productsRepositoryProvider = Provider<ProductsRepository>((ref) {
  return ProductsRepositoryImpl(ref.watch(dioProvider));
});
```

## Read-only async list with a parameter

```dart
@riverpod
Future<List<Product>> productsByCategory(Ref ref, String categoryId) {
  return ref.watch(productsRepositoryProvider).fetchByCategory(categoryId);
}

// usage
final products = ref.watch(productsByCategoryProvider(categoryId));
```

Manual equivalent:

```dart
final productsByCategoryProvider =
    FutureProvider.autoDispose.family<List<Product>, String>((ref, categoryId) {
  return ref.watch(productsRepositoryProvider).fetchByCategory(categoryId);
});
```

## AsyncNotifier with mutations

```dart
@riverpod
class Cart extends _$Cart {
  @override
  Future<List<CartItem>> build() {
    return ref.watch(cartRepositoryProvider).getItems();
  }

  Future<void> add(Product p) async {
    final repo = ref.read(cartRepositoryProvider);
    state = const AsyncLoading();
    state = await AsyncValue.guard(() async {
      await repo.add(p);
      return repo.getItems();
    });
  }

  Future<void> remove(String id) async {
    final previous = state.valueOrNull ?? [];
    // optimistic update
    state = AsyncData(previous.where((i) => i.id != id).toList());
    try {
      await ref.read(cartRepositoryProvider).remove(id);
    } catch (e, st) {
      if (!ref.mounted) return;
      state = AsyncError(e, st);
      state = AsyncData(previous); // roll back
    }
  }
}
```

> In Riverpod 3, `valueOrNull` was renamed to `value` (which now returns null when there is no data). Match the project's version.

Manual equivalent:

```dart
class CartNotifier extends AsyncNotifier<List<CartItem>> {
  @override
  Future<List<CartItem>> build() => ref.watch(cartRepositoryProvider).getItems();
  // same methods
}

final cartProvider =
    AsyncNotifierProvider<CartNotifier, List<CartItem>>(CartNotifier.new);
```

## Sync Notifier (form / filter state)

```dart
@freezed
class ProductFilter with _$ProductFilter {
  const factory ProductFilter({
    @Default('') String query,
    @Default(SortBy.newest) SortBy sort,
  }) = _ProductFilter;
}

@riverpod
class ProductFilterController extends _$ProductFilterController {
  @override
  ProductFilter build() => const ProductFilter();

  void setQuery(String q) => state = state.copyWith(query: q);
  void setSort(SortBy s) => state = state.copyWith(sort: s);
}

// derived provider: re-runs when the filter changes
@riverpod
Future<List<Product>> filteredProducts(Ref ref) async {
  final filter = ref.watch(productFilterControllerProvider);
  return ref.watch(productsRepositoryProvider).search(filter);
}
```

## Debounced search

```dart
@riverpod
Future<List<Product>> searchResults(Ref ref, String query) async {
  if (query.trim().isEmpty) return [];
  var cancelled = false;
  ref.onDispose(() => cancelled = true);
  await Future.delayed(const Duration(milliseconds: 400));
  if (cancelled) throw Exception('cancelled');
  return ref.watch(productsRepositoryProvider).search(query);
}
```

## Stream provider (Firestore / realtime)

```dart
@riverpod
Stream<List<Order>> userOrders(Ref ref) {
  final uid = ref.watch(authStateProvider).value?.uid;
  if (uid == null) return const Stream.empty();
  return ref.watch(ordersRepositoryProvider).watchOrders(uid);
}
```

## Screen

```dart
class CartScreen extends ConsumerWidget {
  const CartScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    ref.listen(cartProvider, (prev, next) {
      if (next is AsyncError) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(next.error.toString())),
        );
      }
    });

    final cart = ref.watch(cartProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Cart')),
      body: switch (cart) {
        AsyncData(:final value) when value.isEmpty =>
          const Center(child: Text('Your cart is empty')),
        AsyncData(:final value) => ListView.builder(
            itemCount: value.length,
            itemBuilder: (_, i) => CartTile(item: value[i]),
          ),
        AsyncError(:final error) => ErrorView(
            message: error.toString(),
            onRetry: () => ref.invalidate(cartProvider),
          ),
        _ => const Center(child: CircularProgressIndicator()),
      },
    );
  }
}
```

## Pull to refresh

```dart
RefreshIndicator(
  onRefresh: () => ref.refresh(cartProvider.future),
  child: ...,
)
```

## Auth-dependent providers

Watch the auth provider inside any provider that depends on the current user, so data clears automatically on logout:

```dart
@Riverpod(keepAlive: true)
Stream<User?> authState(Ref ref) =>
    ref.watch(firebaseAuthProvider).authStateChanges();
```
