# Omni + Proxmox infrastructure provider

Spins up Talos/Kubernetes VMs on the Proxmox cluster (`pve-{1,2,3}.edb-centralen.dk`, Ceph) through
Sidero Omni using [omni-infra-provider-proxmox](https://github.com/siderolabs/omni-infra-provider-proxmox).

| File | Purpose |
|---|---|
| `proxmox-provider/config.yaml.example` | Proxmox API credentials for the provider |
| `proxmox-provider/docker-compose.yaml` | Runs the provider |
| `.env.example` | Omni endpoint + provider key |
| `omni/machine-classes.yaml` | Auto-provision machine classes (CP / worker) |
| `omni/cluster-template.yaml` | 3 control planes + 2 workers |

## Prerequisites to check first

1. **Storage IDs.** `storage_selector` matches the Proxmox *storage ID*, not the Ceph pool name. Check
   with `pvesm status` and adjust `name == "vm-pool"` in `omni/machine-classes.yaml` if it differs.
2. **ISO storage.** The provider has Proxmox download the Talos ISO onto a storage with `iso` content
   (RBD cannot hold ISOs). `local` or a CephFS storage works: Datacenter → Storage → enable *ISO image*.
3. **Network.** PVE nodes need internet access to the image factory (`factory.talos.dev`, or your Omni's
   factory). The VMs (bridge `vmbr0`, DHCP) must reach `https://api.omni.edb-centralen.dk` (443) and
   `edb-centralen.dk:50180/udp` (SideroLink WireGuard).
4. The provider host needs access to the PVE API (8006/tcp) and the Omni API, and can be any Docker host
   (it does not have to be a PVE node).

## 1. Proxmox user, role and API token (on any PVE node)

```bash
pveum role add OmniProvider -privs "VM.Allocate VM.Audit VM.PowerMgmt VM.Console VM.Config.CDROM VM.Config.CPU VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Config.Cloudinit Datastore.Audit Datastore.Allocate Datastore.AllocateSpace Datastore.AllocateTemplate Sys.Audit Sys.Modify SDN.Use Pool.Audit Pool.Allocate"
pveum user add omni@pve
pveum aclmod / -user omni@pve -role OmniProvider
pveum user token add omni@pve provider --privsep 0   # prints the secret once
```

`Sys.Modify` is required by Proxmox for the `download-url` call that fetches the Talos ISO, `Pool.Allocate` for the per-machine-set resource pools the provider creates, and `Datastore.Allocate` for deleting the per-VM cloud-init ISO (`user-data-<vmid>.iso`) on teardown. Without it, deprovisioning loops forever with `not authorized to access endpoint`. To change an existing role, use `pveum role modify OmniProvider -privs "..."`.

Put the token secret in `proxmox-provider/config.yaml` (copy from the `.example`).
If you hit permission errors, check the Proxmox task log; for a quick trial you can use the built-in
`PVEAdmin` role instead and tighten later.

## 2. Register the provider in Omni

```bash
omnictl infraprovider create proxmox
```

Copy the printed `OMNI_ENDPOINT` / `OMNI_SERVICE_ACCOUNT_KEY` into `.env`. This is the *infra provider
key*, not a regular service account key. `OMNI_ENDPOINT` must be the URL you use for `omnictl`/the Omni UI
(assumed `https://omni.edb-centralen.dk`), not the SideroLink machine API URL.

## 3. Run the provider

```bash
cp .env.example .env && cp proxmox-provider/config.yaml.example proxmox-provider/config.yaml
# edit both
docker compose -f proxmox-provider/docker-compose.yaml up -d
docker compose -f proxmox-provider/docker-compose.yaml logs -f
```

"Proxmox" should show up under Omni → Infra Providers as connected.

## 4. Create the cluster

```bash
omnictl apply -f omni/machine-classes.yaml
omnictl cluster template validate -f omni/cluster-template.yaml
omnictl cluster template sync -f omni/cluster-template.yaml
omnictl cluster template status -f omni/cluster-template.yaml
```

Omni requests 5 machines; the provider downloads the ISO, creates the VMs (spread across nodes), and
they register via SideroLink. Tear down with `omnictl cluster template delete -f omni/cluster-template.yaml`
(VMs are removed automatically).

## 5. Test connection to cluster

```
omnictl kubeconfig --cluster <cluster-name> --grant-type=authcode-keyboard
kubectl get nodes
```

## Notes

- Provider options live in the machine class `providerdata` (`cores`, `memory`, `disk_size`,
  `storage_selector` are required, and set `sockets: 1` too (the provider sends 0 otherwise and Proxmox rejects it); see the provider's `data/schema.json`). The README example in the
  provider repo (`cpu:`, `diskSize:`) is outdated.
- Talos `v1.13.12` / Kubernetes `v1.36.3` are used; the `qemu-guest-agent` extension is added by the provider.
- For Proxmox HA (PVE 9+) add an `ha:` block to the machine class; this disables the provider's own
  node spreading.
