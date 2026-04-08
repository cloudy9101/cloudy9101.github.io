---
date: 2026-03-31
draft: false
title: How I Cut Our GitHub Actions Pipeline Time by More Than 50%
weight: 10
params:
  author: Ray Hao
---
![Sand in an hourglass](https://images.unsplash.com/photo-1499377193864-82682aefed04?crop=entropy&cs=tinysrgb&fit=max&fm=jpg&ixid=M3wzNjAwOTd8MHwxfHNlYXJjaHwyNHx8dGltZXxlbnwwfDB8fHwxNzc0NTE3NTYxfDA&ixlib=rb-4.1.0&q=80&w=1080)
*Photo by [Kenny Eliason](https://unsplash.com/@heyquilia?utm_source=Obsidian%20Image%20Inserter%20Plugin&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=Obsidian%20Image%20Inserter%20Plugin&utm_medium=referral)*

## Problem

Our CI pipeline was taking 20 minutes on average to finish in every pull request. It wasn't reliable either because tools installation would occasionally fail due to network issues or rate limits. Our team needed a faster feedback loop.

```mermaid
flowchart TB
	A[PR opened / pushed] --> B[Install tools]
	B --> C[Kubernetes cluster setup]
	C --> D[Build services, Run tests, Lint]
```

The project is a medium-complexity Go codebase with several microservices. For every PR, each workflow went through the same steps: checkout, install a suite of binary tools, build, test etc. Build and unit tests were fast by themselves. Most of the time was consumed by installing those binary tools on every single run. And since these installations depended on external networks, they were a constant source of inconsistent failures which led workflows failing for unrelated reasons.
We had been recording per-step timing via OpenTelemetry. Looking at the data, tool installation was repeated on every single run despite never changing between PRs. That was the real target.
## Solution

The fix was straightforward in principle: build a Docker image with all tools pre-installed, host it on GitHub Container Registry, and run all workflows inside that image.

```mermaid
flowchart TB
    A[PR opened / pushed] --> B

	B[Compute content hash]
	B --> C{Image exists}

	D[Build & push]

    C -- no --> D
    D --> E
    C -- yes --> E

	E[All other workflows]

		style B fill:#fbbf24,stroke:#d97706,color:#000
		style C fill:#fbbf24,stroke:#d97706,color:#000
		style D fill:#fbbf24,stroke:#d97706,color:#000
```

> **Why Docker**
> I needed a solution that would let me install tools once and share them across every PR and every workflows. Docker is the most popular and stable option to fulfill this requirement because every GitHub Actions runner already has it available, and GitHub Container Registry makes hosting the image seamless.

In practice, I added two new workflows:

- **resolve-image** computes a hash of the files the base image Dockerfile depends on, checks if an image with that hash tag already exists in the registry, and outputs the result.
- **build-image** builds and pushes the base image with that hash tag, only triggered when resolve-image reports the image doesn't exist yet.

The hash is computed from three files: the Dockerfile itself, `go.mod`, and `go.sum`. If none of these change, the tag is identical and the build is skipped entirely.

```YAML
- name: Compute content hash
  id: hash
  run: |
    HASH=$(cat \
      Dockerfile \
      go.mod \
      go.sum \
      | sha256sum | cut -d' ' -f1 | head -c 16)
    echo "tag=sha-${HASH}" >> "$GITHUB_OUTPUT"
```

All other workflows first run resolve-image. If the image already exists, build-image is skipped and every subsequent job runs on the existing image immediately. If not, build-image runs first, then the rest follow.

```YAML
# In resolve-image: check whether the image already exists
- name: Check if image exists in GHCR
  id: check
  run: |
    if docker manifest inspect ghcr.io/acme/base:${{ steps.hash.outputs.tag }} \
      > /dev/null 2>&1; then
      echo "exists=true" >> "$GITHUB_OUTPUT"
    else
      echo "exists=false" >> "$GITHUB_OUTPUT"
    fi

# In build-image: skip the build if the image already exists
- name: Build and push ci-base image
  if: needs.resolve-image.outputs.exists != 'true'
  uses: docker/build-push-action@v7
  with:
    context: .
    file: Dockerfile
    push: true
    tags: ghcr.io/acme/base:${{ needs.resolve-image.outputs.tag }}
```

## Result

Building the base image takes around 10 minutes but in practice, most PRs never trigger a rebuild. Thanks to [Renovatebot](https://github.com/renovatebot/renovate) handling automatic dependency updates, base image builds happen almost exclusively on Renovatebot PRs (Go version upgrades, tool version bumps, etc.). PRs created by humans almost always skip the build entirely and go straight to running on the existing image.

Inconsistent failures from tool installation have essentially disappeared.

One unexpected benefit: since the base image already has all the tools pre-installed, it doubles as a foundation for our CodeSpaces development environment. Developers get the same tool versions locally as they do in CI, no more "works on my machine" surprises. I'll cover that setup in the next post.

## What Else We Tried

Before landing on the base image approach, my first assumption was that the Kubernetes cluster setup was the bottleneck - we use [kind](https://kind.sigs.k8s.io/) to run dependencies like PostgreSQL and NATS. I replaced kind with [k3s](https://k3s.io/). It saved 1–2 minutes, but nothing significant on its own.

That change is still worth revisiting. After the base image work and the base image adoption in CodeSpaces, replacing kind with k3s across both environments might tell a different story.

**This is part 1 of a 3-part series:**

- **How I Cut Our GitHub Actions Pipeline Time by More Than 50%** ← you are here
- Reusing the CI base image in CodeSpaces
- Replacing kind with k3s in CI and CodeSpaces
