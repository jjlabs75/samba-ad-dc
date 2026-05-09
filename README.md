# Samba 4 AD DC - Docker image

[![Docker](https://img.shields.io/badge/docker-ready-blue?logo=docker)](https://www.docker.com/)
[![Debian](https://img.shields.io/badge/debian-13--slim-red?logo=debian)](https://www.debian.org/)
[![BackupPC](https://img.shields.io/badge/Samba--AD--DC-4.22-green)](https://backuppc.github.io/backuppc/)

Containerized [Samba Active Directory](https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller) on `debian:13-slim`.  
Installed from the official Debian repository.

---

<br>

> ⚠️ **Security notice:** This container runs with `privileged: true` because Samba Active Directory Domain Controller requires elevated privileges to manage ACLs, extended attributes, Kerberos, DNS, and other system-level services. As a result, Docker does **not provide strong isolation** in this setup. This project uses Docker to simplify deployment and maintenance, **not as a security boundary**. It is strongly recommended to run this container on a **dedicated host or virtual machine** and to avoid hosting other applications or services on the same system.

---

<br>

## Project Structure

```bash
.
├── build
│   ├── Dockerfile
│   ├── entrypoint.sh
│   ├── samba-join
│   └── samba-provision
├── .env.example
├── docker-compose.yml
└── README.md
```

---

<br>

## Quick Start

### Clone the repo

```bash
git clone https://github.com/jjlabs75/samba-ad-dc.git
```

```bash
cd samba-ad-dc
```

<br>

### Build the image from source

```bash
sudo docker build -t samba-ad-dc:4.22 ./build
```

<br>

### Create the environment file

Create a `.env` file:

```bash
cp .env.example .env
```

<br>

Required variables:

| Variable         | Example          | Description                     |
| ---------------- | ---------------- | ------------------------------- |
| `DOMAIN_REALM`   | `COMPANY.LAN`    | Full domain name                |
| `DOMAIN_NAME`    | `COMPANY`        | NetBIOS domain name             |
| `DNS_FORWARDER`  | `1.1.1.1`        | DNS forwarder server            |
| `DOCKER_HOST_IP` | `192.168.0.10`   | Host server IP                  |
| `DOMAIN_NETWORK` | `192.168.0.0/24` | Domain network                  |
| `TIMEZONE`       | `Europe/Paris`   | Time zone                       |

<br>
<br>

Persistent paths:

| Variable | Example | Description |
|----------|---------|-------------|
| `SAMBA_CONFIG_DIR` | `./data/samba/config` | Samba configuration files (`smb.conf`, etc.) |
| `SAMBA_DATA_DIR` | `./data/samba/lib` | Active Directory database, SYSVOL, and internal Samba data |
| `SAMBA_LOGS_DIR` | `./data/samba/logs` | Samba log files |
| `SAMBA_BACKUP_DIR` | `./data/samba/backups` | Domain backup files |

⚠️ `SAMBA_DATA_DIR` contains the Active Directory database.<br>
Only delete this directory if you know exactly what you are doing.

---

<br>

### Start the container

```bash
sudo docker compose up -d
```

```bash
sudo docker compose logs
```

At startup the container:

* configures `/etc/hosts`
* configures `/etc/resolv.conf`
* links `/etc/krb5.conf`
* starts `chrony`
* waits for domain provisioning if needed

---

<br>

## Domain Setup

### Option A — Provision a New Domain

```bash
sudo docker compose exec -it dc samba-provision
```
`samba-provision` is a script that runs `samba-tool domain provision` using parameters defined via environment variables. The container will detect the provisioning and start within a few seconds.

```bash
sudo docker compose logs
```

#### Set the Administrator account password
```bash
sudo docker compose exec -it dc samba-tool user setpassword administrator
```

#### Disable password expiration for the Administrator account

```bash
sudo docker compose exec -it dc samba-tool user setexpiry administrator --noexpiry
```

#### Create a reverse DNS zone

```bash
sudo docker compose exec -it dc \
   samba-tool dns zonecreate dc01 0.168.192.in-addr.arpa --username=administrator
```
This creates the reverse DNS zone `0.168.192.in-addr.arpa`, which corresponds to the `192.168.0.0/24` subnet, on the domain controller `dc01`.

---

<br>

### Option B — Join an Existing Domain

```bash
sudo docker compose exec -it dc samba-join --server=dc01 -U"JJLABS\administrator"
```
`samba-join` is a script that runs `samba-tool domain join` using parameters defined via environment variables. The container will detect the join process and start within a few seconds.
- `--server`: target domain controller  
- `-U`: account used to join the domain

```bash
sudo docker compose logs
```

---

<br>

### Option C — Manual Commands

```bash
sudo docker compose exec -it dc bash
```

Then:

```bash
samba-tool --help
```

---

<br>

## Useful Commands

### Follow logs

```bash
sudo docker compose logs -f dc
```

### Check Samba processes

```bash
sudo docker compose exec dc samba-tool processes
```

### Show help

```bash
sudo docker compose exec dc samba-tool --help
```

### List computers

```bash
sudo docker compose exec dc samba-tool computer list
```

### List users

```bash
sudo docker compose exec dc samba-tool user list
```

### List groups

```bash
sudo docker compose exec dc samba-tool group list
```

### Backup the AD database

```bash
sudo docker compose exec dc \
   samba-tool dbcheck --cross-ncs
```

Verifies the consistency of the Samba Active Directory database, including cross-checks between naming contexts (Domain, Configuration, and Schema partitions), to detect broken references or orphaned objects before performing a backup.

```bash
sudo docker compose exec dc \
   samba-tool domain backup online --server dc01 --username administrator --targetdir=/backups
```

Creates a consistent online backup of the Samba Active Directory domain controller `dc01` while services are running.  
The backup includes the Active Directory database, SYSVOL, and all required metadata for restoration.

### Open shell

```bash
sudo docker compose exec -it dc bash
```

---

<br>

## Container Settings

The container uses:

```yaml
privileged: true
network_mode: host
```

✅ Recommended for Samba AD because it allows:

* Kerberos to work correctly
* LDAP to bind properly
* DNS SRV records to resolve correctly
* no NAT issues
* no manual port mapping

⚠️ Requires a **dedicated Linux host or VM**

Recommended platforms:

* [Proxmox VM](https://www.jjworld.fr/proxmox-virtual-environment-pve-installation)
* Dedicated Linux server

---

<br>

## Samba AD Ports

| Port        | Protocol | Service              |
| ----------- | -------- | -------------------- |
| 53          | TCP/UDP  | DNS                  |
| 88          | TCP/UDP  | Kerberos             |
| 135         | TCP      | RPC Endpoint Mapper  |
| 137         | UDP      | NetBIOS Name Service |
| 138         | UDP      | NetBIOS Datagram     |
| 139         | TCP      | NetBIOS Session      |
| 389         | TCP/UDP  | LDAP                 |
| 445         | TCP      | SMB                  |
| 464         | TCP/UDP  | Kerberos Password    |
| 636         | TCP      | LDAPS                |
| 3268        | TCP      | Global Catalog LDAP  |
| 3269        | TCP      | Global Catalog LDAPS |
| 49152-65535 | TCP      | Dynamic RPC          |

---

<br>

## Upgrade Samba

Backup first:


```bash
sudo docker compose exec dc \
   samba-tool dbcheck --cross-ncs
```

```bash
sudo docker compose exec dc \
   samba-tool domain backup online --server dc01 --username administrator --targetdir=/backups
```

Rebuild:

```bash
sudo docker compose build --no-cache
sudo docker compose up -d
```

---

<br>

## Recommended Host Requirements

For production:

* **OS**: Debian / Ubuntu
* **RAM**: 2 GB minimum
* **CPU**: 2 vCPU minimum
* **Disk**: 20 GB minimum
* **Network**: Static IP required
* **Docker**: 24+
* **Docker Compose**: v2+

---

<br>

## 🚨 Troubleshooting

### Container stops immediately

```bash
sudo docker compose logs dc
```

---

<br>

### Port 53 already in use

Disable `systemd-resolved` DNS stub on the **host**:

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
echo -e "[Resolve]\nDNSStubListener=no" | sudo tee /etc/systemd/resolved.conf.d/no-stub.conf
sudo systemctl restart systemd-resolved
```

---

<br>

## 📚 Resources

* 📖 [Original tutorial](https://www.jjworld.fr/active-directory-linux-avec-samba-4/)
* 📘 [Official Samba documentation](https://wiki.samba.org/)
* 🏢 [Samba AD DC setup guide](https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller)
