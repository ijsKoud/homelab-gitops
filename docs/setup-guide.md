# Homelab K3s Setup Guide

Last updated Oct 19, 2026 · @Daan · mirrors the files in this repository

## Overview

Four Pi 5's (16 GB RAM, 512 GB NVMe each) become one K3s cluster: rpi-1, rpi-2 and rpi-3 as servers, rpi-4 as agent. Apps are deployed from GitHub by Argo CD and reachable at `<app>.k3s.home.klrnbk.nl` no matter which Pi runs them. The NAS keeps Jellyfin and takes over the download stack; game servers stay on your external server.

```
LAN browser      -> bind9 (*.k3s.home -> 192.168.2.20) -> Traefik -> Ingress -> Service -> Pod (any Pi)
Internet user    -> Cloudflare (auth.klrnbk.nl, panel.klrnbk.nl) -> tunnel -> cloudflared pods -> Service
```

### What runs where

| Where | What |
| --- | --- |
| K3s cluster | Traefik, cert-manager, MetalLB, Longhorn, CNPG, Argo CD, Phase (self-hosted) + operator, Reloader, bind9, Authentik, cloudflared, Outline, ConvertX, draw.io, withoutBG, Calagopus panel, Radarr, Sonarr, Profilarr, Seerr, the Appiecal fleet and your other apps |
| NAS (UGREEN, Docker) | Jellyfin, Gluetun + qBittorrent + Prowlarr, Garage (S3), restic to the Storage Box, NFS shares |
| External server | Calagopus Wings and the game servers |
| Hetzner Storage Box | Offsite restic snapshots (no media) |

### IP plan

Addresses below `.50` are outside the Deco DHCP pool, so nothing else will ever take them.

| IP | What | Notes |
| --- | --- | --- |
| 192.168.2.10 | K3s API VIP (kube-vip) | `k3s.home.klrnbk.nl` |
| 192.168.2.20 | Traefik (MetalLB) | All web traffic |
| 192.168.2.21 | bind9 (MetalLB) | DNS, port 53 TCP/UDP |
| 192.168.2.22 - .29 | MetalLB spare | Future TCP/UDP services |
| 192.168.2.100 | rpi-1 | K3s server, `--cluster-init` |
| 192.168.2.101 | rpi-2 | K3s server |
| 192.168.2.102 | rpi-3 | K3s server |
| 192.168.2.103 | rpi-4 | K3s agent |
| 192.168.2.104 | NAS | Jellyfin, downloads, NFS, Garage |

Make sure the Pi's and NAS have **address reservations** in the Deco app (More > Advanced > Address Reservation).

### Hostnames

`*.home.klrnbk.nl` only keeps host records for SSH (`rpi-1.home` to `rpi-4.home`, `nas.home`) plus `auth.home`. All apps live under `*.k3s.home.klrnbk.nl`; old app URLs get updated by hand.

| App | Hostname(s) | Reachable from |
| --- | --- | --- |
| Authentik | `auth.klrnbk.nl`, `auth.home.klrnbk.nl`, `auth.k3s.home.klrnbk.nl` | Internet (tunnel) + LAN (direct) |
| Calagopus panel | `panel.klrnbk.nl`, `panel.k3s.home.klrnbk.nl` | Internet (tunnel) + LAN |
| Phase | `phase.k3s.home.klrnbk.nl` | LAN only |
| Outline | `outline.k3s.home.klrnbk.nl` | LAN |
| ConvertX | `convert.k3s.home.klrnbk.nl` | LAN |
| draw.io | `draw.k3s.home.klrnbk.nl` | LAN |
| withoutBG | `removebg.k3s.home.klrnbk.nl` | LAN |
| Radarr / Sonarr / Profilarr / Seerr | `radarr.`, `sonarr.`, `profilarr.`, `seerr.k3s.home.klrnbk.nl` | LAN |
| Longhorn UI | `longhorn.k3s.home.klrnbk.nl` | LAN |
| qBittorrent / Prowlarr | `192.168.2.104:8080` / `:9696` | LAN |

## Part 0: Migration plan

K3s gets installed next to Docker on the same Pi's, apps move over one at a time, and each Docker container is switched off only after its replacement works. Nothing goes down for longer than a single app's cutover, and every step can be rolled back by starting the old container again.

### Why Docker and K3s can share a Pi

K3s brings its own container runtime (containerd), separate from Docker's. Cluster Traefik answers on the MetalLB IP 192.168.2.20, not on the Pi's own IPs where your Docker Traefik listens, so ports don't clash. With 16 GB per Pi there's room for both during the migration. Before installing, check nothing in Docker publishes the ports K3s needs:

```bash
sudo ss -tulpn | grep -E ':(6443|2379|2380|10250|8472|9500|7946) '
```

The output should be empty. Those are the K3s API, etcd, kubelet, Flannel VXLAN, Longhorn and MetalLB.

### Phase A: prepare

1. **Back up what runs today**: `pg_dump` every Docker Postgres (Authentik, Calagopus, Appiecal, others) and copy Docker volumes to the NAS. These dumps are also what you import later.
2. **Move the download stack to the NAS** (Part 11c): Gluetun, qBittorrent and Prowlarr. Stop them on rpi-4 once downloads work from the NAS.
3. **Empty rpi-3** and remove its old k3s with `/usr/local/bin/k3s-uninstall.sh`.

### Phase B: build the cluster

4. Parts 1 and 2 on all four Pi's: K3s servers on rpi-1, rpi-2, rpi-3, agent on rpi-4. Docker keeps running.
5. Parts 3, 4, 5 and 5b: Argo CD, Traefik, certificates, Phase, Reloader, storage, CNPG.
6. Part 6: bind9 in the cluster on 192.168.2.21. Test it, switch the Deco's DNS to 192.168.2.21 with the old bind9 on rpi-1 as secondary, and stop the old one after a week without problems.

### Phase C: move apps, one at a time

Recommended order, from foundation to leaves:

1. **Authentik** + Cloudflare Tunnel (Parts 7, 7b): everything else logs in through it
2. **Calagopus panel** (Part 11b), from rpi-2
3. **ConvertX, Outline, draw.io, withoutBG** (Parts 8 to 11), ConvertX from rpi-1
4. **Radarr, Sonarr, Profilarr, Seerr** (Part 11c)
5. **The Appiecal fleet** and your other own apps (Part 12)

The same recipe for every app:

1. Deploy it in the cluster through Argo CD at its `*.k3s.home.klrnbk.nl` name.
2. Stop the Docker container (so no new data is written), then import its data: `pg_dump` into CNPG, or copy files into its volume (`kubectl cp`, or a one-off pod that mounts the PVC).
3. Test it at the new URL, then update anything that points at the old URL (bookmarks, OAuth redirect URLs in Authentik, other apps' configs).
4. Leave the old container stopped but not deleted for a week; that's your rollback.

### Phase D: remove Docker

When a Pi has no containers left: stop the old Traefik and bind9, run `docker system prune -a` to reclaim disk, and optionally `sudo apt purge docker-ce docker-ce-cli containerd.io` (K3s uses its own containerd, so this doesn't affect the cluster). Finally, check that no `*.home` records point at old services anymore.

## Part 1: Prepare the Pi's

The Pi's already boot from NVMe and run Docker, so there's no reflashing: K3s installs on top of the existing OS. Run this on every Pi.

1. **Hostnames**: make sure they're `rpi-1` to `rpi-4` (`hostnamectl`), matching the `*.home` records.

   *Why:* K3s registers each node under its hostname, and that name shows up everywhere (`kubectl get nodes`, Longhorn, logs). Matching the DNS names avoids confusion.
2. **Enable memory cgroups** if they aren't yet. Append to the single line in `/boot/firmware/cmdline.txt`:

   ```
   cgroup_memory=1 cgroup_enable=memory
   ```

   *Why:* cgroups are how Linux enforces a container's memory limit. Raspberry Pi OS ships with the memory controller disabled, and K3s refuses to start without it. Docker may already have needed this, so it might be set.
3. **Install what Longhorn and NFS need:**

   ```bash
   sudo apt update && sudo apt full-upgrade -y
   sudo apt install -y open-iscsi nfs-common curl
   sudo systemctl enable --now iscsid
   ```

   *Why:* Longhorn attaches its volumes to pods over iSCSI (`open-iscsi`); without it Longhorn volumes can't mount. `nfs-common` is needed to mount NFS shares from the NAS (the media share and the Longhorn backup target). Add `cryptsetup` only if you want encrypted Longhorn volumes.
4. **Disable swap:**

   ```bash
   sudo systemctl disable --now dphys-swapfile 2>/dev/null || true
   sudo swapoff -a
   ```

   *Why:* Kubernetes places pods by their memory requests and kills a pod that exceeds its limit. With swap, a leaking app gets pushed to disk instead, and the whole Pi slows down rather than one pod restarting cleanly. etcd is also latency-sensitive: if it gets swapped out on a server node it misses heartbeats, which can trigger leader elections or make the node look dead. With 16 GB RAM swap adds little, and as a bonus there's less NVMe wear and secrets in memory never land on disk. (Recent Kubernetes versions can run with swap, but it's opt-in and needs extra kubelet config; no swap is simpler.)
5. **Keep upstream DNS on the Pi's themselves.** Point them at the Deco or a public resolver, not at 192.168.2.21.

   *Why:* if the Pi's depended on bind9 and the cluster was down, they couldn't resolve `ghcr.io` to pull the images needed to bring the cluster (and bind9) back up. Pods still resolve your internal names through CoreDNS (Part 6).
6. **Check free disk space**: `df -h /`, and keep at least 150 GB free.

   *Why:* during the migration Longhorn volumes, K3s images and Docker images all share the same 512 GB NVMe.
7. **Reboot** and check: `cat /proc/cgroups | grep memory` shows `1` in the last column.

   *Why:* the cgroup change only takes effect after a reboot; this confirms it did.

On rpi-3, remove the old k3s first (Part 0, Phase A): `/usr/local/bin/k3s-uninstall.sh`.

## Part 2: Install K3s

rpi-1 starts the cluster, rpi-2 and rpi-3 join as servers through the VIP, rpi-4 joins as an agent. Docker keeps running on all of them during the migration. Generate one shared token first and keep it in your password manager:

```bash
openssl rand -hex 32
```

### rpi-1: first server

Create `/etc/rancher/k3s/config.yaml`:

```yaml
token: <your-token>
cluster-init: true
tls-san:
  - 192.168.2.10
  - k3s.home.klrnbk.nl
disable:
  - traefik
  - servicelb
write-kubeconfig-mode: "0644"
```

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=stable sh -s - server
```

### rpi-1: kube-vip for the API VIP

k3s auto-applies anything in its manifests folder. This makes 192.168.2.10 float between the three servers. Note there is no `--services` flag: MetalLB handles app IPs, kube-vip only the API.

```bash
export VIP=192.168.2.10 INTERFACE=eth0
KV=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases/latest | grep -m1 tag_name | cut -d'"' -f4)

curl -s https://kube-vip.io/manifests/rbac.yaml | sudo tee /var/lib/rancher/k3s/server/manifests/kube-vip-rbac.yaml
sudo k3s ctr image pull ghcr.io/kube-vip/kube-vip:$KV
sudo k3s ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KV vip /kube-vip manifest daemonset \
  --interface $INTERFACE --address $VIP --inCluster --taint \
  --controlplane --arp --leaderElection \
  | sudo tee /var/lib/rancher/k3s/server/manifests/kube-vip.yaml
```

Check with `ping 192.168.2.10` from your laptop.

### rpi-2 and rpi-3: extra servers

`/etc/rancher/k3s/config.yaml`:

```yaml
token: <your-token>
server: https://192.168.2.10:6443
tls-san:
  - 192.168.2.10
  - k3s.home.klrnbk.nl
disable:
  - traefik
  - servicelb
```

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=stable sh -s - server
```

### rpi-4: agent

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=stable \
  K3S_URL=https://192.168.2.10:6443 K3S_TOKEN=<your-token> sh -s - agent
```

### kubectl on your laptop

```bash
scp rpi-1.home.klrnbk.nl:/etc/rancher/k3s/k3s.yaml ~/.kube/config
sed -i '' 's/127.0.0.1/192.168.2.10/' ~/.kube/config   # drop the '' on Linux
kubectl get nodes   # expect 4 nodes, all Ready
```

Add `k3s.home IN A 192.168.2.10` to bind9 if you want to use the hostname instead.

## Part 3: GitOps repo and Argo CD

From here on you don't run `kubectl apply` by hand. You commit YAML to a GitHub repo, and **Argo CD**, running inside the cluster, keeps the cluster in sync with it. Every component is an Argo CD **Application**: a pointer to "this Helm chart" or "this folder in Git", which you can see, diff and sync in the web UI.

```
Your laptop --git push--> GitHub (homelab-gitops) <--polls-- Argo CD (in the cluster) --applies--> K3s
```

You do everything in this part from your **laptop**. It takes about 45 minutes.

### How the pieces fit

- **Root app ("app of apps")**: one Application you apply by hand, once. It watches `clusters/pi/` **including its subfolders** (`directory.recurse`) and creates every Application it finds there. The subfolders (`core/`, `storage/`, `secrets/`, `platform/`) are only for you; the order comes from sync waves, not folders.
- **One Application per component** in a subfolder of `clusters/pi/`: Helm charts (MetalLB, Traefik, Authentik, ...) point at a chart; your own YAML points at a folder in the repo.
- **One Argo CD Project per project**: `apps/<project>/<app>/`. Each project folder (`documentation`, `media`, `platform`, later `appiecal`, ...) automatically gets its own Argo CD Project and namespace, and every app folder inside it becomes its own Application. The Appiecal microservices are four Applications in project `appiecal`. New app = new folder; new project = new folder. Nothing else to edit.
- **Sync waves** set the order. Lower waves are synced and healthy before higher ones start:

```
wave -4  MetalLB, and the Projects (one per folder in apps/)
wave -3  Argo CD itself, and the MetalLB IP pool
wave -2  controllers: cert-manager, Traefik, Longhorn, NFS driver, CNPG, Reloader, Phase operator
wave -1  infra-configs: ClusterIssuer, certificate, CoreDNS config
wave  0  Phase's database, Redis and route (phase-deps)
wave  1  Phase itself, and Authentik
wave  2  all other apps (one ApplicationSet over apps/<project>/<app>)
```

The order matters because some things only work once something else exists: an `IPAddressPool` is a MetalLB resource, so MetalLB must be installed first.

### Step 1: Install the tools on your laptop

| Tool | What it's for |
| --- | --- |
| `kubectl` | Talk to the cluster (set up at the end of Part 2) |
| `helm` | Install Argo CD the first time |
| `argocd` | Optional CLI, handy for quick checks |

```bash
# macOS
brew install helm argocd

# Linux / WSL
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
# argocd: download the binary from https://github.com/argoproj/argo-cd/releases
```

### Step 2: Create the repo and a deploy key

1. On GitHub, create a **private** repo `ijskoud/homelab-gitops` (empty, with a README so `main` exists). Clone it and create the folders:

   ```bash
   git clone git@github.com:ijskoud/homelab-gitops.git
   cd homelab-gitops
   mkdir -p bootstrap charts/app-project/templates clusters/pi/{core,storage,secrets,platform} \
     infrastructure/{argocd,configs,metallb,phase} apps docs
   ```
2. Create an SSH key for Argo CD. It only needs **read** access: Argo CD pulls from the repo, it never writes to it.

   ```bash
   ssh-keygen -t ed25519 -N "" -C argocd -f ~/argocd-deploy-key
   ```

   On GitHub: repo **Settings > Deploy keys > Add deploy key**, paste `~/argocd-deploy-key.pub`, and leave **Allow write access** unticked. Store the private key in your password manager.

The repo will end up like this:

```
homelab-gitops/              # contains no secrets, not even encrypted ones
├── bootstrap/
│   └── root.yaml            # the root app; applied by hand once
├── charts/
│   └── app-project/         # small Helm chart: the standard Argo CD Project
├── clusters/pi/             # Applications only; the root app reads every subfolder
│   ├── core/                # argocd, metallb, metallb-config, cert-manager, traefik, reloader
│   ├── storage/             # longhorn, csi-driver-nfs, cnpg, cnpg-barman
│   ├── secrets/             # phase-operator, phase (phase-deps + phase)
│   ├── platform/            # authentik, tailscale-operator
│   ├── infra-configs.yaml   # Application for infrastructure/configs
│   ├── projects.yaml        # ApplicationSet: one Argo CD Project per folder in apps/
│   └── apps.yaml            # ApplicationSet: one Application per apps/<project>/<app>
├── infrastructure/          # plain YAML the Applications above point at
│   ├── argocd/values.yaml   # Argo CD's own settings
│   ├── configs/             # issuer, certificate, StorageClass, Longhorn extras, Tailscale Service
│   ├── metallb/             # the IP pool
│   └── phase/               # Phase's database, Redis and route
├── apps/                    # <project>/<app>; the project is also the namespace
│   ├── documentation/       # outline, draw-io
│   ├── media/               # convertx, withoutbg
│   └── platform/            # authentik, calagopus, cloudflared
├── docs/setup-guide.md      # this guide
└── .github/renovate.json
```

### Step 3: How secrets work (none in Git)

The repo never contains secrets. They reach the cluster in one of two ways:

| Kind | How it gets into the cluster | Examples |
| --- | --- | --- |
| **Tier 0**: things Phase itself depends on | Created by hand with `kubectl`, once; values kept in your password manager | Argo CD's deploy key, Cloudflare token, Phase's own keys, Authentik's secret key |
| **Everything else** | Synced from Phase by the Phase operator (Part 5b) | Outline, ConvertX, Calagopus, cloudflared, S3 credentials |

Tier 0 is a short, fixed list. Here it is in one place; you run each block when you reach the part that needs it:

```bash
# Part 3 (now): Argo CD's access to the repo
kubectl create namespace argocd
kubectl -n argocd create secret generic repo-homelab-gitops \
  --from-literal=type=git \
  --from-literal=url=git@github.com:ijskoud/homelab-gitops.git \
  --from-file=sshPrivateKey=$HOME/argocd-deploy-key
