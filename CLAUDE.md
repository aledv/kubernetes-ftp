# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository is a small set of Kubernetes manifests that deploy an FTP
server on a Kubernetes cluster using the [fauria/vsftpd](https://github.com/fauria/docker-vsftpd)
Docker image. There is no application code, build system, or test suite —
just static YAML resources.

## Deploying

Resources must be created in this order, since later resources depend on
earlier ones (the PVC binds to the PV, and the Deployment mounts the PVC):

```
kubectl create -f task-pv-volume.yaml
kubectl get pv task-pv-volume

kubectl create -f task-pv-claim.yaml
kubectl get pvc task-pv-claim

kubectl create -f ftp-deployment.yaml
kubectl create -f ftp-service.yaml
kubectl create -f ftp-ingress.yaml
```

To validate a manifest without applying changes: `kubectl apply --dry-run=client -f <file>.yaml`
(requires a reachable cluster context — `kubectl` needs API discovery even
in dry-run/client mode, it cannot validate fully offline). For a pure YAML
syntax check with no cluster available, e.g. `ruby -ryaml -e "YAML.load_stream(File.read('<file>.yaml'))"`.

Full cleanup: `kubectl delete -f ftp-service.yaml -f ftp-deployment.yaml -f task-pv-claim.yaml -f task-pv-volume.yaml`
(add `-f ftp-ingress.yaml` if it was created).

## Architecture / resource relationships

- **task-pv-volume.yaml** — a `PersistentVolume` (`task-pv-volume`) backed by
  `hostPath: /tmp/data`, 10Gi capacity, `ReadWriteOnce`. Node-local by
  construction: only correct on a single-node cluster (minikube/kind/Docker
  Desktop/k3d).
- **task-pv-claim.yaml** — a `PersistentVolumeClaim` (`task-pv-claim`)
  requesting 3Gi, `ReadWriteOnce`. Binds to the PV above.
- **ftp-deployment.yaml** — `Deployment` `my-ftp` (`apps/v1`), **1 replica**
  (kept at 1 deliberately — see hostPath note above; the PV isn't guaranteed
  to co-locate with the pod once there's more than one node) of the
  `fauria/vsftpd` image. Mounts `task-pv-claim` at `/home/vsftpd` (the FTP
  chroot/home directory). Sets `FTP_USER`/`FTP_PASS` as literal demo
  credentials (`user`/`pass1234` — not meant to be reused; a real deployment
  should source them from a `Secret` via `secretKeyRef`) and
  `PASV_ADDRESS`/`PASV_MIN_PORT`/`PASV_MAX_PORT` (`21100`-`21110`) for
  passive-mode FTP, which most clients require.
- **ftp-service.yaml** — `NodePort` Service `my-ftp-service` selecting
  `app: my-ftp`. Exposes port 21 (`nodePort: 30080`) plus one explicit
  Service port entry per passive-mode port `21100`-`21110`
  (`nodePort: 31100`-`31110`) — NodePort has no "range" concept, so the
  passive port range on the Deployment and the individual ports listed here
  must be kept in sync by hand.
- **ftp-ingress.yaml** — Ingress `ftp-http-ingress` (`networking.k8s.io/v1`),
  default backend `my-ftp-service:21`. **Does not actually work**: a
  Kubernetes `Ingress` is an HTTP(S) L7 router and cannot proxy the raw TCP
  FTP protocol (control connection + dynamic passive data connections) —
  no stock ingress controller routes this through a plain `Ingress` object.
  It's kept in the repo (API version and the previously-wrong service name
  are fixed) purely as a documented dead end; external access goes through
  the `NodePort` Service instead. Don't "fix" this file further without
  first re-reading the warning comment at its top.

All manifests currently target `apps/v1` / `networking.k8s.io/v1` / `v1`
(the older `apps/v1beta1` / `extensions/v1beta1` this repo originally used
are removed in modern Kubernetes and have already been migrated).
