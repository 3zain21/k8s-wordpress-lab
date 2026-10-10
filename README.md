# k8s-wordpress-lab

Highly available WordPress + MySQL on a kubeadm cluster, with **2 WordPress replicas** and shared NFS storage. Two variants are included:

| Variant | Web stack | Folder | NodePort |
|---|---|---|---|
| 1 | Apache + PHP (official `wordpress` image) | `01-apache-wordpress/` | 30009 |
| 2 | nginx + PHP-FPM (`wordpress:fpm` image) | `02-nginx-fpm-wordpress/` | 30010 |

Both variants run side by side on the same cluster. Each has its own MySQL, NFS folders, Secrets and Service.

## Architecture

```
                 NodePort (30009 / 30010)
                          |
              +-----------+-----------+
              |                       |
        WordPress pod 1         WordPress pod 2
     (Apache, or nginx+FPM)   (Apache, or nginx+FPM)
              |                       |
              +-----------+-----------+
                          |
          +---------------+----------------+
          |                                |
   NFS (ReadWriteMany)              MySQL Service (ClusterIP)
   /var/www/html shared                    |
                                      MySQL pod (1 replica)
                                           |
                                   NFS (ReadWriteOnce)
```

Key design points:

- **Shared storage:** both WordPress replicas mount the same NFS volume at `/var/www/html`, so uploads, plugins and themes are identical on every pod.
- **Shared auth keys and salts:** stored in one Secret and injected into every replica. Without this, logins break when requests alternate between pods.
- **MySQL stays at 1 replica** and is exposed only through a ClusterIP Service.
- **Install on 1 replica first, then scale to 2.** This avoids both pods racing to copy WordPress files into an empty volume.

## Environment

- 3-node kubeadm cluster (1 control plane, 2 workers) on Vagrant VMs
- Flannel (VXLAN) CNI
- NFS server on the control plane
- Recommended: at least 2 GB RAM per VM (3 GB or more if running both variants)

## Repository structure

```
k8s-wordpress-lab/
├── README.md
├── 01-apache-wordpress/
│   ├── storage.yaml
│   ├── mysql.yaml
│   └── wordpress.yaml
├── 02-nginx-fpm-wordpress/
│   ├── wpn-storage.yaml
│   ├── wpn-mysql.yaml
│   ├── wpn-nginx-config.yaml
│   └── wpn-wordpress.yaml
├── docs/
│   └── troubleshooting.md
└── .gitignore
```

## Prerequisites

1. All nodes are `Ready`:
   ```bash
   kubectl get nodes
   ```
2. Flannel registered the correct node IPs (must not be the Vagrant NAT address `10.0.2.15`):
   ```bash
   kubectl describe node <node> | grep public-ip
   ```
3. Pods on different nodes can reach each other.
4. NFS server is installed on the control plane and `nfs-common` is installed on **all** nodes.

## Setup

### 1. NFS exports (control plane)

```bash
sudo apt install -y nfs-kernel-server

# Variant 1
sudo mkdir -p /srv/nfs/wp /srv/nfs/mysql

# Variant 2
sudo mkdir -p /srv/nfs/wpn-wp /srv/nfs/wpn-mysql

sudo chmod 777 /srv/nfs/*
```

Add one line per folder to `/etc/exports` (adjust the subnet to your network):

```
/srv/nfs/wp         192.168.2.0/24(rw,async,no_subtree_check,no_root_squash)
/srv/nfs/mysql      192.168.2.0/24(rw,async,no_subtree_check,no_root_squash)
/srv/nfs/wpn-wp     192.168.2.0/24(rw,async,no_subtree_check,no_root_squash)
/srv/nfs/wpn-mysql  192.168.2.0/24(rw,async,no_subtree_check,no_root_squash)
```

```bash
sudo exportfs -ra
```

On every worker node:

```bash
sudo apt install -y nfs-common
showmount -e <nfs-server-ip>
```

> This is a lab configuration (`no_root_squash`, `chmod 777`). Do not use it in production.

### 2. Secrets

Secrets are **not** stored in this repository. Create them directly in the cluster. Replace the placeholders with your own values.

```bash
kubectl create secret generic mysql-secret \
  --from-literal=root-password='<root-password>' \
  --from-literal=password='<wordpress-db-password>'

kubectl create secret generic wp-salts \
  --from-literal=AUTH_KEY=$(openssl rand -hex 32) \
  --from-literal=SECURE_AUTH_KEY=$(openssl rand -hex 32) \
  --from-literal=LOGGED_IN_KEY=$(openssl rand -hex 32) \
  --from-literal=NONCE_KEY=$(openssl rand -hex 32) \
  --from-literal=AUTH_SALT=$(openssl rand -hex 32) \
  --from-literal=SECURE_AUTH_SALT=$(openssl rand -hex 32) \
  --from-literal=LOGGED_IN_SALT=$(openssl rand -hex 32) \
  --from-literal=NONCE_SALT=$(openssl rand -hex 32)
```

For Variant 2, create the same two Secrets under the names `wpn-mysql-secret` and `wpn-salts`.

### 3. Deploy Variant 1 (Apache)

```bash
cd 01-apache-wordpress
kubectl apply -f storage.yaml
kubectl get pv,pvc                 # both claims must be Bound

kubectl apply -f mysql.yaml
kubectl get pods -l app=mysql -w   # wait for 1/1 Running

kubectl apply -f wordpress.yaml    # starts with 1 replica
kubectl get pods -l app=wordpress -w
```

### 4. Deploy Variant 2 (nginx + PHP-FPM)

```bash
cd 02-nginx-fpm-wordpress
kubectl apply -f wpn-storage.yaml
kubectl apply -f wpn-mysql.yaml
kubectl apply -f wpn-nginx-config.yaml
kubectl apply -f wpn-wordpress.yaml    # starts with 1 replica
kubectl get pods -l app=wpn-wordpress -w   # wait for 2/2 Running
```

Each pod in this variant runs two containers (`php-fpm` and `nginx`) that share the same NFS volume. nginx serves static files and forwards `.php` requests to PHP-FPM on `127.0.0.1:9000`.

### 5. Install WordPress, then scale

1. Open the site through **one** node IP and the NodePort, for example `http://<node-ip>:30009` (Variant 1) or `http://<node-ip>:30010` (Variant 2).
2. Complete the WordPress installer.
3. Scale to 2 replicas:
   ```bash
   kubectl scale deploy wordpress --replicas=2        # Variant 1
   kubectl scale deploy wpn-wordpress --replicas=2    # Variant 2
   ```
4. Confirm both pods are ready and registered:
   ```bash
   kubectl get pods -o wide
   kubectl get endpointslices
   ```

> WordPress stores the site URL from the first visit. Always use the same URL, or put an Ingress or load balancer in front.

## Verification

- Open the site and `/wp-admin` through each node IP and refresh several times. Staying logged in confirms the salts are shared.
- Upload an image in **Media** and refresh repeatedly. It should always be visible, which confirms the storage is shared.
- Delete one WordPress pod. The site should stay available while it is recreated.
- Connect to MySQL from the terminal:
  ```bash
  kubectl exec -it deploy/mysql -- mysql -uroot -p
  ```

## Possible next steps

- Add an Ingress controller and TLS
- Replace the single MySQL instance with a replicated or managed database
- Add automated backups of the NFS folders and the database
- Use a dynamic NFS provisioner instead of static PVs
- Add monitoring (Prometheus and Grafana)
