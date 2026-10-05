# Turiba DevOps labs

Lab guides for **Software Development and IT Operations (DevOps)**, course ELE1013M, Turiba University.

## Labs

| Lab | Session | Guide | You hand in |
|----:|---------|-------|-------------|
| 1 | S1 · From virtualization to containers | [Ship it](lab01-ship-it/README.md) | A public image on GHCR and `lab1/README.md` in your fork |

The next lab guide is published here before its session.

## How the labs work

- **You work on your own laptop, in your fork of the course repository.** No cloud account or payment card is needed.
- **Each guide has the same parts:** goal and time, steps with ✅ checkpoints, a troubleshooting table, stretch tasks and a command summary.
- **You submit on Evaluentis.** Each lab has an assignment where you paste a link to the file, pull request or tag in your repository. Labs are checked pass/fail against the checklist at the end of each guide.
- **Commands are written for bash.**
  - Windows: use the Ubuntu (WSL) terminal.
  - macOS / Linux: use Terminal.
  - Where PowerShell needs something different, the guide says so.
- **Placeholders:** `<you>` means your GitHub username in lowercase.

## Before Lab 1

Finish the pre-course setup:
- Docker Desktop (or Rancher Desktop) installed and running
- Git and VS Code installed
- a GitHub account with two-factor authentication
- the course repository forked and cloned

Check with:

```bash
docker run --rm hello-world
git --version
```

## Stuck?

1. Read the checkpoint you just failed. It says what you should see.
2. Look up your error in the **Troubleshooting** table at the end of the guide.
3. Ask in class, and bring the exact error message.
