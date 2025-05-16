---
title: Kubernetes Tools
description: A guide to our recommended Kubernetes tools.
date: 2025-06-15
tags: [Kubernetes, tools, tutorial]
---

> [!NOTE]
> Primary audience: Skyscrapers customers and anyone interested in Kubernetes tools.

This document provides a list of our recommended tools for interacting with Kubernetes. The components on this list are tools we recommend and have tested or have experience with. If you have any questions about these tools, please reach out to us.

## K9s

[K9s](https://k9scli.io/) is a terminal-based UI for managing Kubernetes clusters. It provides a powerful command-line interface that allows developers to interact with their clusters without needing to switch to a graphical interface.

K9s is available through most package managers. You can find installation instructions [here](https://k9scli.io/topics/install/).

[![K9s demo](https://github.com/derailed/k9s/raw/master/assets/screen_po.png)](https://asciinema.org/a/305944)

## Headlamp

[Headlamp](https://headlamp.dev/), a CNCF sandbox project, is a application that focuses on visibility, modularity and ease of use. Headlamp has a [plugin specifically for Flux](https://github.com/headlamp-k8s/plugins/tree/main/flux), making it a great choice for developers using that framework.

> [!NOTE]
> We don't run Headlamp on our clusters for security reasons

Headlamp is available through most package managers. You can find installation instructions [here](https://headlamp.dev/docs/latest/installation/desktop).

![Headlamp demo](https://raw.githubusercontent.com/kubernetes-sigs/headlamp/screenshots/videos/headlamp_quick_run.gif)