kubectl -n argocd label secret repo-homelab-gitops argocd.argoproj.io/secret-type=repository

# Part 4: cert-manager's Cloudflare API token (Zone:DNS:Edit on klrnbk.nl)
kubectl create namespace cert-manager
kubectl -n cert-manager create secret generic cloudflare-token --from-literal=token=<cloudflare token>

# Part 5: Longhorn's access to Garage (S3) for volume backups
kubectl create namespace longhorn-system
kubectl -n longhorn-system create secret generic longhorn-s3 \
  --from-literal=AWS_ACCESS_KEY_ID=<garage longhorn key id> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<garage longhorn secret> \
  --from-literal=AWS_ENDPOINTS=http://192.168.2.104:3900

# Part 5b: Phase itself (generate values with: openssl rand -hex 32)
kubectl create namespace phase
kubectl -n phase create secret generic phase-db-credentials --type=kubernetes.io/basic-auth \
  --from-literal=username=phase --from-literal=password=<db-password>
kubectl -n phase create secret generic phase-console-secret \
  --from-literal=SECRET_KEY=<hex32> --from-literal=SERVER_SECRET=<hex32> --from-literal=NEXTAUTH_SECRET=<hex32> \
  --from-literal=DATABASE_PASSWORD=<db-password> --from-literal=REDIS_PASSWORD=<hex32>
kubectl -n phase create secret generic phase-s3-creds \
  --from-literal=ACCESS_KEY_ID=<garage key id> --from-literal=ACCESS_SECRET_KEY=<garage secret> \
  --from-literal=REGION=garage

# Part 5b, once Phase runs: the operator's service token
kubectl create namespace phase-system
kubectl -n phase-system create secret generic phase-service-token --from-literal=token=<pss_service:v2:...>

# Part 14: Tailscale operator OAuth client
kubectl create namespace tailscale
kubectl -n tailscale create secret generic operator-oauth \
  --from-literal=client_id=<oauth client id> --from-literal=client_secret=<tskey-client-...>

# Part 7: Authentik's secret key (the one your Docker install uses now)
kubectl create namespace platform
kubectl -n platform create secret generic authentik-secret \
  --from-literal=AUTHENTIK_SECRET_KEY='<current key>' \
  --from-literal=AUTHENTIK_EMAIL__USERNAME='<smtp username>' \
  --from-literal=AUTHENTIK_EMAIL__PASSWORD='<smtp password>'
```

Keep this list, with the real values filled in, as a **secure note in your password manager**. After a rebuild you paste it back in and the cluster has everything Git doesn't. Two things to know:

- Argo CD doesn't manage these secrets, so it won't delete them either; they simply exist next to what it syncs.
- To change one later, delete and recreate it (`kubectl -n <ns> delete secret <name>`, then the create command). Reloader (Part 5b) restarts whatever uses it.

### Step 4: Install Argo CD

Create `infrastructure/argocd/values.yaml` with this start; the complete file, as it ends up after Part 7, is shown there. It's used for this first install and later by Argo CD to manage itself, so it lives in the repo:

```yaml
configs:
  params:
    server.insecure: true            # Traefik terminates TLS in front of it
  cm:
    # Let the root app wait until each child Application is Healthy (makes sync waves work)
    resource.customizations.health.argoproj.io_Application: |
      hs = {}
      hs.status = "Progressing"
      hs.message = ""
      if obj.status ~= nil and obj.status.health ~= nil then
        hs.status = obj.status.health.status
        if obj.status.health.message ~= nil then
          hs.message = obj.status.health.message
        end
      end
      return hs
```

Create the repo secret (the first block in step 3), then install:

```bash
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm install argocd argo/argo-cd -n argocd -f infrastructure/argocd/values.yaml
helm list -n argocd       # note the CHART version, e.g. argo-cd-9.x.y
```

### Step 5: Open the UI

Until Traefik exists (Part 4), use a port-forward:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:80
```

Open `http://localhost:8080`, log in as `admin` with that password, change it under **User Info**, then delete the secret: `kubectl -n argocd delete secret argocd-initial-admin-secret`. Under **Settings > Repositories** your repo should show **Successful**.

### Step 6: Write the Applications

There are two kinds. A **Helm chart** Application (used in Parts 4, 5 and 5b):

```yaml
# clusters/pi/core/metallb.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metallb
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "-4" }   # before metallb-config (-3), which needs its CRDs
spec:
  project: default
  source:
    repoURL: https://metallb.github.io/metallb
    chart: metallb
    targetRevision: 0.16.1
    helm:
      valuesObject: {}
  destination: { server: https://kubernetes.default.svc, namespace: metallb-system }
  syncPolicy:
    automated: { prune: true, selfHeal: true }   # apply Git automatically, undo manual changes
    syncOptions: [CreateNamespace=true, ServerSideApply=true]
```

And a **folder in Git** Application:

```yaml
# clusters/pi/infra-configs.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: infra-configs
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "-1" }
spec:
  project: default
  source:
    repoURL: git@github.com:ijskoud/homelab-gitops.git
    targetRevision: main
    path: infrastructure/configs
  destination: { server: https://kubernetes.default.svc }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [ServerSideApply=true, SkipDryRunOnMissingResource=true]
```

Argo CD manages **itself** with a chart Application that reads its values from the repo (two sources: the chart, and your repo as `$values`):

```yaml
# clusters/pi/core/argocd.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "-3" }
spec:
  project: default
  sources:
    - repoURL: https://argoproj.github.io/argo-helm
      chart: argo-cd
      targetRevision: 10.10.5
      helm: { valueFiles: [$values/infrastructure/argocd/values.yaml] }
    - repoURL: git@github.com:ijskoud/homelab-gitops.git
      targetRevision: main
      ref: values
  destination: { server: https://kubernetes.default.svc, namespace: argocd }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [ServerSideApply=true]
```

**Projects, created automatically.** Every folder in `apps/` gets a matching Argo CD Project. A Project groups its Applications in the UI (filter by project) and fences them in: by default a project may only deploy into **its own namespace** and may not create cluster-wide resources such as CRDs or ClusterRoles. A mistake in one of the Appiecal services can't touch Authentik or another project.

The Project comes from a tiny Helm chart in the repo:

```yaml
# charts/app-project/Chart.yaml
apiVersion: v2
name: app-project
version: 1.0.0
```

```yaml
# charts/app-project/values.yaml
name: ""
description: ""
sourceRepos: ["*"]
clusterResourceWhitelist:
  - { group: "", kind: Namespace }
```

```yaml
# charts/app-project/templates/appproject.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: {{ .Values.name }}
  namespace: argocd
spec:
  description: {{ .Values.description | default .Values.name | quote }}
  sourceRepos: {{- toYaml .Values.sourceRepos | nindent 4 }}
  destinations:
    - server: https://kubernetes.default.svc
      namespace: {{ .Values.name }}
  clusterResourceWhitelist: {{- toYaml .Values.clusterResourceWhitelist | nindent 4 }}
```

An ApplicationSet renders that chart once per folder in `apps/`. If the folder contains a `project-values.yaml`, those values override the defaults; if not, the defaults apply:

```yaml
# clusters/pi/projects.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: projects
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "-4" }
spec:
  goTemplate: true
  generators:
    - git:
        repoURL: git@github.com:ijskoud/homelab-gitops.git
        revision: main
        directories:
          - path: apps/*
  template:
    metadata:
      name: "project-{{.path.basename}}"
    spec:
      project: default
      source:
        repoURL: git@github.com:ijskoud/homelab-gitops.git
        targetRevision: main
        path: charts/app-project
        helm:
          valuesObject:
            name: "{{.path.basename}}"
          valueFiles: ["../../apps/{{.path.basename}}/project-values.yaml"]
          ignoreMissingValueFiles: true
      destination: { server: https://kubernetes.default.svc, namespace: argocd }
      syncPolicy:
        automated: { prune: true, selfHeal: true }
```

Each project folder has a `project-values.yaml`. All three currently allow every cluster-wide resource (`*`); `platform` needs that for Authentik's chart, the other two would work with the defaults, so you can tighten them later:

```yaml
# apps/platform/project-values.yaml
description: DNS, login, tunnel
clusterResourceWhitelist:
  - { group: "*", kind: "*" }
```

```yaml
# apps/media/project-values.yaml
description: Media
clusterResourceWhitelist:
  - { group: "*", kind: "*" }
```

```yaml
# apps/documentation/project-values.yaml
description: Documentation
clusterResourceWhitelist:
  - { group: "*", kind: "*" }
```

Optionally give any project a `project-values.yaml` with just a `description`, or tighten `sourceRepos` (e.g. `appiecal` to your gitops repo only). In the UI the generated Projects show up as Applications named `project-<name>`.

**One ApplicationSet** turns every `apps/<project>/<app>/` folder into an Application. The first path segment picks the Project and the namespace; the Application is named `<project>-<app>` (e.g. `documentation-outline`, `platform-calagopus`), so names never collide. Authentik is excluded because it's a Helm chart with its own Application (Part 7):

```yaml
# clusters/pi/apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "2" }
spec:
  goTemplate: true
  generators:
    - git:
        repoURL: git@github.com:ijskoud/homelab-gitops.git
        revision: main
        directories:
          - path: apps/*/*
          - path: apps/platform/authentik
            exclude: true
  template:
    metadata:
      name: "{{index .path.segments 1}}-{{.path.basename}}"
    spec:
      project: "{{index .path.segments 1}}"
      source:
        repoURL: git@github.com:ijskoud/homelab-gitops.git
        targetRevision: main
        path: "{{.path.path}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{index .path.segments 1}}"
      syncPolicy:
        automated: { prune: true, selfHeal: true }
        syncOptions: [CreateNamespace=true, ServerSideApply=true]
```

Rules that keep this working:

