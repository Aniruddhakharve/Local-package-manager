# Local-package-manager

Professional, hands-on lab documentation for building a **fully offline local DNF/Yum repository** on **CentOS Stream 10** and **RHEL 10** using an installation ISO.

---

## Lab Objective

Build and validate a local package source that:
- serves BaseOS and AppStream content directly from mounted installation media,
- allows package operations without internet repositories,
- and persists the mount across reboots for operational reliability.

---

## Prerequisites

- CentOS Stream 10 or RHEL 10 host
- Root or sudo privileges
- Installation ISO attached as virtual/physical media (`/dev/sr0`)
- DNF/Yum stack available (default on both platforms)

> Run all commands as `root` (or with `sudo`).

---

## Step-by-Step Implementation

### 1) Create a fixed mount point

```bash
mkdir -p /media/cdrom
```

**What you are doing:**  
Creating a stable filesystem location where the ISO content will be mounted.

**Why we do it:**  
A fixed mount path ensures repository definitions and automation (like `/etc/fstab`) always reference the same location, reducing operational drift and configuration errors.

---

### 2) Attach the installation media (read-only)

```bash
mount -o ro /dev/sr0 /media/cdrom
```

**What you are doing:**  
Mounting the installation media from `/dev/sr0` to `/media/cdrom` in **read-only** mode.

**Why we do it:**  
Read-only (`-o ro`) protects source media integrity and matches expected behavior for ISO-backed repositories. DNF only needs to read packages and metadata; write access is unnecessary and risky.

---

### 3) Author the repository configuration map

Create `/etc/yum.repos.d/local.repo`:

```ini
[local-baseos]
name=Local BaseOS Repository
baseurl=file:///media/cdrom/BaseOS/
enabled=1
gpgcheck=0

[local-appstream]
name=Local AppStream Repository
baseurl=file:///media/cdrom/AppStream/
enabled=1
gpgcheck=0
```

Example command to write it directly:

```bash
cat > /etc/yum.repos.d/local.repo <<'EOF'
[local-baseos]
name=Local BaseOS Repository
baseurl=file:///media/cdrom/BaseOS/
enabled=1
gpgcheck=0

[local-appstream]
name=Local AppStream Repository
baseurl=file:///media/cdrom/AppStream/
enabled=1
gpgcheck=0
EOF
```

**What you are doing:**  
Defining two explicit local repository channels:
- `[local-baseos]` for core OS packages
- `[local-appstream]` for application streams and modular content

**Why we do it:**  
CentOS Stream/RHEL repositories are logically split. Mapping both channels ensures dependency resolution works correctly across core and layered packages in offline mode.

---

### 4) Disable online profiles for true offline simulation

```bash
sed -i 's/enabled=1/enabled=0/g' /etc/yum.repos.d/centos.repo
```

**What you are doing:**  
Turning off all repositories defined in `centos.repo` by replacing `enabled=1` with `enabled=0`.

**Why we do it:**  
This enforces a realistic disconnected environment so validation is guaranteed to use only your local ISO-backed repositories.

> For RHEL systems, equivalent online repo files may differ (for example, Red Hat repo definitions). Apply the same disablement strategy to active online `.repo` files.

---

### 5) Flush the package metadata cache

```bash
dnf clean all
```

**What you are doing:**  
Removing cached metadata/packages from prior repository states.

**Why we do it:**  
Prevents stale cache data from masking configuration mistakes and forces DNF to evaluate your new local repo setup cleanly.

---

### 6) Generate a fresh metadata index

```bash
dnf makecache
```

**What you are doing:**  
Instructing DNF to download/read and store current metadata from enabled repositories.

**Why we do it:**  
Confirms repository readability and prepares dependency/transaction data for fast subsequent package operations.

---

### 7) Verify configuration status

```bash
dnf repolist
```

**What you are doing:**  
Listing all enabled repositories currently visible to DNF.

**Why we do it:**  
This is the primary validation checkpoint. You should see `local-baseos` and `local-appstream` enabled, with online repos disabled for true offline behavior.

---

## Automation & Persistence: Permanent Mount via `/etc/fstab`

To ensure the media mounts automatically on boot, append this entry to `/etc/fstab`:

```bash
echo '/dev/sr0 /media/cdrom iso9660 ro,nofail 0 0' >> /etc/fstab
```

Validate the fstab configuration immediately:

```bash
umount /media/cdrom
mount -a
mount | grep /media/cdrom
```

**What you are doing:**  
Creating a persistent mount rule for the ISO media.

**Why we do it:**  
Without persistence, reboot events break repository paths and package operations. `/etc/fstab` guarantees repeatable infrastructure behavior and minimizes manual recovery.

---

## Final Validation Checklist

- [ ] `/media/cdrom` exists
- [ ] ISO is mounted read-only from `/dev/sr0`
- [ ] `/etc/yum.repos.d/local.repo` has both BaseOS and AppStream blocks
- [ ] Online repositories are disabled for offline simulation
- [ ] `dnf clean all` and `dnf makecache` complete successfully
- [ ] `dnf repolist` shows local repositories enabled
- [ ] `/etc/fstab` contains the persistent mount entry

If all checks pass, your system is ready to operate package management workflows in an offline/local-repository model.