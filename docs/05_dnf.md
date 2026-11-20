# DNF — Dandified YUM

> All examples in this document were captured on **Fedora 41**. To follow along, start a container:
>
> ```
> docker run -it fedora:41 bash
> ```
>
> `dnf` and `rpm` are already preinstalled. No additional packages are needed.

DNF is the **high-level** package management tool used in Red Hat-based Linux distributions such as Fedora, RHEL, CentOS Stream, Rocky Linux, and AlmaLinux. It downloads packages from remote repositories, resolves dependencies automatically, and then calls [rpm](04_rpm.md) to perform the actual installation or removal.

DNF replaced **YUM** (Yellowdog Updater, Modified), which served the same role from the early 2000s until Fedora 22 (2015). DNF was a complete rewrite with faster dependency resolution, lower memory usage, and a cleaner API. The relationship between DNF and `rpm` mirrors the relationship between [APT](03_apt.md) and [dpkg](02_dpkg.md) in the Debian family.

> **DNF5:** Starting with Fedora 41, the `dnf` command is provided by **DNF5** (`dnf5`), a further rewrite in C++ that is significantly faster than the original Python-based DNF. The commands and options are largely the same, with minor differences noted where relevant.

```
$ dnf --version
dnf5 version 5.2.17.0
```


## Basic Workflow

The most common DNF workflow has three steps:

**1. Check for updates** — See which packages have newer versions available:

```
$ dnf check-update
xz.x86_64      1:5.8.1-4.fc41 updates
xz-libs.x86_64 1:5.8.1-4.fc41 updates
```

Unlike APT, DNF does **not** require a separate "update index" step. DNF automatically refreshes its metadata cache when it is expired (controlled by the `metadata_expire` setting, typically 48 hours). You can force a refresh with `dnf makecache`.

**2. Install a package** — Download and install a package along with all its dependencies:

```
$ sudo dnf install htop
Total size of inbound packages is 3 MiB. Need to download 3 MiB.
After this operation, 4 MiB extra will be used (install 4 MiB, remove 0 B).
Package     Arch   Version       Repository      Size
Installing:
 htop       x86_64 3.3.0-4.fc41  fedora     448.6 KiB
Installing dependencies:
 hwloc-libs x86_64 2.12.0-1.fc41 updates      2.9 MiB
 libnl3     x86_64 3.11.0-1.fc41 updates      1.0 MiB

Transaction Summary:
 Installing:         3 packages
...
Complete!
```

Notice that DNF automatically resolved the dependencies (`hwloc-libs`, `libnl3`) and installed them alongside `htop`.

**3. Upgrade all packages** — Upgrade every installed package to its latest available version:

```
$ sudo dnf upgrade
```

DNF will show a summary of what will be upgraded and ask for confirmation before proceeding.

> **Key difference from APT:** In APT, `update` refreshes the index and `upgrade` installs new versions. In DNF, `check-update` checks for updates (optional) and `upgrade` does both refreshing and upgrading in one step.


## Common Commands

| Command | Purpose |
|---------|---------|
| `dnf install <package>` | Installs a package and its dependencies. |
| `dnf remove <package>` | Removes a package and its unused dependencies. |
| `dnf upgrade` | Upgrades all installed packages. |
| `dnf upgrade <package>` | Upgrades a specific package. |
| `dnf downgrade <package>` | Reverts a package to the previous version. |
| `dnf reinstall <package>` | Reinstalls a package (replaces all its files). |
| `dnf search <term>` | Searches package names and descriptions. |
| `dnf info <package>` | Displays detailed metadata for a package. |
| `dnf list --installed` | Lists all installed packages. |
| `dnf list --available` | Lists all packages available in repositories. |
| `dnf check-update` | Lists packages with available upgrades. |
| `dnf provides <file>` | Finds which package provides a file or capability. |
| `dnf repoquery --requires <package>` | Lists the dependencies of a package. |
| `dnf autoremove` | Removes packages that were installed as dependencies and are no longer needed. |
| `dnf clean all` | Clears all cached data (metadata and packages). |
| `dnf makecache` | Downloads and caches metadata from all enabled repositories. |


