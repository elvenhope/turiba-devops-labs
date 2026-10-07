| Image | Size | Distro (`PRETTY_NAME`) | Default user |
|---|---:|---|---|
| `node:24` | 1.65 GB | Debian GNU/Linux 12 (bookworm) | `root` |
| `node:24-slim` | 351 MB | Debian GNU/Linux 12 (bookworm) | `root` |
| `node:24-alpine` | 238 MB | Alpine Linux v3.24 | `root` |

Note: `User=` is empty for these images, so the default user is `root`.

| IMAGE | ID | DISK USAGE | CONTENT SIZE | EXTRA |
|---|---|---:|---:|---:|
| `cow:bad` | `d67717f2fb4b` | 66MB | 4.24MB |  |
| `cow:good` | `209867305690` | 13.5MB | 4.19MB |  |


| Build | Change | `RUN` step cached? | Build time |
|---|---|---|---:|
| a | Initial | No | 0.1s |
| b | Changed `app.txt` | Yes | 0.0s |
| c | Changed `deps.txt` | No | 0.1s |
| d | Copied the app before dependencies | No | 0.1s |

Digest for node:24-alpine: sha256:ebfe2f90462722a7a4de65e91990e97fe0d401c70e0e762c5b53302f905ec1c1

Size of course-api:naive: 1.74GB
Size of course-api:step1: 1.64GB
docker stop took 10.132 seconds
docker new stop after adding sigterm handler: 0.107 seconds and code 0

Size of course-api:lab2: 244MB, about 15% reduction. (didn't do math in my head just rough 244 / 1680)

| CONTAINER ID | IMAGE | COMMAND | CREATED | STATUS | PORTS | NAMES |
|---|---|---|---|---|---|---|
| `26a3e5b444d6` | `course-api:lab2` | `"docker-entrypoint.s…"` | About a minute ago | Up About a minute (healthy) | `0.0.0.0:8080->5000/tcp, [::]:8080->5000/tcp` | `api` |

```text
uid=1000(node) gid=1000(node) groups=1000(node),1000(node)
```

### Environment details

| Property | Value |
|---|---|
| Distribution | Ubuntu 24.04.5 LTS (Noble Numbat) |
| Version ID | `24.04` |
| Codename | `noble` |
| Distribution ID | `ubuntu` (`debian`-like) |
| Ubuntu codename | `noble` |
| Kernel | `6.18.37-0-virt` |

> The kernel version `6.18.37-0-virt` belongs to my laptop's VM.


Cat output of /lab/newtools.txt
chef tools
ansible tools
docker tools

output of "docker logs web 2>/dev/null | grep -c '" 404 '" -> 20

output of trying to add student:
root@b98b94ace51f:/# echo secret > /lab/f && chmod 750 /lab/f && ls -l /lab/f
-rwxr-x--- 1 root root 7 Oct  7 16:13 /lab/f
root@b98b94ace51f:/# useradd -m student
root@b98b94ace51f:/# su - student -c 'cat /lab/f'
cat: /lab/f: Permission denied
root@b98b94ace51f:/# 

output of setting an app_env:
root@b98b94ace51f:/# APP_ENV=staging; sh -c 'echo "child sees: $APP_ENV"'
child sees: 
root@b98b94ace51f:/# export APP_ENV=staging
root@b98b94ace51f:/# APP_ENV=staging; sh -c 'echo "child sees: $APP_ENV"'
child sees: staging
root@b98b94ace51f:/# 

Stoptime for lab -> 0.01s
PID 1 is bash.

container output:
root@b98b94ace51f:/# getent hosts web
172.18.0.2      web
root@b98b94ace51f:/# curl -sI http://web/ | head -1
HTTP/1.1 200 OK
root@b98b94ace51f:/# 

host output:
davit@Davits-MacBook-Air-M4 turiba-devops-labs % docker exec web netstat -ltn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 127.0.0.11:44307        0.0.0.0:*               LISTEN      
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      
tcp        0      0 :::80                   :::*                    LISTEN      
davit@Davits-MacBook-Air-M4 turiba-devops-labs % curl -sI http://localhost:8081/ | head -1
HTTP/1.1 200 OK
davit@Davits-MacBook-Air-M4 turiba-devops-labs % 

two different addresses reach the same nginx because they are both in the same network and the docker is translating the "web" string into a proper address.



Debugging Section

ghcr.io/mleitass/lab2-broken:2

Symptom -> exec /entrypoint.sh: no such file or directory

Cause -> bash is not loading properly because hidden symbols in the entrypoint.sh file 

Fix -> change the entrypoint.sh to be properly written without weird symbols like ^M$

ghcr.io/mleitass/lab2-broken:3

Symptom -> curl localhost:8082 returns empty response from the server

Cause -> in the server.js, it only accepts requests from localhost const server = app.listen(PORT, "127.0.0.1", () => {
  console.log(`API listening on port ${PORT}`);
});

Fix -> remove 127.0.0.1 from the code indicated.


ghcr.io/mleitass/lab2-broken:4

Symptom -> Database not reachable (connect ECONNREFUSED … 127.0.0.1:5432)

Cause -> there's no posgress server running anywhere for 1, and two if it was it was it's prbably in a nother container and localhost wouldnt work

Fix -> start another server and give env DB_HOST as db or something the name of the service

ghcr.io/mleitass/lab2-broken:5

Symptom -> EACCES: permission denied, open '/app/data/todos.log'

Cause -> the directory is owned by root (drwxr-xr-x    2 0        0             4096 Oct  5 18:54 /app/data), and we are using the filesystem as a node user (non-root)

Fix -> make the dir chmod to node

ghcr.io/mleitass/lab2-broken:6

Symptom -> docker stop takes 10s and end 137s

Cause -> /bin/sh is calling node server.js

Fix -> change the dockerfile to not do the whole first do /bin/sh, and then node server.js, just do node server.js directly

ghcr.io/mleitass/lab2-broken:7

Symptom -> its unhealthy perma but app works 

Cause ->. dockerfile has a run command in it that calls curl but alpine version doesn't have curl

Fix - remove the line from dockerfile that does curl