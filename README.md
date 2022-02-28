# Documentation

## Quick note
This repo was templated from https://github.com/k8s-at-home/template-cluster-k3s and reduced/modified to match my own needs.
## Cluster components

The following components will be installed in the [k3s](https://k3s.io/) cluster by default. 

- [cert-manager](https://cert-manager.io/) - SSL selfsigned certificates
- [calico](https://www.tigera.io/project-calico/) - CNI (container network interface)
- [flux](https://toolkit.fluxcd.io/) - GitOps tool for deploying manifests from the `cluster` directory
- [hajimari](https://github.com/toboshii/hajimari) - start page with ingress discovery
- [kube-vip](https://kube-vip.io/) - layer 2 load balancer for the Kubernetes control plane
- [local-path-provisioner](https://github.com/rancher/local-path-provisioner) - local storage class provided by k3s
- [nfs-subdir-external-provisioner](https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/) - nfs default storage class provided by k3s (existing NFS server must be setup beforehand)
- [metallb](https://metallb.universe.tf/) - bare metal load balancer
- [reloader](https://github.com/stakater/Reloader) - restart pods when Kubernetes `configmap` or `secret` changes
- [system-upgrade-controller](https://github.com/rancher/system-upgrade-controller) - upgrade k3s
- [traefik](https://traefik.io) - ingress controller
- [prometheus-kube-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) - prometheus operator with alertmanager
- [blackbox-exporter](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-blackbox-exporter) - Exporter for prometheus to monitor HTTP/ICMP endpoints
- [alertmanager-discord](https://github.com/benjojo/alertmanager-discord) - Allows alerting to discord
- [grafana](https://github.com/grafana/helm-charts/tree/main/charts/grafana) Webdashboard to visualize prometheus metrics

For provisioning the following tools will be used:

- [Ubuntu](https://ubuntu.com/download/server) - this is a pretty universal operating system that supports running all kinds of home related workloads in Kubernetes
- [Ansible](https://www.ansible.com) - this will be used to provision the Ubuntu operating system to be ready for Kubernetes and also to install k3s

## Prerequisites

### Systems

- One or more raspberry pis with a fresh install of [Ubuntu Server 20.04](https://ubuntu.com/download/server).
- Some experience in debugging problems and a positive attitude ;)

### Tools

Tools that needs to be installed on the local workstation.

| Tool                                                   | Purpose                                                                                                                                 |
|--------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| [ansible](https://www.ansible.com)                     | Preparing Ubuntu for Kubernetes and installing k3s                                                                                      |
| [direnv](https://github.com/direnv/direnv)             | Exports env vars based on present working directory                                                                                     |
| [flux](https://toolkit.fluxcd.io/)                     | Operator that manages your k8s cluster based on your Git repository                                                                     |
| [age](https://github.com/FiloSottile/age)              | A simple, modern and secure encryption tool (and Go library) with small explicit keys, no config options, and UNIX-style composability. |
| [go-task](https://github.com/go-task/task)             | A task runner / simpler Make alternative written in Go (snap install task --classic)                                                    |
| [ipcalc](http://jodies.de/ipcalc)                      | Used to verify settings in the configure script                                                                                         |
| [jq](https://stedolan.github.io/jq/)                   | Used to verify settings in the configure script                                                                                         |
| [kubectl](https://kubernetes.io/docs/tasks/tools/)     | Allows you to run commands against Kubernetes clusters                                                                                  |
| [sops](https://github.com/mozilla/sops)                | Encrypts k8s secrets with Age                                                                                                           |
| [helm](https://helm.sh/)                               | Manage Kubernetes applications                                                                                                          |
| [kustomize](https://kustomize.io/)                     | Template-free way to customize application configuration                                                                                |
| [pre-commit](https://github.com/pre-commit/pre-commit) | Runs checks pre `git commit`                                                                                                            |
| [prettier](https://github.com/prettier/prettier)       | Prettier is an opinionated code formatter.                                                                                              |

#### pre-commit

It is advisable to install [pre-commit](https://pre-commit.com/) and the pre-commit hooks that come with this repository.
[sops-pre-commit](https://github.com/k8s-at-home/sops-pre-commit) will check to make sure you are not by accident committing your secrets un-encrypted.

After pre-commit is installed on your machine run:

```sh
pre-commit install-hooks
```

## Repository structure

The Git repository contains the following directories under `cluster` and are ordered below by how Flux will apply them.

- **base** directory is the entrypoint to Flux
- **crds** directory contains custom resource definitions (CRDs) that need to exist globally in your cluster before anything else exists
- **core** directory (depends on **crds**) are important infrastructure applications (grouped by namespace) that should never be pruned by Flux
- **apps** directory (depends on **core**) is where your common applications (grouped by namespace) could be placed, Flux will prune resources here if they are not tracked by Git anymore

```
cluster
├── apps
│   ├── default
|   ├── kube-system
|   ├── monitoring
│   ├── networking
│   └── system-upgrade
├── base
│   └── flux-system
├── core
│   ├── cert-manager
│   ├── metallb-system
│   ├── namespaces
|   ├── nfs-subdir-external-provisioner
│   └── system-upgrade
└── crds
    ├── traefik
    └── cert-manager
```

## Installation

### Setting up Age

Create a Age Private and Public key for encrypting and decrypting secrets.

1. Create a Age Private / Public Key

```sh
age-keygen -o age.agekey
```

2. Set up the directory for the Age key and move the Age file to it

```sh
mkdir -p ~/.config/sops/age
mv age.agekey ~/.config/sops/age/keys.txt
```

3. Export the `SOPS_AGE_KEY_FILE` in `zshrc` and source it

```sh
echo "export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt" >> ~/.zshrc
source ~/.zshrc
```

4. Fill out the Age public key in the `.config.env` under `BOOTSTRAP_AGE_PUBLIC_KEY`, **note** the public key should start with `age`...

### Configuration

The `.config.env` file contains necessary configuration files that are needed by Ansible and Flux.

1. Start filling out all the environment variables. **All are required** and read the comments they will explain further what is required.

2. Copy tmpl folder from [k8s-at-home/template-cluster-k3s](https://github.com/k8s-at-home/template-cluster-k3s) into the root directory of that repository

3. Verify the configuration is correct:
 ```sh
 ./configure.sh --verify
 ````
(If ssh test failed, make sure the ssh key is copied onto the servers)

4. Run `./configure.sh` to start having the script wire up the templated files and place them where they need to be.

### Preparing Ubuntu with Ansible

(Nodes are not security hardened by default, you can do this with [dev-sec/ansible-collection-hardening](https://github.com/dev-sec/ansible-collection-hardening) or something similar.)

1. Install the deps by running `task ansible:deps`

2. Verify Ansible can view your config by running `task ansible:list`

3. Verify Ansible can ping your nodes by running `task ansible:adhoc:ping`

4. Finally, run the Ubuntu Prepare playbook by running `task ansible:playbook:ubuntu-prepare`

5. If everything goes as planned you should see Ansible running the Ubuntu Prepare Playbook against your nodes.

6. Must be done manually for now. Open /boot/firmware/cmdline.txt and add `cgroup_enable=memory cgroup_memory=1` to all raspberry pis. Reboot afterwards.
https://rancher.com/docs/k3s/latest/en/advanced/#enabling-cgroups-for-raspbian-buster

### Installing k3s with Ansible

:round_pushpin: Here we will be running a Ansible Playbook to install [k3s](https://k3s.io/) with [this](https://galaxy.ansible.com/xanmanning/k3s) wonderful k3s Ansible galaxy role. After completion, Ansible will drop a `kubeconfig` in `./provision/kubeconfig` for use with interacting with your cluster with `kubectl`. Copy kubeconfig to ~/.kube/

1. Verify Ansible can view your config by running `task ansible:list`

2. Verify Ansible can ping your nodes by running `task ansible:adhoc:ping`

3. Run the k3s install playbook by running `task ansible:playbook:k3s-install`

4. Verify the nodes are online
   
```sh
kubectl --kubeconfig=./provision/kubeconfig get nodes
# NAME           STATUS   ROLES                       AGE     VERSION
# k8s-0          Ready    control-plane,master      4d20h   v1.21.5+k3s1
# k8s-1          Ready    <none>                    4d20h   v1.21.5+k3s1
# k8s-2          Ready    <none>                    4d20h   v1.21.5+k3s1
```
5. Optionally label the workers so their role is displayed correctly
```
kubectl --kubeconfig=./provision/kubeconfig label node k8s-1 node-role.kubernetes.io/worker=''
kubectl --kubeconfig=./provision/kubeconfig label node k8s-2 node-role.kubernetes.io/worker=''
kubectl --kubeconfig=./provision/kubeconfig get nodes
# NAME           STATUS   ROLES                       AGE     VERSION
# k8s-0          Ready    control-plane,master      4d20h   v1.21.5+k3s1
# k8s-1          Ready    worker                    4d20h   v1.21.5+k3s1
# k8s-2          Ready    worker                    4d20h   v1.21.5+k3s1
```
### GitOps with Flux

1. Verify Flux can be installed

```sh
flux --kubeconfig=./provision/kubeconfig check --pre
# ► checking prerequisites
# ✔ kubectl 1.21.5 >=1.18.0-0
# ✔ Kubernetes 1.21.5+k3s1 >=1.16.0-0
# ✔ prerequisites checks passed
```

2. Pre-create the `flux-system` namespace

```sh
kubectl --kubeconfig=./provision/kubeconfig create namespace flux-system --dry-run=client -o yaml | kubectl --kubeconfig=./provision/kubeconfig apply -f -
```

3. Add the Age key in-order for Flux to decrypt SOPS secrets

```sh
cat ~/.config/sops/age/keys.txt |
    kubectl --kubeconfig=./provision/kubeconfig -n flux-system create secret generic sops-age \
    --from-file=age.agekey=/dev/stdin
```

4. **Verify** all the above files are **encrypted** with SOPS
Commands for encryption / decryption:
```sh
# Decrypt secrets
sops --decrypt cluster/base/cluster-secrets.sops.yaml > cluster/base/cluster-secrets.yaml     
# Encrypt secrets 
sops --encrypt cluster/base/cluster-secrets.yaml > cluster/base/cluster-secrets.sops.yaml
```

6.  Push you changes to git

```sh
git add -A
git commit -m "initial commit"
git push
```

7. Install Flux
Due to race conditions with the Flux CRDs you will have to run the below command twice. There should be no errors on this second run.

```sh
# Optional:
cp ./provision/kubeconfig ~/.kube/config && chmod 0600 ~/.kube/config

kubectl --kubeconfig=./provision/kubeconfig apply --kustomize=./cluster/base/flux-system
# namespace/flux-system configured
# customresourcedefinition.apiextensions.k8s.io/alerts.notification.toolkit.fluxcd.io created
# ...
# unable to recognize "./cluster/base/flux-system": no matches for kind "Kustomization" in version "kustomize.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "GitRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
```

Workaround for error `error GitRepository/flux-system.flux-system - Reconciler error auth secret error: Secret &#34;flux-system&#34; not found`
```sh
export GITHUB_USER=<GITHUB_USER>
export GITHUB_TOKEN=<GITHUB_TOKEN>
flux bootstrap github --owner=$GITHUB_USER --repository=k3s-rpi-gitops --branch=main --path=./cluster/base --personal --private=false
```

8. Verify Flux components are running in the cluster

```sh
kubectl --kubeconfig=./provision/kubeconfig get pods -n flux-system
# NAME                                       READY   STATUS    RESTARTS   AGE
# helm-controller-5bbd94c75-89sb4            1/1     Running   0          1h
# kustomize-controller-7b67b6b77d-nqc67      1/1     Running   0          1h
# notification-controller-7c46575844-k4bvr   1/1     Running   0          1h
# source-controller-7d6875bcb4-zqw9f         1/1     Running   0          1h
```
## Post installation

### helpful commands
 **TODO**
```sh
# Force reconciliation
flux reconcile helmrelease <HELMRELEASE> -n <NAMESPACE>
flux reconcile kustomization apps

# Get statuses of flux resources
flux get all
flux get helmrelease -A

# Follow flux logs
flux logs --level=error
flux logs --follow

# Monitor helm-controller
kubectl get pods -n flux-system
klf -n flux-system helm-controller-55896d6ccf-d9w8p
# klf -n flux-system $(kubectl get pods -n flux-system | grep -E 'helm-controller.*Running' | cut -d ' ' -f1)

# Fix message: "Helm upgrade failed: another operation (install/upgrade/rollback) is in progress"
helm delete <HELMRELEASE> -n <NAMESPACE>
flux reconcile helmrelease <HELMRELEASE> -n <NAMESPACE>

flux delete helmrelease <HELMRELEASE> -n <NAMESPACE>
flux reconcile source helm <HELMRELEASE>

# Fix error message (re-create kustomization): 
# ✗ Kustomization reconciliation failed: Secret/flux-system/cluster-secrets is SOPS encryted, configuring decryption is required for this secret to be reconciled
flux create kustomization cluster-secrets --source=flux-system --path=./cluster/base --prune=true --interval=10m --decryption-provider=sops --decryption-secret=sops-age
```
