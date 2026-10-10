# Troubleshooting

Problems hit while building the WordPress + MySQL lab on a 3-node kubeadm cluster (Vagrant VMs, Flannel, NFS), with the symptom, diagnosis, cause and fix for each.

## Contents

1. [Pods on different nodes cannot communicate](#1-pods-on-different-nodes-cannot-communicate)
2. [Requests to a pod IP hang, but the Service looks correct](#2-requests-to-a-pod-ip-hang-but-the-service-looks-correct)
3. [Flannel rollout stuck after editing the DaemonSet](#3-flannel-rollout-stuck-after-editing-the-daemonset)
4. [WordPress pods stay 0/1 Ready](#4-wordpress-pods-stay-01-ready)
5. [Readiness probe uses the wrong port](#5-readiness-probe-uses-the-wrong-port)
6. [Node goes NotReady after OOM, then the other node follows](#6-node-goes-notready-after-oom-then-the-other-node-follows)
7. [Second MySQL crash-loops with exit code 1](#7-second-mysql-crash-loops-with-exit-code-1)
8. [PVC binds to the wrong PV](#8-pvc-binds-to-the-wrong-pv)

---

## 1. Pods on different nodes cannot communicate

**Symptom**
`curl http://<pod-ip>` from the control plane hangs when the pod runs on a worker node. The same request works when run on the node that hosts the pod.

**Diagnosis**
```bash
kubectl get pods -o wide                       # which node hosts each pod
kubectl get nodes -o wide                      # node INTERNAL-IPs were correct (192.168.2.x)
kubectl describe node <node> | grep public-ip  # Flannel's registered IP
```
Flannel's `public-ip` annotation was `10.0.2.15` on every node.

**Cause**
`10.0.2.15` is the Vagrant NAT address. It is identical on every VM and not routable between them, so VXLAN packets never reached the other node. Flannel had picked the wrong interface even though the kubelet node IPs were correct.

**Fix**
Point Flannel at the private-network interface. In the `kube-flannel` container args of the DaemonSet, add one of:
```yaml
- --iface=<interface>                      # e.g. eth1 (same name on every node)
- --iface-regex=192\.168\.2\..*            # matches by IP, works with different names
```
Then restart the Flannel pods and re-check `public-ip`.

---

## 2. Requests to a pod IP hang, but the Service looks correct

**Symptom**
`kubectl get svc` and `kubectl describe svc` looked right (correct selector, endpoints listed), but requests still hung.

**Diagnosis**
- `Endpoints:` listed both pod IPs, so the selector and labels were fine.
- Testing from the node that hosts the pod worked, which pointed to cross-node networking (see issue 1).
- A NodePort was tested against a pod IP (`10.244.x.x:30008`), which is wrong.

**Cause**
Two separate things: the overlay network problem above, and a test mistake. A NodePort listens on the **node's** IP, not the pod's IP.

**Fix**
Test the right targets:
```bash
curl http://<pod-ip>:80          # pod IP uses the container port
curl http://<node-ip>:30008      # NodePort uses a node IP
```

---

## 3. Flannel rollout stuck after editing the DaemonSet

**Symptom**
`kubectl rollout status ds kube-flannel-ds -n kube-flannel` kept reporting `0 of 3 updated pods are available`, and the node `public-ip` annotation did not change.

**Diagnosis**
```bash
kubectl get pods -n kube-flannel -o wide
kubectl logs -n kube-flannel <pod> --previous
kubectl get ds kube-flannel-ds -n kube-flannel -o yaml | grep -A6 "args:"
```
Typical causes found in the logs or the YAML:
- `Failed to find interface with specified name`: the interface name differed between nodes.
- A malformed edit: wrong indentation, or the flag added to the init container instead of the main container.

**Cause**
The new args made the Flannel pods crash or never become ready, so they never re-registered the node IP.

**Fix**
Use `--iface-regex` instead of an interface name, make sure it is in the main `kube-flannel` container, and keep only one `--iface` or `--iface-regex` line.

---

## 4. WordPress pods stay 0/1 Ready

**Symptom**
The WordPress pods were `Running` but `0/1`. The site worked for a few minutes and then stopped. Events showed:
```
Readiness probe failed: ... context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

**Diagnosis**
```bash
kubectl describe pod <wordpress-pod>        # events and probe settings
kubectl get endpoints wordpress-service     # empty while pods are unready
kubectl logs deploy/wordpress --tail=30
```

**Cause**
- The probe had a 1-second timeout, but WordPress on NFS is slow, especially on first start while it copies about 60 MB of files onto the share.
- Pods that fail readiness are removed from the Service endpoints, so the site "stopped working".
- Two replicas started at once and both copied files into the same empty volume.

**Fix**
- Use a TCP readiness probe with generous settings:
  ```yaml
  readinessProbe:
    tcpSocket:
      port: 80
    initialDelaySeconds: 20
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 6
  ```
- Switch the NFS exports from `sync` to `async` for the lab.
- Start with **1 replica**, finish the installer, then scale to 2.

---

## 5. Readiness probe uses the wrong port

**Symptom**
Probe failures in the events referenced `:30009` (the NodePort) and `/wp-admin/install.php` instead of port 80 and `/`:
```
Get "http://10.244.1.9:30009/": dial tcp ...:30009: connect: connection refused
```

**Diagnosis**
```bash
kubectl get deploy wordpress -o yaml | grep -B2 -A8 readinessProbe
```
The live Deployment did not match the intended manifest.

**Cause**
A probe runs inside the pod network against the **container** port. A NodePort exists only on the node, so probing it always fails.

**Fix**
Probe the container port (80, or 9000 for php-fpm). Re-apply the corrected manifest, or patch the live object. Always compare the live YAML with the file you think you applied.

---

## 6. Node goes NotReady after OOM, then the other node follows

**Symptom**
- The MySQL pod was killed (`OOMKilled`, exit code 137).
- One node became `NotReady`. Logs failed with `TLS handshake timeout` against port 10250.
- After the node was reloaded, the other node went `NotReady` as well.

**Diagnosis**
```bash
kubectl describe pod <mysql-pod> | grep -A8 "Last State"
kubectl get nodes
# on the node:
free -h
sudo dmesg -T | grep -i -E "oom|killed process" | tail
```

**Cause**
- The VMs had too little RAM and the pods had no memory requests or limits.
- While the first node was down, its pods were rescheduled onto the second node, which then ran out of memory too. Kubernetes does not rebalance pods after the first node returns.

**Fix**
- Raise `vb.memory` in the Vagrantfile (at least 2 GB per VM, 3 GB or more when running both WordPress variants).
- Reload nodes **one at a time**, waiting for each to show `Ready`.
- Add requests and limits so the scheduler accounts for memory:
  ```bash
  kubectl set resources deploy mysql --requests=memory=512Mi --limits=memory=1Gi
  kubectl set resources deploy wordpress --requests=memory=256Mi --limits=memory=512Mi
  kubectl rollout restart deploy wordpress
  ```
- Temporarily scale down unneeded workloads to relieve pressure.

---

## 7. Second MySQL crash-loops with exit code 1

**Symptom**
A newly created MySQL Deployment went `Error`, then `CrashLoopBackOff`, with exit code 1 a few seconds after each start.

**Diagnosis**
```bash
kubectl logs deploy/wpn-mysql --tail=30
kubectl describe pod <pod>
ls -la /srv/nfs/wpn-mysql      # on the NFS server
```

**Cause**
The first start of this MySQL had been killed mid-initialization by the memory problem in issue 6. The PV reclaim policy is `Retain`, so the half-initialized data directory stayed on the NFS share, and MySQL refused to start on a non-empty, inconsistent directory.

**Fix**
```bash
kubectl scale deploy wpn-mysql --replicas=0
ls -la /srv/nfs/wpn-mysql                          # confirm it is the right folder first
sudo rm -rf /srv/nfs/wpn-mysql/{*,.[!.]*}
kubectl scale deploy wpn-mysql --replicas=1
kubectl logs deploy/wpn-mysql -f                   # wait for "ready for connections"
```
Check the path carefully before deleting. The other MySQL's folder must not be touched.

Other errors worth knowing:

| Log message | Likely cause |
|---|---|
| `Permission denied`, `Can't create/write to file` | NFS export missing `no_root_squash`, or `exportfs -ra` not run |
| `Unable to lock ./ibdata1` | NFS locking problem, check `nfs-common` on the node |
| `OOMKilled` in `describe pod` | Not enough memory on the node |

---

## 8. PVC binds to the wrong PV

**Symptom**
Risk rather than an observed failure: with several static PVs and `storageClassName: ""`, a claim can bind to any compatible volume.

**Cause**
A ReadWriteMany PV also satisfies a ReadWriteOnce request, so a MySQL claim could bind to the WordPress volume (or the reverse).

**Fix**
Bind each claim to its volume explicitly:
```yaml
spec:
  storageClassName: ""
  volumeName: wp-pv
```
Verify with `kubectl get pv,pvc` that each claim is `Bound` to the intended volume.

---

## Handy commands

```bash
kubectl get pods -o wide                     # placement and status
kubectl describe pod <pod>                   # events, probes, last state
kubectl logs <pod> --previous                # logs from the crashed container
kubectl logs deploy/<name> -c <container>    # one container in a multi-container pod
kubectl get endpointslices                   # which pods back a Service
kubectl get nodes -o wide                    # node IPs and status
kubectl describe node <node> | grep -i flannel
showmount -e <nfs-server-ip>                 # list NFS exports (from a node)
```
