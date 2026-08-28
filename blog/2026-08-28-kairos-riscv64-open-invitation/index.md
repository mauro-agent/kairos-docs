---
authors:
  - mauro-morales
description: An open invitation to the RISC-V community to help test Kairos on riscv64, plus everything you need to build and boot your own image.
slug: 2026/08/28/kairos-riscv64-open-invitation
tags:
  - kairos
title: "Kairos on RISC-V: An Open Invitation to Test It"
---

Dear community,

For the past few months I've been working on RISC-V support for Kairos. I think this is a very exciting technology and one where I'd enjoy seeing the easy lifecycle management that Kairos brings paired with. In this post you'll see an emulated k3s cluster on an Ubuntu 24.04 Kairos image. I think it's time to get out of the VM, and here's where you come in. If you have a RISC-V board, I'd like to invite you to do some testing to help us run on metal.

{/* truncate */}

I've teased the progress a few times, but I never had anything concrete to actually show, until now. I want you to keep an open mind. This is a work in progress, and there are probably more rough edges than I'd like.

![Terminal showing a Kairos riscv64 system booted on Ubuntu 24.04, with `kubectl get nodes` reporting the node Ready as control-plane, running k3s v1.36.3](./kairos-riscv64-k3s-ready.png)

As you can see, this is a Kairos system based on Ubuntu 24.04, running k3s, on riscv64. Exciting, but it's way too soon to call victory: none of this has been tested on real hardware yet. Report what breaks, and we'll work through the fixes together.

## Where things stand

The Ubuntu base comes straight from upstream. k3s comes from [CARV-ICS-FORTH's actively-maintained riscv64 fork](https://github.com/CARV-ICS-FORTH/k3s), since k3s has no official riscv64 release yet. There's an open upstream ticket to change that: [k3s-io/k3s#7151](https://github.com/k3s-io/k3s/issues/7151). The same is true for k0s: [k0sproject/k0s#1919](https://github.com/k0sproject/k0s/issues/1919). One way to help without touching any hardware: go show interest on those tickets, it moves us closer to official releases.

Worth mentioning too: [Hadron](https://kairos.io/blog/2025/12/17/introducing-hadron-the-minimal-upstream-first-linux-base-for-kairos/), the minimal, upstream-first base Kairos is moving toward, already builds and passes its own tests on riscv64 in CI. There's just no public release with it yet.

## The easy path

Download the ISO from the [latest release](https://github.com/mauromorales/homelab/releases/tag/kairos-riscv64-v0.1.3). Boot it (`qemu-system-riscv64`, or real riscv64 hardware if you have it) and tell me if and where it breaks, with as much detail as you can give me. We'll work through a fix together, and you can help me keep testing from there.

The raw disk image (for flashing straight to hardware, no installer step) is too big for a GitHub release asset, so it's published as a scratch OCI image on Quay instead:

```bash
export IMAGE=quay.io/mauromorales/kairos-riscv64:0.1.3-img
container=$(docker create "$IMAGE" noop)   # scratch image, never executed
docker cp "$container:/output/." .
docker rm "$container"
```

## The long path: build your own

You can choose between Debian, Ubuntu, and openSUSE. All three have been manually tested and confirmed booting on native riscv64 (see the [tracking issue](https://github.com/kairos-io/kairos/issues/4083) for the details and known rough edges, including a graphics gap right after GRUB on all three that serial console doesn't show). Fedora is trickier: there's no official Fedora riscv64 base image yet, only a community-maintained one people have used successfully for testing, so treat it as experimental on top of experimental. Details in that same tracking issue.

Here's the Kairos Dockerfile I use for the Ubuntu flavor, from [my homelab repo](https://github.com/mauromorales/homelab/tree/main/nodes/kairos-riscv64). This is also where k3s gets installed manually instead of through `kairos-init`'s usual provider mechanism, exactly because it's not an upstream release yet:

```dockerfile
ARG KAIROS_INIT=latest

FROM quay.io/kairos/kairos-init:${KAIROS_INIT} AS kairos-init
FROM ubuntu:24.04 AS base-kairos
ARG MODEL=generic
ARG VERSION

RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

RUN --mount=type=bind,from=kairos-init,src=/kairos-init,dst=/kairos-init \
    /kairos-init -l debug -s install -m "${MODEL}" --version "${VERSION}" && \
    /kairos-init -l debug -s init -m "${MODEL}" --version "${VERSION}"

# k3s has no official riscv64 release yet, so this fetches CARV-ICS-FORTH's
# fork and installs it the same way kairos-init's own k3s provider would.
ARG K3S_RISCV64_RELEASE=20260817
RUN curl -fL -o /usr/bin/k3s \
        "https://github.com/CARV-ICS-FORTH/k3s/releases/download/${K3S_RISCV64_RELEASE}/k3s-riscv64" && \
    chmod +x /usr/bin/k3s && \
    curl -sfL https://get.k3s.io -o /tmp/k3s-install.sh && \
    chmod +x /tmp/k3s-install.sh && \
    INSTALL_K3S_BIN_DIR=/usr/bin \
    INSTALL_K3S_SKIP_DOWNLOAD=true \
    INSTALL_K3S_SKIP_ENABLE=true \
    /tmp/k3s-install.sh && \
    rm -f /tmp/k3s-install.sh
```

The full file has a bit more (provider-kairos wiring for the cloud-config `k3s:` stanza), all commented, in the [homelab repo](https://github.com/mauromorales/homelab/blob/main/nodes/kairos-riscv64/kairos.Dockerfile).

Build it directly:

```bash
docker buildx build --platform linux/riscv64 \
  -f kairos.Dockerfile \
  --build-arg VERSION=0.1.0 \
  -t kairos-riscv64:kairos --load .
```

And finally produce an ISO with [AuroraBoot](https://github.com/kairos-io/AuroraBoot):

```bash
sudo ./auroraboot --debug build-iso --arch riscv64 --output ./output/ \
  --override-name kairos-riscv64-0.1.0 \
  docker:kairos-riscv64:kairos
```

Or a raw disk image instead:

```bash
sudo ./auroraboot --debug \
  --set "disable_http_server=true" \
  --set "disable_netboot=true" \
  --set "container_image=docker:kairos-riscv64:kairos" \
  --set "state_dir=./output-raw" \
  --set "disk.raw=true" \
  --set "arch=riscv64"
```

That's it. Now you should have an image to test on your device.

## If your device boots successfully

The next step is to actually install Kairos. I've left a very basic [cloud-config](https://github.com/mauromorales/homelab/blob/main/nodes/kairos-riscv64/cloud-config.yaml) you can use as a starting point, but feel free to write your own, backed by the [Kairos configuration reference](https://kairos.io/docs/reference/configuration).

You can install it manually with the agent:

```bash
kairos-agent manual-install --device auto path/to/your/cloud-config.yaml
```

Then shut down and make sure the CD isn't mounted, if that's how you installed. The next time the system boots, it shouldn't show the installer option on GRUB anymore, and your system should be installed.

Open issues on [kairos-io/kairos](https://github.com/kairos-io/kairos), and I'll address them directly with you. If what you hit is actually about k3s or k0s themselves, rather than Kairos, open it upstream instead: [k3s-io/k3s#7151](https://github.com/k3s-io/k3s/issues/7151) or [k0sproject/k0s#1919](https://github.com/k0sproject/k0s/issues/1919).

Thanks!
