# Lab 5 · Your first CI pipeline

GitHub user: Abhirajdhingrax · Reference files used (Lab 2 fallback): no

![ci](https://github.com/Abhirajdhingrax/turiba-devops-labs/actions/workflows/ci.yml/badge.svg?branch=main)

## 1 · Graph predictions

My drawing of the `lab05 graph` jobs, before the first run:

```text
graph-build ----> graph-test ----> graph-package <---- graph-lint
graph-report needs graph-build, graph-lint, graph-test and graph-package, with if: always()
```

| # | Question | I predicted, and why | What the run showed |
|---|----------|----------------------|---------------------|
| P1 | Which jobs start together? | graph-build and graph-lint, because they have no needs | Run with break = none: ____ |
| P2 | Will `graph-package` find `build.txt`? | No: every job runs on a fresh runner, and graph-package has no download step | graph-package printed: ____ |
| P3 | With `break = build`: which jobs run, which fail, which are skipped? | graph-build fails, graph-lint succeeds, graph-test and graph-package are skipped, graph-report still runs because of if: always() | ____ |

## 2 · Pipeline

- B5, before I filled it in: `localhost` or `db`? I predicted localhost because the integration steps run directly on the runner VM (no container: key), and the service publishes port 5432 to it
- My first red run in Part 2 (the job, the first red step, the error line copied from the log): ____
- What I changed to make it green: see the commits of the PR
- My lint findings, and how I fixed them (fixed, not silenced): No findings: npm run lint printed no problems.
- Part 2 pull request (URL): ____
- Part 3: will the PR run push an image? I predicted no, because a pull_request run only builds. What happened: the step "Log in to GHCR (main only)" was skipped and push was false, so the PR only built the amd64 image
- My image on GHCR, from the `main` run: `ghcr.io/abhirajdhingrax/turiba-devops-labs/api:____`

## 3 · Red PR

- Red PR (URL): ____
- The test that failed, with Expected and Received (copied): ____
- The cause line from "On failure: the API log" (copied): ____
- The merge box while it was red (the text, copied): ____
- Which commit did the red check test: your branch alone, or your branch merged with `main`? How do you know? My branch merged with main: a pull_request run checks out refs/pull/N/merge, a temporary merge commit, and the run summary shows that merge commit, not my branch head

## 4 · Answers

**A1.** With break = build: ____. graph-build failed, so graph-test (needs graph-build) and graph-package (needs graph-test) were skipped; graph-lint needs nothing, so it ran. graph-report still ran because if: always() replaces the rule that every needed job must succeed.

**A2.** Every job runs on its own fresh runner and jobs don't share files. graph-test has an actions/download-artifact step for the build-output artifact, graph-package doesn't. Adding the same download-artifact step (name: build-output) to graph-package would make it find build.txt.

**A3.** In CI the steps run on the runner VM, and the PostgreSQL service publishes 5432 to that VM, so the database is at localhost. In Compose the API runs in a container on the Compose network, where the service name db resolves by DNS. If the job ran in a container (container: node:24-alpine), it would be db, the service label, because then the job and the service share a network.

**A4.** The required check integration failed. The run tested the temporary merge commit of my branch with main. The ruleset protect-main requires lint, unit, integration and image, so the merge button was blocked until all four were green. On my Part 2 PR the image job ran but did not push, because login and push only happen when github.event_name is push, i.e. on main.
