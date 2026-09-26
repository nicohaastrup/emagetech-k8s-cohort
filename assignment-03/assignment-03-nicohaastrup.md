# Assignment 03 — Nicodemus Haastrup

**GitHub username:** nicohaastrup  
**Date completed:** 2026-06-24  
**Git SHA of submitted app:** bde8add

## 1. Size comparison table

| Variant            | Size  | Layers | Stop time | Exit code |
|--------------------|-------|--------|-----------|-----------|
| `cohort-greet:naive` | 1.63GB | 15     | 5.163 total     | 137         |
| `cohort-greet:multi` | 264MB | 10     | 0.230 total     | 0         |

(Layers counted from `docker image history` output minus 1 for the header.)

## 2. Final image digest

`sha256:3cc85e19a9a05aedf95fe6299c495a69d0a1a72af6a1db2e7d2d7e871c401f0c`

## 3. Answers to the 7 questions

**Q1 — naive size + stop behaviour + why:**  
The naive image size is **1.63GB**. The stop time is **5.163 total** (about 5 seconds). The exit code is **137** (SIGKILL). This happens because the Dockerfile uses **shell form** `CMD gunicorn -b 0.0.0.0:8080 app:app` instead of exec form. Shell form runs gunicorn inside a `/bin/sh` wrapper, so when `docker stop` sends SIGTERM, it goes to the shell process, not directly to gunicorn. The shell doesn't forward SIGTERM properly, so after the timeout expires, Docker sends SIGKILL (code 137). The ~5 second stop time matches the default timeout of 5 seconds, confirming the app didn't shut down gracefully.

**Q2 — build output, CACHED vs rebuilt:**  
Build output after touching only `app.py`:

```
#3 CACHED
#8 CACHED
#9 CACHED
#10 CACHED
#11 CACHED
#12 CACHED
#13 CACHED
#14 CACHED
```

Layers #3, #8, #9, #10, #11, #12, #13, #14 were all `CACHED`. The `RUN pip install` layer was cached because `requirements.txt` didn't change — only `app.py` was modified. The `COPY app.py .` layer was rebuilt (not shown as CACHED) because that's the only layer that depends on app.py content. This demonstrates proper layer ordering: `requirements.txt` is copied before `app.py`, so code changes don't bust the dependency install layer.

**Q3 — new stop time/exit + which change:**  
The multi-stage image stop time is **0.230 total** (about 0.2 seconds). The exit code is **0** (successful shutdown). This is dramatically faster than naive's ~5 seconds! The Dockerfile change responsible is using **exec form** `CMD ["gunicorn", "-b", "0.0.0.0:8080", "app:app"]` instead of shell form. Exec form passes the command directly to the process manager, so SIGTERM reaches gunicorn/python directly, allowing graceful shutdown. The ~0.2 second stop time shows the app received SIGTERM and shut down immediately.

**Q4 — size reduction breakdown:**  
Naive: **1.63GB** (1.63GB)  
Multi: **264MB** (264MB)  
Reduction: **83.8%** smaller

Size savings from:
1. **Base image**: `python:3.11.10-slim` (runtime stage) vs `python:3.11` — slim is significantly smaller than the full Python image
2. **Multi-stage build**: The runtime stage only includes `/opt/venv` copied from build, not the entire build toolchain (pip, build dependencies, etc.)
3. **`--no-cache-dir`**: Prevents pip's cache directory from being included in the image layer
4. **Clean venv**: Using `/opt/venv` instead of `/root/.local` or `pip install --user` gives a more controlled, smaller dependency set

The ~73MB pip install layer in naive vs the smaller multi-stage layers account for most of the difference.

**Q5 — cache-mount timings + CI relevance:**  
Cold build #1: **5.598 total** (5.598 seconds)  
Cold build #2: **4.242 total** (4.242 seconds)  
Saved: **~1.36 seconds** (about 24% faster)

The cache mount persists pip's download cache across builds, so even with `--no-cache` on the build itself, pip can reuse downloaded packages from the cache mount. In CI pipelines, this matters when:
- Your layer cache is cold (fresh CI runner)
- But you can persist a remote cache mount (e.g., to a shared volume or registry)
- Dependencies haven't changed, so pip can reuse cached downloads
- This reduces CI build time significantly, especially for Python with many dependencies

**Q6 — secret marker + what `ARG` would leak:**  
Token marker output: `d018`  
Leak check: **no leak**

Full token: `d018c4c155a1d869a3de1a75a9a79f2e`

If I'd used `ARG PYPI_TOKEN` instead of a secret mount, the token would appear in:
- `docker history` output (visible as an ARG value)
- The image layer metadata (accessible via `docker image inspect`)
- Anyone who can inspect the image can see the token

Secret mounts (`--mount=type=secret`) don't persist in the final image — the secret is only available during the build step and is not written to any layer. This keeps build-time secrets out of the image entirely.

**Q7 — tag vs digest for k8s manifest:**  
For a production Kubernetes manifest, I would use the **digest pin** (`sha256:3cc85e19a9a05aedf95fe6299c495a69d0a1a72af6a1db2e7d2d7e871c401f0c`) or the `:0.1.0-bde8add` tag.

