# Oracle 26ai installation on Oracle- Linux 9, 10 and Redhat 10 in Container

This repository contains Dockerfiles to build a containerized **Oracle AI Database 26ai (Free edition)** on three base images:

| Base OS | Dockerfile in this repo | Status |
|---|---|---|
| Oracle Linux 9 (OL9) | `Oracle Linux 9` | ✅ Officially supported |
| Oracle Linux 10 (OL10) | `Oracle Linux 10` | ⚠️ Requires workaround |
| Red Hat Enterprise Linux 10 (UBI10) | `Redhat 10` | ⚠️🔺 Hardest — requires workaround + RHEL subscription |

> The files `Oracle Linux 9`, `Oracle Linux 10`, and `Redhat 10` are Dockerfiles without a file extension. Rename them (e.g. `Dockerfile.ol9`) or pass them explicitly with `docker build -f "Oracle Linux 9" .` when building.

---

## Repository structure

```
.
├── Oracle Linux 9      # Dockerfile — Oracle Linux 9 base image
├── Oracle Linux 10     # Dockerfile — Oracle Linux 10 base image (CV_ASSUME_DISTID workaround)
├── Redhat 10            # Dockerfile — RHEL 10 UBI base image (subscription + workaround)
├── Build commands       # Example docker build / docker run commands
└── README.md
```

Each Dockerfile expects the required RPMs to be placed under:

```
./rpm/${TARGETARCH}/
    oracle-ai-database-preinstall-26ai*.rpm
    oracle-ai-database-free-26ai-*.rpm
```

`TARGETARCH` is the Docker buildx architecture (`amd64` or `arm64`) — set automatically when using `docker buildx build`, or pass it manually with `--build-arg TARGETARCH=amd64`.

---

## ⚠️ Before you build — known issues in this repo

1. **RHEL credentials are passed as build args.** `Redhat 10` calls `subscription-manager register --username $ID --password $KEY` baked into a `RUN` layer. Build secrets passed via `--build-arg` are visible in `docker history`/image layers — prefer `docker build --secret` or `subscription-manager` with an activation key + `unregister` cleanup step instead, especially if you ever push this image anywhere.
2. **`--nodeps` on the preinstall RPM.** All three Dockerfiles install the preinstall RPM with `rpm -Uvh --nodeps`. This sidesteps dependency errors but means missing prerequisite packages won't be reported — if the database configuration step fails later, dependency gaps are the first thing to check.

---

## Oracle RPM downloads

Two RPMs are required for every build: the **preinstallation RPM** (creates the `oracle` user, sets kernel/sysctl parameters) and the **Free Database RPM** (the actual database software).

| Package | Direct link |
|---|---|
| Preinstall RPM — OL9 / RHEL9 | `https://yum.oracle.com/repo/OracleLinux/OL9/appstream/x86_64/getPackage/oracle-ai-database-preinstall-26ai-1.0-1.el9.x86_64.rpm` |
| Preinstall RPM — OL10 / RHEL10 | `https://yum.oracle.com/repo/OracleLinux/OL10/appstream/x86_64/getPackage/oracle-ai-database-preinstall-26ai-1.0-1.el10.x86_64.rpm` |
| Oracle AI Database 26ai **Free** RPM (all platforms) | https://www.oracle.com/database/technologies/free-downloads.html |
| Oracle AI Database 26ai **Enterprise Edition** RPM (requires Oracle account) | https://www.oracle.com/database/technologies/oracle26ai-linux-downloads.html |
| Official RPM installation guide (Oracle Docs) | https://docs.oracle.com/en/database/oracle/oracle-database/26/ladbi/running-rpm-packages-to-install-oracle-database.html |
| Oracle's own prebuilt container image (alternative to building your own) | `docker pull container-registry.oracle.com/database/free:latest` |

The `yum.oracle.com` links above can be fetched directly with `curl`/`wget`, no Oracle account needed. The Free and EE RPMs from `oracle.com` typically require accepting the license (and an Oracle account for EE).

Example for OL9:
```bash
mkdir -p rpm/amd64
curl -o rpm/amd64/oracle-ai-database-preinstall-26ai-1.0-1.el9.x86_64.rpm \
  https://yum.oracle.com/repo/OracleLinux/OL9/appstream/x86_64/getPackage/oracle-ai-database-preinstall-26ai-1.0-1.el9.x86_64.rpm
# Download oracle-ai-database-free-26ai-*.rpm manually from the Free downloads page above
# and place it in the same rpm/amd64/ folder.
```

Exact RPM file names change with each release update (e.g. `23.26.0-1` vs later `23.26.x`), so always confirm the current file name on the download page rather than hardcoding a version.

