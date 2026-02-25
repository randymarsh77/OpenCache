---
sidebar_position: 5
---

# GitHub Actions

OpenCache provides two composable actions for integrating Nix binary caching into your CI workflows. The **setup** action snapshots the Nix store before builds, enabling automatic detection of new store paths — no need to manually capture build output.

## Standalone (Single Build)

Use **setup** before your build and **deploy** after. New store paths are auto-detected:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: DeterminateSystems/nix-installer-action@main

      - uses: randymarsh77/OpenCache/setup@v1

      - name: Build
        run: nix build

      - uses: randymarsh77/OpenCache/deploy@v1
        with:
          backend: github-releases
          github-token: ${{ secrets.GITHUB_TOKEN }}
          static: ./site
```

## Matrix Builds

Each matrix job runs **setup** + **deploy** independently — store paths and narinfo are pushed directly to the backend from every job. During static site generation, narinfo from all previous deploys (including other matrix jobs) is fetched from the release, so the generated site is always a complete manifest:

```yaml
jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: DeterminateSystems/nix-installer-action@main

      - uses: randymarsh77/OpenCache/setup@v1

      - name: Build
        run: nix build

      - uses: randymarsh77/OpenCache/deploy@v1
        with:
          backend: github-releases
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

## With magic-nix-cache

If you already use [DeterminateSystems/magic-nix-cache-action](https://github.com/DeterminateSystems/magic-nix-cache-action), you can hook the deploy action into the magic-nix-cache daemon to discover built paths automatically — no separate setup step needed. The deploy action detects new store paths using the daemon's startup timestamp:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: DeterminateSystems/nix-installer-action@main
      - uses: DeterminateSystems/magic-nix-cache-action@main

      - name: Build
        run: nix build

      - uses: randymarsh77/OpenCache/deploy@v1
        with:
          magic-nix-cache-addr: '127.0.0.1:37515'
          backend: github-releases
          github-token: ${{ secrets.GITHUB_TOKEN }}
          static: ./site
```

This lets you benefit from both magic-nix-cache (fast GitHub Actions cache for CI) and OpenCache (permanent binary cache via GitHub Releases + Pages).

## Explicit Paths (Legacy)

You can still pass explicit store paths if preferred:

```yaml
      - name: Build
        run: nix build --print-out-paths | tee /tmp/store-paths.txt

      - uses: randymarsh77/OpenCache/deploy@v1
        with:
          paths-file: /tmp/store-paths.txt
```

## Action Reference

### `setup`

Snapshots the current Nix store so new paths can be auto-detected by the deploy action.

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `store-dir` | no | `/nix/store` | Path to the Nix store directory |

| Output | Description |
|--------|-------------|
| `snapshot-path` | Path to the file containing the initial store snapshot |

### `deploy`

Starts a temporary OpenCache server, pushes store paths to the configured backend, and optionally generates a static site.

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `paths` | ¹ | | Newline-separated store paths |
| `paths-file` | ¹ | | File listing store paths |
| `export-dir` | no | | Binary cache export dir. When set, NARs are read from this directory instead of the local nix store. |
| `snapshot-path` | no | `/tmp/opencache-setup/store-paths-before.txt` | Store snapshot from `setup` (for auto-detection). The default matches the `setup` action output. |
| `store-dir` | no | `/nix/store` | Path to the Nix store directory |
| `magic-nix-cache-addr` | ¹ | | Address of a running magic-nix-cache daemon (default for magic-nix-cache-action is `127.0.0.1:37515`). Notifies the daemon and detects new paths automatically — no `setup` action needed. |
| `backend` | no | `github-releases` | Storage backend |
| `github-token` | no | | GitHub token (required for `github-releases`) |
| `github-owner` | no | *current owner* | Repository owner |
| `github-repo` | no | *current repo* | Repository name |
| `github-release-tag` | no | `nix-cache` | Release tag for NAR storage |
| `signing-key` | no | | Nix signing key |
| `upload-secret` | no | | Bearer token for upload auth |
| `static` | no | | Output dir for static site generation |
| `port` | no | `18734` | Temporary server port |
| `compression` | no | `none` | Compression for `nix copy` |

¹ One of `paths`, `paths-file`, or `magic-nix-cache-addr` is required unless the `setup` action was used (which provides the default `snapshot-path`).
