---
name: docker-disk-hygiene
description: Use when Docker or Docker Desktop takes unexpectedly large disk space, the disk is filling up, a build fails with "no space left on device", Docker.raw is huge on a Mac, after months of running coding agents or test builds in containers, when asked to clean up Docker, or for a periodic Docker disk check.
---

# Docker Disk Hygiene

## Overview

Docker keeps images, stopped containers and build cache until someone deletes them. `--rm` removes the container, not its image or the build cache. The built-in build-cache garbage collection can fail to hold its limit, and nothing warns you. Months of agent sandboxes and test builds can quietly take 50 GB.

**Core principle:** measure first, delete by name only after the user picks an option, never touch volumes, then set limits so the pile does not come back.

## Step 1: Measure (read-only)

Use short output forms. Verbose forms such as `docker system df -v` can produce output too large to read.

```bash
docker system df
docker image ls --format "{{.Size}}\t{{.Repository}}:{{.Tag}}\t{{.CreatedSince}}"
docker ps -a --format "{{.State}}\t{{.Names}}\t{{.Image}}"
docker volume ls
```

On a Mac, Docker Desktop keeps everything in one disk file. If your sandbox cannot read it, give the user this command:

```bash
du -h ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw
```

If Docker Desktop is not running, the commands fail. Ask the user to start it.

## Step 2: Sort what you found into piles

| Pile | How to spot it | Risk of deleting |
|---|---|---|
| Build cache | `Build Cache` row in `docker system df`, ACTIVE = 0 | Low: next build is slower, nothing else breaks |
| Images of finished projects and agent sandboxes | old `CreatedSince`, no running container uses them | Medium: rebuild needed if the project comes back |
| Old test or tagged builds | tags like `:test`, `:old`, branch names | Medium: same as above |
| Stopped containers | `docker ps -a` with State `exited` | They block deletion of their images; check they hold no needed state |
| Volumes | `docker volume ls` | **High: this is data (databases, uploads)** |

Totals in `docker system df` overlap: images and build cache share layers, so the sum of rows is larger than the real disk use.

## Step 3: Propose options and wait

Give the user a table of options, from narrowest to widest, with size per option and an explicit "not touched" list:

- **A.** Build cache only.
- **B.** A + named images of projects no longer in use (list each name and size).
- **C.** B + old test tags (list each).
- **Not touched:** running containers, their images, all volumes, base images the user keeps on purpose.

Delete nothing until the user picks an option.

## Step 4: Delete by name

```bash
# Stopped containers that hold images from the chosen option
docker rm <container> <container>

# Images by name, without -f (Docker refuses to remove an image a running container uses)
docker rmi <image> <image>

# Whole build cache
docker builder prune -a -f
```

## Step 5: Verify

Run `docker system df` again and report before / after for each pile. On a Mac the `Docker.raw` file shrinks a few minutes after deletion, not at once.

## Step 6: Prevent the next pile

Tell the user which of these are set and which are not. Docker Desktop settings belong to the user; recommend them, do not change them yourself without permission.

1. **Disk limit for Docker.** Docker Desktop → Settings → Resources → Advanced → Disk usage limit (wording can differ by version). Docker cannot grow past it, and a build that hits it fails with a clear error.
2. **Build cache cap.** Docker Desktop → Settings → Docker Engine, add `"builder": {"gc": {"enabled": true, "defaultKeepStorage": "10GB"}}` → Apply & restart. The default cap did not hold in practice, so the monthly check (item 4) still matters.
3. **Clean up when a project or experiment ends.** In its folder: `docker compose down --rmi all` (also removes base images that compose file pulled), or `docker rmi <image>`.
4. **Monthly check.** `docker system df` once a month. Add it to the user's recurring routine if they have one.

## Never

| Command | Why |
|---|---|
| `docker system prune -a --volumes` | Deletes volumes, which means databases and other data |
| `docker rmi -f` | Removes an image from under a container that still uses it |
| Any prune with `-a` on a shared server | Cache and images there belong to every project on that server |
| Deleting before the user chose | The user decides what is no longer needed |

## Common mistakes

| Assumption | Reality |
|---|---|
| `docker compose run --rm` keeps Docker clean | It removes the container only; the image and build cache stay |
| `docker image prune -a` will remove an old image | It skips any image referenced by a container, even a stopped one; remove the container first |
| The default build-cache GC keeps the cache small | It can fail to hold its limit; check `docker system df` |
| Freed space shows up on a Mac at once | `Docker.raw` shrinks later |
