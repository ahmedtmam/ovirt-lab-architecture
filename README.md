# oVirt Lab Architecture

> **Self-Hosted Engine on Bare-Metal Nodes**

A complete reference for building an oVirt 4.5 lab environment using the
Self-Hosted Engine (SHE) model. The Engine VM runs inside the cluster it manages,
eliminating the need for a dedicated physical management server.

---

## Lab Infrastructure

| Component    | Hostname           | IP Address     | Role                  |
|--------------|--------------------|----------------|-----------------------|
| DNS Server   | dns.lab.local      | 192.168.5.140  | Name Resolution       |
| NFS Server   | nfs.lab.local      | 192.168.5.141  | Shared Storage        |
| oVirt Node 1 | ovirt1.lab.local   | 192.168.5.142  | Engine Host (primary) |
| Engine VM    | engine.lab.local   | 192.168.5.145  | oVirt Engine          |
| oVirt Node 2 | ovirt2.lab.local   | 192.168.5.143  | Compute Host          |
| oVirt Node 3 | ovirt3.lab.local   | 192.168.5.144  | Compute Host          |

- **Network:** `192.168.5.0/24` — `lab.local` domain
- **OS:** AlmaLinux 8 / RHEL 8 / Centos 8
- **oVirt:** 4.5.x (latest stable)

---

## Repository Structure
  
```
ovirt-lab-architecture/
├── README.md                        # This file
├── docs/
│   └── ovirt_lab_architecture_v2.docx   # Full architecture & build guide
├── configs/
│   ├── dns/
│   │   ├── named.conf               # BIND9 main config
│   │   ├── lab.local.zone           # Forward zone
│   │   └── 5.168.192.arpa           # Reverse zone
│   ├── nfs/
│   │   └── exports                  # /etc/exports for NFS server
│   └── hosts/
│       └── hosts                    # /etc/hosts fallback for all nodes
└── ansible/
    ├── inventory/
    │   └── hosts.ini                # Ansible inventory
    ├── playbooks/
    │   ├── 01_dns.yml               # Configure DNS server
    │   ├── 02_nfs.yml               # Configure NFS server
    │   ├── 03_ovirt_hosts.yml       # Prepare oVirt nodes
    │   └── 04_site.yml              # Full site playbook (runs all)
    └── roles/
        ├── dns/                     # BIND9 role
        ├── nfs/                     # NFS server role
        ├── ovirt-host/              # oVirt node preparation role
        └── ovirt-engine/            # Post-engine configuration role
```

---

## Quick Start

### 1. Clone the repository
```bash
git clone https://github.com/ahmedtmam/ovirt-lab-architecture.git
cd ovirt-lab-architecture
```

### 2. Install Ansible dependencies
```bash
pip install ansible
ansible-galaxy collection install ovirt.ovirt
```

### 3. Edit the inventory
```bash
vi ansible/inventory/hosts.ini
# Verify IPs match your environment
```

### 4. Run the full site playbook
```bash
# DNS first — all other steps depend on it
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/01_dns.yml

# NFS storage
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/02_nfs.yml

# Prepare oVirt hosts
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/03_ovirt_hosts.yml

# Or run everything at once
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/04_site.yml
```

### 5. Deploy the hosted engine (manual step)
```bash
# SSH to ovirt1 and run:
hosted-engine --deploy
```
Follow the wizard using the values in `docs/ovirt_lab_architecture_v2.docx` Section 6.

---

## Software Architecture (Kernel to User Space)

```
┌─────────────────────────────────────────────┐  Layer 7
│   Web UI / REST API / Ansible / Python SDK  │  (User)
├─────────────────────────────────────────────┤  Layer 6
│   oVirt Engine — WildFly + PostgreSQL        │
├─────────────────────────────────────────────┤  Layer 5
│   VDSM — Host Agent (port 54321)            │
├─────────────────────────────────────────────┤  Layer 4
│   libvirt — Hypervisor Abstraction API      │
├──────────────────────┬──────────────────────┤  Layer 3
│   QEMU (per-VM proc) │   KVM (/dev/kvm)     │
├─────────────────────────────────────────────┤  Layer 2
│   Linux Kernel EL8 — kvm.ko, virtio, NFS   │
├─────────────────────────────────────────────┤  Layer 1
│   Physical Hardware — VT-x / AMD-V CPU      │  (HW)
└─────────────────────────────────────────────┘
        ↕ NFS v4.1 — nfs.lab.local
┌─────────────────────────────────────────────┐
│  Shared Storage: data | iso | hosted-engine │
└─────────────────────────────────────────────┘
```

See `docs/ovirt_lab_architecture_v2.docx` Section 12 for the full layer-by-layer breakdown.

---

## Prerequisites Checklist

- [ ] All hostnames resolve forward and reverse via `dns.lab.local`
- [ ] NFS exports visible from all nodes (`showmount -e nfs.lab.local`)
- [ ] CPU virtualization enabled in BIOS (`grep -E 'vmx|svm' /proc/cpuinfo`)
- [ ] SELinux in Enforcing mode on all hosts
- [ ] Time synchronized (chrony) across all nodes
- [ ] oVirt 4.5 repo enabled on all hosts

---

## References

- [oVirt Documentation](https://www.ovirt.org/documentation/)
- [Red Hat Virtualization 4.4 Admin Guide](https://access.redhat.com/documentation/en-us/red_hat_virtualization/)
- [Ansible ovirt.ovirt Collection](https://galaxy.ansible.com/ovirt/ovirt)
- [VDSM Developer Guide](https://github.com/oVirt/vdsm)

