<div align="center">

# Homelab

![license](https://img.shields.io/github/license/ametis70/homelab?style=flat-square)
![k3s](https://img.shields.io/badge/k3s-FFC61C?style=flat-square&logo=k3s&logoColor=white)
![flux](https://img.shields.io/badge/flux-5468FF?style=flat-square&logo=flux&logoColor=white)

GitOps-managed Kubernetes cluster running on [k3s](https://k3s.io),<br />
reconciled by [Flux](https://fluxcd.io) from this repository

</div>

## Architecture

The cluster is deployed on k3s and fully managed via GitOps using the [Flux Operator](https://github.com/controlplaneio-fluxcd/flux-operator). All application manifests live in this repository and are automatically reconciled to the cluster.

- **Storage** — [Longhorn](https://longhorn.io) provides replicated block storage with three storage classes for different durability and backup needs
- **Secrets** — [External Secrets Operator](https://external-secrets.io) syncs secrets from an external store
- **Database** — [CloudNativePG](https://cloudnative-pg.io) manages PostgreSQL clusters with continuous WAL archiving and automated backups via barman-cloud
- **Ingress** — [Traefik](https://traefik.io) handles routing and TLS termination
- **Auth** — [Authentik](https://goauthentik.io) provides SSO and identity management
- **Monitoring** — [Prometheus](https://prometheus.io), [Grafana](https://grafana.com), [Loki](https://grafana.com/oss/loki), and [Alloy](https://grafana.com/oss/alloy) for metrics, dashboards, and log aggregation
- **Notifications** — Flux alerts are forwarded via Telegram

## Notes

This setup is heavily inspired by the patterns found in [onedr0p/flux-cluster-template](https://github.com/onedr0p/flux-cluster-template) and the broader [k8s-at-home](https://github.com/topics/k8s-at-home) community.
