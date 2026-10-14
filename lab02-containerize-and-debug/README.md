# DevOps Lab Notes

## 1. Environment

| Property | Value |
|---|---|
| Distribution | Ubuntu 24.04.5 LTS (Noble Numbat) |
| Version ID | `24.04` |
| Codename | `noble` |
| Distribution ID | `ubuntu` (`debian`-like) |
| Kernel | `6.18.37-0-virt` |

> **Note:** The kernel `6.18.37-0-virt` belongs to my laptop's VM, not to the container. Containers share the host kernel.

---

## 2. Base Image Comparison

| Image | Size | Distro (`PRETTY_NAME`) | Default user |
|---|---:|---|---|
| `node:24` | 1.65 GB | Debian GNU/Linux 12 (bookworm) | `root` |
| `node:24-slim` | 351 MB | Debian GNU/Linux 12 (bookworm) | `root` |
| `node:24-alpine` | 238 MB | Alpine Linux v3.24 | `root` |

> `User=` is empty for all three images, so the default user is `root`.

**Pinned digest for `node:24-alpine`:**

```text
sha256:ebfe2f90462722a7a4de65e91990e97fe0d401c70e0e762c5b53302f905ec1c1
```

---

## 3. Copy-on-Write Layers

| Image | ID | Disk usage | Content size |
|---|---|---:|---:|
| `cow:bad` | `d67717f2fb4b` | 66 MB | 4.24 MB |
| `cow:good` | `209867305690` | 13.5 MB | 4.19 MB |

---

## 4. Build Cache Behaviour

| Build | Change | `RUN` step cached? | Build time |
|---|---|:---:|---:|
| a | Initial build | ❌ No | 0.1 s |
| b | Changed `app.txt` | ✅ Yes | 0.0 s |
| c | Changed `deps.txt` | ❌ No | 0.1 s |
| d | Copied the app before dependencies | ❌ No | 0.1 s |

---

## 5. Optimising `course-api`

### Image size

| Tag | Size |
|---|---:|
| `course-api:naive` | 1.74 GB |
| `course-api:step1` | 1.64 GB |
| `course-api:lab2` | **244 MB** |

The final image is roughly **15% of the original size**, i.e. about an **85% reduction** (244 MB / ~1,680 MB, rough estimate).

### Graceful shutdown

| Scenario | `docker stop` time | Exit code |
|---|---:|:---:|
| Before SIGTERM handler | 10.132 s | — |
| After adding SIGTERM handler | 0.107 s | `0` |

### Running container

| Container ID | Image | Command | Status | Ports | Name |
|---|---|---|---|---|---|
| `26a3e5b444d6` | `course-api:lab2` | `"docker-entrypoint.s…"` | Up (healthy) | `0.0.0.0:8080->5000/tcp`, `[::]:8080->5000/tcp` | `api` |

### Running as non-root

```text
uid=1000(node) gid=1000(node) groups=1000(node),1000(node)
```

---

## 6. Linux Fundamentals

### Reading a file

```console
$ cat /lab/newtools.txt
chef tools
ansible tools
docker tools
```

### Counting 404s in logs

```console
$ docker logs web 2>/dev/null | grep -c '" 404 '
20
```

### File permissions

```console
root@b98b94ace51f:/# echo secret > /lab/f && chmod 750 /lab/f && ls -l /lab/f
-rwxr-x--- 1 root root 7 Oct  7 16:13 /lab/f
root@b98b94ace51f:/# useradd -m student
root@b98b94ace51f:/# su - student -c 'cat /lab/f'
cat: /lab/f: Permission denied
```

`student` is neither the owner nor in the `root` group, so the "other" bits (`---`) apply and access is denied.

### Environment variables

```console
root@b98b94ace51f:/# APP_ENV=staging; sh -c 'echo "child sees: $APP_ENV"'
child sees:
root@b98b94ace51f:/# export APP_ENV=staging
root@b98b94ace51f:/# APP_ENV=staging; sh -c 'echo "child sees: $APP_ENV"'
child sees: staging
```

