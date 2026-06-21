# Assignment 02 — Nicodemus Haastrup

**GitHub username:** nicohaastrup  
**Date completed:** 2026-06-07  
**Language chosen:** Python

## 1. The image I built

- Final image ID: `sha256:393ac0c394ace903002cbbd497fab2fc0bafe12d4063ca844a2a23aca635259a`
- Image size: `212MB` (disk usage), `45.7MB` (content size)  
- Number of layers: `5` (from `docker image history` — 6 rows including the header, so 5 image layers)

### Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY app.py .

ENV PORT=8000
EXPOSE 8000

CMD ["python", "app.py"]


.dockerignore

.git
.gitignore
node_modules
__pycache__
*.pyc
*.log
README.md


2. Answers to the 8 questions

Q1 — what .dockerignore affects:
.dockerignore affects what files are included in the build context Docker sends to the build engine when running docker build. It excludes matching files from the context before any COPY or ADD happens. Excluding .git matters even though I didn’t COPY .git explicitly because .git is still part of the build context if present; it increases the context size and can slow builds or accidentally be copied if I later change the Dockerfile to COPY . .. Keeping it out keeps the build context lean and avoids leaking repository history.

Q2 — what is the image ID a hash of:
The image ID is a SHA256 hash of the image configuration object, which includes metadata like Cmd, Env, ExposedPorts, WorkingDir, and the digests of all layers. It is effectively a content hash of the image config + layer references, not just a random identifier.

Q3 — largest layer and why:
From docker image history cohort-greet:0.1.0, the largest layer is:

<missing>  11 days ago  # debian.sh --arch 'arm64' out/ 'trixie' '@1…  109MB

This layer comes from the base image python:3.11-slim, which is built on a minimal Debian (“trixie”) image. It’s the largest because it contains the entire OS base (kernel headers, basic utilities, etc.) needed to run Python, while my app.py is only ~12.3KB.

Q4 — --memory 64m shows up as what value:
docker container inspect -f '{{.HostConfig.Memory}}' greet returned 67108864.
This is 64 MiB in bytes:
$64 \times 1{,}048{,}576 = 67{,}108{,}864$ bytes.
Docker’s inspect returns memory in bytes, not megabytes, so the unit is different.

Q5 — PID of my app inside the container:
From cat /proc/1/cmdline | tr '\0' ' ', I saw:

python app.py

So my app process is PID 1 inside the container. This tells me that the container was started directly with my app as the init process (the main process Docker runs), with no extra shell or wrapper process.

Q6 — stop vs kill, and which for a database:

stop: Sends SIGTERM to the main process, allowing graceful shutdown (close connections, flush data), then waits for a grace period and sends SIGKILL if needed.
kill: Immediately sends SIGKILL (by default), terminating the process instantly with no graceful shutdown.

For a database container, I would use stop, because databases need to flush buffers, commit transactions, and close connections cleanly. An immediate kill can cause data corruption or incomplete writes.

Q7 — what same-IMAGE-ID-across-tags proves:
From docker image ls cohort-greet:

IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
cohort-greet:0.1     393ac0c394ac       212MB         45.7MB   U
cohort-greet:0.1.0   393ac0c394ac       212MB         45.7MB   U
cohort-greet:latest  393ac0c394ac       212MB         45.7MB   U

All three tags (cohort-greet:0.1, cohort-greet:0.1.0, cohort-greet:latest) have the same IMAGE ID (393ac0c394ac). This proves that tags are just mutable human-readable pointers to the same image, not separate copies. Docker stores one image (with one ID), and multiple tags can point to it. Tags are metadata, not distinct images.

Q8 — tag vs digest mutability:
No. If Docker Hub re-tags alpine:3.19 tomorrow to point at a different image, pulling alpine:3.19 later will give you the new image, because tags are mutable pointers.

Pulling by digest (alpine@sha256:6baf43584bcb78f2e5847d1de515f23499913ac9f12bdf834811a3145eb11ca1) will always give you the exact same image with that specific content hash, because digests are immutable content addresses.

3. Evidence

docker image history cohort-greet:0.1.0

docker image history cohort-greet:0.1.0

