# BuildKit cache mounts not exported by `--cache-to`

**Upstream:** [moby/buildkit#1512](https://github.com/moby/buildkit/issues/1512)

## Problem

`RUN --mount=type=cache` contents aren't exported by `--cache-to`, so ephemeral CI
runners start every build with empty package/compiler caches.

## Repro — confirmed 2026-09-28

**Result:** Reproduced. A fresh builder that imports the saved cache gets an empty cache
mount, while the layer cache imports correctly.

### Environment

| Component | Version |
|---|---|
| Docker Engine | 29.8.1 |
| buildx | v0.37.1 |
| BuildKit (builder image `moby/buildkit:buildx-stable-1`) | v0.32.2 |
| Driver | `docker-container` |
| Host | WSL2 Ubuntu 24.04 on Windows 11 Home |

### Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.20
ARG BUST=0
RUN --mount=type=cache,target=/cache,id=demo \
    [ -f /cache/marker ] && echo WARM || echo COLD; \
    date +%s > /cache/marker
```

- The `RUN` step prints `WARM` if `/cache/marker` exists (mount survived) or `COLD` if not,
  then writes the marker.
- Changing `BUST` changes the step's inputs, forcing it to re-run instead of `CACHED`.
  The cache mount's contents are not a step input.

### Steps and actual output

**1. Builder 1: first build, export cache**

```bash
docker buildx create --name b1 --driver docker-container
docker buildx build --builder b1 --progress=plain --build-arg BUST=1 \
  --cache-to type=local,dest=./bk,mode=max .
```
```
#7 0.098 COLD
```
Expected — first use of the mount.

**2. Builder 1 again: control, mount persists on same builder**

```bash
docker buildx build --builder b1 --progress=plain --build-arg BUST=2 .
```
```
#7 0.074 WARM
```

**3. Inspect exported cache for the marker**

```bash
for f in ./bk/blobs/sha256/*; do
  tar -tzf "$f" 2>/dev/null | grep -q marker && echo "marker found in $f"
done; echo "search done"
```
```
search done
```

Control — same search for a file known to be in the saved Alpine layer:

```bash
for f in ./bk/blobs/sha256/*; do
  tar -tzf "$f" 2>/dev/null | grep -q "bin/sh" && echo "bin/sh found in $f"
done; echo "search done"
```
```
bin/sh found in ./bk/blobs/sha256/25f1d6b1951ac8eb3740558fe94cb83d377bdadf95fd9f98b50d2e1b96130471
search done
```
The export contains image files but not the cache-mount contents.

**4. Simulate a fresh CI runner**

```bash
docker buildx rm b1
docker buildx create --name b2 --driver docker-container
```

**5. Builder 2: import cache, force re-run — the bug**

```bash
docker buildx build --builder b2 --progress=plain --build-arg BUST=3 \
  --cache-from type=local,src=./bk .
```
```
#7 importing cache manifest from local:10418442000894224608
#9 0.093 COLD
```
Cache imported, but the mount is empty.

**6. Builder 2: control, import works**

```bash
docker buildx build --builder b2 --progress=plain --build-arg BUST=1 \
  --cache-from type=local,src=./bk .
```
```
#6 importing cache manifest from local:10418442000894224608
#8 CACHED
```
Builder 2 never ran the step with `BUST=1`; the result came from `./bk`.

### Summary

| Step | Builder | Result | Proves |
|---|---|---|---|
| 1 | b1 (new) | COLD | Baseline |
| 2 | b1 (same) | WARM | Cache mounts persist on the same builder |
| 3 | — | marker absent, `bin/sh` present | Export excludes cache-mount contents |
| 5 | b2 (fresh + import) | COLD | **Bug:** mount empty on fresh builder |
| 6 | b2 (fresh + import) | CACHED | Layer cache import works |

**Expected:** `WARM` in step 5, or a supported way to carry mount contents to a new builder.
**Actual:** `COLD`, with no warning at export or import.

### Cleanup

```bash
docker buildx rm b2 && rm -rf ./bk
```

## Status

Open, help wanted. Design accepted upstream (pluggable cache-mount storage backend),
nobody working on it.

## Current workaround

reproducible-containers/buildkit-cache-dance (slow on large caches)

## Fix options (undecided)

- **A.** Implement upstream design (Go, buildkit + buildx)
- **B.** Improve cache-dance with incremental extraction (TS)
- **C.** CI-agnostic Go CLI for cache mount export/import