- **Projects follow the folders.** A new folder in `apps/` gets its Project within a few minutes; right after a push its apps can briefly show "project not found" and then sync on their own. Deleting a project folder removes its apps and then its Project.
- **Apps in the same project share a namespace**, so resource names must be unique within the project (e.g. two apps can't both have a Service called `redis`), and no app ships its own `namespace.yaml`: Argo CD creates the namespace.
- **Services in the same project reach each other by short name** (`http://api`), across projects by `<service>.<project>.svc` (e.g. `radarr.media.svc`).

An app folder **with** a `kustomization.yaml` only deploys the files listed in it: **files that aren't listed are ignored**, the most common reason a new file "does nothing". A folder **without** one (like `withoutbg` and `calagopus`) deploys every YAML file in it.

For now, create `infrastructure/configs/kustomization.yaml` and `infrastructure/phase/kustomization.yaml` with `resources: []`; you fill them in Parts 4 to 6.

### Step 7: Apply the root app

`bootstrap/root.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:ijskoud/homelab-gitops.git
    targetRevision: main
    path: clusters/pi
    directory: { recurse: true } # Applications are grouped in subfolders (core/, storage/, ...)
  destination: { server: https://kubernetes.default.svc, namespace: argocd }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

Commit, push, and apply it, the only `kubectl apply` you'll ever do for this repo (and again only if you change `root.yaml` itself, since nothing manages the root app):

```bash
git add . && git commit -m "Bootstrap Argo CD" && git push
kubectl apply -f bootstrap/root.yaml
```

In the UI you now see `root`, and under it `argocd`, `infra-configs` and `apps`, all **Synced** and **Healthy** (the ApplicationSet has no apps yet). Part 3 is done; the next parts add files to `clusters/pi/` and to the project folders.

### Your workflow from now on

1. Add or change a file; list it in the folder's `kustomization.yaml`. Secrets never go in the repo: put them in Phase (or, for tier 0, `kubectl`).
2. Commit and push.
3. Argo CD notices within about a minute (`timeout.reconciliation: 60s` in its values); click **Refresh** on the app to make it check now. If an app never picks up a change, it's usually stuck in an earlier sync: **Terminate** it, then **Sync**.
4. In the UI, the app's tree shows every resource; **App Diff** shows exactly what differs from Git.

**When something isn't Synced or Healthy:**

| You see | Check |
| --- | --- |
| `OutOfSync` that won't go away | Click the app > **App Diff**. Often a field the cluster fills in; fix the YAML to match |
| `ComparisonError` / `Unknown` | The app's top banner shows the error. Usually a YAML typo, a missing file in `kustomization.yaml`, or a Secret the app expects that doesn't exist yet (check Phase or the tier 0 list) |
| `Degraded` | Click the red resource, then **Events** or **Logs** |
| Waiting forever | A lower sync wave isn't Healthy yet: fix that one first |

From the CLI: `argocd login localhost:8080 --insecure` (or your hostname later), then `argocd app list` and `argocd app get <name>`.

## Part 4: Core infrastructure

Every Helm chart in this guide is an Argo CD Application in a subfolder of `clusters/pi/`, built from the chart template in Part 3, step 6: the chart's repo, name and version go under `source`, its values under `source.helm.valuesObject`, and infrastructure charts get sync wave `-2`. Plain YAML that *uses* those charts goes in `infrastructure/configs/` (wave `-1`); list each file in that folder's `kustomization.yaml`. Pin chart versions and let Renovate bump them (Part 12).

### MetalLB

`clusters/pi/core/metallb.yaml` is exactly the chart example from Part 3, step 6; MetalLB needs no values. It's wave `-4` because the next file needs its CRDs.

The IP pool must exist **before** Traefik: Argo CD only marks Traefik Healthy once its LoadBalancer Service has an IP, and only then moves on to the next wave. So the pool gets its own small Application at wave `-3`, between MetalLB (`-4`) and Traefik (`-2`). It's the folder-in-Git template from Part 3 with `name: metallb-config`, `path: infrastructure/metallb` and sync wave `"-3"`, saved as `clusters/pi/core/metallb-config.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metallb-config
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "-3" }
spec:
  project: default
  source:
    repoURL: git@github.com:ijskoud/homelab-gitops.git
    targetRevision: main
    path: infrastructure/metallb
  destination: { server: https://kubernetes.default.svc }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [ServerSideApply=true]
```

`infrastructure/metallb/pool.yaml` (plus a `kustomization.yaml` listing it):

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata: { name: home, namespace: metallb-system }
spec:
  addresses: ["192.168.2.20-192.168.2.29"]
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata: { name: home, namespace: metallb-system }
```

`infrastructure/metallb/kustomization.yaml`:

```yaml
resources: [pool.yaml]
```

### cert-manager

`clusters/pi/core/cert-manager.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "-2" }
spec:
  project: default
  source:
    repoURL: https://charts.jetstack.io
    chart: cert-manager
    targetRevision: v1.*
    helm:
      valuesObject:
        crds: { enabled: true }
        extraArgs:
          - --dns01-recursive-nameservers-only
          - --dns01-recursive-nameservers=1.1.1.1:53,9.9.9.9:53
  destination: { server: https://kubernetes.default.svc, namespace: cert-manager }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true, ServerSideApply=true]
```

The ClusterIssuer uses the **DNS-01** challenge, because Let's Encrypt can't reach your internal hosts. The example uses Cloudflare; swap the solver for wherever `klrnbk.nl` is hosted ([supported providers](https://cert-manager.io/docs/configuration/acme/dns01/)).

`infrastructure/configs/cluster-issuer.yaml` (listed first in `infrastructure/configs/kustomization.yaml`):

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: { name: letsencrypt }
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: programming@daan.klrnbk.nl
    privateKeySecretRef: { name: letsencrypt-account }
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef: { name: cloudflare-token, key: token }
```

The `cloudflare-token` secret is tier 0: create it by hand with the command in Part 3, step 3 (namespace `cert-manager`, key `token`). Tip: test with the staging server `https://acme-staging-v02.api.letsencrypt.org/directory` first to avoid rate limits.

### Traefik

`clusters/pi/core/traefik.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: traefik
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "-2" }
spec:
  project: default
  source:
    repoURL: https://traefik.github.io/charts
    chart: traefik
    targetRevision: 41.*
    helm:
      valuesObject:
        deployment: { replicas: 2 }
        ingressClass: { enabled: true, isDefaultClass: true }
        providers:
          kubernetesCRD: { allowCrossNamespace: true }
        service:
          type: LoadBalancer
          annotations:
            metallb.io/loadBalancerIPs: 192.168.2.20
        additionalArguments:
        - "--entryPoints.web.http.redirections.entryPoint.to=websecure"
        - "--entryPoints.web.http.redirections.entryPoint.scheme=https"
        - "--entryPoints.web.http.redirections.entryPoint.permanent=true"
  destination: { server: https://kubernetes.default.svc, namespace: traefik }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true, ServerSideApply=true]
```

### Wildcard certificate as Traefik default

`infrastructure/configs/wildcard-cert.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata: { name: wildcard-home, namespace: traefik }
spec:
  secretName: wildcard-home-tls
  dnsNames:
    - "home.klrnbk.nl"
    - "*.home.klrnbk.nl"
    - "*.k3s.home.klrnbk.nl"
    - "auth.klrnbk.nl"
  issuerRef: { name: letsencrypt, kind: ClusterIssuer }
---
apiVersion: traefik.io/v1alpha1
kind: TLSStore
metadata: { name: default, namespace: traefik }
spec:
  defaultCertificate: { secretName: wildcard-home-tls }
```

Check: `kubectl get certificate -n traefik` shows `READY True`, and `kubectl get svc -n traefik` shows external IP 192.168.2.20.

### Argo CD UI on its own hostname

With Traefik and the certificate in place, drop the port-forward. Add to `infrastructure/argocd/values.yaml` and push; Argo CD updates itself:

```yaml
server:
  ingress:
    enabled: true
    ingressClassName: traefik
    hostname: argocd.k3s.home.klrnbk.nl
```

The UI is then at `https://argocd.k3s.home.klrnbk.nl`. Login through Authentik follows in Part 7.

## Part 5: Storage and Postgres

Three storage classes, each for its own job. Always set `storageClassName` explicitly in your apps so nothing lands on the wrong one by accident.

| StorageClass | Backed by | Use for |
| --- | --- | --- |
| `longhorn` | Pi NVMe, 2 replicas | App data (uploads, configs). Survives a Pi dying |
| `local-path` | One Pi's NVMe (k3s built-in) | Postgres only. CNPG replicates itself |
| `nfs-nas` | NAS share | Bulk data, media |

### Step 1: Prepare the NAS

The UGREEN NAS supports **NFSv3** at most. That's fine for app data and media; Longhorn's backups (which require NFSv4) go to Garage S3 instead (step 3). In UGOS (menu names can differ slightly between versions):

1. **Create a shared folder** `k3s-nfs-storage` for app data that lives on the NAS. Your `media` share already exists and is used in Part 11c.
2. **Enable NFS**: Control Panel > File Services > NFS > enable (NFSv3).
3. **Give the Pi's access** on both shares (`k3s-nfs-storage` and `media`): an NFS rule for client `192.168.2.0/24` (the whole subnet, not one Pi: pods can land on any of them), **read/write**, squash set to **no mapping** (`no_root_squash`). Some pods write as root; with squashing on they get "permission denied".
4. **On all four Pi's**, enable the helper NFSv3 needs for file locking:

   ```bash
   sudo systemctl enable --now rpc-statd
   ```

Test from one of the Pi's:

```bash
showmount -e 192.168.2.104           # lists /volume1/k3s-nfs-storage and /volume1/media
sudo mount -t nfs -o nfsvers=3 192.168.2.104:/volume1/k3s-nfs-storage /mnt
sudo touch /mnt/test && ls -l /mnt/test && sudo rm /mnt/test
sudo umount /mnt
```

If the mount says "No such file or directory", use the exact path `showmount -e` prints. If `touch` says "Permission denied", the squash setting isn't on "no mapping" yet.

### Step 2: NFS driver and StorageClass

`clusters/pi/storage/csi-driver-nfs.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: csi-driver-nfs
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "-2" }
spec:
  project: default
  source:
    repoURL: https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
    chart: csi-driver-nfs
    targetRevision: v4.*
  destination: { server: https://kubernetes.default.svc, namespace: kube-system }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [ServerSideApply=true]
```

`infrastructure/configs/nfs-storageclass.yaml` (add it to that folder's `kustomization.yaml`):

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: nfs-nas }
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.2.104
  share: /volume1/k3s-nfs-storage
reclaimPolicy: Retain # deleting a PVC never deletes data on the NAS
volumeBindingMode: Immediate
mountOptions: [nfsvers=3]
```

Check: `kubectl get pods -n kube-system | grep csi-nfs` shows Running pods on every Pi, and `kubectl get storageclass` lists `nfs-nas`.

### Step 3: S3 storage on the NAS (Garage)

Longhorn and CNPG both back up to S3-compatible storage, which Garage provides as a single Docker container on the NAS. Follow the separate **[Garage Setup on the UGREEN NAS](https://claude.ai/code/artifact/7c94c42c-27c7-4a6b-aca3-04c268853ce0)** guide, then come back here.

When you're done, you have:

| What | Value |
| --- | --- |
| S3 endpoint | `http://192.168.2.104:3900`, region `garage` |
| Bucket `cnpg-backups` | with key `cnpg`, for the Postgres template below |
| Bucket `longhorn-backups` | with key `longhorn`, for step 4 |
| Key `offsite-reader` | read-only, for restic/rclone in Part 13 |

Key IDs and secrets go in your password manager; the next steps use them.

The NAS sits next to the Pi's, so also sync the bucket offsite to the Hetzner Storage Box with restic (Part 13). A backup in the same room doesn't survive a fire.

### Step 4: Longhorn

Longhorn backs up to the `longhorn-backups` bucket in Garage. Its credentials are tier 0 (restoring volumes must never depend on Phase), so create them by hand first (also listed in Part 3, step 3):

```bash
kubectl create namespace longhorn-system
kubectl -n longhorn-system create secret generic longhorn-s3 \
  --from-literal=AWS_ACCESS_KEY_ID=<garage longhorn key id> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<garage longhorn secret> \
  --from-literal=AWS_ENDPOINTS=http://192.168.2.104:3900
```

`clusters/pi/storage/longhorn.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: longhorn
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "-2" }
spec:
  project: default
  source:
    repoURL: https://charts.longhorn.io
    chart: longhorn
    targetRevision: 1.13.0
    helm:
      valuesObject:
        preUpgradeChecker: { jobEnabled: false } # its Helm hook job doesn't work under Argo CD
        persistence:
          defaultClass: false
          defaultClassReplicaCount: 2
        defaultSettings:
          defaultReplicaCount: 2
        defaultBackupStore:
          backupTarget: s3://longhorn-backups@garage/ # bucket@region; region = s3_region in garage.toml
          backupTargetCredentialSecret: longhorn-s3
  destination: { server: https://kubernetes.default.svc, namespace: longhorn-system }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true, ServerSideApply=true]
```

Then two files in `infrastructure/configs/` (add both to its `kustomization.yaml`).

`longhorn-backup-job.yaml`: a nightly backup of every Longhorn volume, keeping 7:

```yaml
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata: { name: nightly-backup, namespace: longhorn-system }
spec:
  task: backup
  cron: "0 2 * * *" # 02:00, before the 03:00 database backups and 04:00 restic
  retain: 7
  concurrency: 2
  groups: [default] # "default" = every volume that has no other job assigned
```

`longhorn-ingress.yaml`: the UI at `longhorn.k3s.home.klrnbk.nl`. It has no login of its own, so put it behind Authentik forward auth once Part 7 is done by adding the annotation `traefik.ingress.kubernetes.io/router.middlewares: platform-forward-auth@kubernetescrd`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: longhorn, namespace: longhorn-system }
spec:
  ingressClassName: traefik
  rules:
    - host: longhorn.k3s.home.klrnbk.nl
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: longhorn-frontend, port: { number: 80 } } }
```

Check:

```bash
kubectl -n longhorn-system get pods                 # all Running (takes a few minutes)
kubectl -n longhorn-system get backuptargets        # AVAILABLE true
kubectl get storageclass                            # longhorn listed
```

If the backup target stays unavailable, check its reason with `kubectl -n longhorn-system get backuptarget default -o jsonpath='{.status.conditions}'`. Usual causes: Garage not reachable on `192.168.2.104:3900`, a typo in the `longhorn-s3` secret, or the key lacking access to the bucket. If the URL shown doesn't match `longhorn.yaml` after a push, set it once by hand:

```bash
kubectl -n longhorn-system patch backuptarget default --type merge \
  -p '{"spec":{"backupTargetURL":"s3://longhorn-backups@garage/","credentialSecret":"longhorn-s3"}}'
```

### Step 5: CloudNativePG (the Postgres operator)

CloudNativePG (CNPG) runs Postgres inside the cluster: you describe a database in a small YAML file (step 6), and the operator creates it, keeps a replica on another Pi, fails over automatically and makes backups. It comes in two parts:

- the **operator** itself (chart `cloudnative-pg`)
- the **Barman Cloud plugin** (chart `plugin-barman-cloud`), which sends backups to S3, i.e. Garage. It needs cert-manager, which is why it runs one wave later.

`clusters/pi/storage/cnpg.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cnpg
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "-2" }
spec:
  project: default
  source:
    repoURL: https://cloudnative-pg.github.io/charts
    chart: cloudnative-pg
    targetRevision: "0.*"
  destination: { server: https://kubernetes.default.svc, namespace: cnpg-system }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true, ServerSideApply=true]
```

`clusters/pi/storage/cnpg-barman.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cnpg-barman
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "-1" } # after cert-manager and the operator
spec:
  project: default
  source:
    repoURL: https://cloudnative-pg.github.io/charts
    chart: plugin-barman-cloud
    targetRevision: "0.*"
  destination: { server: https://kubernetes.default.svc, namespace: cnpg-system } # must be the operator's namespace
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true, ServerSideApply=true]
```

Push both, then check:

```bash
kubectl -n cnpg-system get pods
# cnpg-cloudnative-pg-...   1/1 Running
# barman-cloud-...          1/1 Running

kubectl get crd | grep -E "postgresql.cnpg.io|barmancloud.cnpg.io"
# clusters.postgresql.cnpg.io, scheduledbackups..., objectstores.barmancloud.cnpg.io, ...
```

Install the `kubectl cnpg` plugin on the machine you run kubectl from. It gives a readable status of every database, and makes backups and restores one command:

```bash
curl -sSfL https://github.com/cloudnative-pg/cloudnative-pg/raw/main/hack/install-cnpg-plugin.sh \
  | sudo sh -s -- -b /usr/local/bin
kubectl cnpg version
```

### Step 6: The Postgres template for your apps

This is what you copy into any app folder that needs a database. Replace `APP` with the app name and PROJECT with its project (which is also the namespace). Every name starts with APP, so several apps with a database can live in one project. It gives you a primary + replica on different Pi's, continuous WAL archiving (point-in-time recovery to any second) and a nightly base backup.

`apps/PROJECT/APP/postgres.yaml`:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: { name: APP-db, namespace: PROJECT }
spec:
  instances: 2
  storage: { size: 5Gi, storageClass: local-path }
  bootstrap:
    initdb: { database: APP, owner: APP }
  plugins:
    - name: barman-cloud.cloudnative-pg.io
      enabled: true          # CNPG's default; written out so Argo CD sees no diff
      isWALArchiver: true
      parameters: { barmanObjectName: APP-s3 }
---
apiVersion: barmancloud.cnpg.io/v1
kind: ObjectStore
metadata: { name: APP-s3, namespace: PROJECT }
spec:
  retentionPolicy: "30d"
  configuration:
    destinationPath: s3://cnpg-backups/APP
    endpointURL: http://192.168.2.104:3900
    s3Credentials:
      accessKeyId: { name: APP-s3-creds, key: ACCESS_KEY_ID }
      secretAccessKey: { name: APP-s3-creds, key: ACCESS_SECRET_KEY }
      region: { name: APP-s3-creds, key: REGION }
    wal: { compression: gzip }
---
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata: { name: APP-db-nightly, namespace: PROJECT }
spec:
  schedule: "0 0 3 * * *"
  cluster: { name: APP-db }
  method: plugin
  pluginConfiguration: { name: barman-cloud.cloudnative-pg.io }
```

The `APP-s3-creds` secret comes from Phase: a second `PhaseSecret` in the app's folder syncs the shared Phase app `s3-backups` into a secret with that name (Part 5b). The only exception is Phase's own database, whose `phase-s3-creds` is created by hand (tier 0).

Until Phase is running (Part 5b), a database works fine without that secret: CNPG starts it as normal, and only the backups wait until the secret appears.

**Connecting the app:** CNPG creates a secret `APP-db-app` with `username`, `password`, `host`, `uri` and `jdbc-uri`. Connect to service `APP-db-rw` (writes) or `APP-db-ro` (read replicas).

**Restoring** (after a disaster, or to clone a database): create a new Cluster with a recovery bootstrap instead of `initdb`:

```yaml
  bootstrap:
    recovery: { source: origin }
  externalClusters:
    - name: origin
      plugin:
        name: barman-cloud.cloudnative-pg.io
        parameters: { barmanObjectName: APP-s3, serverName: APP-db }
```

Test a restore once before you depend on it. A backup you never restored is a hope, not a backup.

**Migrating existing Docker databases:** use `bootstrap.initdb.import` with `type: microservice` pointing at your old Postgres container, or `pg_dump` / `psql` through `kubectl port-forward svc/APP-db-rw 5432`.

### Step 7: Test it with a throwaway database

Before any real data depends on it, prove the whole chain once: create a database, back it up to Garage, and restore it into a second one. This runs by hand with `kubectl`, outside Git, and gets deleted at the end.

**1. Namespace and S3 credentials** (the `cnpg` key from Garage):

```bash
kubectl create namespace cnpg-test
kubectl -n cnpg-test create secret generic test-s3-creds \
  --from-literal=ACCESS_KEY_ID=<GK... of the cnpg key> \
  --from-literal=ACCESS_SECRET_KEY=<secret of the cnpg key> \
  --from-literal=REGION=garage
```

**2. The database**: the step 6 template with `APP` = `test` and `PROJECT` = `cnpg-test`. Save it as `test-db.yaml` and apply it:

```bash
sed -e 's/APP/test/g' -e 's/PROJECT/cnpg-test/g' template.yaml > test-db.yaml   # or edit by hand
kubectl apply -f test-db.yaml
kubectl cnpg status test-db -n cnpg-test
```

After a minute or two the status shows **Cluster in healthy state**, with a primary and a replica on different Pi's, and **Continuous Backup status** working (WAL archiving to Garage).

**3. Put some data in it:**

```bash
kubectl cnpg psql test-db -n cnpg-test -- -d test -c \
  "CREATE TABLE hello (msg text); INSERT INTO hello VALUES ('it works');"
```

**4. Take a backup** (normally the nightly ScheduledBackup does this):

```bash
kubectl cnpg backup test-db -n cnpg-test --method plugin --plugin-name barman-cloud.cloudnative-pg.io
kubectl get backups -n cnpg-test          # PHASE should become completed
```

In the Garage web UI you should now see files under `cnpg-backups/test/`.

**5. Restore it into a new database.** Save as `test-restore.yaml` and apply:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: { name: test-restore, namespace: cnpg-test }
spec:
  instances: 1
  storage: { size: 5Gi, storageClass: local-path }
  bootstrap:
    recovery: { source: origin }
  externalClusters:
    - name: origin
      plugin:
        name: barman-cloud.cloudnative-pg.io
        parameters: { barmanObjectName: test-s3, serverName: test-db }
