---
title: Designing external secrets for homelab clusters
draft: true
breadcrumbs: false
---

# Considerations

- Needs to keep secrets secure, including away from the repository.
- Use of ESO is preferred as it is the standard approach and allows to switch out secret providers at a later date.
- GitHub Actions secrets would be ideal because the secrets can be kept with the repository, however this mechanism is write-only so can only be used to push secrets to GitHub.
- Use of in-cluster storage not an option due 