Why:
- **Digest pin**: Guarantees exact reproducibility — the image won't change even if someone re-tags the semver. The digest is a content hash, so it's immutable.
- **Version-SHA tag** (`:0.1.0-bde8add`): Also very safe, ties the image to both a version and a specific git commit. Slightly more human-readable than a digest.

If security mandates *exact* reproducibility, **digest pin is mandatory**. Tags are mutable pointers; digests are immutable content addresses. Only the digest guarantees you're running the exact same bytes.

## 4. Files

### Final `Dockerfile`

```dockerfile
# syntax=docker/dockerfile:1.7

FROM python:3.11.10-slim AS build
RUN python -m venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt

FROM python:3.11.10-slim AS runtime
COPY --from=build /opt/venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH
WORKDIR /app
RUN useradd --uid 1000 app && chown -R app /app
COPY app.py .
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/healthz')"
USER app
CMD ["gunicorn", "-b", "0.0.0.0:8080", "app:app"]
```

### `Dockerfile.naive`

```dockerfile
FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 8080
CMD gunicorn -b 0.0.0.0:8080 app:app
```

### `Dockerfile.secret`

```dockerfile
# syntax=docker/dockerfile:1.7

FROM python:3.11.10-slim AS build
RUN python -m venv /opt/venv
ENV PATH=/opt/venv/bin:$PATH
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=secret,id=pypi_token,required=true \
    pip install --no-cache-dir -r requirements.txt && \
    token=$(cat /run/secrets/pypi_token | cut -c1-4) && \
    echo "$token" > /where-token-was-used

FROM python:3.11.10-slim AS runtime
COPY --from=build /opt/venv /opt/venv
COPY --from=build /where-token-was-used /where-token-was-used
ENV PATH=/opt/venv/bin:$PATH
WORKDIR /app
RUN useradd --uid 1000 app && chown -R app /app
COPY app.py .
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/healthz')"
USER app
CMD ["gunicorn", "-b", "0.0.0.0:8080", "app:app"]
```

### `.dockerignore`

```
.git/
.gitignore
__pycache__/
*.pyc
Dockerfile*
*.md
.env*
```

## 5. Evidence

### `docker image ls cohort-greet` (all tags)

```
REPOSITORY     TAG             IMAGE ID
cohort-greet   secret          4fcc9de1c1a0
cohort-greet   0.1.0           3cc85e19a9a0
cohort-greet   0.1.0-bde8add   3cc85e19a9a0
cohort-greet   git-bde8add     3cc85e19a9a0
cohort-greet   multi           3cc85e19a9a0
cohort-greet   naive           b74fadad6801
```

All tags `0.1.0`, `0.1.0-bde8add`, `git-bde8add`, and `multi` share the same IMAGE ID `3cc85e19a9a0`.

### Docker build output

```
[+] Building 5.7s (17/17) FINISHED
 => [build 5/5] RUN --mount=type=cache,target=/root/.cache/pip     pip install --no-cache-dir -r requirements.t  1.1s
 => [runtime 2/5] COPY --from=build /opt/venv /opt/venv                                                          0.1s
 => [runtime 3/5] WORKDIR /app                                                                                   0.0s
 => [runtime 4/5] RUN useradd --uid 1000 app && chown -R app /app                                                0.1s
 => [runtime 5/5] COPY app.py .                                                                                  0.0s
```

### `docker container run --rm cohort-greet:secret cat /where-token-was-used`

```
d018
```

### "no leak" check

```
no leak
```

### hadolint output

```
No warnings (empty output)
```

### Cold build timings (Part 3.1)

```
Build #1: 5.598 total
Build #2: 4.242 total
```

### Pushed image URL (optional)

```
https://hub.docker.com/r/nicodhaas/cohort-greet
```

Tag pushed: `docker.io/nicodhaas/cohort-greet:0.1.0-bde8add`  
Digest: `sha256:3cc85e19a9a05aedf95fe6299c495a69d0a1a72af6a1db2e7d2d7e871c401f0c`

## 6. One trade-off I had to make

I chose `python:3.11.10-slim` over `python:3.11.10-alpine` for the runtime base. Alpine is smaller and uses musl libc, which can reduce image size further. However, slim is based on Debian (glibc), which has better compatibility with Python packages and fewer edge cases with binary wheels. Some packages (like certain cryptography libs) can have issues on Alpine due to musl compatibility. I traded ~10-20MB of size for better reliability and fewer "it works on slim but breaks on alpine" scenarios. If the security team mandated minimum size, I'd switch to alpine or distroless, but for general production use, slim is the safer default.

## 7. One thing I'm still unsure about

I'm initially unsure why the Mac zsh `time` command outputs `total` instead of `real`, but after asking, I learned this is macOS/zsh-specific behavior. The `grep real` filter removed all output because zsh doesn't output `real` — it outputs `total`. I fixed this by using `grep total` instead, and successfully captured both stop times.
