# Assignment 01 — CLI Essentials & Docker Commands

**Name:** Nicodemus Haastrup  
**GitHub:** `nicohaastrup`  
**Date:** 2026-05-21

---

## Part 1 Reflection

I've used most of these CLI commands before, but some of them like `wc -l`, pipes, and redirection are not things I use every day. This assignment was a good refresh on the basics I sometimes move through too quickly when working in the terminal. Using `ls -la | grep ".txt"` and `wc -l commands.txt` helped bring those small but useful commands back into regular practice. I also noticed a platform difference on macOS when `ls --help` did not work the same way I expected, so using `man ls` was a useful reminder to adapt to the environment.

---

## Part 2 — Questions

### Q1: After pulling `alpine:3.19` and running `docker container run alpine echo hi` — how many images and how many containers?

**Answer:**

```bash
$ docker image pull alpine:3.19
3.19: Pulling from library/alpine
...
Status: Downloaded newer image for alpine:3.19
docker.io/library/alpine:3.19

$ docker image ls
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
alpine:3.19          6baf43584bcb   11.9MB       3.45MB
alpine:latest        5b10f432ef3d   13.6MB       4.29MB
hello-world:latest   96498ffd522e   22.6kB       10.3kB
nginx:latest         5aca99593157   259MB        64.3MB

$ docker container run alpine echo hi
hi

$ docker container ls -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                      PORTS     NAMES
70893c005053   alpine        "echo hi"               47 seconds ago   Exited (0) 46 seconds ago             competent_ride
d02d466cdb27   hello-world   "/hello"                20 minutes ago   Exited (0) 20 minutes ago             unruffled_shamir
e01aa9ccc665   nginx         "/docker-entrypoint.…"  6 days ago       Exited (0) 6 days ago                 gifted_bardeen
713426830ae1   alpine        "/bin/sh"               6 days ago       Exited (0) 6 days ago                 mystifying_liskov
```

By the time I ran this section, I had **4 images** and **4 containers** already present locally. The important point is that `docker container run alpine echo hi` created a **new container** from the alpine image, and that container exited immediately after the command finished. An image is the template, while a container is an instance created from that image.

### Q2: What's the difference between `docker run -it alpine sh` and `docker exec -it <name> sh`?

**Answer:**

- `docker run -it alpine sh` creates a new container from the alpine image and starts a shell inside it.
- `docker exec -it <name> sh` opens a shell inside an existing running container.

Use `docker run` when you want a fresh container to work in. Use `docker exec` when you want to inspect or work inside a container that is already running.

### Q3: What flag mounts a file from the host into a container as read-only?

```bash
$ docker container run --help | grep -i mount
      --mount mount                    Attach a filesystem mount to the container
  -v, --volume list                    Bind mount a volume
```

The flag can be used either as `-v` / `--volume` with `:ro`, or as `--mount` with `readonly`.

```bash
docker run -v /host/file.txt:/container/file.txt:ro nginx
```

```bash
docker run --mount type=bind,source=/host/file.txt,target=/container/file.txt,readonly nginx
```

The `:ro` or `readonly` option makes the mounted file read-only inside the container.

---

## Part 3 — Evidence

### Step 3 — `docker container ls` after starting `practice-web`

```bash
$ docker container run -d --name practice-web -p 8081:80 -e STUDENT_NAME="Nicodemus Haastrup" nginx:1.25
bbb9563fab3cba5beb9a07cad6804de8ed30c46fe7746c7bcfe4f5d4b97c1fd4

$ docker container ls
CONTAINER ID   IMAGE        COMMAND                  CREATED          STATUS          PORTS                                     NAMES
bbb9563fab3c   nginx:1.25   "/docker-entrypoint.…"   33 seconds ago   Up 32 seconds   0.0.0.0:8081->80/tcp, [::]:8081->80/tcp   practice-web
```

### Step 5 — Log streaming

```bash
$ docker container logs -f web
...
192.168.65.1 - - [13/Jun/2026:18:25:18 +0000] "GET / HTTP/1.1" 200 615 "-" "Mozilla/5.0 ..."
192.168.65.1 - - [13/Jun/2026:18:27:14 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 ..."
192.168.65.1 - - [13/Jun/2026:18:27:16 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 ..."
^C
```

The nginx logs showed my browser requests after refreshing the page, which confirmed the container was serving traffic correctly.

### Step 6 — Shell inside container, verify `STUDENT_NAME`

```bash
$ docker container exec -it practice-web sh

# env | grep STUDENT_NAME
STUDENT_NAME=Nicodemus Haastrup

# exit
```

### Step 7 & 8 — Custom index.html, verified in browser

```bash
$ docker container exec -it practice-web sh

# echo "Hello from Nicodemus Haastrup" > /usr/share/nginx/html/index.html

# exit
```

After refreshing `http://localhost:8081`, the browser displayed:

```text
Hello from Nicodemus Haastrup
```

### Step 9 — Container IP address via `docker inspect`

```bash
$ docker container inspect -f '{{.NetworkSettings.IPAddress}}' practice-web
template parsing error: template: :1:18: executing "" at <.NetworkSettings.IPAddress>: map has no entry for key "IPAddress"

$ docker container inspect -f '{{.NetworkSettings.Networks.bridge.IPAddress}}' practice-web
172.17.0.2
```

The first inspect format did not work in my Docker version, so I used the bridge network path instead and got the container IP successfully.

### Step 10 — Stop and remove in one command

```bash
$ docker container stop web
web

$ docker container ls -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                          PORTS     NAMES
6c0934ea7992   nginx:1.25    "/docker-entrypoint.…"   7 minutes ago    Exited (0) Less than a second ago         web
...

$ docker container rm web
web
```

### Step 11 & 12 — Image cleanup

```bash
$ docker image ls
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
alpine:3.19          6baf43584bcb   11.9MB       3.45MB
alpine:latest        5b10f432ef3d   13.6MB       4.29MB
hello-world:latest   96498ffd522e   22.6kB       10.3kB
nginx:1.25           a484819eb602   278MB        70.6MB
nginx:latest         5aca99593157   259MB        64.3MB

$ docker image rm nginx:1.25
Untagged: nginx:1.25
Deleted: sha256:a484819eb60211f5299034ac80f6a681b06f89e65866ce91f356ed7c72af059c
```

I was able to remove the `nginx:1.25` image after removing the container that used it.

---

## One thing that surprised me

I was surprised that changes inside a container disappear when the container is removed and recreated. After writing a custom `index.html` inside the running nginx container, the change worked immediately in the browser, but it did not persist after the container was removed. That made it clearer to me why volumes are important when data needs to remain available beyond the life of a single container.

---

## Validation checklist

- [x] Open a terminal, navigate to a directory, and create/read/append to a file
- [x] Use a pipe to filter the output of one command with another
- [x] Set and read an environment variable in your shell
- [x] Pull an image and start a detached container with a name and a published port
- [x] List running and stopped containers
- [x] Stream a container's logs in real time
- [x] Open an interactive shell inside a running container
- [x] Print one specific field from `docker inspect` using `-f`
- [x] Stop and remove a container
- [x] Reclaim disk space with Docker cleanup commands
