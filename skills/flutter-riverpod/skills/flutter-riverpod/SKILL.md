---
name: flutter-riverpod
description: Build and refactor Flutter features with Riverpod (v2/v3, codegen or manual) using a feature-first clean architecture. Use when creating a new screen or feature, adding providers/notifiers, wiring repositories, handling async state, or fixing Riverpod errors like "ref used after dispose" or unwanted rebuilds.
---

# Flutter + Riverpod feature builder

Build Flutter features that are consistent, testable, and easy to read, using Riverpod for state and dependency injection.

## 1. Check the project first

Before writing code, inspect the project so new code matches it:

1. Read `pubspec.yaml`: note the versions of `flutter_riverpod` / `hooks_riverpod`, `riverpod_annotation`, `riverpod_generator`, `freezed`, `json_serializable`, `go_router`, `dio`.
   - `riverpod_generator` present → use `@riverpod` codegen.
   - Not present → use manual providers (`NotifierProvider`, `AsyncNotifierProvider`, `Provider`, `FutureProvider`, `StreamProvider`).
   - Major version 3.x → use the unified `Ref` (no `XxxRef` types), `ref.mounted`, and be aware of automatic retry on failed providers.
2. Look at one existing feature folder and copy its structure, naming and error handling. **Project conventions beat this skill.**
3. If the project has no structure yet, use the layout below.

## 2. Default folder layout (feature-first)

```
lib/
  core/            # shared: network client, errors, theme, router, utils
  features/
    <feature>/
      data/
        models/          # DTOs (freezed + json_serializable)
        datasources/     # remote/local sources (Dio, Firestore, Hive...)
        repositories/    # implementation of domain repository
      domain/
        entities/        # pure Dart classes (optional for small apps)
        repositories/    # abstract repository
      presentation/
        providers/       # notifiers + providers
        screens/
        widgets/
```

For small features, `domain/` can be skipped: the repository class in `data/` is enough. Don't add layers that carry no logic.

## 3. Provider recipes

Pick the smallest provider that fits:

| Need | Codegen | Manual |
|---|---|---|
| Dependency (repo, client) | `@riverpod Repo repo(Ref ref)` | `Provider<Repo>` |
| Read-only async data | `@riverpod Future<T> x(Ref ref)` | `FutureProvider` |
| Stream (Firestore, socket) | `@riverpod Stream<T> x(Ref ref)` | `StreamProvider` |
| Sync state with methods | `class X extends _$X` returning `T` | `NotifierProvider` |
| Async state with methods | `class X extends _$X` returning `Future<T>` | `AsyncNotifierProvider` |
| Parameterized (id, filter) | add function/build params | `.family` |

Do not use `StateProvider`, `StateNotifierProvider` or `ChangeNotifierProvider` in new code — they are legacy. Migrate them only when asked.

See `references/patterns.md` for full examples of each.

## 4. Rules to follow

- **`ref.watch` in `build`, `ref.read` in callbacks.** Never `ref.read` inside `build` to get reactive values; never `ref.watch` inside `onPressed`.
- **Use `select`** when a widget needs one field of a big state: `ref.watch(userProvider.select((u) => u.name))`.
- **AsyncValue in the UI**: render with `switch` on `AsyncValue` (`AsyncData`, `AsyncError`, `AsyncLoading`) or `.when(...)`. Always handle error and loading.
- **Mutations in AsyncNotifier**: set `state = const AsyncLoading()` then `state = await AsyncValue.guard(() => ...)`. If keeping old data while reloading matters, use `AsyncLoading<T>().copyWithPrevious(state)` (v2) or check `state.isLoading && state.hasValue` in the UI.
- **After `await`, check `ref.mounted`** (v3) before touching `state` or other providers.
- **Keep alive deliberately**: codegen providers are auto-dispose by default. Use `@Riverpod(keepAlive: true)` only for app-wide things (auth, settings, repositories).
- **Side effects (snackbars, navigation)** go in `ref.listen` inside `build`, not in the notifier.
- **No `BuildContext` in notifiers.** Notifiers must stay testable without widgets.
- **Repositories are injected through providers** so tests can override them.
- **Errors**: convert low-level exceptions (DioException, FirebaseException) into a project `Failure`/`AppException` type in the repository layer, not in the UI.

## 5. Workflow for a new feature

1. Model: `freezed` data class + `fromJson`.
2. Data source + repository (abstract + implementation), with a provider for each.
3. Notifier/provider for the screen state.
4. Screen as `ConsumerWidget` (or `ConsumerStatefulWidget` if it needs controllers/animations; `HookConsumerWidget` if the project uses hooks).
5. Route registration (follow the project's router).
6. If codegen is used, remind to run:
   `dart run build_runner build -d` (or `watch -d` during development).
7. Add at least one notifier unit test using `ProviderContainer` with an overridden repository (see the `flutter-review-testing` skill).

## 6. Common errors → fixes

- **"Cannot use ref after the widget was disposed" / "ref used after dispose"** → check `ref.mounted` (or `context.mounted` in widgets) after every `await`.
- **Provider rebuilds too often** → use `select`, split the provider, or move the `watch` lower in the widget tree.
- **State resets when leaving the screen** → auto-dispose is working as designed; use `keepAlive: true` or `ref.keepAlive()` only if the data really should survive.
- **"Bad state: Tried to read a provider from a ProviderScope that was disposed"** → a callback runs after navigation; guard with `mounted`.
- **Generated file missing (`part 'x.g.dart'`)** → add the `part` line and run build_runner.
- **Infinite loop / circular dependency** → two providers watch each other; extract shared logic into a third provider.