## Searching and Inspecting Packages

To search for a package by name or description:

```
$ dnf search htop
Matched fields: name (exact)
 htop.x86_64  Interactive process viewer
```

To view detailed metadata about a package:

```
$ dnf info htop
Available packages
Name           : htop
Epoch          : 0
Version        : 3.3.0
Release        : 4.fc41
Architecture   : x86_64
Download size  : 195.7 KiB
Installed size : 448.6 KiB
Source         : htop-3.3.0-4.fc41.src.rpm
Repository     : fedora
Summary        : Interactive process viewer
URL            : http://hisham.hm/htop/
License        : GPLv2+
Description    : htop is an interactive text-mode process viewer for Linux, similar to
               : top(1).
```

To find which package provides a specific file:

```
$ dnf provides /usr/bin/htop
htop-3.3.0-4.fc41.x86_64 : Interactive process viewer
Matched from:
Filename     : /usr/bin/htop
```


## Removing Packages

To remove a package:

```
$ sudo dnf remove htop
Package     Arch   Version       Repository      Size
Removing:
 htop       x86_64 3.3.0-4.fc41  fedora     448.6 KiB
Removing unused dependencies:
 hwloc-libs x86_64 2.12.0-1.fc41 updates      2.9 MiB
 libnl3     x86_64 3.11.0-1.fc41 updates      1.0 MiB

Transaction Summary:
 Removing:           3 packages
...
Complete!
```

By default, DNF also removes unused dependencies that were pulled in when the package was installed. This is similar to running `apt autoremove` after `apt remove` in the Debian world, but DNF does it automatically.


## Transaction History

One of DNF's most powerful features is its **transaction history**. Every install, upgrade, and removal is recorded as a numbered transaction that can be inspected and rolled back.

```
$ dnf history info
Transaction ID : 2
Begin time     : 2026-11-21 00:54:14
End time       : 2026-11-21 00:54:15
User           : 0 Super User <root>
Status         : Ok
Releasever     : 41
Description    : dnf install -y htop
Packages altered:
  Action  Package                            Reason     Repository
  Install htop-0:3.3.0-4.fc41.x86_64        User       fedora
  Install hwloc-libs-0:2.12.0-1.fc41.x86_64 Dependency updates
  Install libnl3-0:3.11.0-1.fc41.x86_64     Dependency updates
```

To undo a specific transaction (roll back the changes):

```
$ sudo dnf history undo <transaction-id>
```

This will remove packages that were installed in that transaction or reinstall packages that were removed. Transaction history is stored in the RPM database, so it persists across reboots.

> **Comparison with APT:** APT logs transactions to `/var/log/apt/history.log` but does not support rolling them back. DNF's `history undo` is a significant advantage for system recovery.


## Package Groups

DNF can install predefined **groups** of related packages at once, which is useful for setting up a complete environment:

```
$ dnf group list
ID                          Name                                        Installed
kde-desktop                 KDE                                                no
office                      Office/Productivity                                no
network-server              Network Servers                                    no
python-science              Python Science                                     no
...
```

To install a group:

```
$ sudo dnf group install "Development Tools"
```

To see which packages are in a group:

```
$ dnf group info "Development Tools"
```

Groups are categorized as:

- **Mandatory** — Always installed with the group.
- **Default** — Installed by default but can be excluded.
- **Optional** — Not installed by default; must be explicitly requested.


## Repository Configuration

### Repository Files

DNF reads repository definitions from `.repo` files in `/etc/yum.repos.d/`:

```
$ ls /etc/yum.repos.d/
fedora-cisco-openh264.repo
fedora-updates-testing.repo
fedora-updates.repo
fedora.repo
```

