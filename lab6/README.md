# Lab 6 · Test what you ship, ship a release

GitHub user: ElvenHope · Reference pipeline used (Part 0 fallback): no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores), and <you> in the first line and in section 3. -->

## 1 · Smoke test

- The names of my smoke-test steps in the `image` job: __Build for the smoke test (amd64, kept on the runner) and Smoke test the image__
- What my smoke test asks, and what makes it fail: ____
- 🔮 Part 1, before my first run: will my code pass the smoke test? I predicted __Yes__ because __it passes in the runner so why not__. What happened: ____
- 🔮 Part 2, before the checks finished: which checks will go red after `break.sh`? I predicted __Image?__ because __It's what we are trying to break__

## 2 · What broke and how I found it

- The check and the step that went red: __Image, smoke step__
- The log line that showed the cause (copied from the log):
Error: Cannot find module './banner'
Require stack:
- /app/server.js
```text
____
```

- The cause, in my own words: __docker ignore was ignoring itself and the banner.js__
- Why `lint`, `unit` and `integration` stayed green: __cause they didn't care about banner.js__
- My fix, and the commit that made it: __I removed .dockerignore and banner.js from the .dockerignore 80c16dcbda5326b66ca2334dc9b4af96c5abaf70__

## 3 · Release

- Release page: https://github.com/elvenhope/turiba-devops-labs/releases/tag/v0.2.0
- Image: `ghcr.io/elvenhope/turiba-devops-labs/api:0.2.0`, platforms (from `docker buildx imagetools inspect`): __linux/arm64 linux/amd64__
- One thing in Part 3 that didn't work the first time, and how I fixed it: __It worked, I checked it first manually and by claude later__
