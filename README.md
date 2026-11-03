# Omni + Proxmox infrastructure provider

Spins up Talos/Kubernetes VMs on the Proxmox cluster (`pve-{1,2,3}.edb-centralen.dk`, Ceph) through
Sidero Omni using [omni-infra-provider-proxmox](https://github.com/siderolabs/omni-infra-provider-proxmox).

Also: https://www.youtube.com/watch?v=0z7s9kBA2V4&t=1096s

## Prerequisites to check first

1. **Storage IDs.** `storage_selector` matches the Proxmox *storage ID*, not the Ceph pool name. Check
   with `pvesm status` and adjust `name == "vm-pool"` in `cluster/machine-classes/*.yaml` if it differs.
2. **ISO storage.** The provider has Proxmox download the Talos ISO onto a storage with `iso` content
   (RBD cannot hold ISOs). `local` or a CephFS storage works: Datacenter → Storage → enable *ISO image*.
3. **Omni infrastructure provider for Proxmox.** Must be configured with token for Proxmox with required 
   permissions and registered to Omni. This is set up in a separate repo: 
   https://github.com/wt2v/mgmt/tree/main/services/omni-infra-provider
4. **SSH-keys.** Create keys for ArgoCD to deploy.
   `ssh-keygen -t ed25519 -C "argocd-deploy-key" -f argocd_deploy_key -N ""`
   Add the public key as a read-only deploy key in the GitHub UI: Repo → Settings → Deploy keys → Add deploy key
5. Create service account in omni
   `omnictl serviceaccount create --use-user-role=false --role Operator github-actions`

Repository secrets:

| Secret | Purpose |
|---|---|
| `OMNI_ENDPOINT` | Omni URL, e.g. `https://omni.edb-centralen.dk` |
| `OMNI_SERVICE_ACCOUNT_KEY` | Omni service account key (`omnictl serviceaccount create --use-user-role=false --role Operator github-actions`) |
| `ARGOCD_REPO_SSH_KEY` | Private half of a read-only deploy key on this repo, so Argo CD can clone it |
| `TS_OAUTH_CLIENT_ID` | 
| `TS_OAUTH_SECRET` | 

## 1. Create the cluster and bootstrap the platform

Run the **Create cluster** workflow (Actions → Run workflow). It has two jobs; the steps are in
[create_cluster.yaml](.github/workflows/create_cluster.yaml) if you want to run them locally.

1. **cluster:** renders `cluster/cluster-templates/cilium.yaml` (same chart version and values Argo CD uses later), applies the
   machine classes, syncs the cluster template and waits until Omni reports the cluster ready.
2. **platform:** installs Argo CD with Helm, creates the repo credentials and applies `platform/appset.yaml`.
   From then on Argo CD manages Cilium, itself and every other `platform/<app>/` folder from `main`; it adopts
   the Cilium objects Omni applied, since it renders the same chart version and values. If this job fails, you
   can re-run just it.

Omni requests 6 machines; the provider downloads the ISO, creates the VMs (spread across nodes), and
they register via SideroLink. Omni applies the Cilium manifest once (`mode: one-time`) when the Kubernetes
API is up, which makes the nodes Ready. Tear down with `omnictl cluster template delete -f cluster/cluster-templates/cluster-template.yaml`
(VMs are removed automatically).

To add a platform app, create a folder `platform/<app>/` with an `app.yaml` (name, namespace, chart
repo/name/version) and a `values.yaml`. Bump versions in `app.yaml`; the workflow and Argo CD read the same
files. Changes to `platform/appset.yaml` itself must be applied manually with `kubectl apply`.

## 2. Test connection to cluster

```
omnictl kubeconfig --cluster <cluster-name> --grant-type=authcode-keyboard
kubectl get nodes
```

## Notes

- For Proxmox HA (PVE 9+) add an `ha:` block to the machine class; this disables the provider's own
  node spreading.