Each `.repo` file can define one or more repositories. Here is a typical repository definition:

```
$ cat /etc/yum.repos.d/fedora.repo
[fedora]
name=Fedora $releasever - $basearch
metalink=https://mirrors.fedoraproject.org/metalink?repo=fedora-$releasever&arch=$basearch
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-$releasever-$basearch
```

| Field       | Description |
|-------------|-------------|
| `[fedora]`  | Repository ID — a unique short name used internally. |
| `name`      | Human-readable description. |
| `baseurl`   | Direct URL to the repository. |
| `metalink`  | URL that returns a list of mirrors (alternative to `baseurl`). |
| `enabled`   | `1` to enable, `0` to disable. |
| `gpgcheck`  | `1` to verify package signatures against the GPG key. |
| `gpgkey`    | Path or URL to the GPG key for signature verification. |

Variables like `$releasever` (e.g., `41`) and `$basearch` (e.g., `x86_64`) are automatically substituted.

### Managing Repositories

```
# List all enabled repositories
$ dnf repolist
repo id               repo name
fedora                Fedora 41 - x86_64
fedora-cisco-openh264 Fedora 41 openh264 (From Cisco) - x86_64
updates               Fedora 41 - x86_64 - Updates

# List all repositories (including disabled)
$ dnf repolist --all

# Enable a repository
$ sudo dnf config-manager setopt fedora-updates-testing.enabled=1

# Disable a repository
$ sudo dnf config-manager setopt fedora-updates-testing.enabled=0
```

### Adding a Third-Party Repository

On Fedora, third-party repositories are often provided as `.repo` files. The most common way to add one is to install an RPM that drops a `.repo` file into `/etc/yum.repos.d/`:

```
$ sudo dnf install https://example.com/example-repo-release.rpm
```

You can also add a repository directly:

```
$ sudo dnf config-manager addrepo --from-repofile=https://example.com/example.repo
```

### COPR Repositories

**COPR** (Cool Other Package Repositories) is Fedora's equivalent of Ubuntu's PPAs — community-maintained repositories for packages not in the official repos:

```
# Enable a COPR repository
$ sudo dnf copr enable <owner>/<project>

# Disable a COPR repository
$ sudo dnf copr disable <owner>/<project>
```


## The DNF Cache

DNF caches repository metadata and downloaded packages locally. In DNF5, the cache is stored at `/var/cache/libdnf5/`:

```
$ du -sh /var/cache/libdnf5/
105M    /var/cache/libdnf5/
```

To manage the cache:

```
# Remove all cached data (metadata and packages)
$ sudo dnf clean all

# Remove only cached package files
$ sudo dnf clean packages

# Remove only cached metadata
$ sudo dnf clean metadata

# Download and cache fresh metadata
$ sudo dnf makecache
```

The `metadata_expire` setting in each `.repo` file controls how long cached metadata is considered valid before DNF automatically refreshes it.


## DNF vs YUM

DNF replaced YUM as the default package manager starting with Fedora 22 (2015). RHEL 8 shipped with DNF as the default, and RHEL 9 removed YUM entirely (the `yum` command is a symlink to `dnf`).

| Feature | YUM | DNF | DNF5 |
|---------|-----|-----|------|
| Language | Python | Python + C (`libdnf`) | C++ (`libdnf5`) |
| Dependency resolver | Custom | libsolv (SAT-based) | libsolv (SAT-based) |
| Performance | Slow with large repos | Faster | Significantly faster |
| API stability | Unstable | Stable Python API | Stable C++ API |
| Default in | RHEL 5–7, Fedora ≤21 | RHEL 8–9, Fedora 22–40 | Fedora 41+ |

In practice, the `dnf` commands shown in this document work on all three implementations. If you are on an older system that still uses `yum`, most commands are identical — just replace `dnf` with `yum`.

---

**Previous**: [← RPM](04_rpm.md) · **Next**: [Building Packages →](06_building_packages.md)