A plain shell variable stays in the current shell. Only after `export` is it passed to child processes.

### PID 1

- **Stop time:** 0.01 s
- **PID 1:** `bash`

---

## 7. Container Networking

**From inside a container on the same network:**

```console
root@b98b94ace51f:/# getent hosts web
172.18.0.2      web
root@b98b94ace51f:/# curl -sI http://web/ | head -1
HTTP/1.1 200 OK
```

**From the host:**

```console
% docker exec web netstat -ltn
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 127.0.0.11:44307        0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN
tcp        0      0 :::80                   :::*                    LISTEN

% curl -sI http://localhost:8081/ | head -1
HTTP/1.1 200 OK
```

**Why two different addresses reach the same nginx:** inside the network, Docker's embedded DNS resolves the name `web` to the container's IP. From the host, the published port `8081` is forwarded to port `80` in the container. Both paths end at the same nginx listening on `0.0.0.0:80`.

---

## 8. Debugging Broken Images

### `ghcr.io/mleitass/lab2-broken:2`

| | |
|---|---|
| **Symptom** | `exec /entrypoint.sh: no such file or directory` |
| **Cause** | `entrypoint.sh` has Windows line endings (`\r\n`, visible as `^M$`). The shebang becomes `#!/bin/sh\r`, an interpreter that doesn't exist. |
| **Fix** | Convert the file to Unix line endings (e.g. `dos2unix entrypoint.sh` or `sed -i 's/\r$//' entrypoint.sh`). |

### `ghcr.io/mleitass/lab2-broken:3`

| | |
|---|---|
| **Symptom** | `curl localhost:8082` returns an empty response from the server. |
| **Cause** | `server.js` binds only to `127.0.0.1`, so it is unreachable from outside the container. |
| **Fix** | Remove the host argument (or use `0.0.0.0`) so the server listens on all interfaces. |

```js
// Before
const server = app.listen(PORT, "127.0.0.1", () => { ... });

// After
const server = app.listen(PORT, () => { ... });
```

### `ghcr.io/mleitass/lab2-broken:4`

| | |
|---|---|
| **Symptom** | `Database not reachable (connect ECONNREFUSED … 127.0.0.1:5432)` |
| **Cause** | 1) No Postgres server is running. 2) Even if one were, it would be in another container, where `localhost` points to the app container itself. |
| **Fix** | Run Postgres in its own container on the same network and set `DB_HOST` to its service/container name (e.g. `DB_HOST=db`). |

### `ghcr.io/mleitass/lab2-broken:5`

| | |
|---|---|
| **Symptom** | `EACCES: permission denied, open '/app/data/todos.log'` |
| **Cause** | `/app/data` is owned by `root` (`drwxr-xr-x 0 0`), but the app runs as the non-root `node` user. |
| **Fix** | Change ownership to `node` in the Dockerfile, e.g. `RUN chown node:node /app/data`. |

### `ghcr.io/mleitass/lab2-broken:6`

| | |
|---|---|
| **Symptom** | `docker stop` takes 10 s and the container exits with code `137`. |
| **Cause** | Shell-form `CMD` runs `/bin/sh` as PID 1, which does not forward SIGTERM to `node server.js`, so Docker falls back to SIGKILL. |
| **Fix** | Use exec form so Node is PID 1: `CMD ["node", "server.js"]`. |

### `ghcr.io/mleitass/lab2-broken:7`

| | |
|---|---|
| **Symptom** | Container is permanently `unhealthy`, but the app works. |
| **Cause** | The `HEALTHCHECK` uses `curl`, which isn't installed in the Alpine image. |
| **Fix** | Remove the curl-based healthcheck, or replace it with `wget` (available in Alpine) or install `curl`. |
