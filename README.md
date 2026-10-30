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
4. **Omni infrastructure provider for Proxmox.** Must be configured with token for Proxmox with required 
   permissions and registered to Omni. This is set up in a separate repo: 
   https://github.com/wt2v/mgmt/tree/main/services/omni-infra-provider 

## 1. Create the cluster

```bash
omnictl apply -f omni/machine-classes.yaml
omnictl cluster template validate -f omni/cluster-template.yaml
omnictl cluster template sync -f omni/cluster-template.yaml
omnictl cluster template status -f omni/cluster-template.yaml
```

Omni requests 5 machines; the provider downloads the ISO, creates the VMs (spread across nodes), and
they register via SideroLink. Tear down with `omnictl cluster template delete -f omni/cluster-template.yaml`
(VMs are removed automatically).

## 2. Test connection to cluster

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
