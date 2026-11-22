# Firestore patterns

## Model with Timestamp converter

```dart
class TimestampConverter implements JsonConverter<DateTime?, Object?> {
  const TimestampConverter();
  @override
  DateTime? fromJson(Object? json) => switch (json) {
        Timestamp t => t.toDate(),
        String s => DateTime.tryParse(s),
        _ => null,
      };
  @override
  Object? toJson(DateTime? date) =>
      date == null ? null : Timestamp.fromDate(date);
}

@freezed
class Order with _$Order {
  const factory Order({
    @JsonKey(includeToJson: false) @Default('') String id,
    required String userId,
    required double total,
    required String status,
    @TimestampConverter() DateTime? createdAt,
  }) = _Order;

  factory Order.fromJson(Map<String, dynamic> json) => _$OrderFromJson(json);
}
```

## Typed collection reference

```dart
class OrdersRemoteDataSource {
  OrdersRemoteDataSource(this._db);
  final FirebaseFirestore _db;

  CollectionReference<Order> get _orders =>
      _db.collection('orders').withConverter<Order>(
            fromFirestore: (snap, _) =>
                Order.fromJson({...snap.data()!, 'id': snap.id}),
            toFirestore: (order, _) => order.toJson(),
          );

  Stream<List<Order>> watchUserOrders(String uid) => _orders
      .where('userId', isEqualTo: uid)
      .orderBy('createdAt', descending: true)
      .snapshots()
      .map((s) => s.docs.map((d) => d.data()).toList());

  Future<String> create(Order order) async {
    final ref = _db.collection('orders').doc();
    await ref.set({
      ...order.toJson(),
      'createdAt': FieldValue.serverTimestamp(),
    });
    return ref.id;
  }
}
```

## Cursor pagination

```dart
class Page<T> {
  Page(this.items, this.lastDoc, this.hasMore);
  final List<T> items;
  final DocumentSnapshot? lastDoc;
  final bool hasMore;
}

Future<Page<Order>> fetchPage({DocumentSnapshot? after, int size = 20}) async {
  Query<Order> q = _orders.orderBy('createdAt', descending: true).limit(size);
  if (after != null) q = q.startAfterDocument(after);
  final snap = await q.get();
  return Page(
    snap.docs.map((d) => d.data()).toList(),
    snap.docs.isEmpty ? after : snap.docs.last,
    snap.docs.length == size,
  );
}
```

Hold the pages in an `AsyncNotifier` that appends items and keeps `lastDoc` + `hasMore`; call `loadMore()` from a scroll listener when near the bottom and not already loading.

## Transaction (stock decrement)

```dart
await _db.runTransaction((tx) async {
  final ref = _db.collection('products').doc(productId);
  final snap = await tx.get(ref);
  final stock = (snap.data()?['stock'] as num?)?.toInt() ?? 0;
  if (stock < qty) throw OutOfStockException();
  tx.update(ref, {'stock': stock - qty});
});
```

For simple counters with no condition, `FieldValue.increment(n)` is cheaper than a transaction.

## Batch

```dart
final batch = _db.batch();
for (final id in ids) {
  batch.update(_db.collection('notifications').doc(id), {'read': true});
}
await batch.commit(); // max 500 operations per batch
```
