# Lab 5 · Your first CI pipeline

GitHub user: elvenhope · Reference files used (Lab 2 fallback): no

![ci](https://github.com/<you>/turiba-devops-labs/actions/workflows/ci.yml/badge.svg?branch=main)

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores), and your user name placeholder in the first line and in the badge URL: write your GitHub user name exactly as GitHub shows it. -->

## 1 · Graph predictions

Part 1, step 1.2: fill in this section **before** you run `lab05 graph`. A wrong prediction that you explain earns full marks.

My drawing of the `lab05 graph` jobs, before the first run (boxes and arrows in text are fine):

```text
I am not sure how to draw but like 

first is graph build, then it gets linted with graph lint, and then graph test depends on graph building being successful. graph package needs both build and test to be successful. graph report depends on every step. i am not sure why graph package needs build to succeed. since if it only needs graph test, and graph test depends on graph build, if graph build fails so does graph test and so does graph package. so i don't see why package needs to depend both on build and test seperately. 
```

| # | Question | I predicted, and why | What the run showed |
|---|----------|----------------------|---------------------|
| P1 | Which jobs start together? | __Lint and Build I guess__ | __Lint and Build__ |
| P2 | Will `graph-package` find `build.txt`? | __i guess only if test succeeded?__ | __It didn't, because they don't share files build.txt: not here: jobs don't share files__ |
| P3 | With `break = build`: which jobs run, which fail, which are skipped? | __lint runs, test fails and package fails, and report__ | __lint, runs, build fails, graph report runs. test and package are skipped. Error: Process completed with exit code 1.__ |

## 2 · Pipeline

- B5, before I filled it in: `localhost` or `db`? I predicted __localhost__ because __because each runner step is isolated from others. so that leads me to believe that they are happening in their own containers__
- My first red run in Part 2 (the job, the first red step, the error line copied from the log): __everything was green from the get go__
- What I changed to make it green: __nothing__
- My lint findings, and how I fixed them (fixed, not silenced): ____
- Part 2 pull request (URL): __https://github.com/elvenhope/turiba-devops-labs/pull/7__
- Part 3: will the PR run push an image? I predicted __I think yes__. What happened: ____
- My image on GHCR, from the `main` run: `ghcr.io/____/turiba-devops-labs/api:____`

## 3 · Red PR

- Red PR (URL): ____
- The test that failed, with Expected and Received (copied): ____
- The cause line from "On failure: the API log" (copied): ____
- The merge box while it was red (the text, copied): ____
- Which commit did the red check test: your branch alone, or your branch merged with `main`? How do you know? ____

## 4 · Answers

**A1.** ____

**A2.** ____

**A3.** ____

**A4.** ____
