# oVirt Lab Architecture

> **Self-Hosted Engine on Bare-Metal Nodes**

A complete reference for building an oVirt 4.5 lab environment using the
Self-Hosted Engine (SHE) model. The Engine VM runs inside the cluster it manages,
eliminating the need for a dedicated physical management server.

---
## Contents

| File | Description |
|------|-------------|
| `docs/ovirt_architecture.pdf` | Full architecture & build guide |

## What's covered

- Infrastructure component table (DNS, NFS, 3 nodes, Engine VM)
- Hardware & OS requirements
- DNS setup (BIND9, forward + reverse zones)
- NFS shared storage configuration
- oVirt host preparation (all 3 nodes)
- Self-Hosted Engine deployment (`hosted-engine --deploy`)
- Adding compute hosts (ovirt2, ovirt3)
- Storage domain configuration
- Network port reference
- Full verification checklist
- Software architecture: kernel to user space (Layer 1–7)
- "Start VM" call flow trace through all layers

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

## Option A — Ansible (automated)



#### Repository Structure
  
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

#### Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/ahmedtmam/ovirt-lab-architecture.git
cd ovirt-lab-architecture

# 2. Install Ansible dependencies
pip install ansible
ansible-galaxy collection install ovirt.ovirt

# 3. Edit the inventory
vim ansible/inventory/hosts.ini # Verify IPs match your environment

# 4. Run the full site playbook

# DNS first — all other steps depend on it
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/01_dns.yml
# NFS storage
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/02_nfs.yml

# Prepare oVirt hosts
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/03_ovirt_hosts.yml

# Or run everything at once
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/04_site.yml

# 5. Deploy the hosted engine (manual step)
# SSH to ovirt1 and run:
hosted-engine --deploy
```

Follow the wizard using the values in `docs/ovirt_architecture.pdf` Section 6.

---
## Option B — Bash (manual, step by step)
#### Step 1 — DNS Server (run on dns.lab.local — 192.168.5.140)

```bash
dnf install -y bind bind-utils

cat > /etc/named.conf << 'EOF'
options {
    listen-on port 53 { 127.0.0.1; 192.168.5.140; };
    allow-query     { 192.168.5.0/24; 127.0.0.1; };
    allow-recursion { 192.168.5.0/24; 127.0.0.1; };
    recursion yes;
    allow-update { none; };
    forwarders { 8.8.8.8; 8.8.4.4; };
    dnssec-validation no;
};

zone "lab.local" IN {
    type master;
    file "/var/named/lab.local.zone";
};
zone "5.168.192.in-addr.arpa" IN {
    type master;
    file "/var/named/5.168.192.arpa";
};
EOF

cat > /var/named/lab.local.zone << 'EOF'
$TTL 86400
@   IN SOA dns.lab.local. admin.lab.local. (
              2026051101 3600 1800 604800 86400 )
    IN NS   dns.lab.local.
dns     IN A  192.168.5.140
nfs     IN A  192.168.5.141
ovirt1  IN A  192.168.5.142
ovirt2  IN A  192.168.5.143
ovirt3  IN A  192.168.5.144
engine  IN A  192.168.5.145
EOF

cat > /var/named/5.168.192.arpa << 'EOF'
$TTL 86400
@   IN SOA dns.lab.local. admin.lab.local. (
              2026051101 3600 1800 604800 86400 )
    IN NS   dns.lab.local.
140 IN PTR  dns.lab.local.
141 IN PTR  nfs.lab.local.
142 IN PTR  ovirt1.lab.local.
143 IN PTR  ovirt2.lab.local.
144 IN PTR  ovirt3.lab.local.
145 IN PTR  engine.lab.local.
EOF

chown root:named /etc/named.conf /var/named/lab.local.zone /var/named/5.168.192.arpa
chmod 640        /etc/named.conf /var/named/lab.local.zone /var/named/5.168.192.arpa

systemctl enable --now named
firewall-cmd --permanent --add-service=dns && firewall-cmd --reload

# Verify — all 6 hosts must resolve forward AND reverse
for host in dns nfs ovirt1 ovirt2 ovirt3 engine; do
    nslookup ${host}.lab.local 127.0.0.1 | awk '/^Name:|^Address: / {print}' 
