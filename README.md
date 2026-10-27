# kubernetes-ftp

[![CI](https://github.com/aledv/kubernetes-ftp/actions/workflows/validate.yml/badge.svg)](https://github.com/aledv/kubernetes-ftp/actions/workflows/validate.yml) [![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

A minimal example of deploying an FTP server on Kubernetes, using the
[fauria/vsftpd](https://github.com/fauria/docker-vsftpd) Docker image.

This is a learning/demo project, intended to run on a **single-node**
cluster (minikube, kind, Docker Desktop, k3d, ...). See
[Known limitations](#known-limitations) before using it as a base for
anything real.

## Contents

- [What's in this repo](#whats-in-this-repo)
- [Prerequisites](#prerequisites)
- [Deploying](#deploying)
- [Configuration](#configuration)
- [Connecting to the FTP server](#connecting-to-the-ftp-server)
- [Known limitations](#known-limitations)
- [Cleanup](#cleanup)
- [Validation](#validation)
- [Reference](#reference)
- [Contributing](#contributing)
- [License](#license)

## What's in this repo

| File | Kind | Purpose |
|---|---|---|
| `task-pv-volume.yaml` | `PersistentVolume` | 10Gi volume backed by `hostPath: /tmp/data` on the node. |
| `task-pv-claim.yaml` | `PersistentVolumeClaim` | Claims 3Gi from the volume above, `ReadWriteOnce`. |
| `ftp-deployment.yaml` | `Deployment` | Runs the `fauria/vsftpd` container, mounts the claim at `/home/vsftpd` as the FTP home/chroot directory. |
| `ftp-service.yaml` | `Service` (`NodePort`) | Exposes the FTP control port (21) and the passive-mode data port range (31100-31110) outside the cluster. |
| `ftp-ingress.yaml` | `Ingress` | Present for completeness, but **does not work for FTP** - see [Known limitations](#known-limitations). |

## Prerequisites

- A running Kubernetes cluster and a configured `kubectl` context. For local
  use, any single-node cluster works (minikube, kind, Docker Desktop, k3d).
- An FTP client to test with (e.g. `ftp`, FileZilla, `lftp`).

## Deploying

Resources must be created in this order, since later resources depend on
earlier ones (the PVC binds to the PV, and the Deployment mounts the PVC):

```sh
kubectl create -f task-pv-volume.yaml
kubectl get pv task-pv-volume

kubectl create -f task-pv-claim.yaml
kubectl get pvc task-pv-claim

kubectl create -f ftp-deployment.yaml
kubectl create -f ftp-service.yaml
```

`ftp-ingress.yaml` is intentionally not part of the recommended flow (see
below); create it only if you understand and accept its limitation.

Check that the pod is running:

```sh
kubectl get pods -l app=my-ftp
kubectl logs -l app=my-ftp
```

## Configuration

### Credentials

`ftp-deployment.yaml` sets `FTP_USER`/`FTP_PASS` as plain environment
variables (`user` / `pass1234`) for demo purposes only. Before using this
for anything beyond a local experiment, move them into a Kubernetes
`Secret`:

```sh
kubectl create secret generic ftp-credentials \
  --from-literal=FTP_USER=someuser \
  --from-literal=FTP_PASS=some-strong-password
```

and reference them in the container spec with `valueFrom.secretKeyRef`
instead of a literal `value`.

### Passive mode (PASV)

Most modern FTP clients use passive mode, where the *server* tells the
client which port to connect to for the data channel, instead of the server
connecting back to the client (active mode, which is normally blocked by
NAT/firewalls). This is why `ftp-deployment.yaml` sets:

- `PASV_ADDRESS` - the externally reachable address of the node running the
  pod (e.g. its LAN/public IP). Update this to match your environment;
  `127.0.0.1` only works if the FTP client runs on the same machine as the
  cluster node.
- `PASV_MIN_PORT` / `PASV_MAX_PORT` - the port range vsftpd will use for
  data connections (`31100`-`31110`).

vsftpd announces these port numbers *literally* to the client in its
PASV/EPSV response, and the client connects to that exact number - there is
no translation. That's why this range is `31100`-`31110` and not, say,
`21100`-`21110`: Kubernetes `NodePort` only allows ports in `30000`-`32767`
by default, so the externally-reachable port and the port vsftpd announces
have to be the same numbers from the start, or clients will try to connect
to a port that was never opened externally and passive transfers will fail
with "connection refused" (this was verified against a live cluster - it's
not a hypothetical). Consequently `containerPort` in `ftp-deployment.yaml`
and `port`/`targetPort`/`nodePort` in `ftp-service.yaml` must all use these
same numbers too. Kubernetes `NodePort` has no concept of a port *range*
either - each port has to be listed explicitly as its own Service port
entry, which is why the Service lists 11 separate `ftp-pasv-*` ports. If you
widen the PASV range, widen the Deployment's `containerPort`s and the
Service's ports accordingly (keeping all four - PASV env var, containerPort,
targetPort, nodePort - numerically identical), or passive transfers on the
extra ports will hang.

## Connecting to the FTP server

With the default `NodePort` values, from outside the cluster:

```
ftp <node-ip> 30080
```

using the node's IP (e.g. `minikube ip`) and the credentials configured
above. The client will then be redirected to one of the passive ports
(`31100`-`31110` on the node) for each data transfer.

## Known limitations

This project intentionally keeps things simple; the tradeoffs are called
out here rather than hidden:

- **Single-node storage.** `task-pv-volume.yaml` uses `hostPath`, which
  binds the volume to one node's local disk. `ftp-deployment.yaml` runs a
  single replica (`replicas: 1`) for this reason - Kubernetes gives no
  guarantee that a `hostPath` PV and a pod using it land on the same node
  once there is more than one node, so scaling up in a multi-node cluster
  would risk pods seeing an empty or inconsistent directory. To run more
  than one replica on a real cluster, replace the storage with something
  that supports `ReadWriteMany` (e.g. NFS) and add pod anti-affinity /
  session-affinity handling as needed.
- **`ftp-ingress.yaml` does not proxy FTP.** A Kubernetes `Ingress` is an
  HTTP(S) layer-7 router: it parses HTTP requests and routes them by
  host/path. FTP is a raw TCP protocol (a control connection on port 21,
  plus dynamically-negotiated passive-mode data connections) that no stock
  ingress controller can route through a plain `Ingress` object. This file
  is kept in the repo with its API version and service name fixed (it
  previously pointed at a service that didn't even exist), but it will not
  give you external FTP access. Use the `NodePort` Service instead. The
  only realistic way to front FTP with an ingress controller is
  controller-specific TCP passthrough configuration (e.g. ingress-nginx's
  `tcp-services` ConfigMap), which is unrelated to the `Ingress` resource
  type.
- **Static PV binding depends on `storageClassName: ""`.** Both
  `task-pv-volume.yaml` and `task-pv-claim.yaml` set `storageClassName: ""`
  on purpose. Without it, a cluster with a default `StorageClass` (e.g. k3s
  ships `local-path` as default) auto-assigns that class to the claim, which
  then dynamically provisions its *own* volume instead of binding to the
  hand-created `task-pv-volume` - the pod still works, but silently ends up
  using different storage than this repo documents, and `task-pv-volume`
  sits unused. Verified live: removing `storageClassName: ""` reproduces
  this.
- **Demo credentials.** See [Credentials](#credentials) above.
- **Plain FTP, not FTPS.** Traffic (including the login) is unencrypted.
  `fauria/vsftpd` supports TLS via additional environment variables if
  encryption is required - see its
  [documentation](https://github.com/fauria/docker-vsftpd).

## Cleanup

```sh
kubectl delete -f ftp-service.yaml
kubectl delete -f ftp-deployment.yaml
kubectl delete -f task-pv-claim.yaml
kubectl delete -f task-pv-volume.yaml
```

(and `kubectl delete -f ftp-ingress.yaml` if you created it.)

## Validation

CI runs [yamllint](https://github.com/adrienverge/yamllint) and
[kubeconform](https://github.com/yannh/kubeconform) (schema check against the
Kubernetes API, no cluster needed) on every push. Locally:

```sh
yamllint .
kubeconform -strict -summary *.yaml
```

## Reference

- Docker image used: [fauria/vsftpd](https://github.com/fauria/docker-vsftpd)

## Contributing

Issues and pull requests are welcome. Please run the linters and tests before opening a pull request.

## License

MIT, see [LICENSE](LICENSE).