IMAGE          CREATED          CREATED BY                              SIZE      COMMENT
b2b1c2d83430   20 minutes ago   CMD ["python" "app.py"]                 0B      buildkit.dockerfile.v0
<missing>      20 minutes ago   EXPOSE [8000/tcp]                       0B      buildkit.dockerfile.v0
<missing>      20 minutes ago   ENV PORT=8000                           0B      buildkit.dockerfile.v0
<missing>      20 minutes ago   COPY app.py . # buildkit                12.3kB  buildkit.dockerfile.v0
<missing>      20 minutes ago   WORKDIR /app                            8.19kB  buildkit.dockerfile.v0
<missing>      10 days ago      CMD ["python3"]                         0B      buildkit.dockerfile.v0
<missing>      10 days ago      RUN /bin/sh -c set -eux;  for src in idle3 p…  16.4kB  buildkit.dockerfile.v0
<missing>      10 days ago      RUN /bin/sh -c set -eux;   savedAptMark="$(a…  51.7MB  buildkit.dockerfile.v0
<missing>      10 days ago      ENV PYTHON_SHA256=272179ddd9a2e41a0fc8e42e33…  0B      buildkit.dockerfile.v0
<missing>      10 days ago      ENV PYTHON_VERSION=3.11.15              0B      buildkit.dockerfile.v0
<missing>      10 days ago      ENV GPG_KEY=A035C8C19219BA821ECEA86B64E628F8…  0B      buildkit.dockerfile.v0
<missing>      10 days ago      RUN /bin/sh -c set -eux;  apt-get update;  a…  4.99MB  buildkit.dockerfile.v0
<missing>      10 days ago      ENV LANG=C.UTF-8                        0B      buildkit.dockerfile.v0
<missing>      10 days ago      ENV PATH=/usr/local/bin:/usr/local/sbin:/usr…  0B      buildkit.dockerfile.v0
<missing>      11 days ago      # debian.sh --arch 'arm64' out/ 'trixie' '@1…  109MB   debuerreotype 0.17


Detached run with all flags (Part 2.2)

docker container run -d \
  --name greet \
  -p 8080:8000 \
  -e STUDENT_NAME="Nicodemus Haastrup" \
  -e GREETING="hi" \
  --restart unless-stopped \
  --memory 64m \
  --cpus 0.25 \
  cohort-greet:0.1.0

d9966b1647f6f92d250037a7bc76200d23b4f6f65baa36aeebe16c91ccb5a67f


docker container logs greet after 2 curl requests

curl http://localhost:8080
curl http://localhost:8080
docker container logs greet

listening on :8000
[req] 192.168.65.1 "GET / HTTP/1.1" 200 -

(One request log shown; the second curl produced the same response but the log output shown in my terminal was the same line.)

Actual response:

curl http://localhost:8080

hi, Nicodemus Haastrup — 2026-06-21T18:42:41.855673Z


docker container stats --no-stream greet

docker container stats --no-stream greet

CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT   MEM %     NET I/O         BLOCK I/O     PIDS
0e7788a3faac   greet     0.02%     13.48MiB / 64MiB    21.06%    1.67kB / 722B   0B / 1.95MB   1


docker container inspect for restart policy and memory

docker container inspect -f '{{.HostConfig.RestartPolicy.Name}} {{.HostConfig.Memory}}' greet

unless-stopped 67108864

(Separate commands earlier confirmed:

docker container inspect -f '{{.HostConfig.RestartPolicy.Name}}' greet
# => unless-stopped

docker container inspect -f '{{.HostConfig.Memory}}' greet
# => 67108864

)

docker image ls cohort-greet (three tags)

docker image ls cohort-greet

IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
cohort-greet:0.1     393ac0c394ac       212MB         45.7MB   U
cohort-greet:0.1.0   393ac0c394ac       212MB         45.7MB   U
cohort-greet:latest  393ac0c394ac       212MB         45.7MB   U


Pushed image URL (optional)

I pushed the image to Docker Hub under nicodhaas:

https://hub.docker.com/r/nicodhaas/cohort-greet

Specific tag pushed:

docker.io/nicodhaas/cohort-greet:0.1.0

Digest from push:

sha256:393ac0c394ace903002cbbd497fab2fc0bafe12d4063ca844a2a23aca635259a


4. One thing that surprised me

I was surprised that the base image python:3.11-slim alone makes my image around 212MB disk usage (45.7MB content size), while my app.py is only 744 bytes. Most of the image size comes from the Python runtime and minimal OS, not my code. Also, the image ID shown by docker image ls is a short prefix (393ac0c394ac) of the full SHA256 digest.

5. One thing I'm still unsure about

I’m still unsure about precisely how Docker enforces --memory limits on Linux vs. Docker Desktop, and what exactly happens when a container hits the limit (does it crash immediately, get throttled, or fail specific operations first?).




