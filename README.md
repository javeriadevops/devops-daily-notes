# devops-daily-notes

A daily learning log for DevOps — commands, concepts, and error-fixes I run into while learning.

This is a study repo, not a polished reference. Notes are written as I learn a topic, and I add errors I actually hit along with how I solved them. If something here is wrong, it's because I was wrong that day and haven't corrected it yet.

---

## Structure

```
.
├── docker/       # Images, containers, Dockerfiles, compose
├── kubernetes/   # Pods, deployments, services, kubectl
├── linux/        # Shell, permissions, processes, networking
└── ci-cd/        # GitHub Actions, pipelines, automation
```

Each note is one file, named by date and topic:

```
docker/2026-08-27-dockerfile-basics.md
```

---

## Note format

Most notes follow the same shape:

- **Topic** — what I studied
- **Why** — what I was building or trying to fix
- **Concepts** — the idea in my own words
- **Commands** — what I actually ran
- **Errors and fixes** — what broke, and what fixed it
- **For tomorrow** — the next thing to pick up

The "errors and fixes" section is the part I care about most. Reading docs is easy; remembering why a container refused to start at 1am is the thing that actually sticks.

---

## Current focus

- Docker — containerizing a FastAPI application
- GitHub Actions — building a CI pipeline for that same project
- Linux — server administration fundamentals

---

## Background

I completed a DevOps internship (server setup and configuration, Linux application deployment, CI/CD pipeline basics) and I'm continuing to learn independently. Tools I've worked with so far: Linux, Nginx, Docker, Git, GitHub Actions, Python, Bash, AWS EC2, and DigitalOcean.

I'm a beginner in most of these. This repo is the record of closing that gap.
