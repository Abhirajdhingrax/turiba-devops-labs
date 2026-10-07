# Lab 2 · Containerize it, then debug it

- **Image:** `ghcr.io/abhirajdhingrax/course-api:lab2` (public, linux/amd64 + linux/arm64)
- **Image digest:** `sha256:3d073c53d476af90fe4c0c2d4e97f3ffa4ab144b67074c84b705bdbca66ef853`
- **Base image digest:** `node:24-alpine@sha256:ebfe2f90462722a7a4de65e91990e97fe0d401c70e0e762c5b53302f905ec1c1`

## Part 1 · Images and layers

| Image | Size | Distro | Default user |
|-------|-----:|--------|--------------|
| node:24 | 1.66 GB | Debian GNU/Linux 12 (bookworm) | root (uid=0) |
| node:24-slim | 332 MB | Debian GNU/Linux 12 (bookworm) | root (uid=0) |
| node:24-alpine | 242 MB | Alpine Linux v3.24 | root (uid=0) |

cow:bad = 65.4 MB · cow:good = 12.9 MB

| Build | RUN step CACHED? | Build time |
|-------|------------------|-----------:|
| b · app.txt changed | yes | 1.2 s |
| c · deps.txt changed | no | 16.7 s |
| d · app.txt changed, wrong order | no | 16.5 s |

## Part 2 · The course API image

| Step | Image | Size |
|------|-------|-----:|
| Naive | course-api:naive | 1.75 GB |
| + .dockerignore, npm ci --omit=dev, exec form | course-api:step1 | 1.65 GB |
| Multi-stage, node:24-alpine, non-root | course-api:lab2 | 248 MB |
| Reduction against naive | | 86 % |

docker stop before the SIGTERM handler: 3.4 s, exit code 137 · after: 2.3 s, exit code 0

Output of `docker run --rm course-api:lab2 id`:

~~~
uid=1000(node) gid=1000(node) groups=1000(node),1000(node)
~~~

Output of `docker ps` showing (healthy):

~~~
CONTAINER ID   IMAGE             COMMAND                  CREATED          STATUS                    PORTS                                         NAMES
4414aa2fc889   course-api:lab2   "docker-entrypoint.s…"   37 seconds ago   Up 36 seconds (healthy)   0.0.0.0:8080->5000/tcp, [::]:8080->5000/tcp   api
~~~

## Part 3 · Linux drills

**3.1**

~~~
PRETTY_NAME="Ubuntu 24.04.5 LTS"
6.18.40.1-microsoft-standard-WSL2
~~~

`/etc/os-release` comes from the Ubuntu image's files. `uname -r` shows the kernel of my laptop's WSL 2 VM, which all containers share.

**3.2**

~~~
chef tools
ansible tools
docker tools
~~~

**3.3** Number of 404 lines in the nginx log: **20**

**3.4**

~~~
-rwxr-x--- 1 root root 7 Oct  7 17:01 /lab/f
cat: /lab/f: Permission denied
~~~

Mode 750 gives the owner (root) rwx, the group r-x and others nothing. `student` is neither the owner nor in the group, so reading is denied. Root bypasses file permissions.

**3.5**

~~~
child sees:
child sees: staging
~~~

Without `export` the variable exists only in the current shell. After `export` it is passed to child processes.

**3.6** PID 1 is `bash`. `docker stop lab` took 3.4 s: bash as PID 1 does not react to SIGTERM, so Docker waits for the grace period and then sends SIGKILL.

**3.7**

~~~
172.18.0.2      web
HTTP/1.1 200 OK
tcp   0   0 0.0.0.0:80   0.0.0.0:*   LISTEN
tcp   0   0 :::80        :::*        LISTEN
HTTP/1.1 200 OK
~~~

Inside `labnet`, Docker's DNS resolves `web` to the container's own IP (172.18.0.2) on port 80. From the host, the port mapping `8081:80` forwards to the same nginx, which listens on all addresses (0.0.0.0:80).

## Part 4 · Broken containers

