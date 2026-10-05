# Lab 1 · Ship it

- **Image:** `ghcr.io/elvenhope/lab1.0-web:1.0`
- **Digest:** `sha256:68aff2ff44f2e7ef5a3eabcb14a83f9662c8b0a5217ad2fd9369e3aa1b730e25`
- **Platforms:** linux/amd64, linux/arm64
- **Partner's image I ran:** `ghcr.io/johnthunderr/lab1-web:1.0` (digest matched: yes)

![My partner's image running on my laptop](partner-run.png)

## Answers

1. Where does the kernel used by your containers come from on your laptop? Paste the `docker info` / `uname -r` output that proves it.

Alpine Linux v3.24

Server:
 Containers: 26
  Running: 11
  Paused: 0
  Stopped: 15
 Images: 10
 Server Version: 29.5.3
 Storage Driver: overlayfs
  driver-type: io.containerd.snapshotter.v1
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Cgroup Version: 2
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local splunk syslog
 CDI spec directories:
  /etc/cdi
  /var/run/cdi
 Swarm: inactive
 Runtimes: io.containerd.runc.v2 runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: fff62f14765df376e5fc36f5a8f8e795b5670f61
 runc version: bb14dabeb7185bb72c8c86735d090dcb20f36587
 init version: 
 Security Options:
  seccomp
   Profile: /etc/rancher-desktop/seccomp.json
  cgroupns
 Kernel Version: 6.18.37-0-virt
 Operating System: Alpine Linux v3.24
 OSType: linux
 Architecture: aarch64
 CPUs: 2
 Total Memory: 3.824GiB
 Name: lima-rancher-desktop
 ID: a4020047-f3b4-44b3-b124-14bb02c2fbe4
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Experimental: false
 Insecure Registries:
  ::1/128
  127.0.0.0/8
 Live Restore Enabled: false
 Firewall Backend: iptables
  EnableUserlandProxy: true
  UserlandProxyPath: /usr/bin/docker-proxy


2. What is the difference between `lab1-web:1.0` and `mypage`?

none really, mypage is just the name we gave to the image we downloaded and ran, the imageitself is called lab1-web the version tag being 1.0

3. In Part 2 your edit to `index.html` survived `docker stop` but not `docker rm`. Why?

because docker stop doesn't delete the writes, it only stops the process. docker rm actually removes the write layer.

4. Your page is about 1 KB, the image is tens of MB. What do you think the rest is?
all the nginx, and alpine and other dependacy layers that are in the nginx:stable-alpine image that we are putting our code on top of.
