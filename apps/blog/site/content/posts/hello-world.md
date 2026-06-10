---
title: "Hello, world"
date: 2026-06-10
description: "First post, served from the homelab it's written about."
---

This blog runs on the same AKS homelab it writes about: a Hugo site built in
CI, baked into a Caddy image, and deployed through FluxCD behind Envoy
Gateway, CrowdSec, and Anubis.

Posts are Markdown files in the [homelab repo](https://github.com/Seazeeee/aks-homelab) —
publishing one looks like this:

```bash
hugo new content posts/my-post.md
git commit -S -m "post: my post"
```

More to come on how the cluster is put together.
