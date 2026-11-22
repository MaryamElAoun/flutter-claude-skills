---
name: flutter-review-testing
description: Review Flutter/Dart code and write tests for it — unit tests, Riverpod notifier tests with ProviderContainer, widget tests, golden tests and integration tests using flutter_test and mocktail. Use when asked to review a PR or file, find bugs or performance issues in Flutter code, add tests, raise coverage, or set up CI checks.
---

# Flutter code review & testing

## Part A — Code review

### How to review

1. Run the tools first (if a shell is available) and include real output:
   - `dart format --output=none --set-exit-if-changed .`
   - `flutter analyze`
   - `flutter test`
2. Read the diff/file top-down, then go through the checklist below.
3. Report findings grouped by severity:
   - **🔴 Must fix** — bugs, crashes, security, data loss.
   - **🟡 Should fix** — performance, architecture, missing tests, error handling.
   - **🟢 Nit** — naming, style, small readability tweaks.
   Each finding: file:line, what's wrong, why it matters, and a concrete code fix.
4. End with what is good about the change (briefly) and the top 1–3 things to do next.

Don't pad the review with style nits that `flutter analyze` / `dart format` already enforce.

### Checklist

**Correctness & safety**
- `BuildContext` used after `await` without `if (!context.mounted) return;`.
- `setState` after dispose / missing `mounted` check.
- Controllers, `StreamSubscription`, `AnimationController`, `Timer`, `FocusNode` not disposed.
- Unhandled futures (no `await`, no error handling) — `unawaited_futures` lint.
- Force unwrap `!` on values that can really be null (API/JSON data).
- `late` fields that may be read before init.
- `catch (e)` that swallows errors silently.
- Wrong equality: models without `==`/`hashCode` (use freezed/equatable) used in state or sets.
- Riverpod: `ref.read` inside `build`, `ref.watch` inside callbacks, missing `ref.mounted` after `await`, side effects inside `build` instead of `ref.listen`.

**Performance**
- Missing `const` constructors on static widgets.
- Huge `build` methods → extract widgets (classes, not helper methods returning widgets).
- `ListView(children: [...])` with many/unbounded items → `ListView.builder`.
- Heavy work (JSON parsing of big payloads, image processing) on the UI isolate → `compute`/`Isolate.run`.
- `Image.network` without caching/size hints in lists → `cached_network_image`, `cacheWidth`.
- `Opacity`/`ClipRRect`/shadows animated in long lists; `MediaQuery.of(context)` where `MediaQuery.sizeOf(context)` would avoid rebuilds.
- Watching a whole provider where `select` would do.

**Architecture**
- UI calling HTTP/Firebase directly instead of through a repository.
- Business logic inside widgets.
- Hardcoded strings (should be in l10n), colors/text styles (should be in theme), magic numbers.
- Duplicate code that already exists in `core/`.

**Security**
- API keys/secrets in source code; tokens stored in `SharedPreferences` instead of `flutter_secure_storage`.
- Logging sensitive data (`print`/`debugPrint` of tokens, passwords).
- Missing input validation; trusting client-side checks for things the backend must enforce.

**UX & accessibility**
- No loading / empty / error states.
- Tap targets < 48dp, missing `Semantics`/`tooltip` on icon buttons.
- Layout overflow on small screens or with large text scale; RTL issues (use `EdgeInsetsDirectional`, `AlignmentDirectional`, `start/end`).
- TV apps: every interactive element reachable by D-pad focus.

## Part B — Writing tests

Packages: `flutter_test` (SDK), `mocktail`, and as needed `fake_cloud_firestore`, `firebase_auth_mocks`, `integration_test` (SDK). Check `pubspec.yaml` and follow what the project already uses (e.g. `mockito` instead of `mocktail`).

Mirror `lib/` under `test/`: `lib/features/cart/...` → `test/features/cart/..._test.dart`.

### What to test first (highest value)
1. Notifiers / business logic (pure Dart, fast).
2. Repositories: mapping of data and errors.
3. Widget tests for screens with states (loading, data, empty, error) and key interactions.
4. A few integration tests for critical flows (login, checkout).

See `references/test-examples.md` for templates of each.

### Rules
- Arrange / Act / Assert, one behavior per test, descriptive names: `'add() emits loading then data with new item'`.
- Override providers instead of mocking Riverpod internals.
- Always test the error path, not only the happy path.
- `registerFallbackValue` for custom types used with `any()` in mocktail.
- In widget tests, wrap in `ProviderScope(overrides: [...])` + `MaterialApp` (plus localization delegates if the app uses them); use `pumpAndSettle` carefully (it hangs on infinite animations like `CircularProgressIndicator` → use `pump()`).
- Use `find.byKey(const Key('...'))` for elements that are hard to find by text.
- Goldens: generate with `flutter test --update-goldens`, run on one fixed platform in CI (fonts differ between OSes).

### Coverage & CI
```bash
flutter test --coverage
# optional HTML report
genhtml coverage/lcov.info -o coverage/html
```
Typical CI steps: `flutter pub get` → `dart format --set-exit-if-changed .` → `flutter analyze` → `flutter test --coverage`. A GitHub Actions template is in `references/ci.md`.
