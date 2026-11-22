# GitHub Actions CI for Flutter

`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          channel: stable
          cache: true

      - run: flutter pub get

      # only if the project uses codegen
      - run: dart run build_runner build -d

      - name: Format
        run: dart format --output=none --set-exit-if-changed .

      - name: Analyze
        run: flutter analyze

      - name: Test
        run: flutter test --coverage

      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/lcov.info
```

Pin the Flutter version (`flutter-version: 3.x.y`) instead of `channel: stable` if the team needs reproducible builds.