---

## Build & run commands

```bash
# For Oracle Linux 9, 10
docker build -t oracle-26ai:latest .

# For Red Hat 9, 10 (subscription credentials required)
docker build --build-arg ID=${YOUR_REDHAT_ID} --build-arg KEY=${YOUR_REDHAT_PASSWORD} -t oracle-26ai:latest .

# Basic run (single container, no persistence)
docker run -d --name ol9-oracle-26ai \
  -e ORACLE_PWD='ORACLE26AI' \
  -p 1521:1521 -p 5500:5500 \
  oracle-26ai:latest

# Run with persistent data + Fast Recovery Area volumes
docker run -d \
  --name oracle26ai-free \
  -e ORACLE_PWD='ORACLE26AI' \
  -p 1521:1521 \
  -p 5500:5500 \
  -v oracle_data:/opt/oracle/oradata \
  -v oracle_fra:/opt/oracle/fast_recovery_area \
  oracle-26ai:latest
```

Verify pluggable databases are open once the container is running:
```sql
select name, open_mode from v$pdbs;
```

---

## Why Oracle Linux 10 and RHEL 10 are harder than Oracle Linux 9

Oracle Linux 9 is the straightforward path — it is the platform Oracle's preinstall RPM and 26ai Free RPM were originally certified against, so the Dockerfile is a plain `dnf install` + `rpm -Uvh`. OL10 and RHEL10 are more difficult for a few concrete reasons:

**1. Late/uncertain certification.** For a stretch after the OL10/RHEL10 release, Oracle's installer (Cluster Verification Utility checks bundled with the preinstall RPM and the DB installer) did not recognize OL10/RHEL10 as a supported distribution ID, even though the underlying packages mostly worked. The common workaround — used in this repo's `Oracle Linux 10` and `Redhat 10` Dockerfiles — is to spoof the detected distribution with an environment variable so the installer "thinks" it's running on OL9/RHEL9:
```bash
export CV_ASSUME_DISTID=OL9     # or RHEL9
```
Oracle has since published a dedicated `oracle-ai-database-preinstall-26ai-*.el10.x86_64.rpm` on `yum.oracle.com`, so native OL10/RHEL10 preinstall support is improving — but certification status changes over time. Check **My Oracle Support Doc ID 3053981.1** for the current certification matrix before relying on this for anything beyond a lab/dev container.

**2. RHEL has no preinstall RPM repo by default.** Oracle's preinstall RPM is built and distributed for Oracle Linux. On RHEL, `dnf install oracle-ai-database-preinstall-26ai` doesn't resolve from any default repo — you must manually download the RPM from `yum.oracle.com` (same file works on RHEL) and install it with `rpm -Uvh --nodeps`, skipping normal dependency resolution. That's exactly the pattern in the `Redhat 10` Dockerfile here.

**3. RHEL requires an active subscription/entitlement.** The UBI10 base image alone has very limited package repos. To `dnf install` the gcc/glibc/libaio/etc. prerequisites, the container must register against Red Hat's subscription service during the build (`subscription-manager register --username --password`), which means a valid Red Hat Developer or paid subscription is a hard prerequisite — something OL9/OL10 don't need at all.

**4. Library/ABI drift.** OL10/RHEL10 ship newer glibc, OpenSSL, and compatibility libraries than what the 26ai installer's prerequisite checks expect (e.g. `compat-openssl11`, `libnsl2` naming/availability differs across EL9 → EL10). This is part of why `--nodeps` and manually curated package lists (rather than the preinstall RPM's automatic dependency resolution) show up in the OL10/RHEL10 Dockerfiles.

**Summary of relative difficulty:**

| | OL9 | OL10 | RHEL10 |
|---|---|---|---|
| Preinstall RPM available | Yes, certified | Yes, but newer/less battle-tested | Yes (same RPM), manual install only |
| Needs `CV_ASSUME_DISTID` workaround | No | Yes (historically) | Yes (historically) |
| Needs subscription/entitlement | No | No | Yes |
| Relative effort | Low | Medium | High |

---

## Disclaimer

- This setup uses **Oracle AI Database 26ai Free**, which is capped at 2 CPUs, 2 GB RAM (SGA+PGA), and 12 GB of user data — fine for development/testing, not for production.
- Spoofing the distro ID via `CV_ASSUME_DISTID` is an unsupported workaround. Don't use these images for production workloads until Oracle's official certification matrix confirms full OL10/RHEL10 support.
- Always re-check current RPM file names and certification status on Oracle's download pages and My Oracle Support before building.
