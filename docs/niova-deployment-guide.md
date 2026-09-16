# Niova Deployment Guide

This guide walks you through deploying a **Niova test cluster** end to end:

1. Bring up the **Control Plane (CP)** as a single-node container.
2. Register your **physical infrastructure** and create **virtual devices (VDEVs)**.
3. Install and start the **Niova server (NISD)** and **Niova client**.
4. (Optional) Create **tenants** and run **NBT** tests.

> **Deployment methods for NISD.** There are two ways to run the NISD (Niova server):
> **(A) NISD on bare metal / host** (covered in this guide) and **(B) NISD in a
> container** (documented separately, coming soon). The Control Plane itself always
> runs as a container.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Part 1 — Control Plane Setup (Single-Node Container)](#part-1--control-plane-setup-single-node-container)
  - [Step 1 — Download the container image](#step-1--download-the-container-image)
  - [Step 2 — Extract the setup tarball](#step-2--extract-the-setup-tarball)
  - [Step 3 — Start the container](#step-3--start-the-container)
  - [Step 4 — Verify Serf discovery (optional)](#step-4--verify-serf-discovery-optional)
  - [Step 5 — Log in as admin](#step-5--log-in-as-admin)
  - [Step 6 — Insert infra keys](#step-6--insert-infra-keys)
  - [Step 7 — Create a VDEV](#step-7--create-a-vdev)
  - [Step 8 — Create a VDEV under a PFS (optional)](#step-8--create-a-vdev-under-a-pfs-optional)
- [Part 2 — Query the Control Plane](#part-2--query-the-control-plane)
- [Part 3 — Niova Server (NISD) & Client Setup](#part-3--niova-server-nisd--client-setup)
  - [Download the Niova packages](#download-the-niova-packages)
  - [Install niova-block RPMs](#install-niova-block-rpms)
  - [Install Serf (Rocky Linux 10.2)](#install-serf-rocky-linux-102)
  - [Start the NISD process (Niova server)](#start-the-nisd-process-niova-server)
  - [Start the Niova client](#start-the-niova-client)
- [Part 4 — Multi-Tenancy](#part-4--multi-tenancy)
- [Part 5 — Running NBT Tests](#part-5--running-nbt-tests)

---

## Architecture Overview

The **Niova Control Plane (CP)** is Niova's highly available metadata service. It owns
the durable record of physical infrastructure (PDUs, racks, hypervisors, devices, NISDs)
and the logical-to-physical mapping for every virtual device (VDEV). It is backed by
**TiDB** (the `mdsvc-tidb` / `mdsvc-api` service) and advertises itself over **Serf**
gossip so Niova clients can discover a live CP endpoint without static configuration.

**Single-node CP layout:** one Docker container running an embedded single-node TiDB
(via `tiup playground`) + `mdsvc-api` + Serf.

---

## Prerequisites

Install the following on the host:

- **Docker** + **Docker Compose v2** (`docker compose`).
- **Serf**, **curl**, **jq**, and a **mysql/mariadb** client.
- **uuidgen** (or any UUID generator) — used for `MDSVC_INSTANCE_ID` and test UUIDs.

Supported container architectures: **amd64** / **arm64**.

---

## Part 1 — Control Plane Setup (Single-Node Container)

The setup ships a `docker-compose.yml` that runs a single container with an embedded
TiDB, a multi-tenant `mdsvc-api`, and Serf.

### Step 1 — Download the container image

**1.a** Pull the image for your architecture (`amd64` or `arm64`):

```bash
docker pull bhanu0407/niova-mdsvc:v1.0.1.<arch>
```

**1.b** Tag the image as `mdsvc-tidb:latest`:

```bash
docker tag bhanu0407/niova-mdsvc:v1.0.1.<arch> mdsvc-tidb:latest
```

**1.c** Verify the image is available locally:

```bash
docker images | grep mdsvc-tidb
```

### Step 2 — Extract the setup tarball

```bash
mkdir -p mdsvc-tidb && cd mdsvc-tidb
tar -xzvf /path/to/niova-mdsvc-setup.tar.gz
```

Extracting the setup package creates:

- `docker-compose.yml`
- `.env`
- `niova-infra-keys.json`

> **Notes**
> - `docker-compose.yml` includes default settings for local development, so **no
>   configuration changes are required** to get started. To see the default values,
>   read the `.env` file.
> - To customize the configuration, edit `.env` and keep it in the **same directory**
>   as `docker-compose.yml` — Docker Compose loads it automatically.
> - Before using this setup outside local testing, update: `JWT_SECRET`,
>   `ADMIN_DEFAULT_PASSWORD`, and `TENANT_ADMIN_PASSWORD`.
> - The service uses the local image tagged `mdsvc-tidb:latest` (`pull_policy: never`).
>   If your image has a different tag, retag it before starting the container.

### Step 3 — Start the container

```bash
docker compose up -d
docker logs -f mdsvc-tidb
```

Wait for **`TiDB to be ready!`** and **`Starting mdsvc-api server`** in the logs.

### Step 4 — Verify Serf discovery (optional)

```bash
serf members -rpc-addr=127.0.0.1:8029 -rpc-auth=dummy -tag Type=niova-mdsvc -format json -status alive
```

> No output? Try the other RPC ports: `8025`, `8026`, `8027`, `8028`, `8030`.

### Step 5 — Log in as admin

Logging in returns a token used for subsequent CP operations.

> The username and password must match the values in `.env`.

```bash
BASE="${MDSVC_API_URL:-http://localhost:8081}"

TA_TOKEN=$(
  curl -s -X POST "$BASE/users/login" \
    -H "Content-Type: application/json" \
    -d '{"username":"admin","password":"admin"}' \
  | jq -r '.payload.access_token'
)
```

Validate the token:

```bash
echo $TA_TOKEN
```

Example output:

```text
eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJ1aWQiOiJkOWFlNmFlOC02NmU2LTRkY2YtOTRhYi1mZTA0MDM3ODdlYjkiLCJzdWIiOiJhZG1pbiIsInJvbGUiOiJhZG1pbiIsInR2IjowLCJpc3MiOiJtZHN2Yy10aWRiIiwiYXVkIjpbIm1kc3ZjLXRpZGIiXSwiZXhwIjoxNzg1MzE1ODM1LCJpYXQiOjE3ODUzMTQ5MzV9.rLra25zdC7OhE67hnTYG-F1oOWKTttMPnTPHgbCye8faVQf_NzptgzRnsT3n-lY1abWLecqaOK0SeOy7yXYr1Q
```

### Step 6 — Insert infra keys

This creates the physical hierarchy (PDU → Rack → Hypervisor → Device → Partition →
NISD), scoped to this tenant.

Edit `niova-infra-keys.json` from the extracted setup package to describe your infra.
**Every entity (PDU, Rack, Hypervisor, Device, Partition, NISD) must have a unique
UUID.** Update the file to match your environment.

```jsonc
{
  "pdus": [
    {
      "name": "PDU-1",
      "location": "e2e",
      "specification": "e2e",
      "power_cap": "n/a",
      "racks": [
        {
          "name": "Rack-1",
          "location": "e2e",
          "specification": "42U",
          "hypervisors": [
            {
              "name": "HV-1",
              "port_range": "17667-17757",   // Port range for NISD processes on this hypervisor
              "ssh_port": "22",
              "rdma_enabled": false,
              "ip_addrs": ["127.0.0.1"],     // IP addresses for NISD communication on the hypervisor
              "devices": [
                {
                  "name": "nvme4n1",
                  "device_path": "/dev/nvme4n1",
                  "serial_number": "e2e",
                  "state": 2,
                  "size": 1200243695616,     // Device size in bytes
                  "partitions": [
                    {
                      "partition_path": "/dev/nvme4n1p1",
                      "size": 120023154688,
                      "nisd": {
                        "id": "c7f7ea89-9f57-4c02-a359-f390e5934e8a",
                        "peer_port": 17667,
                        "total_size": 120023154688,     // Total size of the partition; this is the NISD size
                        "available_size": 120023154688, // Keep available_size the same as total_size
                        "socket_path": "/var/run/nisd.sock",
                        "net_info": [{ "ip_addr": "127.0.0.1", "port": 17667 }] // IP address and port for the NISD
                      }
                    }
                  ]
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

> The block above is annotated (`//` comments) to explain each field. A real JSON file
> **must not contain comments** — remove them before posting, or edit the shipped
> `niova-infra-keys.json` directly.

Insert the infra keys with a single POST:

```bash
curl -s -X POST "$BASE/api/infra" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TA_TOKEN" \
  -d @/path/to/niova-mdsvc-setup.tar.extract/niova-infra-keys.json | jq .
```

Expected output:

```json
{
  "status": 0
}
```

At this point the infra hierarchy is registered up to the NISD (Niova server) level.
Next, create VDEVs.

### Step 7 — Create a VDEV

Create a 100 GiB VDEV with replica factor 1 (allocates chunks across the infra you just
registered):

```bash
curl -s -X POST "$BASE/api/vdev" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TA_TOKEN" \
  -d '{"name": "vdev01", "size_bytes": 107374182400, "data_blk_cnt": 1}' | jq .
```

Expected output:

```json
{
  "status": 0,
  "payload": {
    "vdev_id": "1676545e-674a-4f24-94b1-3c145d0a85cd",
    "name": "vdev01",
    "chunk_cnt": 13,
    "failure_domain": "PDU"
  }
}
```

> Save the `vdev_id` — you'll need it for `niova-block` / `niova-ctl` testing. The
> payload also includes `chunk_cnt` and `failure_domain`.

### Step 8 — Create a VDEV under a PFS (optional)

First create a PFS:

```bash
curl -s -X POST "$BASE/api/pfs" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TA_TOKEN" \
  -d '{"name": "primary-pfs"}' | jq .
```

Expected output:

```json
{
  "status": 0,
  "payload": {
    "pfs_id": "997867eb-49c1-4da6-8397-d9dd793bbe96",
    "name": "primary-pfs"
  }
}
```

Then create a VDEV named `meta0` under that PFS:

```bash
curl -s -X POST "$BASE/api/vdev" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TA_TOKEN" \
  -d '{"name": "meta0", "size_bytes": 107374182400, "data_blk_cnt": 1, "pfs": "997867eb-49c1-4da6-8397-d9dd793bbe96"}' | jq .
```

---

## Part 2 — Query the Control Plane

Use these queries to fetch resource details from the CP (for example, NISD IDs needed to
start the NISD process).

**All NISD details:**

```bash
curl -s "$BASE/api/resource?type=nisd" \
  -H "Authorization: Bearer $TA_TOKEN"
```

**NISD IDs only:**

```bash
curl -s "$BASE/api/resource?type=nisd" \
  -H "Authorization: Bearer $TA_TOKEN" | jq -r '.payload.resources[].id'
```

**All VDEV details:**

```bash
curl -s -X GET "$BASE/api/resource?type=vdev" \
  -H "Authorization: Bearer $TA_TOKEN"
```

Example output:

```json
{
  "status": 0,
  "payload": {
    "type": "vdev",
    "resources": [
      {
        "chunk_cnt": 13,
        "data_blk_cnt": 1,
        "failure_domain": "PDU",
        "id": "1676545e-674a-4f24-94b1-3c145d0a85cd",
        "last_mounted_at": null,
        "mount_counter": 0,
        "name": "pfsVdev01",
        "owner_id": "d9ae6ae8-66e6-4dcf-94ab-fe0403787eb9",
        "parity_blk_cnt": 0,
        "pfs_id": null,
        "redundancy": 0,
        "size": 107374182400
      }
    ]
  }
}
```

**VDEV IDs only:**

```bash
curl -s "$BASE/api/resource?type=vdev" \
  -H "Authorization: Bearer $TA_TOKEN" | jq -r '.payload.resources[].id'
```

Example output:

```text
1676545e-674a-4f24-94b1-3c145d0a85cd
```

---

## Part 3 — Niova Server (NISD) & Client Setup

### Download the Niova packages

Use the `download-niova.sh` helper to fetch the `niova-core` and `niova-block` packages
from the public GCS bucket. It auto-detects your architecture and package type, and can
optionally install the packages for you.

Fetch the script:

```bash
wget https://storage.googleapis.com/niova-public-bucket/download-niova.sh
```

View the available options:

```bash
bash download-niova.sh --help
```

```text
Usage: download-niova.sh [OPTIONS]

Download niova-core and/or niova-block packages from the public GCS bucket.

Options:
  -v, --version VERSION   Specific version to download (e.g. 0.1.0-beta.4)
                          Default: fetch the latest published version
  -p, --package PKG       Which package(s): all | core | block  (default: all)
  -t, --type TYPE         Package type: auto | rpm | deb        (default: auto)
  -a, --arch ARCH         Architecture: auto | x86_64 | aarch64 | amd64 | arm64
                          Default: auto-detect from $(uname -m)
  -o, --output-dir DIR    Directory to save downloads            (default: .)
  -i, --install           Install packages after download (requires sudo)
  -l, --list              List all available versions and exit
  -n, --dry-run           Print what would be downloaded without fetching
  -q, --quiet             Suppress wget progress output
  -h, --help              Show this help
```

Common examples:

```bash
# Download latest RPMs for this machine (auto-detected)
bash download-niova.sh

# Download and install latest DEBs
bash download-niova.sh --type deb --install

# Download a specific version into /tmp
bash download-niova.sh --version 0.1.0-beta.4 --output-dir /tmp

# Only download niova-block, x86_64 RPM, dry-run
bash download-niova.sh --package block --type rpm --arch x86_64 --dry-run

# List all available versions
bash download-niova.sh --list
```

> Run the script with `bash download-niova.sh ...` (not `download-niova.sh ...`), unless
> it is on your `PATH`.

Preview the download (dry-run):

```bash
bash download-niova.sh --type rpm --arch x86_64 --dry-run
```

```text
[niova] Fetching latest version...
[niova] Version : 0.1.0-beta.4
[niova] Type    : rpm
[niova] Arch    : x86_64
[niova] Output  : .
[niova] Resolving filename for niova-core...
[niova] Downloading niova-core-1.0.0-1.el10_2.x86_64.rpm...
  [dry-run] wget "https://storage.googleapis.com/niova-public-bucket/releases/rpm/el10.2/x86_64/0.1.0-beta.4/x86_64/niova-core-1.0.0-1.el10_2.x86_64.rpm"
[niova] Resolving filename for niova-block...
[niova] Downloading niova-block-1.0.0-1.el10_2.x86_64.rpm...
  [dry-run] wget "https://storage.googleapis.com/niova-public-bucket/releases/rpm/el10.2/x86_64/0.1.0-beta.4/x86_64/niova-block-1.0.0-1.el10_2.x86_64.rpm"
[niova] Dry-run complete — no files were downloaded.
```

Download the RPMs:

```bash
bash download-niova.sh --type rpm --arch x86_64
```

```text
[niova] Fetching latest version...
[niova] Version : 0.1.0-beta.4
[niova] Type    : rpm
[niova] Arch    : x86_64
[niova] Output  : .
[niova] Resolving filename for niova-core...
[niova] Downloading niova-core-1.0.0-1.el10_2.x86_64.rpm...
[niova] Saved: ./niova-core-1.0.0-1.el10_2.x86_64.rpm
[niova] Resolving filename for niova-block...
[niova] Downloading niova-block-1.0.0-1.el10_2.x86_64.rpm...
[niova] Saved: ./niova-block-1.0.0-1.el10_2.x86_64.rpm
[niova] Done.
```

Verify the downloaded packages:

```bash
ls *.rpm
```

```text
niova-block-1.0.0-1.el10_2.x86_64.rpm  niova-core-1.0.0-1.el10_2.x86_64.rpm
```

> To download **and** install in one step, add `--install` (requires `sudo`), which
> skips the manual `dnf install` in the next section.

### Install niova-block RPMs

Enable CRB and install EPEL dependencies:

```bash
dnf config-manager --set-enabled crb
dnf -y install epel-release
```

Install the Niova packages (from the directory where you downloaded them):

```bash
dnf -y install ./niova-core-1.0.0-1.el10_2.x86_64.rpm
dnf -y install ./niova-block-1.0.0-1.el10_2.x86_64.rpm
```

This installs the required `niova-core` and `niova-block` RPM packages.

### Install Serf (Rocky Linux 10.2)

Serf is required for the Niova client to communicate with the Control Plane.

**a.** Install the required packages:

```bash
dnf -y install dnf-plugins-core git golang gcc make
```

**b.** Enable the CRB repository:

```bash
dnf config-manager --set-enabled crb
```

**c.** Clone the Serf source:

```bash
cd /tmp
git clone https://github.com/hashicorp/serf.git
cd serf
```

**d.** Build the Serf binary:

```bash
go mod download
go build -o serf-agent ./cmd/serf
```

**e.** Install the binary:

```bash
install -m 755 serf-agent /usr/bin/serf
```

**f.** Verify the installation:

```bash
serf version
```

### Start the NISD process (Niova server)

Initialize the device. **Make sure the device is not in use and has no filesystem
mounted.**

```bash
sudo niova-block-ctl -u 01ab0004-0011-7011-d011-dddddddddd1a -d /dev/nvme0n1p1 -i
```

Make sure `niova-block-ctl` has terminated properly, then start the Niova server (NISD):

```bash
sudo NIOVA_MDSVC_URL="http://127.0.0.1:8081" \
  NIOVA_BLOCK_TCP_PEER_PORT="18000" \
  NIOVA_NISD_DO_TOKEN_VALIDATION=0 \
  nisd -u 01ab0004-0011-7011-d011-dddddddddd1a -d /dev/nvme0n1p1 -g -r 32 -x
```

> `NIOVA_BLOCK_TCP_PEER_PORT` must match the `peer_port` value for this NISD in
> `niova-infra-keys.json`.

### Start the Niova client

```bash
sudo NIOVA_BLOCK_UBLK_UNIFIED=0 \
  NIOVA_GOSSIP_PATH=/path_for_setup_tar_extract/gossipNodes \
  NIOVA_GOSSIP_KEY=dummy \
  NIOVA_BLOCK_CP_AUTH_USERNAME=admin \
  NIOVA_BLOCK_CP_AUTH_SECRET=admin \
  NIOVA_BLOCK_MDSVC_GET_CHUNKS_LIMIT=256 \
  niova-ublk -t cp -v b41f3f8e-2a5a-4e0d-9dfe-3cc733e96777 -q 128 -b 1048576 -T
```

> **Notes**
> - The `-v` parameter can be either the **VDEV UUID** or the **VDEV name**:
>   ```bash
>   niova-ublk -t cp -v vdev1 -q 128 -b 1048576 -T
>   ```
> - `NIOVA_BLOCK_CP_AUTH_USERNAME` and `NIOVA_BLOCK_CP_AUTH_SECRET` must match
>   `ADMIN_DEFAULT_USERNAME` and `ADMIN_DEFAULT_PASSWORD` from `.env` (or the default
>   values in `docker-compose.yml`).

---

## Part 4 — Multi-Tenancy

### Step 1 — Log in as the master tenant-admin

The global tenant-admin credentials are configured via `TENANT_ADMIN_USERNAME` /
`TENANT_ADMIN_PASSWORD` (dev defaults: `tenant-admin` /
`integration-test-tenant-admin-secret`).

```bash
curl -s -X POST "$BASE/tenant-admin/login" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "tenant-admin",
    "password": "integration-test-tenant-admin-secret"
  }'
```

This returns a JWT `access_token`.

### Step 2 — Create the tenant

Call `POST /cp/tenants` using the `access_token` from Step 1:

```bash
curl -s -X POST "$BASE/cp/tenants" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <access_token>" \
  -d '{"display_name": "example-tenant"}'
```

Response:

```json
{
  "tenant": {
    "tenant_uuid": "c3e9811f-829d-47a3-b4aa-713280c441f3",
    "display_name": "example-tenant",
    "schema_name": "mdsvc_tenant_c3e9811f_829d_47a3_b4aa_713280c441f3",
    "status": "active",
    "is_default": false,
    "created_at": "2026-08-27T06:00:00Z"
  },
  "admin_username": "admin",
  "admin_password": "<randomly_generated_password>"
}
```

> **IMPORTANT:** Save the `tenant_uuid` and `admin_password` from the response
> immediately. The `admin_password` is generated randomly and shown **only once**.

### Step 3 — Log in as the tenant's admin

To access the tenant's data plane (`/api/…`), log in with the `tenant_uuid` and the
returned `admin_password`:

```bash
curl -s -X POST "$BASE/users/login" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "<admin_password>",
    "tenant_uuid": "c3e9811f-829d-47a3-b4aa-713280c441f3"
  }'
```

### Step 4 — Add infra for this tenant admin

```bash
curl -s -X POST "$BASE/api/infra" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <access_token>" \
  -d @io01-io02-infra.json | jq .
```

---

## Part 5 — Running NBT Tests

Username and password are for users to run tests within a given tenant.

```bash
unset NIOVA_BLOCK_AUTH_ENABLED
export NIOVA_BLOCK_CP_AUTH_CLUSTER_UUID=c3e9811f-829d-47a3-b4aa-713280c441f3

sudo -E NIOVA_BLOCK_CP_AUTH_USERNAME=admin \
  NIOVA_BLOCK_CP_AUTH_SECRET=admin \
  NIOVA_GOSSIP_PATH=/path/to/gossip/gossipNodes \
  NIOVA_GOSSIP_KEY=dummy \
  NIOVA_BLOCK_MDSVC_GET_CHUNKS_LIMIT=256 \
  NIOVA_BLOCK_PROXY_TAG=mdsvc-tidb \
  /path/to/niova-block-bin/bin/niova-block-testx -c cp -v "vdevname" -u "$(uuidgen)" -r 0 -Z 0 -N 400000 -a 123456
```

Example log output:

```text
<...:warn:niova-block-tes:env_parse@230> env-var NIOVA_GOSSIP_PATH value /path/to/gossip/gossipNodes applied from environment
<...:warn:niova-block-tes:env_parse@230> env-var NIOVA_GOSSIP_KEY value dummy applied from environment
<...:warn:niova-block-tes:env_parse@230> env-var NIOVA_BLOCK_CP_AUTH_USERNAME value admin applied from environment
<...:warn:niova-block-tes:env_parse@230> env-var NIOVA_BLOCK_CP_AUTH_SECRET value admin applied from environment
<...:warn:niova-block-tes:env_parse_long@208> env-var NIOVA_NISD_DO_TOKEN_VALIDATION value 0 applied from environment
<...:warn:niova-block-tes:env_parse_long@208> env-var NIOVA_BLOCK_MDSVC_GET_CHUNKS_LIMIT value 256 applied from environment
```