```

```bash
kubectl apply -f test-restore.yaml
kubectl cnpg status test-restore -n cnpg-test      # wait until healthy
kubectl cnpg psql test-restore -n cnpg-test -- -d test -c "SELECT * FROM hello;"
# it works
```

If you see `it works`, backups and restores both work end to end.

**6. Clean up:**

```bash
kubectl delete namespace cnpg-test
```

The backup files stay in Garage under `cnpg-backups/test/`; delete that folder in the Garage web UI.

**If something fails:**

| Symptom | Check |
| --- | --- |
| Cluster stuck in `Setting up primary` | `kubectl -n cnpg-test get pods` and the pod's logs; often the `local-path` volume or an image pull |
| Continuous Backup status not OK | `kubectl -n cnpg-test describe objectstore test-s3`, then the S3 settings: endpoint `http://192.168.2.104:3900`, region `garage`, the `cnpg` key |
| Backup stays `running` or `failed` | `kubectl -n cnpg-system logs deploy/barman-cloud` (the plugin's log) |
| Restore can't find the backup | `serverName` must be the original cluster name (`test-db`), and `barmanObjectName` an ObjectStore in the same namespace |

## Part 5b: Secrets with Phase and automatic restarts

Phase becomes the source of truth for app secrets: you change a value in the Phase Console, the Phase operator syncs it into a Kubernetes Secret within seconds, and Reloader restarts the pods that use it. The few "tier 0" secrets that Phase itself depends on are created by hand (Part 3, step 3); nothing secret is ever stored in Git.

### What lives where

| Tier | Secret | Where | Why |
| --- | --- | --- | --- |
| 0 | Phase's own secrets, the Phase service token | `kubectl`, values in your password manager | Phase can't fetch the secrets it needs to start |
| 0 | S3 credentials for Phase's database backups | `kubectl` | Restoring Phase must not depend on Phase |
| 0 | cert-manager Cloudflare token | `kubectl` | Phase's HTTPS depends on it |
| 0 | Authentik secret key, Argo CD's OIDC secret and deploy key | `kubectl` | You can always log in and sync, even if Phase is broken |
| 1 | Everything else: Outline, ConvertX, Calagopus, cloudflared, S3 credentials for other apps | Phase | Edit in one UI, synced automatically |
| - | Postgres passwords | CNPG generates them | Nothing to manage |

The rule: **anything Phase depends on is created by hand.** Then a full rebuild works in order: Argo CD, the tier 0 secrets, Phase, and only after that everything that reads from Phase.

### Self-host Phase in the cluster

Phase runs from its official Helm chart ([docs](https://docs.phase.dev/self-hosting/kubernetes)) in namespace `phase`, internal only at `phase.k3s.home.klrnbk.nl`. It asks for about 2.4 CPU cores and 3.8 GB RAM in total, which the cluster has room for. It's split over two Applications in one file (step 6): `phase-deps` (wave `0`) with its database, Redis and route from `infrastructure/phase/`, then `phase` (wave `1`) with the chart, so Phase is up before any app needs it.

Phase consists of the chart (frontend, backend, worker) plus three things you provide yourself: a Postgres database from CNPG, a Redis, and a Traefik route. Everything below goes in `infrastructure/phase/`, except the Applications in `clusters/pi/secrets/phase.yaml`.

#### 1. The tier 0 secrets (by hand)

Generate each value with `openssl rand -hex 32` and store them in your password manager. Use the **same** value for the database password in both secrets:

```bash
kubectl create namespace phase

kubectl -n phase create secret generic phase-db-credentials --type=kubernetes.io/basic-auth \
  --from-literal=username=phase --from-literal=password=<db-password>

kubectl -n phase create secret generic phase-console-secret \
  --from-literal=SECRET_KEY=<hex32> \
  --from-literal=SERVER_SECRET=<hex32> \
  --from-literal=NEXTAUTH_SECRET=<hex32> \
  --from-literal=DATABASE_PASSWORD=<db-password> \
  --from-literal=REDIS_PASSWORD=<hex32>

kubectl -n phase create secret generic phase-s3-creds \
  --from-literal=ACCESS_KEY_ID=<GK... of the cnpg key> \
  --from-literal=ACCESS_SECRET_KEY=<secret of the cnpg key> \
  --from-literal=REGION=garage
```

`SERVER_SECRET` decrypts everything stored in Phase: without it, a database backup is useless. Guard it as carefully as your password manager itself.

#### 2. `infrastructure/phase/postgres.yaml`

The Part 5 template with `APP` = `phase` and `PROJECT` = `phase`. The only addition is `secret:` under `initdb`: the **bootstrap line**. It tells CNPG to create the `phase` user with the password from `phase-db-credentials`, instead of generating a random one, so the chart can use the same password:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: { name: phase-db, namespace: phase }
spec:
  instances: 2
  storage: { size: 5Gi, storageClass: local-path }
  bootstrap:
    initdb: { database: phase, owner: phase, secret: { name: phase-db-credentials } }
  plugins:
    - name: barman-cloud.cloudnative-pg.io
      enabled: true
      isWALArchiver: true
      parameters: { barmanObjectName: phase-s3 }
---
apiVersion: barmancloud.cnpg.io/v1
kind: ObjectStore
metadata: { name: phase-s3, namespace: phase }
spec:
  retentionPolicy: "30d"
  configuration:
    destinationPath: s3://cnpg-backups/phase
    endpointURL: http://192.168.2.104:3900
    s3Credentials:
      accessKeyId: { name: phase-s3-creds, key: ACCESS_KEY_ID }
      secretAccessKey: { name: phase-s3-creds, key: ACCESS_SECRET_KEY }
      region: { name: phase-s3-creds, key: REGION }
    wal: { compression: gzip }
---
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata: { name: phase-db-nightly, namespace: phase }
spec:
  schedule: "0 0 3 * * *"
  cluster: { name: phase-db }
  method: plugin
  pluginConfiguration: { name: barman-cloud.cloudnative-pg.io }
```

#### 3. `infrastructure/phase/redis.yaml`

Phase uses Redis as a cache and job queue. It holds nothing that needs a backup, so a small Deployment without a volume is enough. The password comes from `phase-console-secret`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: redis, namespace: phase }
spec:
  selector: { matchLabels: { app: redis } }
  template:
    metadata: { labels: { app: redis } }
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          args: ["--requirepass", "$(REDIS_PASSWORD)"]
          env:
            - name: REDIS_PASSWORD
              valueFrom: { secretKeyRef: { name: phase-console-secret, key: REDIS_PASSWORD } }
          ports: [{ containerPort: 6379 }]
          resources:
            requests: { cpu: 50m, memory: 64Mi }
            limits: { memory: 256Mi }
---
apiVersion: v1
kind: Service
metadata: { name: redis, namespace: phase }
spec:
  selector: { app: redis }
  ports: [{ port: 6379 }]
```

#### 4. `infrastructure/phase/ingressroute.yaml`

Phase serves its UI on `/` and its API on `/service/*`, and the API expects the `/service` prefix removed. The chart's own ingress only does that for NGINX, so this is the Traefik equivalent:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata: { name: strip-service, namespace: phase }
spec:
  stripPrefix: { prefixes: [/service] }
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata: { name: phase, namespace: phase }
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`phase.k3s.home.klrnbk.nl`) && PathPrefix(`/service`)
      kind: Rule
      middlewares: [{ name: strip-service }]
      services: [{ name: phase-console-backend, port: 8000 }]
    - match: Host(`phase.k3s.home.klrnbk.nl`)
      kind: Rule
      services: [{ name: phase-console-frontend, port: 3000 }]
  tls: {}
```

#### 5. `infrastructure/phase/kustomization.yaml`

```yaml
resources: [postgres.yaml, redis.yaml, ingressroute.yaml]
```

#### 6. `clusters/pi/secrets/phase.yaml`: two Applications in one file

The Phase chart runs its database migration as a Helm **pre-install hook**. Argo CD runs such hooks *before* it creates anything else in the same Application, and the migration waits for Postgres and Redis. So the folder above and the chart must be **separate** Applications, with the folder one wave earlier; otherwise the migration waits forever for a database Argo CD hasn't created yet.

One Application can't do it safely (turning Postgres and Redis into hooks too would let Argo CD recreate your database), but one **file** can hold both Applications: the root app reads every YAML document in it, separated by `---`. `phase-deps` (wave `0`) brings the database, Redis and route; `phase` (wave `1`) is the chart only. `releaseName: phase-console` makes the chart's Services `phase-console-frontend` and `phase-console-backend`, matching the IngressRoute, and the `database` and `redis` values point it at CNPG and your Redis instead of its own:

```yaml
# clusters/pi/secrets/phase.yaml
# 1/2: database, Redis and route first (the chart's migration hook needs them)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: phase-deps
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "0" }
spec:
  project: default
  source:
    repoURL: git@github.com:ijskoud/homelab-gitops.git
    targetRevision: main
    path: infrastructure/phase
  destination: { server: https://kubernetes.default.svc, namespace: phase }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true, ServerSideApply=true]
---
# 2/2: the Phase chart, once the above is Healthy
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: phase
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "1" }
spec:
  project: default
  source:
    repoURL: https://helm.phase.dev
    chart: phase
    targetRevision: "*" # pin the exact version once it runs
    helm:
      releaseName: phase-console
      valuesObject:
        global:
          host: phase.k3s.home.klrnbk.nl
          version: "latest"
        sso:
          providers: "authentik"
        phaseSecrets: phase-console-secret
        ingress: { enabled: false }
        certManager: { enabled: false }
        database:
          external: true # use CNPG, not the chart's own Postgres
          host: phase-db-rw
          port: "5432"
          name: phase
          user: phase
          sslmode: prefer
        redis:
          external: true # use redis.yaml, not the chart's own Redis
          host: redis
          port: "6379"
  destination: { server: https://kubernetes.default.svc, namespace: phase }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true, ServerSideApply=true]
```

Phase in wave `1` is still before your apps (wave `2`), which read their secrets from it. `sso.providers: authentik` adds a **Log in with Authentik** button; its `AUTHENTIK_*` keys go in `phase-console-secret` (below). After any change to these chart values, delete the ConfigMap `phase-console-config` (it's a Helm hook, so Argo CD never updates it), sync, and restart Phase's Deployments.

#### 7. Push and check

```bash
kubectl cnpg status phase-db -n phase          # healthy, primary + replica
kubectl -n phase get pods                      # frontend, backend, worker, redis: Running
kubectl -n phase get svc                       # phase-console-frontend, phase-console-backend, redis
curl -s https://phase.k3s.home.klrnbk.nl/service/health/    # {"status": "alive", ...}
```

The first sync runs a database migration job; Phase's pods only become Ready after that. If the migration hangs, `kubectl -n phase logs job/phase-console-migrations --all-containers` almost always shows a database connection problem: check that `DATABASE_PASSWORD` matches the password in `phase-db-credentials`.

### First login and setup

1. Open `https://phase.k3s.home.klrnbk.nl` and create your account. **Download the recovery kit**; it's the only way back in if you forget your sudo password.
2. Enable 2FA on your account. Authentik SSO is enabled (`sso.providers` in step 6, with `AUTHENTIK_CLIENT_ID`, `AUTHENTIK_CLIENT_SECRET`, `AUTHENTIK_URL` and `AUTHENTIK_APP_SLUG` added to `phase-console-secret`), but keep your local password login too, so a broken Authentik never locks you out of the secrets you'd need to fix it.
3. Create **one Phase app per service**, each with an environment `production`, and store its secrets at the root path `/`: `documentation-outline`, `media-convertx`, `platform-calagopus`, `cloudflared`, plus one shared app `s3-backups` with the Garage `cnpg` key for every database backup. Naming them `<project>-<app>` matches the Argo CD Application names.
4. Create a **service account** with read access to the `production` environment of each of these apps, generate a service token, and create the `phase-service-token` secret with it (Part 3, step 3).
5. Phase's database is backed up by CNPG like any other (Part 5), and those backups go offsite with restic (Part 13). Restoring Phase is step 1 of any disaster recovery.

### Install the operator

`clusters/pi/secrets/phase-operator.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: phase-operator
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "-2" }
spec:
  project: default
  source:
    repoURL: https://helm.phase.dev
    chart: phase-kubernetes-operator
    targetRevision: "*"
    helm:
      valuesObject:
        image: { tag: v2.0.1 }
  destination: { server: https://kubernetes.default.svc, namespace: phase-system }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true, ServerSideApply=true]
```

The operator authenticates with the `phase-service-token` secret in `phase-system` (tier 0, created by hand in Part 3, step 3).

Every `PhaseSecret` in every namespace points at this one token through `secretNamespace: phase-system`.

### Syncing an app's secrets

Each app gets a `PhaseSecret` in its folder, one per Phase app it reads, named after the secret it creates. Example for Outline, `apps/documentation/outline/phase-secret-outline.yaml` (the Phase app `documentation-outline` holds Outline's whole environment, see Part 8):

```yaml
apiVersion: secrets.phase.dev/v1alpha1
kind: PhaseSecret
metadata: { name: outline-env, namespace: documentation }
spec:
  phaseApp: documentation-outline
  phaseAppEnv: production
  phaseAppEnvPath: /
  phaseHost: https://phase.k3s.home.klrnbk.nl
  pollingInterval: 30
  authentication:
    serviceToken:
      serviceTokenSecretReference:
        secretName: phase-service-token
        secretNamespace: phase-system
  managedSecretReferences:
    - secretName: outline-env
      secretNamespace: documentation
      secretType: Opaque
```

Do the same for the other apps:

| App | Phase app (path `/`) | PhaseSecret file > managed secret (namespace) | Keys |
| --- | --- | --- | --- |
| Outline | `documentation-outline` | `phase-secret-outline.yaml` > `outline-env` (`documentation`) | see Part 8 |
| ConvertX | `media-convertx` | `phase-secret-convertx.yaml` > `convertx-env` (`media`) | `JWT_SECRET`, `ACCOUNT_REGISTRATION`, ... |
| Calagopus | `platform-calagopus` | `phase-secret-calagopus.yaml` > `calagopus-env` (`platform`) | `APP_ENCRYPTION_KEY`, ... |
| cloudflared | `cloudflared` | `phase-secret.yaml` > `cloudflared-token` (`platform`) | `TUNNEL_TOKEN`, `TUNNEL_ID` |
| Database backups | `s3-backups` (shared) | `phase-secret-s3.yaml` > `<app>-s3-creds`, in each app with a database | `ACCESS_KEY_ID`, `ACCESS_SECRET_KEY`, `REGION` |

Names are unique per namespace: two PhaseSecrets in one project can't share a name, so every name starts with the app (`outline-s3-secret`, `calagopus-s3-secret`). The `secretType: Opaque` line is the CRD's default, written out so Argo CD sees no diff; it belongs under `managedSecretReferences`, never under `serviceTokenSecretReference`.

Authentik, cert-manager and Phase itself are tier 0 and use the hand-made secrets from Part 3. The operator syncs Phase's `UPPER_SNAKE_CASE` keys as-is; use a processor when an app expects a different key name, e.g.:

```yaml
  managedSecretReferences:
    - secretName: some-app-secret
      secretNamespace: some-project
      secretType: Opaque
      processors:
        SOME_APP_API_KEY: { asName: api-key }
```

### Automatic restarts with Reloader

Most apps only read secrets at startup, so a changed secret does nothing until the pod restarts. Reloader watches Secrets and ConfigMaps and does a rolling restart of every workload that uses one that changed ([docs](https://docs.stakater.com/reloader/main/)).

`clusters/pi/core/reloader.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: reloader
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "-2" }
spec:
  project: default
  source:
    repoURL: https://stakater.github.io/stakater-charts
    chart: reloader
    targetRevision: "*" # pin
    helm:
      valuesObject:
        reloader:
          reloadStrategy: annotations
  destination: { server: https://kubernetes.default.svc, namespace: reloader }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true, ServerSideApply=true]
```

Then opt each workload in with one annotation on the Deployment's own metadata (not the pod template):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: outline
  namespace: documentation
  annotations:
    reloader.stakater.com/auto: "true"
```

cloudflared and Calagopus have it; add it to Outline, ConvertX and your own apps too. For Helm charts, use the chart's annotation value, e.g. for Authentik:

```yaml
    server:
      deploymentAnnotations: { reloader.stakater.com/auto: "true" }
    worker:
      deploymentAnnotations: { reloader.stakater.com/auto: "true" }
```

The chain is now: edit a value in Phase, the operator updates the Secret within `pollingInterval` seconds, Reloader rolls the pods.

Phase also has its own redeploy annotation (`secrets.phase.dev/redeploy: "true"`), but it only works for Deployments that load the secret with `envFrom`. Reloader also catches `secretKeyRef`, volume mounts, ConfigMaps and non-Phase secrets (CNPG, cert-manager), so use Reloader and leave Phase's annotation off; using both would restart pods twice.

## Part 6: Bind9 in the cluster

bind9 runs as 2 replicas on different Pi's behind MetalLB IP 192.168.2.21, and the zone file lives in Git. Editing a record = a commit.

**Watch out for the chicken-and-egg:** if every device on your network uses only this DNS and the cluster is down, nothing resolves. That's why the Pi's themselves keep using upstream DNS (Part 1). Keep your old bind9 running as a second DNS server until you trust the new one.

### Files

`apps/platform/bind9/kustomization.yaml`. The `configMapGenerator` adds a hash to the ConfigMap name, so every zone change automatically restarts bind9:

```yaml
namespace: platform
resources: [deployment.yaml, service.yaml]
configMapGenerator:
  - name: bind9-config
    files: [named.conf, db.home.klrnbk.nl, db.auth.klrnbk.nl]
```

`apps/platform/bind9/named.conf`:

```
options {
  directory "/var/cache/bind";
  recursion yes;
  allow-recursion { 192.168.2.0/24; 10.42.0.0/16; 127.0.0.1; };
  forwarders { 1.1.1.1; 9.9.9.9; };
  dnssec-validation auto;
  listen-on { any; };
  listen-on-v6 { none; };
};

zone "home.klrnbk.nl" {
  type primary;
  file "/etc/bind/zones/db.home.klrnbk.nl";
};

// Split-horizon: inside the LAN, auth.klrnbk.nl goes straight to Traefik.
// The internet keeps getting Cloudflare's answer (the tunnel).
zone "auth.klrnbk.nl" {
  type primary;
  file "/etc/bind/zones/db.auth.klrnbk.nl";
};
```

`apps/platform/bind9/db.home.klrnbk.nl` (bump the serial on every change, format `YYYYMMDDnn`; copy your existing records in here):

```
$TTL 300
@          IN SOA ns1.home.klrnbk.nl. admin.klrnbk.nl. (
             2026101301 3600 600 604800 300 )
           IN NS  ns1.home.klrnbk.nl.
ns1        IN A   192.168.2.21
k3s        IN A   192.168.2.10
rpi-1      IN A   192.168.2.100
rpi-2      IN A   192.168.2.101
rpi-3      IN A   192.168.2.102
rpi-4      IN A   192.168.2.103
nas        IN A   192.168.2.104
auth       IN A   192.168.2.20
*.k3s      IN A   192.168.2.20
```

No `*.home` wildcard: only these host records. `apps/platform/bind9/db.auth.klrnbk.nl`:

```
$TTL 300
@   IN SOA ns1.home.klrnbk.nl. admin.klrnbk.nl. ( 2026101301 3600 600 604800 300 )
    IN NS  ns1.home.klrnbk.nl.
@   IN A   192.168.2.20
```

`apps/platform/bind9/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bind9
  annotations:
    reloader.stakater.com/auto: "true"
spec:
  replicas: 2
  selector: { matchLabels: { app: bind9 } }
  template:
    metadata: { labels: { app: bind9 } }
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector: { matchLabels: { app: bind9 } }
      containers:
        - name: bind9
          image: ubuntu/bind9:9.20-24.04_edge   # multi-arch; pin and let Renovate bump
          ports:
            - { containerPort: 53, protocol: UDP }
            - { containerPort: 53, protocol: TCP }
          volumeMounts:
            - { name: config, mountPath: /etc/bind/named.conf, subPath: named.conf }
            - { name: config, mountPath: /etc/bind/zones/db.home.klrnbk.nl, subPath: db.home.klrnbk.nl }
            - { name: config, mountPath: /etc/bind/zones/db.auth.klrnbk.nl, subPath: db.auth.klrnbk.nl }
          readinessProbe: { tcpSocket: { port: 53 } }
      volumes:
        - name: config
          configMap: { name: bind9-config }
```

`apps/platform/bind9/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bind9
  annotations: { metallb.io/loadBalancerIPs: 192.168.2.21 }
spec:
  type: LoadBalancer
  selector: { app: bind9 }
  ports:
    - { name: dns-udp, port: 53, protocol: UDP }
    - { name: dns-tcp, port: 53, protocol: TCP }
```

### Switch clients over

Your current bind9 on rpi-1 keeps running during the whole switch:

1. Test from your laptop: `dig @192.168.2.21 test.k3s.home.klrnbk.nl` returns 192.168.2.20, `dig @192.168.2.21 auth.klrnbk.nl` returns 192.168.2.20, and `dig @192.168.2.21 github.com` resolves too.
2. In the Deco app, set the DHCP DNS server to 192.168.2.21, with the old bind9 (192.168.2.100) as secondary.
3. After a week without problems, stop the old bind9 container and remove it as secondary DNS in the Deco. The two bind9 replicas on different Pi's are the redundancy from then on.

### Let pods resolve internal names (CoreDNS)

Pods use the cluster's CoreDNS, which forwards to the Pi's upstream DNS and therefore doesn't know `*.k3s.home.klrnbk.nl`. The Phase operator needs `phase.k3s.home.klrnbk.nl`, so forward the internal zone to bind9. K3s reads extra CoreDNS config from a ConfigMap named `coredns-custom`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: coredns-custom, namespace: kube-system }
data:
  home.server: |
    home.klrnbk.nl:53 {
      forward . 192.168.2.21
      cache 30
    }
```

Put it in `infrastructure/configs/coredns-custom.yaml`. `auth.klrnbk.nl` deliberately isn't forwarded: pods use the public answer, which works the same.

## Part 7: Authentik

Authentik is your single sign-on: Outline logs in through it with OIDC, and apps without their own login (draw.io, withoutBG, Longhorn) get a login screen in front of them through Traefik forward auth. Since version 2025.10 Authentik needs only Postgres, no Redis ([release notes](https://docs.goauthentik.io/releases/2025.10/)), so it uses the CNPG template.

### Files in `apps/platform/authentik/`

Four files in `apps/platform/authentik/` (plus `middleware.yaml` once you add forward auth), and the Application in `clusters/pi/platform/`. The secret key is tier 0: the `authentik-secret` secret, created by hand (Part 3, step 3) **before** you push anything below.

#### 1. `postgres.yaml`

The Part 5 template with `APP` = `authentik` and `PROJECT` = `platform`. CNPG generates the password and puts it in the secret `authentik-db-app`, which the chart reads (step 6):

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: { name: authentik-db, namespace: platform }
spec:
  instances: 2
  storage: { size: 5Gi, storageClass: local-path }
  bootstrap:
    initdb: { database: authentik, owner: authentik }
  plugins:
    - name: barman-cloud.cloudnative-pg.io
      enabled: true
      isWALArchiver: true
      parameters: { barmanObjectName: authentik-s3 }
---
apiVersion: barmancloud.cnpg.io/v1
kind: ObjectStore
metadata: { name: authentik-s3, namespace: platform }
spec:
  retentionPolicy: "30d"
  configuration:
    destinationPath: s3://cnpg-backups/authentik
    endpointURL: http://192.168.2.104:3900
    s3Credentials:
      accessKeyId: { name: authentik-s3-creds, key: ACCESS_KEY_ID }
      secretAccessKey: { name: authentik-s3-creds, key: ACCESS_SECRET_KEY }
      region: { name: authentik-s3-creds, key: REGION }
    wal: { compression: gzip }
---
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata: { name: authentik-db-nightly, namespace: platform }
spec:
  schedule: "0 0 3 * * *"
  cluster: { name: authentik-db }
  method: plugin
  pluginConfiguration: { name: barman-cloud.cloudnative-pg.io }
```

#### 2. `phase-secret.yaml`

Syncs the Garage credentials for the database backups from the shared Phase app `s3-backups` into `authentik-s3-creds`. Authentik starts without it; only the backups wait for it.

```yaml
apiVersion: secrets.phase.dev/v1alpha1
kind: PhaseSecret
metadata: { name: authentik-s3-secret, namespace: platform }
spec:
  phaseApp: s3-backups
  phaseAppEnv: production
  phaseAppEnvPath: /
  phaseHost: https://phase.k3s.home.klrnbk.nl
  pollingInterval: 30
  authentication:
    serviceToken:
      serviceTokenSecretReference:
        secretName: phase-service-token
        secretNamespace: phase-system
  managedSecretReferences:
    - secretName: authentik-s3-creds
      secretNamespace: platform
      secretType: Opaque
```

#### 3. `media-pvc.yaml`

Authentik stores uploaded files (icons, branding, custom backgrounds) under `/data/media`. The volume is **ReadWriteMany**, because the two server pods and the worker run on different Pi's and all mount it:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: authentik-data, namespace: platform }
spec:
  accessModes: [ReadWriteMany]
  storageClassName: longhorn
  resources: { requests: { storage: 2Gi } }
```

#### 4. `middleware.yaml` (not in the repo yet)

The Traefik middleware other apps use for forward auth. Add it, and list it in `kustomization.yaml`, when you set up forward auth. It's explained under "Forward auth for apps without a login" below:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata: { name: forward-auth, namespace: platform }
spec:
  forwardAuth:
    address: http://authentik-server.platform.svc.cluster.local/outpost.goauthentik.io/auth/traefik
    trustForwardHeader: true
    authResponseHeaders:
      - X-authentik-username
      - X-authentik-groups
      - X-authentik-email
```

#### 5. `kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources: [postgres.yaml, phase-secret.yaml, media-pvc.yaml]
```

#### 6. `clusters/pi/platform/authentik.yaml`

The Application combines this folder with the chart. The issuer and every OAuth config use `auth.home.klrnbk.nl`, because that's the name both your browser and the pods resolve to the same Authentik (that's why the ApplicationSet in Part 3 skips `apps/platform/authentik`). All keys below are checked against the chart's [values.yaml](https://github.com/goauthentik/helm/blob/main/charts/authentik/values.yaml):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: authentik
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "1" }
spec:
  project: platform
  sources:
    - repoURL: git@github.com:ijskoud/homelab-gitops.git
      targetRevision: main
      path: apps/platform/authentik
    - repoURL: https://charts.goauthentik.io
      chart: authentik
      targetRevision: "2026.8"
      helm:
        valuesObject:
          authentik:
            postgresql:
              host: authentik-db-rw
              name: authentik
              user: authentik
              port: 5432
            email:
              host: smtp.forwardemail.net
              port: 465
              use_tls: false
              use_ssl: true
              timeout: 10
              from: "noreply@auth.klrnbk.nl"
          global:
            env:
              - name: AUTHENTIK_SECRET_KEY
                valueFrom: { secretKeyRef: { name: authentik-secret, key: AUTHENTIK_SECRET_KEY } }
              - name: AUTHENTIK_POSTGRESQL__PASSWORD
                valueFrom: { secretKeyRef: { name: authentik-db-app, key: password } }
              - name: AUTHENTIK_EMAIL__USERNAME
                valueFrom:
                  { secretKeyRef: { name: authentik-secret, key: AUTHENTIK_EMAIL__USERNAME } }
              - name: AUTHENTIK_EMAIL__PASSWORD
                valueFrom:
                  { secretKeyRef: { name: authentik-secret, key: AUTHENTIK_EMAIL__PASSWORD } }
            volumes:
              - name: data
                persistentVolumeClaim: { claimName: authentik-data }
            volumeMounts:
              - name: data
                mountPath: /data
          postgresql:
            enabled: false # we use CNPG instead of the bundled chart
          server:
            replicas: 2
            ingress:
              enabled: true
              ingressClassName: traefik
              hosts:
                - auth.home.klrnbk.nl
                - auth.k3s.home.klrnbk.nl
  destination: { server: https://kubernetes.default.svc, namespace: platform }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true, ServerSideApply=true]
```

Check after the full migration below:

```bash
kubectl cnpg status authentik-db -n platform        # healthy, primary + replica
kubectl -n platform get pods -l app.kubernetes.io/name=authentik   # 2x server, 1x worker: Running
kubectl -n platform get pvc authentik-data          # Bound
curl -s -o /dev/null -w "%{http_code}\n" https://auth.k3s.home.klrnbk.nl/-/health/live/   # 200 or 204
```

#### Email (SMTP)

Authentik sends mail for password resets, invitations and MFA recovery. The server, port, TLS mode and sender are in the `email:` block of the Application above; the **username and password** are in `authentik-secret` as `AUTHENTIK_EMAIL__USERNAME` and `AUTHENTIK_EMAIL__PASSWORD` (tier 0, Part 3, step 3), so neither ends up in Git. Already created the secret without them? Add both keys:

```bash
kubectl -n platform patch secret authentik-secret --type merge -p '{
  "stringData": {
    "AUTHENTIK_EMAIL__USERNAME": "<smtp username>",
    "AUTHENTIK_EMAIL__PASSWORD": "<smtp password>"
  }
}'
```

Send a test mail once Authentik runs:

```bash
kubectl -n platform exec deploy/authentik-worker -- ak test_email <your email address>
```

| Error contains | Fix |
| --- | --- |
| `Authentication failed` / `535` | Wrong username or password; for Gmail and Outlook use an app password |
| `wrong version number` / `SSL` | TLS mode doesn't match the port: 587 needs `use_tls: true`, 465 needs `use_ssl: true` (never both) |
| Timeout | Wrong host or port, or your provider blocks that port |

### Migrating your existing Authentik from rpi-1

You keep all users, MFA devices, flows and OAuth providers (including Calagopus') by moving the database instead of starting fresh.

1. Note the exact version of the running Authentik (admin UI footer, or the image tag in your compose file) and pin the same chart version in `clusters/pi/platform/authentik.yaml`.
2. Create the `authentik-secret` with the **current** `AUTHENTIK_SECRET_KEY` from your Docker setup (Part 3, step 3). A different key breaks sessions and encrypted fields.
3. First push `clusters/pi/platform/authentik.yaml` with **only the Git source** (leave out the chart for now). Argo CD creates the CNPG cluster `authentik-db`, the volume and the middleware, but no Authentik yet, so nothing writes an empty schema into the database. Wait until `kubectl cnpg status authentik-db -n platform` is healthy.
4. Stop the Docker Authentik (server and worker), then dump and import:

   ```bash
   # on rpi-1: dump the old database
   docker exec <authentik-postgres> pg_dump -U authentik -Fc authentik > authentik.dump

   # restore inside the CNPG primary pod: it has pg_restore in the matching version,
   # so nothing extra needs to be installed on the Pi
   PRIMARY=$(kubectl -n platform get cluster authentik-db -o jsonpath='{.status.currentPrimary}')
   kubectl -n platform cp authentik.dump $PRIMARY:/var/lib/postgresql/data/authentik.dump -c postgres
   kubectl -n platform exec $PRIMARY -c postgres -- \
     pg_restore -U postgres -d authentik --no-owner --role=authentik \
     /var/lib/postgresql/data/authentik.dump

   # check, then remove the dump from the pod
   kubectl -n platform exec $PRIMARY -c postgres -- psql -U postgres -d authentik -c '\dt' | head
   kubectl -n platform exec $PRIMARY -c postgres -- rm /var/lib/postgresql/data/authentik.dump
   ```
5. Add the chart source to `clusters/pi/platform/authentik.yaml` and push. Wait until the pods are Running and log in at `https://auth.k3s.home.klrnbk.nl` with your existing account.
6. Copy your uploaded files (icons, branding) into the new volume. In the Docker setup they're in the folder mounted at `/media` (older installs) or `/data/media` (newer ones); either way they go to `/data/media` now:

   ```bash
   POD=$(kubectl -n platform get pods -l app.kubernetes.io/component=server -o name | head -1)
   kubectl -n platform cp ./media ${POD#pod/}:/data/media -c server
   kubectl -n platform exec ${POD#pod/} -c server -- ls /data/media     # your files
   ```

   The volume is shared, so copying into one pod is enough. Then check under **Customization > Files** in the admin UI that the files show up and uploads work.
7. Switch the public route: point `auth.klrnbk.nl` at the cluster's Cloudflare Tunnel (Part 7b) and check a login from outside (phone on mobile data).
8. Only after a week, upgrade to the current Authentik version, one major release at a time per the release notes.

### Forward auth for apps without a login

One-time setup in the Authentik admin UI:

1. **Applications > Providers > Create > Proxy Provider**, mode **Forward auth (domain level)**. Authentication URL `https://auth.k3s.home.klrnbk.nl`, cookie domain `k3s.home.klrnbk.nl`.
2. **Applications > Create**, name `Homelab`, pick that provider.
3. **Outposts > authentik Embedded Outpost > Edit**, add the `Homelab` application.

The Traefik side is `middleware.yaml` (file 4 above): it sends every request to the embedded outpost, which checks the login.

Ingresses in other namespaces can reference it because Traefik has `allowCrossNamespace: true` (Part 4).

Any Ingress you want behind the login then gets:

```yaml
metadata:
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: platform-forward-auth@kubernetescrd
```

The name format is `<namespace>-<middleware name>@kubernetescrd`.

### Argo CD login through Authentik

1. In Authentik: **Applications > Create with provider**, type **OAuth2/OpenID Connect**, name `Argo CD`, slug `argo-cd`, client type **Confidential**, strict redirect URI `https://argocd.k3s.home.klrnbk.nl/auth/callback`. Create two groups, `Deployment / ArgoCD / Admin` (full control) and `Deployment / ArgoCD / Default` (view, sync and restart), and add people to them. The group names in `policy.csv` must match exactly.
2. The client secret is tier 0 (Argo CD must work when Phase doesn't), and Argo CD reads it from its own `argocd-secret`. Add it once from your laptop, and keep a copy in your password manager:

   ```bash
   kubectl -n argocd patch secret argocd-secret --type merge \
     -p '{"stringData":{"oidc.authentik.clientSecret":"<client secret>"}}'
   ```
3. The complete `infrastructure/argocd/values.yaml`; push it and Argo CD updates itself:

   ```yaml
   configs:
     params:
       server.insecure: true # Traefik terminates TLS in front of it
     cm:
       timeout.reconciliation: 60s # poll Git every minute instead of every 3
       # Let the root app wait until each child Application is Healthy (makes sync waves work)
       resource.customizations.health.argoproj.io_Application: |
         hs = {}
         hs.status = "Progressing"
         hs.message = ""
         if obj.status ~= nil and obj.status.health ~= nil then
           hs.status = obj.status.health.status
           if obj.status.health.message ~= nil then
             hs.message = obj.status.health.message
           end
         end
         return hs
       url: https://argocd.k3s.home.klrnbk.nl
       oidc.config: |
         name: KLRNBK
         issuer: https://auth.home.klrnbk.nl/application/o/argo-cd/
         clientID: NuIeSl6lCx1MGnjDYbgbY9VS8QFERSeWD7486TZe
         clientSecret: $oidc.authentik.clientSecret
         requestedScopes: [openid, profile, email]
     rbac:
       policy.csv: |
         p, role:user, applications, get, */*, allow
         p, role:user, applications, sync, */*, allow
         p, role:user, applications, action/apps/Deployment/restart, */*, allow
         p, role:user, logs, get, */*, allow
         g, Deployment / ArgoCD / Admin, role:admin
         g, Deployment / ArgoCD / Default, role:user

   server:
     ingress:
       enabled: true
       ingressClassName: traefik
       hostname: argocd.k3s.home.klrnbk.nl
   ```
4. Log out and choose **Log in via Authentik**. Keep the local `admin` account as break-glass: if Authentik is down, you still get into Argo CD to fix it.

## Part 7b: Cloudflare Tunnel for Authentik

A Cloudflare Tunnel makes Authentik reachable from the internet at `auth.klrnbk.nl` without opening any port on your router: `cloudflared` pods in the cluster dial out to Cloudflare, and Cloudflare forwards requests back through that connection. Only Authentik and the Calagopus panel are public; everything else stays internal-only.

```
Internet -> auth.klrnbk.nl (Cloudflare edge, public TLS)
         -> tunnel -> cloudflared pods (2 replicas, any Pi)
         -> authentik-server.platform.svc:80
```

This requires the DNS for `klrnbk.nl` to be managed by Cloudflare.

### Step 1: create the tunnel

1. Cloudflare dashboard > **Zero Trust > Networks > Tunnels > Create a tunnel**, type **Cloudflared**, name `homelab`.
2. Skip the install instructions, but copy the **tunnel token** (the long string after `--token`).
3. Add **public hostnames**, service type `HTTP`:
   - `auth.klrnbk.nl` to `authentik-server.platform.svc.cluster.local:80`
   - `panel.klrnbk.nl` to `calagopus.platform.svc.cluster.local:80` (Part 11b)
4. Store the token in the Phase app `cloudflared` as `TUNNEL_TOKEN`, and the tunnel's ID as `TUNNEL_ID` (synced to secret `cloudflared-token`, see Part 5b).
5. If `auth.klrnbk.nl` currently points at your old setup (a DNS record or another tunnel), the new public hostname replaces it. Do this at step 7 of the Authentik migration (Part 7).

Routing is configured in the dashboard (a "remotely managed" tunnel), so adding a public hostname later needs no change in the cluster.

### Step 2: run cloudflared

`apps/platform/cloudflared/` with `phase-secret.yaml` (Phase app `cloudflared`, managed secret `cloudflared-token` with `TUNNEL_TOKEN` and `TUNNEL_ID`), `cloudflared.yaml` and a `kustomization.yaml` listing both:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: platform
  annotations:
    reloader.stakater.com/auto: "true"
spec:
  replicas: 2
  selector: { matchLabels: { app: cloudflared } }
  template:
    metadata: { labels: { app: cloudflared } }
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector: { matchLabels: { app: cloudflared } }
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:latest
          args: [tunnel, --no-autoupdate, --metrics, 0.0.0.0:2000, run]
          env:
            - name: TUNNEL_TOKEN
              valueFrom: { secretKeyRef: { name: cloudflared-token, key: TUNNEL_TOKEN } }
            - name: TUNNEL_ID
              valueFrom: { secretKeyRef: { name: cloudflared-token, key: TUNNEL_ID } }
          livenessProbe:
            httpGet: { path: /ready, port: 2000 }
            failureThreshold: 1
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests: { cpu: 50m, memory: 64Mi }
            limits: { memory: 256Mi }
```

`phase-secret.yaml`:

```yaml
apiVersion: secrets.phase.dev/v1alpha1
kind: PhaseSecret
metadata: { name: cloudflared, namespace: platform }
spec:
  phaseApp: cloudflared
  phaseAppEnv: production
  phaseAppEnvPath: /
  phaseHost: https://phase.k3s.home.klrnbk.nl
  pollingInterval: 30
  authentication:
    serviceToken:
      serviceTokenSecretReference:
        secretName: phase-service-token
        secretNamespace: phase-system
  managedSecretReferences:
    - secretName: cloudflared-token
      secretNamespace: platform
      secretType: Opaque
```

`kustomization.yaml`:

```yaml
resources: [phase-secret.yaml, cloudflared.yaml]
```

Push it; the ApplicationSet picks up the new folder automatically as `platform-cloudflared`. The tunnel shows **Healthy** in the Cloudflare dashboard with 2 connectors once both pods are up.

### Step 3: harden Authentik now that it's public

- **Enforce MFA**: in Authentik, edit the `default-authentication-flow` and set the MFA validation stage to *force* (TOTP or WebAuthn/passkey) for all users.
- **Invitation-only enrollment**: external users get a single-use invitation link (Part 11b), so there's no open sign-up page.
- **Block the admin UI from outside**: a Cloudflare WAF custom rule on `auth.klrnbk.nl` with path starting with `/if/admin` > Block. At home the same URL bypasses Cloudflare (split-horizon, Part 6), so the admin UI keeps working on your LAN.
- **Rate limit** the login path (`/if/flow/` and `/api/v3/flows/`) with a Cloudflare rate-limiting rule.
- **Real client IPs**: Authentik trusts private ranges as proxies by default, so the `X-Forwarded-For` header from cloudflared (pod IPs in 10.42.0.0/16) is honoured and logs show the visitor's real IP.

### One URL inside and outside

`auth.klrnbk.nl` resolves differently depending on where you are: the internet gets Cloudflare (and the tunnel), your LAN gets 192.168.2.20 from bind9 and talks to Traefik directly, with a valid Let's Encrypt certificate for that name (Part 4). Because the name is the same everywhere, use `https://auth.klrnbk.nl` in every OAuth/OIDC config (Calagopus, Outline): the issuer URL then matches whether a login starts at home or outside. `auth.home.klrnbk.nl` and `auth.k3s.home.klrnbk.nl` keep working as LAN aliases.

## Part 8: Outline

Outline has no username/password login of its own, so Authentik must be running first. It needs Postgres (CNPG), Redis and file storage; uploads go to a Longhorn volume, so you don't need S3 for it.

### Step 1: OIDC provider in Authentik

1. **Applications > Providers > Create > OAuth2/OpenID Provider**, name `Outline`.
2. Client type **Confidential**, redirect URI (strict) `https://outline.k3s.home.klrnbk.nl/auth/oidc.callback`.
3. **Applications > Create**, name `Outline`, slug `outline`, pick that provider.
4. Copy the client ID and secret.

### Step 2: files in `apps/documentation/outline/`

- `postgres.yaml`: the Part 5 template with `APP` = `outline`, `PROJECT` = `documentation`
- `phase-secret-outline.yaml`: Phase app `documentation-outline` into `outline-env` (the example in Part 5b)
- `phase-secret-s3.yaml`: the shared Phase app `s3-backups` into `outline-s3-creds`
- `redis.yaml`, `outline.yaml`, and a `kustomization.yaml` listing all five

The Deployment only sets the database and file storage; **everything else comes from the Phase app `documentation-outline`**, so you change settings without a commit: `SECRET_KEY` and `UTILS_SECRET` (each `openssl rand -hex 32`), `URL=https://outline.k3s.home.klrnbk.nl`, `REDIS_URL=redis://redis:6379`, `PGSSLMODE=disable`, `FORCE_HTTPS=false`, and the OIDC settings: `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, `OIDC_AUTH_URI`, `OIDC_TOKEN_URI`, `OIDC_USERINFO_URI`, `OIDC_LOGOUT_URI` (all under `https://auth.home.klrnbk.nl/application/o/`), `OIDC_USERNAME_CLAIM=preferred_username`, `OIDC_DISPLAY_NAME=Authentik`, `OIDC_SCOPES=openid profile email`.

The Service is called `redis` in namespace `documentation`; a second app in this project that needs Redis must use another name.

`redis.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: redis, namespace: documentation }
spec:
  selector: { matchLabels: { app: redis } }
  template:
    metadata: { labels: { app: redis } }
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports: [{ containerPort: 6379 }]
---
apiVersion: v1
kind: Service
metadata: { name: redis, namespace: documentation }
spec:
  selector: { app: redis }
  ports: [{ port: 6379 }]
```

`outline.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: outline-data, namespace: documentation }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn
  resources: { requests: { storage: 10Gi } }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: outline, namespace: documentation }
spec:
  replicas: 1 # local file storage on a RWO volume = 1 replica
  strategy: { type: Recreate }
  selector: { matchLabels: { app: outline } }
  template:
    metadata: { labels: { app: outline } }
    spec:
      securityContext: { fsGroup: 1001 }
      containers:
        - name: outline
          image: outlinewiki/outline:1.11.0
          ports: [{ containerPort: 3000 }]
          envFrom: [{ secretRef: { name: outline-env } }]
          env:
            - name: DATABASE_URL
              valueFrom: { secretKeyRef: { name: outline-db-app, key: uri } }
            - { name: FILE_STORAGE, value: local }
            - { name: FILE_STORAGE_LOCAL_ROOT_DIR, value: /var/lib/outline/data }
          volumeMounts: [{ name: data, mountPath: /var/lib/outline/data }]
      volumes:
        - name: data
          persistentVolumeClaim: { claimName: outline-data }
---
apiVersion: v1
kind: Service
metadata: { name: outline, namespace: documentation }
spec:
  selector: { app: outline }
  ports: [{ port: 80, targetPort: 3000 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: outline, namespace: documentation }
spec:
  ingressClassName: traefik
  rules:
    - host: outline.k3s.home.klrnbk.nl
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: outline, port: { number: 80 } } }
```

`postgres.yaml`:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: { name: outline-db, namespace: documentation }
spec:
  instances: 2
  storage: { size: 5Gi, storageClass: local-path }
  bootstrap:
    initdb: { database: outline, owner: outline }
  plugins:
    - name: barman-cloud.cloudnative-pg.io
      enabled: true
      isWALArchiver: true
      parameters: { barmanObjectName: outline-s3 }
---
apiVersion: barmancloud.cnpg.io/v1
kind: ObjectStore
metadata: { name: outline-s3, namespace: documentation }
spec:
  retentionPolicy: "30d"
  configuration:
    destinationPath: s3://cnpg-backups/outline
    endpointURL: http://192.168.2.104:3900
    s3Credentials:
      accessKeyId: { name: outline-s3-creds, key: ACCESS_KEY_ID }
      secretAccessKey: { name: outline-s3-creds, key: ACCESS_SECRET_KEY }
      region: { name: outline-s3-creds, key: REGION }
    wal: { compression: gzip }
---
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata: { name: outline-db-nightly, namespace: documentation }
spec:
  schedule: "0 0 3 * * *"
  cluster: { name: outline-db }
  method: plugin
  pluginConfiguration: { name: barman-cloud.cloudnative-pg.io }
```

`phase-secret-s3.yaml` (`phase-secret-outline.yaml` is the example in Part 5b):

```yaml
apiVersion: secrets.phase.dev/v1alpha1
kind: PhaseSecret
metadata: { name: outline-s3-secret, namespace: documentation }
spec:
  phaseApp: s3-backups
  phaseAppEnv: production
  phaseAppEnvPath: /
  phaseHost: https://phase.k3s.home.klrnbk.nl
  pollingInterval: 30
  authentication:
    serviceToken:
      serviceTokenSecretReference:
        secretName: phase-service-token
        secretNamespace: phase-system
  managedSecretReferences:
    - secretName: outline-s3-creds
      secretNamespace: documentation
      secretType: Opaque
```

`kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  [postgres.yaml, phase-secret-outline.yaml, phase-secret-s3.yaml, outline.yaml, redis.yaml]
```

Check the [Outline releases](https://github.com/outline/outline/releases) for the current version. Outline runs its database migrations on startup.

### Step 3: first login

The **first person to sign in becomes the admin**, so log in yourself right after it comes up. Then, in Outline settings, turn off sign-ups or restrict them to your Authentik users.

## Part 9: ConvertX

ConvertX converts between 1000+ file formats and publishes ARM64 images ([GitHub](https://github.com/C4illin/ConvertX)). It has its own accounts; **anyone can register the first account**, so create yours right after deploying, and keep `ACCOUNT_REGISTRATION=false` in its Phase app. The image is large (about 1.3 GB), so the first pull on each Pi takes a while.

`apps/media/convertx/convertx.yaml`, plus `phase-secret-convertx.yaml` that syncs the Phase app `media-convertx` into `convertx-env` (`JWT_SECRET` from `openssl rand -hex 32`, `ACCOUNT_REGISTRATION=false`, `AUTO_DELETE_EVERY_N_HOURS=24`), and a `kustomization.yaml` listing both:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: convertx-data, namespace: media }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn
  resources: { requests: { storage: 10Gi } }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: convertx, namespace: media }
spec:
  replicas: 1
  strategy: { type: Recreate }
  selector: { matchLabels: { app: convertx } }
  template:
    metadata: { labels: { app: convertx } }
    spec:
      containers:
        - name: convertx
          image: ghcr.io/c4illin/convertx:v0.19.0
          ports: [{ containerPort: 3000 }]
          envFrom: [{ secretRef: { name: convertx-env } }]
          resources:
            requests: { cpu: 250m, memory: 512Mi }
            limits: { memory: 4Gi } # video conversions with FFmpeg are heavy
          volumeMounts: [{ name: data, mountPath: /app/data }]
      volumes:
        - name: data
          persistentVolumeClaim: { claimName: convertx-data }
---
apiVersion: v1
kind: Service
metadata: { name: convertx, namespace: media }
spec:
  selector: { app: convertx }
  ports: [{ port: 80, targetPort: 3000 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: convertx, namespace: media }
spec:
  ingressClassName: traefik
  rules:
    - host: convert.k3s.home.klrnbk.nl
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: convertx, port: { number: 80 } } }
```

`phase-secret-convertx.yaml`:

```yaml
apiVersion: secrets.phase.dev/v1alpha1
kind: PhaseSecret
metadata: { name: convertx-env, namespace: media }
spec:
  phaseApp: media-convertx
  phaseAppEnv: production
  phaseAppEnvPath: /
  phaseHost: https://phase.k3s.home.klrnbk.nl
  pollingInterval: 30
  authentication:
    serviceToken:
      serviceTokenSecretReference:
        secretName: phase-service-token
        secretNamespace: phase-system
  managedSecretReferences:
    - secretName: convertx-env
      secretNamespace: media
      secretType: Opaque
```

`kustomization.yaml`:

```yaml
resources: [convertx.yaml, phase-secret-convertx.yaml]
```

Want one login for everything? Add the `platform-forward-auth@kubernetescrd` middleware annotation from Part 7 as an extra layer in front of ConvertX's own login.

## Part 10: draw.io

The official `jgraph/drawio` image is built for ARM64 and runs the editor on Tomcat, port 8080 ([self-hosting docs](https://github.com/jgraph/drawio/wiki/FAQ)). It stores nothing itself: diagrams are saved to your device or a storage backend you connect (GitHub, for example), so no volume is needed and it can run 2 replicas. Self-hosted draw.io has no real-time co-editing and PDF export goes through the browser's print dialog.

It has no login. Once forward auth is set up (Part 7), add the `platform-forward-auth@kubernetescrd` annotation to its Ingress; until then it's open to anyone on your LAN and tailnet.

`apps/documentation/draw-io/drawio.yaml` (plus a `kustomization.yaml` listing it):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: drawio, namespace: documentation }
spec:
  replicas: 2
  selector: { matchLabels: { app: drawio } }
  template:
    metadata: { labels: { app: drawio } }
    spec:
      containers:
        - name: drawio
          image: jgraph/drawio:32.3.0
          ports: [{ containerPort: 8080 }]
          resources:
            requests: { cpu: 100m, memory: 384Mi }
            limits: { memory: 1Gi }
          readinessProbe:
            httpGet: { path: /, port: 8080 }
---
apiVersion: v1
kind: Service
metadata: { name: drawio, namespace: documentation }
spec:
  selector: { app: drawio }
  ports: [{ port: 80, targetPort: 8080 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: drawio
  namespace: documentation
spec:
  ingressClassName: traefik
  rules:
    - host: draw.k3s.home.klrnbk.nl
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: drawio, port: { number: 80 } } }
```

`kustomization.yaml`:

```yaml
resources: [drawio.yaml]
```

Tip: open it as `https://draw.k3s.home.klrnbk.nl/?offline=1` to hide the cloud storage options and keep everything local.

## Part 11: Background removal (withoutBG)

**withoutBG** is the closest self-hosted match to remove.bg: a drag-and-drop web UI plus an API, running open-weights models on the CPU with no GPU or cloud calls. The CPU image `withoutbg/withoutbg-openweights-v3-app-cpu` is published for ARM64 ([Docker Hub](https://hub.docker.com/r/withoutbg/withoutbg-openweights-v3-app-cpu)), and the models are bundled in the image, so there's nothing to download or configure.

Expect a few seconds per image on a Pi 5. Alternatives if you want to compare: `rembg` (CLI/API, many models) and ConvertX for simple format work.

`apps/media/withoutbg/withoutbg.yaml` (the only file in its folder, so no `kustomization.yaml` is needed). Like draw.io it has no login; add the forward auth annotation once Part 7's middleware exists:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: withoutbg, namespace: media }
spec:
  replicas: 1
  selector: { matchLabels: { app: withoutbg } }
  template:
    metadata: { labels: { app: withoutbg } }
    spec:
      containers:
        - name: withoutbg
          image: withoutbg/withoutbg-openweights-v3-app-cpu:latest
          ports: [{ containerPort: 8080 }]
          resources:
            requests: { cpu: "1", memory: 4Gi }
            limits: { memory: 8Gi }
          readinessProbe:
            httpGet: { path: /api/ready, port: 8080 }
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /health, port: 8080 }
            initialDelaySeconds: 60
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata: { name: withoutbg, namespace: media }
spec:
  selector: { app: withoutbg }
  ports: [{ port: 80, targetPort: 8080 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: withoutbg
  namespace: media
spec:
  ingressClassName: traefik
  rules:
    - host: removebg.k3s.home.klrnbk.nl
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: withoutbg, port: { number: 80 } } }
```

The API is available at `https://removebg.k3s.home.klrnbk.nl/api/v1/remove-background`, handy if one of your own apps needs background removal. Requests from apps inside the cluster can call `http://withoutbg.media.svc/api/...` directly and skip the login.

## Part 11b: Calagopus panel

The Calagopus panel moves from Docker on rpi-2 into the cluster, public at `panel.klrnbk.nl` through the tunnel; Wings and the game servers stay on your external server. The panel needs Postgres (CNPG) and Valkey, and its built-in **Wings proxy** (`APP_ENABLE_WINGS_PROXY`) relays console traffic to Wings, so players' browsers only ever talk to the panel ([compose reference](https://raw.githubusercontent.com/calagopus/panel/refs/heads/main/compose.yml)).

### Files in `apps/platform/calagopus/`

Calagopus lives in project `platform`, so its namespace is `platform` and its in-cluster address `calagopus.platform.svc.cluster.local`. The folder has no `kustomization.yaml`, so Argo CD applies every file in it.

- `postgres.yaml`: the Part 5 template with `APP` = `calagopus` and `PROJECT` = `platform`, plus `imageName: ghcr.io/cloudnative-pg/postgresql:18` under `spec:` (Calagopus ships Postgres 18; the major must be at least the one you migrate from)
- `phase-secret-calagopus.yaml`: Phase app `platform-calagopus` into `calagopus-env` (`APP_ENCRYPTION_KEY`, ...)
- `phase-secret-s3.yaml`: the shared Phase app `s3-backups` into `calagopus-s3-creds`
- `valkey.yaml`, `panel.yaml`

The Ingress in `panel.yaml` is commented out for now: the panel is reached through the tunnel. Uncomment it for `panel.k3s.home.klrnbk.nl`.

`postgres.yaml`:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: { name: calagopus-db, namespace: platform }
spec:
  instances: 2
  imageName: ghcr.io/cloudnative-pg/postgresql:18
  storage: { size: 5Gi, storageClass: local-path }
  bootstrap:
    initdb: { database: calagopus, owner: calagopus }
  plugins:
    - name: barman-cloud.cloudnative-pg.io
      enabled: true
      isWALArchiver: true
      parameters: { barmanObjectName: calagopus-s3 }
---
apiVersion: barmancloud.cnpg.io/v1
kind: ObjectStore
metadata: { name: calagopus-s3, namespace: platform }
spec:
  retentionPolicy: "30d"
  configuration:
    destinationPath: s3://cnpg-backups/calagopus
    endpointURL: http://192.168.2.104:3900
    s3Credentials:
      accessKeyId: { name: calagopus-s3-creds, key: ACCESS_KEY_ID }
      secretAccessKey: { name: calagopus-s3-creds, key: ACCESS_SECRET_KEY }
      region: { name: calagopus-s3-creds, key: REGION }
    wal: { compression: gzip }
---
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata: { name: calagopus-db-nightly, namespace: platform }
spec:
  schedule: "0 0 3 * * *"
  cluster: { name: calagopus-db }
  method: plugin
  pluginConfiguration: { name: barman-cloud.cloudnative-pg.io }
```

`phase-secret-calagopus.yaml`:

```yaml
apiVersion: secrets.phase.dev/v1alpha1
kind: PhaseSecret
metadata: { name: calagopus-env, namespace: platform }
spec:
  phaseApp: platform-calagopus
  phaseAppEnv: production
  phaseAppEnvPath: /
  phaseHost: https://phase.k3s.home.klrnbk.nl
  pollingInterval: 30
  authentication:
    serviceToken:
      serviceTokenSecretReference:
        secretName: phase-service-token
        secretNamespace: phase-system
  managedSecretReferences:
    - secretName: calagopus-env
      secretNamespace: platform
      secretType: Opaque
```

`phase-secret-s3.yaml`:

```yaml
apiVersion: secrets.phase.dev/v1alpha1
kind: PhaseSecret
metadata: { name: calagopus-s3-secret, namespace: platform }
spec:
  phaseApp: s3-backups
  phaseAppEnv: production
  phaseAppEnvPath: /
  phaseHost: https://phase.k3s.home.klrnbk.nl
  pollingInterval: 30
  authentication:
    serviceToken:
      serviceTokenSecretReference:
        secretName: phase-service-token
        secretNamespace: phase-system
  managedSecretReferences:
    - secretName: calagopus-s3-creds
      secretNamespace: platform
      secretType: Opaque
```

`valkey.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: valkey, namespace: platform }
spec:
  selector: { matchLabels: { app: valkey } }
  template:
    metadata: { labels: { app: valkey } }
    spec:
      containers:
        - name: valkey
          image: valkey/valkey:8-alpine
          ports: [{ containerPort: 6379 }]
---
apiVersion: v1
kind: Service
metadata: { name: valkey, namespace: platform }
spec:
  selector: { app: valkey }
  ports: [{ port: 6379 }]
```

`panel.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: calagopus-data, namespace: platform }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn
  resources: { requests: { storage: 5Gi } }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: calagopus
  namespace: platform
  annotations: { reloader.stakater.com/auto: "true" }
spec:
  replicas: 1
  strategy: { type: Recreate }
  selector: { matchLabels: { app: calagopus } }
  template:
    metadata: { labels: { app: calagopus } }
    spec:
      containers:
        - name: panel
          image: ghcr.io/calagopus/panel:1.2.6
          ports: [{ containerPort: 8000 }]
          envFrom: [{ secretRef: { name: calagopus-env } }]
          env:
            - name: DATABASE_URL
              valueFrom: { secretKeyRef: { name: calagopus-db-app, key: uri } }
            - { name: REDIS_URL, value: redis://valkey:6379 }
            - { name: DATABASE_MIGRATE, value: "true" }
            - { name: PORT, value: "8000" }
            - { name: TZ, value: Europe/Amsterdam }
            - { name: APP_PRIMARY, value: "true" }
            - { name: APP_ENABLE_WINGS_PROXY, value: "true" }
            - { name: APP_LOG_DIRECTORY, value: /var/log/calagopus }
          volumeMounts:
            - { name: data, mountPath: /var/lib/calagopus }
            - { name: logs, mountPath: /var/log/calagopus }
      volumes:
        - name: data
          persistentVolumeClaim: { claimName: calagopus-data }
        - name: logs
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata: { name: calagopus, namespace: platform }
spec:
  selector: { app: calagopus }
  ports: [{ port: 80, targetPort: 8000 }]
# ---
# apiVersion: networking.k8s.io/v1
# kind: Ingress
# metadata: { name: calagopus, namespace: platform }
# spec:
#   ingressClassName: traefik
#   rules:
#     - host: panel.k3s.home.klrnbk.nl
#       http:
#         paths:
#           - path: /
#             pathType: Prefix
#             backend: { service: { name: calagopus, port: { number: 80 } } }
```

### Migrating from rpi-2

1. Put the **current** `APP_ENCRYPTION_KEY` from rpi-2's compose file into the Phase app `platform-calagopus`. A new key makes every encrypted value in the database (node tokens, OAuth secrets) unreadable.
2. In `panel.yaml`, set the panel Deployment to `replicas: 0` **in Git**, then push the whole folder. Argo CD creates the database, Valkey, the volume and the secrets, but no panel, so nothing writes an empty schema into the database. (`kubectl scale` doesn't stick: Argo CD's self-heal puts it back.) Check the database is still empty:

   ```bash
   kubectl -n platform get pods | grep -E 'calagopus|valkey'   # calagopus-db-* and valkey, no panel
   PRIMARY=$(kubectl -n platform get cluster calagopus-db -o jsonpath='{.status.currentPrimary}')
   kubectl -n platform exec $PRIMARY -c postgres -- psql -U postgres -d calagopus -c '\dt'
   # "Did not find any relations."  If tables exist, the panel already ran: see step 3
   ```
3. Stop the panel on rpi-2 (Wings keeps running the game servers; only the panel is briefly gone), then import the database. The first command empties the database, needed if the panel ever started against it; on an empty one it's harmless:

   ```bash
   # on rpi-2: dump the old database
   docker exec <calagopus-db> pg_dump -U panel -Fc panel > calagopus.dump

   # make sure the target database is empty
   kubectl -n platform exec $PRIMARY -c postgres -- psql -U postgres \
     -c "DROP DATABASE calagopus WITH (FORCE);" \
     -c "CREATE DATABASE calagopus OWNER calagopus;"

   # restore inside the CNPG primary pod (it has pg_restore in the matching version)
   kubectl -n platform cp calagopus.dump $PRIMARY:/var/lib/postgresql/data/calagopus.dump -c postgres
   kubectl -n platform exec $PRIMARY -c postgres -- \
     pg_restore -U postgres -d calagopus --no-owner --role=calagopus \
     /var/lib/postgresql/data/calagopus.dump
   kubectl -n platform exec $PRIMARY -c postgres -- psql -U postgres -d calagopus -c '\dt' | head -20
   kubectl -n platform exec $PRIMARY -c postgres -- rm /var/lib/postgresql/data/calagopus.dump
   ```
4. Copy the old `./data` folder into the `calagopus-data` volume. With the panel at 0 replicas nothing mounts it, so use a temporary pod. Save as `data-copy.yaml`:

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata: { name: data-copy, namespace: platform }
   spec:
     containers:
       - name: copy
         image: busybox:1.36
         command: [sleep, "3600"]
         volumeMounts: [{ name: data, mountPath: /data }]
     volumes:
       - name: data
         persistentVolumeClaim: { claimName: calagopus-data }
   ```

   ```bash
   # on rpi-2: pack the old data folder (from the panel's compose directory)
   tar czf calagopus-data.tgz -C ./data .

   kubectl apply -f data-copy.yaml
   kubectl -n platform wait --for=condition=Ready pod/data-copy
   kubectl -n platform cp calagopus-data.tgz data-copy:/tmp/data.tgz
   kubectl -n platform exec data-copy -- tar xzf /tmp/data.tgz -C /data
   kubectl -n platform exec data-copy -- ls -la /data
   kubectl -n platform delete pod data-copy
   ```
5. Set `replicas: 1` in `panel.yaml`, push, and test (through the tunnel, or at `https://panel.k3s.home.klrnbk.nl` once the Ingress is uncommented): log in with an existing account and check your servers and the node are listed.
6. Add `panel.klrnbk.nl` to the tunnel (Part 7b). If the panel's URL changes, update it in the panel settings and in Wings' config on the external server, and check the node shows as connected.

### Login through Authentik, invitation-only

1. In Authentik: **Applications > Create with provider**, type **OAuth2/OpenID Connect**, name `Calagopus`. Use the callback URL shown in Calagopus' OAuth provider form as the strict redirect URI, with base `https://panel.klrnbk.nl`.
2. Create a group `calagopus-users` and **bind it to the application** (Application > Policy / Group / User Bindings). Only members can authorize, even with a valid Authentik account.
3. In Calagopus: **Admin > Settings > OAuth providers**, add Authentik with issuer `https://auth.home.klrnbk.nl/application/o/calagopus/` and the client ID and secret, and map the `groups` claim to a Calagopus role.
4. **Invitations**: create an enrollment flow with an Invitation stage (Authentik ships an example flow to import), whose User Write stage adds new users to `calagopus-users`. Then send each player a single-use link from **Directory > Invitations**.
5. Keep one **local Calagopus admin** with 2FA as a break-glass login, so a broken Authentik never locks you out of the panel.

## Part 11c: Media stack

Downloads happen on the NAS behind the VPN (Gluetun + qBittorrent + Prowlarr), the managers run in the cluster (Radarr, Sonarr, Profilarr, Seerr), and Jellyfin stays on the NAS. The one rule that makes it work: **everything sees `/volume1/media` as `/media`**. Downloads and library then sit on one filesystem, so imports are instant hardlinks instead of copies, and no remote path mappings are needed.

```
/volume1/media            ->  /media in every container
├── downloads/
│   ├── incomplete/
│   └── complete/{radarr,sonarr}   # qBittorrent categories
├── movies/                        # Radarr root folder
└── series/                        # Sonarr root folder
```

**File ownership**: run everything with the same UID/GID as the NAS user that owns `/volume1/media` (check with `id <user>` on the NAS; examples use 1000). The NAS containers get `PUID`/`PGID`, the cluster pods the same values.

### On the NAS: Gluetun, qBittorrent, Prowlarr

`compose.yml` in the UGREEN Docker app. qBittorrent and Prowlarr share Gluetun's network, so all their traffic goes through the VPN:

```yaml
services:
  gluetun:
    image: qmcgaw/gluetun:latest
    cap_add: [NET_ADMIN]
    devices: [/dev/net/tun:/dev/net/tun]
    environment:
      - VPN_SERVICE_PROVIDER=<your provider>   # copy your current Gluetun env from rpi-4
      - FIREWALL_OUTBOUND_SUBNETS=192.168.2.0/24   # allow LAN traffic (cluster, Jellyfin)
    extra_hosts:                                   # Gluetun's own DNS doesn't know internal names
      - radarr.k3s.home.klrnbk.nl:192.168.2.20
      - sonarr.k3s.home.klrnbk.nl:192.168.2.20
    ports:
      - 8080:8080      # qBittorrent web UI
      - 9696:9696      # Prowlarr
      - 6881:6881/udp  # torrents (or your provider's forwarded port)
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    network_mode: service:gluetun
    environment: [PUID=1000, PGID=1000, TZ=Europe/Amsterdam, WEBUI_PORT=8080]
    volumes:
      - ./qbittorrent:/config
      - /volume1/media:/media
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    network_mode: service:gluetun
    environment: [PUID=1000, PGID=1000, TZ=Europe/Amsterdam]
    volumes: [./prowlarr:/config]
    restart: unless-stopped
```

Move the existing `/config` folders of all three from rpi-4 so settings and indexers carry over. In qBittorrent set the default save path to `/media/downloads/complete`, the incomplete path to `/media/downloads/incomplete`, and create categories `radarr` and `sonarr`.

### In the cluster: one shared media volume

All four apps live in namespace `media` and share one ReadWriteMany NFS volume. `apps/media/storage/storage.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata: { name: media-nfs }
spec:
  capacity: { storage: 10Ti }
  accessModes: [ReadWriteMany]
  persistentVolumeReclaimPolicy: Retain
  mountOptions: [nfsvers=3]
  csi:
    driver: nfs.csi.k8s.io
    volumeHandle: nas-media
    volumeAttributes: { server: 192.168.2.104, share: /volume1/media }
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: media, namespace: media }
spec:
  accessModes: [ReadWriteMany]
  storageClassName: ""
  volumeName: media-nfs
  resources: { requests: { storage: 10Ti } }
```

### Radarr (Sonarr is identical)

Config (SQLite) goes on Longhorn, never on NFS. `apps/media/radarr/radarr.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: radarr-config, namespace: media }
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn
  resources: { requests: { storage: 2Gi } }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: radarr, namespace: media }
spec:
  replicas: 1
  strategy: { type: Recreate }
  selector: { matchLabels: { app: radarr } }
  template:
    metadata: { labels: { app: radarr } }
    spec:
      containers:
        - name: radarr
          image: lscr.io/linuxserver/radarr:latest   # pin; Renovate bumps it
          env:
            - { name: PUID, value: "1000" }
            - { name: PGID, value: "1000" }
            - { name: TZ, value: Europe/Amsterdam }
          ports: [{ containerPort: 7878 }]
          volumeMounts:
            - { name: config, mountPath: /config }
            - { name: media, mountPath: /media }
      volumes:
        - name: config
          persistentVolumeClaim: { claimName: radarr-config }
        - name: media
          persistentVolumeClaim: { claimName: media }
---
apiVersion: v1
kind: Service
metadata: { name: radarr, namespace: media }
spec:
  selector: { app: radarr }
  ports: [{ port: 80, targetPort: 7878 }]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: radarr, namespace: media }
spec:
  ingressClassName: traefik
  rules:
    - host: radarr.k3s.home.klrnbk.nl
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: radarr, port: { number: 80 } } }
```

The others follow the same pattern:

| App | Image | Port | Config path | Needs `/media` |
| --- | --- | --- | --- | --- |
| Sonarr | `lscr.io/linuxserver/sonarr` | 8989 | `/config` | Yes |
| Profilarr | `santiagosayshey/profilarr` | 6868 | `/config` | No |
| Seerr | `ghcr.io/seerr-team/seerr` | 5055 | `/app/config` (runs as UID 1000) | No |

### Wiring them together

| In | Setting | Value |
| --- | --- | --- |
| Radarr | Root folder | `/media/movies` |
| Sonarr | Root folder | `/media/series` |
| Radarr / Sonarr | Download client qBittorrent | `192.168.2.104:8080`, category `radarr` / `sonarr` |
| Prowlarr | Apps > Radarr / Sonarr | Prowlarr server `http://192.168.2.104:9696`, app server `https://radarr.k3s.home.klrnbk.nl` / `https://sonarr.k3s.home.klrnbk.nl` |
| Profilarr | Arr instances | `http://radarr.media.svc`, `http://sonarr.media.svc` |
| Seerr | Jellyfin | `http://192.168.2.104:8096` |
| Seerr | Radarr / Sonarr | `radarr.media.svc` / `sonarr.media.svc`, port 80 |

**Migrating existing configs**: copy each app's old `/config` folder into its PVC before the first start (temporary pod + `kubectl cp`). If the old containers used different paths than `/media/...`, change the root folders afterwards with Radarr's and Sonarr's mass editor, choosing **not** to move files.

## Part 12: Your own apps, updates and troubleshooting

Your own apps deploy themselves when you push a version tag; third-party apps get update PRs from Renovate that you merge when you're ready. Both replace Tugtainer, with the bonus that every change is a commit you can revert.

### Your own apps: build on GitHub

`.github/workflows/release.yml` in each app repo. Push a tag like `v1.4.0` and it builds an ARM64 image to GHCR:

```yaml
name: release
on:
  push:
    tags: ["v*"]
jobs:
  image:
    runs-on: ubuntu-latest
    permissions: { contents: read, packages: write }
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/ijskoud/${{ github.event.repository.name }}
          tags: type=semver,pattern={{version}}
      - uses: docker/build-push-action@v6
        with:
          push: true
          platforms: linux/arm64
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

QEMU emulation is slow for big builds (Spring Boot, Kotlin). For public repos, `runs-on: ubuntu-24.04-arm` builds natively, and you can drop the QEMU step.

### Your own apps: auto-deploy through your GitHub App

Your GitHub App writes the new version into the gitops repo, and Argo CD's automatic sync (on by default in the ApplicationSet from Part 3) rolls it out. Nothing in the cluster needs write access to Git.

```
git tag v1.5.0 -> GitHub Actions builds ghcr.io/ijskoud/myapp:1.5.0
               -> GitHub App commits newTag: 1.5.0 to homelab-gitops
               -> Argo CD sees the commit (within ~1 min) and deploys it
```

1. Give each own app a `kustomization.yaml` with an `images:` entry. That's the one line the GitHub App changes:

   ```yaml
   # apps/appiecal/myapp/kustomization.yaml   (apps/<project>/<app>/)
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources: [deployment.yaml, service.yaml, ingress.yaml, postgres.yaml]
   images:
     - name: ghcr.io/ijskoud/myapp
       newTag: 1.4.0
   ```
2. Install the GitHub App on `homelab-gitops` with **Contents: read and write** (plus **Pull requests** if it opens PRs instead of committing directly). Store its app ID and private key as secrets in the app repos, never in `homelab-gitops`.
3. If the bump is driven from the release workflow, add this job after the image build in `.github/workflows/release.yml`. It uses the app's token to edit that one line with `yq` (preinstalled on GitHub runners):

   ```yaml
     deploy:
       needs: image
       runs-on: ubuntu-latest
       steps:
         - id: app-token
           uses: actions/create-github-app-token@v2
           with:
             app-id: ${{ vars.DEPLOY_APP_ID }}
             private-key: ${{ secrets.DEPLOY_APP_PRIVATE_KEY }}
             owner: ijskoud
             repositories: homelab-gitops
         - uses: actions/checkout@v4
           with:
             repository: ijskoud/homelab-gitops
             token: ${{ steps.app-token.outputs.token }}
         - name: Bump image tag
           env:
             APP: ${{ github.event.repository.name }}
             PROJECT: appiecal                # the project folder this app lives in
           run: |
             export VERSION="${GITHUB_REF_NAME#v}"
             export IMAGE="ghcr.io/ijskoud/$APP"
             yq -i '(.images[] | select(.name == strenv(IMAGE))).newTag = strenv(VERSION)' "apps/$PROJECT/$APP/kustomization.yaml"
             git config user.name "deploy-bot[bot]"
             git config user.email "deploy-bot[bot]@users.noreply.github.com"
             git commit -am "deploy: $APP $VERSION"
             git push
   ```

   If your GitHub App already does the bumping on its own, it only needs to target that `newTag` line; the rest of the flow is the same.
4. **Private images**: if the GHCR packages are private, the cluster needs a pull secret per namespace (`kubectl create secret docker-registry ghcr-pull ...`, tier 0) referenced as `imagePullSecrets` in the Deployment. Making the packages public avoids that.

Every step is visible: the tag in the app repo, the bump commit in `homelab-gitops`, and the sync in the Argo CD UI. Rolling back is reverting the bump commit.

For the rest of a custom app (Service, Ingress, Postgres), copy the ConvertX files from Part 9 and the Postgres template from Part 5. A Spring Boot app reads its database from the CNPG secret:

```yaml
env:
  - name: SPRING_DATASOURCE_URL
    valueFrom: { secretKeyRef: { name: myapp-db-app, key: jdbc-uri } }
```

### Third-party apps: Renovate

Install the [Renovate GitHub app](https://github.com/apps/renovate) on `homelab-gitops` and add `renovate.json`:

```json
{
	"extends": ["config:recommended"],
	"argocd": { "managerFilePatterns": ["/clusters/pi/.+\\.yaml$/"] },
	"kubernetes": { "managerFilePatterns": ["/(apps|infrastructure)/.+\\.yaml$/"] },
	"packageRules": [{ "matchPackageNames": ["ghcr.io/ijskoud/**"], "enabled": false }]
}
```

The `argocd` manager picks up chart versions in your Applications, the `kubernetes` manager image tags in plain manifests. The package rule leaves your own images to your GitHub App, so the two never bump the same tag.

Renovate then opens a PR for every new Helm chart version (Authentik, Traefik, Longhorn, ...) and container tag (Outline, ConvertX, draw.io, bind9). Merge = deploy, revert = roll back.

### Rollout order

Follows Part 0; tick off as you go.

- [ ] Download stack running on the NAS, stopped on rpi-4
- [ ] Rpi-3 emptied, old k3s uninstalled
- [ ] K3s on all four Pi's next to Docker, `kubectl get nodes` shows 4 Ready
- [ ] Argo CD (UI on argocd.k3s.home.klrnbk.nl), tier 0 secrets created and saved in your password manager, Traefik on 192.168.2.20 with valid certificates (incl. `auth.klrnbk.nl`)
- [ ] Longhorn, NFS, Garage, CNPG; a throwaway database backed up **and restored**
- [ ] Phase self-hosted, recovery kit saved, operator syncing; Reloader running
- [ ] bind9 on 192.168.2.21 with CoreDNS forwarding, Deco switched over
- [ ] Authentik migrated, tunnel live for `auth.klrnbk.nl`, MFA enforced
- [ ] Calagopus panel migrated, `panel.klrnbk.nl` live, Wings node connected, invitations work
- [ ] ConvertX, Outline, draw.io, withoutBG (forward auth in front of draw.io, withoutBG and Longhorn)
- [ ] Radarr, Sonarr, Profilarr, Seerr wired to the NAS
- [ ] Appiecal fleet and other own apps in `apps/`, images built by GitHub Actions
- [ ] Nightly restic to the Storage Box, first restore drill done
- [ ] Docker removed from all Pi's

**The Appiecal fleet** is its own project: one folder per microservice under `apps/appiecal/` (e.g. `apps/appiecal/api/`), each becoming an Application in Argo CD Project `appiecal`, all in namespace `appiecal` so the services reach each other by short name. Give each service with a database its own CNPG cluster from the Part 5 template (`PROJECT` = `appiecal`; `pg_dump` from its current Postgres, import as in Part 7), its secrets in its own Phase app (`appiecal-<service>`), and the release workflow + GitHub App bump above. Another project of your own is just another folder under `apps/`; its Project is created automatically.

### Troubleshooting

| Symptom | Check |
| --- | --- |
| Argo CD didn't apply my change | The app in the UI: **Refresh**, then the status banner and **App Diff**. Or `argocd app get <name>` |
| 404 from Traefik | Ingress host typo, or `ingressClassName` missing |
| 502 / Bad Gateway | `kubectl get endpoints -n <ns>`: empty = Service selector doesn't match pod labels, or wrong `targetPort` |
| Browser shows a certificate warning | `kubectl describe certificate -n traefik wildcard-home`; DNS-01 token permissions |
| Pod stuck `Pending` | `kubectl describe pod`: usually a PVC that can't bind (wrong storageClassName) |
| Pod `CrashLoopBackOff` | `kubectl logs <pod> --previous` |
| `exec format error` | The image has no ARM64 build |
| Middleware not found | `allowCrossNamespace: true` missing in Traefik values |
| Database won't come up | `kubectl get cluster -n <ns>` and `kubectl cnpg status <name> -n <ns>` (install the `kubectl-cnpg` plugin) |

## Part 13: Offsite backups with restic

Every night the NAS takes an encrypted, deduplicated restic snapshot of everything that can't be recreated and sends it to a Hetzner Storage Box (about €4/month for 1 TB). Snapshots keep history, so a deleted or ransomware-encrypted file can be restored from before the damage, which a plain mirror can't do.

| Included | Source on the NAS | Rough size |
| --- | --- | --- |
| Postgres backups of every app, including Phase | Garage bucket `cnpg-backups` (mirrored to a staging folder first) | 1-10 GB |
| Longhorn volume backups (Outline, ConvertX, Calagopus, \*arr configs) | Garage bucket `longhorn-backups` (mirrored to the staging folder first) | 5-30 GB |
| NAS Docker configs (qBittorrent, Prowlarr, Garage, compose files) | `/volume1/docker` | <1 GB |
| **Not** included | `/volume1/media` | Re-downloadable |

Keep these **outside** the backup, in your password manager: the restic password, the tier 0 secrets note from Part 3 (including Phase's `SERVER_SECRET` and Argo CD's deploy key), and the Storage Box login. Lose the restic password and the backup is useless.

### Setup

1. Order the Storage Box, enable **SSH support** in the Hetzner console, and add the NAS's public SSH key.
2. On the NAS, add an SSH config entry (Storage Boxes use port 23):

   ```
   Host storagebox
     HostName uXXXXXX.your-storagebox.de
     User uXXXXXX
     Port 23
   ```
3. Initialise the repository once: `restic -r sftp:storagebox:/restic init`.
4. Add an `rclone.conf` with a `garage` remote (type `s3`, provider `Other`, endpoint `http://192.168.2.104:3900`, region `garage`, a read-only Garage key).

### Nightly script

`/volume1/docker/restic/backup.sh`, scheduled at 04:00 (after the 03:00 CNPG base backups) through the UGOS task scheduler or crontab:

```bash
#!/bin/sh
set -eu
R=/volume1/docker/restic
RUN="docker run --rm -v $R/password:/pw:ro -v $HOME/.ssh:/root/.ssh:ro -v /volume1:/volume1 \
  -e RESTIC_REPOSITORY=sftp:storagebox:/restic -e RESTIC_PASSWORD_FILE=/pw restic/restic"

# 1. Mirror the Postgres and Longhorn backups out of Garage into a plain folder
docker run --rm -v $R/rclone.conf:/config/rclone/rclone.conf:ro -v /volume1/backup-staging:/staging \
  rclone/rclone sync garage:cnpg-backups /staging/cnpg-backups
docker run --rm -v $R/rclone.conf:/config/rclone/rclone.conf:ro -v /volume1/backup-staging:/staging \
  rclone/rclone sync garage:longhorn-backups /staging/longhorn-backups

# 2. Snapshot
$RUN backup /volume1/backup-staging /volume1/docker --tag nightly

# 3. Keep 7 daily, 4 weekly, 6 monthly
$RUN forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune

# 4. Verify a random 5% of the data each night
$RUN check --read-data-subset=5%
```

### Restore drill

Once per quarter, restore one database to prove the chain works: `restic restore latest --target /volume1/restore-test --include /volume1/backup-staging/cnpg-backups/outline`, upload it to a test bucket in Garage, and bootstrap a throwaway CNPG cluster from it (recovery snippet in Part 5).

### Full disaster recovery, in order

1. Pi's: Parts 1 and 2. NAS: restore `/volume1/docker` and the Garage bucket from restic if the NAS itself was lost.
2. Argo CD: run the tier 0 secrets note from your password manager (Part 3, step 3), `helm install` with `infrastructure/argocd/values.yaml`, then `kubectl apply -f bootstrap/root.yaml`.
3. The infrastructure comes up from Git: MetalLB, Traefik, cert-manager, storage, CNPG.
4. **Phase first**: its CNPG cluster restores from the backup (recovery bootstrap), with the original `SERVER_SECRET`.
5. Everything else: the Phase operator fills the app secrets, each CNPG cluster recovers its database, Longhorn restores app volumes from the backup target.

## Part 14: kubectl from anywhere with Tailscale

The Tailscale Kubernetes operator runs in the cluster and joins your tailnet as its own device, `tailscale-operator`. With its **API server proxy** enabled, your laptop talks to the cluster API through that device from any network, with no subnet routes, so a remote network that also uses `192.168.2.x` can't clash. The proxy also logs you in by your Tailscale identity, so the laptop needs no cluster certificate ([docs](https://tailscale.com/docs/kubernetes-operator/api-server-access)).

```
Laptop (anywhere, on Tailscale) -> tailscale-operator (in the cluster) -> K3s API server
```

### Step 1: Prepare your tailnet

In the Tailscale admin console:

1. **DNS**: make sure **MagicDNS** and **HTTPS Certificates** are enabled. The kubectl proxy uses a Tailscale certificate for its HTTPS endpoint.
2. **Access controls**: add the operator's tags, and a grant that makes you cluster admin through the proxy. Save the policy before step 3, because the OAuth client can only use tags that already exist.

   ```json
   "tagOwners": {
     "tag:k8s-operator": [],
     "tag:k8s": ["tag:k8s-operator"]
   },
   "grants": [
     {
       "src": ["autogroup:admin"],
       "dst": ["tag:k8s-operator"],
       "ip": ["tcp:443"],
       "app": {
         "tailscale.com/cap/kubernetes": [{
           "impersonate": { "groups": ["system:masters"] }
         }]
       }
     },
     {
       "src": ["autogroup:member"],
       "dst": ["tag:k8s"],
       "ip": ["tcp:443"]
     }
   ]
   ```

   The first grant is for kubectl (`system:masters` is Kubernetes' built-in admin group). The second lets your devices reach the apps through Traefik's tailnet device (see "Your apps over Tailscale" below). If your policy still has the default allow-all rule, the second grant is optional. If your policy already has a `grants` list, add these entries to it rather than creating a second one.
3. **Create the OAuth client** (this is the "token" for the `operator-oauth` secret):
   1. Go to **Settings > Trust credentials** and click **Generate credential** (pick **OAuth client** if asked for a type).
   2. Under **Scopes**, give **write** access to **General > Services**, **Devices > Core** and **Keys > Auth Keys**. For each, select tag `tag:k8s-operator`.
   3. Click **Generate**. Copy the **client ID** and the **client secret** (it starts with `tskey-client-`). The secret is shown **only once**; put both in your password manager straight away.

   The operator uses this client to add itself and its proxies to your tailnet. OAuth client secrets don't expire like auth keys do, so you only rotate it if it leaks.

### Step 2: Create the OAuth secret (tier 0)

The OAuth client is a secret, so it doesn't go in Git. Create it by hand (also listed in Part 3, step 3):

```bash
kubectl create namespace tailscale
kubectl -n tailscale create secret generic operator-oauth \
  --from-literal=client_id=<oauth client id> --from-literal=client_secret=<tskey-client-...>
```

### Step 3: Install the operator through Argo CD

`clusters/pi/platform/tailscale-operator.yaml`. The OAuth values are left out on purpose: the operator reads the `operator-oauth` secret you just created.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: tailscale-operator
  namespace: argocd
  annotations: { argocd.argoproj.io/sync-wave: "-2" }
spec:
  project: default
  source:
    repoURL: https://pkgs.tailscale.com/helmcharts
    chart: tailscale-operator
    targetRevision: 1.104.1
    helm:
      valuesObject:
        apiServerProxyConfig:
          mode: "true"
  destination: { server: https://kubernetes.default.svc, namespace: tailscale }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=true, ServerSideApply=true]
```

Push it. After a minute, **Machines** in the admin console shows `tailscale-operator` with tag `tag:k8s-operator`. If it doesn't, check `kubectl -n tailscale logs deploy/operator`; an OAuth error there almost always means a typo in the secret or a missing scope.

### Step 4: Point kubectl at it

On your laptop, with Tailscale connected:

```bash
tailscale configure kubeconfig tailscale-operator
kubectl get nodes
```

This adds a context to your kubeconfig next to your existing home one. Switch between them with `kubectl config use-context`; the Tailscale one works both at home and away, so you can also just keep using that. The first request can take a few seconds while the proxy fetches its certificate.

### Your apps over Tailscale

The kubectl proxy only covers the Kubernetes API. To reach your apps away from home with the **same URLs**, Traefik gets a second Service that the operator turns into a tailnet device, `k3s-apps`:

```
At home: convert.k3s.home.klrnbk.nl -> bind9 -> 192.168.2.20 (MetalLB) -> Traefik -> app
Away:    convert.k3s.home.klrnbk.nl -> Cloudflare DNS -> 100.x (k3s-apps, over Tailscale) -> Traefik -> app
```

HTTPS stays exactly as at home: the tailnet device only forwards the encrypted connection, and Traefik serves the Let's Encrypt wildcard certificate from cert-manager. Authentik forward auth applies as usual.

1. Add `infrastructure/configs/traefik-tailscale.yaml` and list it in that folder's `kustomization.yaml`:

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: traefik-tailscale
     namespace: traefik
     annotations:
       tailscale.com/hostname: k3s-apps
   spec:
     type: LoadBalancer
     loadBalancerClass: tailscale
     selector:
       app.kubernetes.io/name: traefik
     ports:
       - { name: websecure, port: 443, targetPort: websecure }
       - { name: web, port: 80, targetPort: web }
   ```
   With every part done, `infrastructure/configs/kustomization.yaml` reads:

   ```yaml
   resources:
     [
       cluster-issuer.yaml,
       wildcard-cert.yaml,
       traefik-tailscale.yaml,
       nfs-storageclass.yaml,
       longhorn-backup-job.yaml,
       longhorn-ingress.yaml,
     ]
   ```

2. Push, then find the device's address: `k3s-apps` appears under **Machines** in the admin console, or run `kubectl -n traefik get svc traefik-tailscale` and read the external IP (a `100.x.y.z` address).
3. In **Cloudflare DNS** for `klrnbk.nl`, add an **A** record `*.k3s.home` pointing at that `100.x.y.z` address, with the proxy status **DNS only** (grey cloud). Cloudflare can't proxy a tailnet address, and it shouldn't.
4. Test away from home (phone on mobile data with Tailscale on): `https://argocd.k3s.home.klrnbk.nl` should load with a valid certificate.

Why this works at home too: your devices ask bind9 first, which answers `192.168.2.20` for `*.k3s.home.klrnbk.nl`, so the public record is never used on your LAN. Two things to know:

- **The hostnames are visible in public DNS**, but they point at a tailnet address that nobody outside your tailnet can reach.
- **Keep the Service** once it exists: if it's deleted and recreated, `k3s-apps` gets a new 100.x address and the Cloudflare record needs updating.

### Good to know

- **Your existing kubeconfig stays** as the break-glass route at home: if the operator pod is down, the Tailscale route is down too, but `https://192.168.2.10:6443` still works on your LAN.
- **Only `autogroup:admin` gets in.** Anyone else on your tailnet reaches the proxy but has no Kubernetes permissions unless you add a grant or RBAC binding for them.
- **Argo CD CLI** works the same way: `argocd login` uses your current kube context with `--core`, or reach the UI through the Argo CD hostname at home.
