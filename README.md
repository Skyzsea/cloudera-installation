# cloudera-installation

Cloudera Manager 7.13.1.1400, OpenJDK 8, and CDH 7.3.1 parcel for RHEL 8 (el8, x86_64), plus an offline lab repo procedure.

Large files are not stored in git. They are attached as assets on GitHub Releases, because GitHub rejects git files over 100 MB.

| Release tag | Contents |
|---|---|
| [v7.13.1.1400](https://github.com/hariramshrestha98400/cloudera-installation/releases/tag/v7.13.1.1400) | Cloudera Manager server, daemons, agent RPMs and OpenJDK 8 RPM |
| [CDH-7.3.1-1](https://github.com/hariramshrestha98400/cloudera-installation/releases/tag/CDH-7.3.1-1) | CDH 7.3.1 parcel, split into 8 parts, plus `parts.sha1` |

| Package | Size |
|---|---|
| cloudera-manager-server-7.13.1.1400-84991238.el8.x86_64.rpm | 21 KB |
| cloudera-manager-daemons-7.13.1.1400-84991238.el8.x86_64.rpm | ~2.0 GB |
| cloudera-manager-agent-7.13.1.1400-84991238.el8.x86_64.rpm | ~243 MB |
| openjdk8-8.0.372_1-cloudera.x86_64.rpm | ~103 MB |
| CDH-7.3.1-1.cdh7.3.1.p900.80138085-el8.parcel (8 parts) | 14.57 GB |

---

## 1. Rebuild the CDH parcel from its parts

The parcel is split into 8 files, `...parcel.part00` through `...parcel.part07`. The first seven are 1,992,294,400 bytes each. The last is 624,803,888 bytes.

### Step 1: Download all parts

Download all 8 `.part` files from the [CDH-7.3.1-1 release](https://github.com/hariramshrestha98400/cloudera-installation/releases/tag/CDH-7.3.1-1) into one empty folder. Also download `parts.sha1`.

### Step 2: Check the parts (optional)

Linux, macOS, or Git Bash:

```bash
cd /path/to/folder
sha1sum -c parts.sha1
```

Every line should end with `OK`.

### Step 3: Combine the parts

Linux, macOS, or Git Bash:

```bash
cat CDH-7.3.1-1.cdh7.3.1.p900.80138085-el8.parcel.part* > CDH-7.3.1-1.cdh7.3.1.p900.80138085-el8.parcel
```

Windows Command Prompt, run in the folder that holds the parts:

```bat
copy /b CDH-7.3.1-1.cdh7.3.1.p900.80138085-el8.parcel.part* CDH-7.3.1-1.cdh7.3.1.p900.80138085-el8.parcel
```

Parts are joined in name order, so keep the `part00`–`part07` names unchanged.

### Step 4: Verify the rebuilt parcel

The original parcel's SHA-1 is `e561f3485d286b50c35869eff6283d4293202aa5`.

Linux, macOS, or Git Bash:

```bash
sha1sum CDH-7.3.1-1.cdh7.3.1.p900.80138085-el8.parcel
```

Windows PowerShell:

```powershell
Get-FileHash -Algorithm SHA1 .\CDH-7.3.1-1.cdh7.3.1.p900.80138085-el8.parcel
```

The output must be `e561f3485d286b50c35869eff6283d4293202aa5`. If it differs, re-download the parts that failed their `parts.sha1` check and combine again.

---

## 2. Offline lab procedure (RHEL 8, Cloudera Manager, CDH parcel)

This covers three parts: an offline YUM repo for RHEL 8 and Cloudera Manager, a local copy of the CDH parcel, and client setup. Run it on a machine with internet access, then move the output to the lab network.

### Disk budget

- RHEL 8 BaseOS + AppStream: about 80–100 GB for full mirrors. For a lab, sync only what the cluster needs, or use the packages already in the dnf cache.
- Cloudera Manager RPMs: about 3 GB (daemons is 2 GB).
- OpenJDK 8 RPM: about 100 MB.
- CDH 7.3.1 parcel: 14.5 GB, plus its `.sha`.

Plan for roughly 25 GB for a minimal lab, more for full mirrors.

### Step 1: Prepare the sync host

Use a RHEL 8 host with a valid subscription and the Cloudera repo access you already have.

```bash
sudo dnf install -y dnf-utils createrepo_c httpd
sudo mkdir -p /srv/offline-repo/{rhel8,cloudera/cm,cloudera/java,parcels}
```

### Step 2: Mirror the RHEL 8 repos

Sync only the repos you need:

```bash
for repo in rhel-8-for-x86_64-baseos-rpms rhel-8-for-x86_64-appstream-rpms; do
  sudo reposync --repoid=$repo --download-metadata --newest-only \
       --download-path=/srv/offline-repo/rhel8
done
```

`--newest-only` keeps just the latest package versions, which saves a lot of space. Add `--arch=x86_64,noarch` if you want to limit architectures.

### Step 3: Mirror the Cloudera repos

These are the repos already configured in `/etc/yum.repos.d/`:

```bash
sudo reposync --repoid=cloudera-manager --download-metadata \
     --download-path=/srv/offline-repo/cloudera/cm
sudo reposync --repoid=cloudera-java --download-metadata \
     --download-path=/srv/offline-repo/cloudera/java
```

If the repo IDs differ on your system, check them with `dnf repolist`.

### Step 4: Generate repo metadata

```bash
for d in /srv/offline-repo/rhel8/* /srv/offline-repo/cloudera/cm /srv/offline-repo/cloudera/java; do
  sudo createrepo_c --update "$d"
done
```

Run this again after each sync so metadata matches the RPMs.

### Step 5: Copy the CDH parcel

Rebuild the parcel from Section 1 first. Then copy the parcel and its checksum into the repo directory:

```bash
sudo mkdir -p /srv/offline-repo/parcels/CDH-7.3.1
sudo cp CDH-7.3.1-1.cdh7.3.1.p900.80138085-el8.parcel \
        /srv/offline-repo/parcels/CDH-7.3.1/
```

Only the `.parcel` and its `.sha` are needed. The `.torrent` is not needed.

**Gap:** Cloudera Manager's remote parcel repo also needs a `manifest.json` index in that directory. None exists in this repo. Generate it from the `.sha` value, or use the manifest from Cloudera's download, which requires your account. Check the Cloudera docs for the exact format, since Cloudera Manager validates it.

### Step 6: Serve the repos over HTTP

```bash
sudo tee /etc/httpd/conf.d/offline-repo.conf >/dev/null <<'EOF'
Alias /repo /srv/offline-repo
<Directory /srv/offline-repo>
    Options Indexes FollowSymLinks
    Require all granted
</Directory>
EOF
sudo systemctl enable --now httpd
sudo firewall-cmd --permanent --add-service=http && sudo firewall-cmd --reload
```

Test with `curl http://<repo-host>/repo/`.

### Step 7: Point lab hosts at the offline repo

Create `/etc/yum.repos.d/lab-offline.repo` on each lab host:

```ini
[lab-rhel8-baseos]
name=Lab RHEL 8 BaseOS
baseurl=http://<repo-host>/repo/rhel8/rhel-8-for-x86_64-baseos-rpms
enabled=1
gpgcheck=0

[lab-rhel8-appstream]
name=Lab RHEL 8 AppStream
baseurl=http://<repo-host>/repo/rhel8/rhel-8-for-x86_64-appstream-rpms
enabled=1
gpgcheck=0

[lab-cloudera-manager]
name=Lab Cloudera Manager
baseurl=http://<repo-host>/repo/cloudera/cm
enabled=1
gpgcheck=0

[lab-cloudera-java]
name=Lab Cloudera JDK
baseurl=http://<repo-host>/repo/cloudera/java
enabled=1
gpgcheck=0
```

Then disable the upstream repos on those hosts (`sudo dnf config-manager --disable <repoid>`), or the hosts will still try the internet:

```bash
sudo dnf clean all && sudo dnf makecache
```

For Cloudera Manager to find the parcel, set the Parcel Repository URL to `http://<repo-host>/repo/parcels/CDH-7.3.1/` in Cloudera Manager.

### Step 8: Verify

- `dnf repolist` on a lab host shows the four lab repos.
- `dnf install cloudera-manager-agent` runs without contacting the internet.
- Cloudera Manager's parcel page shows CDH 7.3.1 as available.

### Notes
 1. Parcel (irreplaceable without the Cloudera login, 14 GB)
  - CDH-7.3.1-1.cdh7.3.1.p900.80138085-el8.parcel
  - its .sha file
  - manifest.json, which the parcel repo needs and which is missing from cm01 today
  - the .torrent is optional

  2. Cloudera RPMs (about 2.3 GB)
  - cloudera-manager-daemons, -agent, -server (7.13.1.1400)
  - openjdk8-8.0+372_1-cloudera from the 7.11.3.0 repo
  - RPM-GPG-KEY-cloudera, so gpgcheck=1 keeps working offline

  3. RHEL packages (size depends on how much you mirror)
  - The 15 or so packages the playbook installs, plus their dependencies. These are small, about 21 MB for the direct list.
  - A full BaseOS and AppStream mirror is tens of gigabytes. Back it up if you want to rebuild the offline lab without a
    subscription.

  4. Cluster state (not part of the repo, but needed to restore)
  - PostgreSQL dumps of the scm and hive databases, which hold the cluster definition and Metastore.
  - The Cloudera licence file.
  - The playbook, its vars.yml, and the encrypted vault with its vault password stored separately.

  Notes
  - Don't back up repodata/. createrepo_c regenerates it from the RPMs, so the RPMs and the .sha checksums are what matter.
  - Record a checksum for each file when you back it up, and verify them again before restoring. Use sha256sum for RPMs and
    the parcel .sha for the parcel.
  - Keep at least two copies on separate disks. The source for all of this is Cloudera's archive, which needs your
    subscription, so a local backup is the only guarantee if that access lapses.



- **Licensing:** Cloudera packages are licensed. Keep this mirror inside the lab, and check that your Cloudera agreement allows internal copies.
- **Updates:** Re-run Steps 2–4 when you want new packages. Keep old copies if clusters are pinned to specific versions.
- **Security:** The example uses plain HTTP and no GPG checks. That's fine for an isolated lab, but use HTTPS and `gpgcheck=1` anywhere else.