done
```

---

#### Step 2 — NFS Server (run on nfs.lab.local — 192.168.5.141)

```bash
dnf install -y nfs-utils

# Directories MUST be owned by UID/GID 36 (vdsm:kvm) — non-negotiable
mkdir -p /exports/ovirt-data /exports/ovirt-iso /exports/ovirt-hosted-engine
chown -R 36:36 /exports/ovirt-data /exports/ovirt-iso /exports/ovirt-hosted-engine
chmod 0755     /exports/ovirt-data /exports/ovirt-iso /exports/ovirt-hosted-engine

cat > /etc/exports << 'EOF'
/exports/ovirt-data           192.168.5.0/24(rw,sync,no_root_squash,no_subtree_check)
/exports/ovirt-iso            192.168.5.0/24(rw,sync,no_root_squash,no_subtree_check)
/exports/ovirt-hosted-engine  192.168.5.0/24(rw,sync,no_root_squash,no_subtree_check)
EOF

systemctl enable --now rpcbind nfs-server
exportfs -rav

firewall-cmd --permanent --add-service={nfs,rpc-bind,mountd}
firewall-cmd --reload

# Verify
showmount -e localhost
```

---

#### Step 3 — Prepare oVirt Hosts (run on ovirt1, ovirt2, ovirt3 — one at a time)

```bash
# Change hostname to match the node — ovirt1 / ovirt2 / ovirt3
hostnamectl set-hostname ovirt1.lab.local

# Point DNS resolver at dns.lab.local
nmcli con mod <interface> ipv4.dns 192.168.5.140 ipv4.dns-search lab.local && nmcli con up <interface>

# Confirm DNS works from this host
nslookup  $(hostname)
nslookup  $(hostname -I)

# Time synchronisation
dnf install -y chrony && systemctl enable --now chronyd

# Confirm CPU virtualisation extensions are present
grep -E 'vmx|svm' /proc/cpuinfo | head -2
# No output = VT-x/AMD-V is OFF in BIOS — stop and fix before proceeding

# Add oVirt 4.5 repository
dnf install -y https://resources.ovirt.org/pub/yum-repo/ovirt-release45.rpm
dnf module enable -y javapackages-tools pki-deps
dnf distro-sync -y

# Install oVirt host packages (VDSM is included, do NOT start it manually)
dnf install -y ovirt-host cockpit-ovirt-dashboard

systemctl enable --now firewalld

# Confirm NFS is reachable from this host
showmount -e nfs.lab.local
```

---

#### Step 4 — Deploy oVirt Engine (run on ovirt1.lab.local ONLY)

```bash
dnf install -y ovirt-hosted-engine-setup
hosted-engine --deploy
```

> Follow the interactive wizard. Full answer reference is in
> `docs/ovirt_lab_architecture_v2.docx` **Section 6**.
>
> Key values:
> | Prompt | Value |
> |--------|-------|
> | Storage type | `nfs` |
> | NFS path | `nfs.lab.local:/exports/ovirt-hosted-engine` |
> | Engine FQDN | `engine.lab.local` |
> | Engine IP | `192.168.5.145` |
> | Engine admin password | *(choose a strong password)* |

---

#### Step 5 — Add ovirt2 and ovirt3 to the cluster

```bash
# Confirm engine VM is up before touching the Web UI
hosted-engine --vm-status
hosted-engine --check-liveliness
```

Then open the Web UI: **https://engine.lab.local/ovirt-engine**

```
Compute → Hosts → New
  Name:       ovirt2
  Hostname:   ovirt2.lab.local
  SSH Port:   22
  Password:   <root password>
  → OK  (wait for status: Up)

Repeat for ovirt3.
```

---

#### Step 6 — Full verification (run on all hosts)

```bash
# Every command must succeed before the lab is considered healthy
nslookup engine.lab.local        # → 192.168.5.145
nslookup 192.168.5.145           # → engine.lab.local
showmount -e nfs.lab.local       # → shows 3 export paths
hosted-engine --vm-status        # → Engine VM: Up
hosted-engine --check-liveliness # → Engine is alive
```
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

See `docs/ovirt_architecture.pdf` Section 12 for the full layer-by-layer breakdown.

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

