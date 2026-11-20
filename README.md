# Linux Package Management

A structured guide to Linux package management — from low-level tools like `dpkg` and `rpm` to high-level managers like APT, covering package formats, dependency resolution, and repository configuration.


## Table of Contents

### Overview

| # | Topic | Description |
|---|-------|-------------|
| 1 | [System Package Managers](docs/01_system_package_managers.md) | What system package managers are, Linux base and derivative distributions, high-level vs low-level architecture, wrappers and frontends, package formats, non-Linux and universal package managers, and repositories. |

### Debian/Ubuntu

| # | Topic | Description |
|---|-------|-------------|
| 2 | [dpkg](docs/02_dpkg.md) | The low-level Debian package manager — `.deb` file anatomy, installing, removing, querying packages, and `update-alternatives`. |
| 3 | [APT](docs/03_apt.md) | The high-level Advanced Package Tool — command-line tools, `apt-get` vs `apt`, repository configuration, and Ubuntu software sources. |

### Fedora/RHEL

| # | Topic | Description |
|---|-------|-------------|
| 4 | [RPM](docs/04_rpm.md) | The low-level RPM Package Manager — `.rpm` file anatomy, installing, upgrading, removing, querying, and verifying packages. |
| 5 | [DNF](docs/05_dnf.md) | The high-level Dandified YUM — installing, upgrading, removing packages, transaction history, groups, and repository configuration. |

### Building Packages

| # | Topic | Description |
|---|-------|-------------|
| 6 | [Building Packages](docs/06_building_packages.md) | Creating `.deb` and `.rpm` packages from source — the `debian/` directory, `.spec` files, and clean build environments. |