### lab2-broken:2
- Symptom: exits at once, `Exited (255)`, `exec /entrypoint.sh: no such file or directory` although the file exists
- Cause: `cat -A` shows `^M$`: the script has Windows (CRLF) line endings, so the shebang is `#!/bin/sh\r` and the kernel looks for an interpreter `/bin/sh\r` that does not exist
- Fix: convert the script to LF (`sed -i 's/\r$//' entrypoint.sh` or `dos2unix`), and add `*.sh text eol=lf` to `.gitattributes`

### lab2-broken:3
- Symptom: runs, but `curl localhost:8082` → `Connection reset by peer`
- Cause: `netstat -ltn` shows the app listens on `127.0.0.1:5000` only. Docker's port mapping delivers traffic to the container's network interface, not its loopback
- Fix: listen on all interfaces: `app.listen(PORT, '0.0.0.0')`

### lab2-broken:4
- Symptom: `Database not reachable (connect ECONNREFUSED 127.0.0.1:5432)`
- Cause: `DB_HOST` is not in the image config (`docker inspect` → Env); it comes from a `.env` file copied into `/app` (`DB_HOST=localhost`). Inside a container, localhost is the container itself. The password is also leaked in the image
- Fix: add `.env` to `.dockerignore` and pass configuration at run time, e.g. `docker run -e DB_HOST=db …`

### lab2-broken:5
- Symptom: exits at once, `EACCES: permission denied, open '/app/data/todos.log'`
- Cause: the image runs as `node` (uid 1000), but `/app/data` is owned by root (`drwxr-xr-x 0 0`), so node cannot write there
- Fix: before `USER node`: `RUN mkdir -p /app/data && chown node:node /app/data` (or `COPY --chown=node:node`)

### lab2-broken:6
- Symptom: `docker stop` waits the full grace period (3.3 s) and ends with exit code 137
- Cause: `CMD ["/bin/sh","-c","node server.js"]` (shell form) and the app has no SIGTERM handler. As PID 1 it ignores SIGTERM, so Docker has to SIGKILL it
- Fix: exec form `CMD ["node", "server.js"]` plus a SIGTERM handler (as in step 2.4), or run with `--init`

### lab2-broken:7
- Symptom: `Up (unhealthy)` forever, but the app answers fine
- Cause: `HEALTHCHECK` runs `curl -fs http://127.0.0.1:5000/`, but the Alpine image has no curl: health log shows `/bin/sh: curl: not found`, FailingStreak 5
- Fix: `HEALTHCHECK CMD wget -qO- http://127.0.0.1:5000/healthz || exit 1` (wget is in BusyBox), or install curl

## Answers

1. **`cow:bad` contains no `/big.file`, yet it is 50 MB bigger than `cow:good`. Why?**
   Every `RUN` creates a new read-only layer. The `dd` layer stores the 50 MB file; the later `rm` only adds a small "whiteout" marker on top that hides it. The file is still inside the lower layer, which is shipped with the image. In `cow:good` the file is created and deleted in the same `RUN`, so it never ends up in any layer.

2. **Why does the order of `COPY` and `RUN` lines decide how long a rebuild takes?**
   Docker reuses a cached layer only if that step and all steps before it are unchanged. When one step changes, it and every step after it run again. Copying the dependency files and running `npm ci` before copying the source code means a code change only rebuilds the last cheap steps (build b: 1.2 s); copying the code first forces the slow install every time (build d: 16.5 s).

3. **Why did `docker stop` take 10 seconds before you added the SIGTERM handler?**
   `docker stop` sends SIGTERM to PID 1. Node ran as PID 1 without a handler, and the kernel does not apply the default "terminate" action to PID 1, so the signal was ignored. Docker waited for the grace period (3.4 s on my setup) and then sent SIGKILL, which gives exit code 137. With the handler, node closes the server and exits with 0.

4. **Name three things the naive image contained that `course-api:lab2` does not.**
   The full Debian base with compilers, Python and git (about 1.7 GB); the devDependencies (`jest`, `nodemon`) from `npm install`; and the `.env` file with the password plus the `Dockerfile` in `/app`. It also ran as root and started through `npm`.
