# Building Packages

> The `.deb` examples were captured on **Ubuntu 24.04 LTS (Noble Numbat)** and the `.rpm` examples on **Fedora 41**. To follow along, start a container and install the build tools:
>
> ```
> # For .deb examples
> docker run -it ubuntu:24.04 bash
> apt-get update && apt-get install -y build-essential devscripts debhelper
>
> # For .rpm examples
> docker run -it fedora:41 bash
> dnf install -y rpm-build rpmdevtools gcc
> ```

The previous documents covered how to *install* packages that someone else has already built. This document covers the other side: how to *create* your own packages from source code for the two major Linux package ecosystems:

- `.deb` (Debian/Ubuntu)
- `.rpm` (Fedora/RHEL)

Both workflows follow the same general pattern: write metadata that describes the package (name, version, dependencies, description), place the source code in the expected location, and run a build tool that compiles the software and produces a single installable file (`.deb` or `.rpm`). The result can be installed, upgraded, and removed using the system package manager just like any other package.

## Why Build a Package?

Why not just compile the code and copy the binary to `/usr/bin`? Because a package gives you:

- **Dependency tracking** — The package manager knows what libraries are needed and installs them automatically.
- **Clean removal** — Every installed file is recorded, so the package manager can remove all of them without leaving anything behind.
- **Upgrade support** — The package manager can replace one version with another while preserving your configuration files.
- **Reproducibility** — Anyone can rebuild the exact same package from the same inputs on any compatible system.

## Overview

|                       | `.deb` (Debian/Ubuntu)                       | `.rpm` (Fedora/RHEL) |
|-----------------------|----------------------------------------------|----------------------|
| **Metadata location** | `debian/` directory inside the source tree   | `.spec` file in `~/rpmbuild/SPECS/` |
| **Build command**     | `dpkg-buildpackage`                          | `rpmbuild` |
| **Key tool packages** | `build-essential`, `devscripts`, `debhelper` | `rpm-build`, `rpmdevtools` |
| **Output**            | `../<name>_<version>_<arch>.deb`             | `~/rpmbuild/RPMS/<arch>/<name>-<version>.<arch>.rpm` |
| **Automation**        | `debhelper` (`dh`) auto-detects build system | Explicit commands in `.spec`, with RPM macros for common patterns |

Both examples below build the same minimal C program — a "Hello, World!" binary — so you can compare the two workflows side by side.

### The Source Code

Both examples use this simple program:

```c
/* hello.c */
#include <stdio.h>
int main() {
    printf("Hello, World!\n");
    return 0;
}
```

And this `Makefile`:

```makefile
PREFIX ?= /usr

hello: hello.c
	gcc -o hello hello.c

install: hello
	install -D -m 755 hello $(DESTDIR)$(PREFIX)/bin/hello

clean:
	rm -f hello
```

The `install` target uses two variables — `PREFIX` (the base install directory) and `DESTDIR` (a staging directory prepended to all paths). During a package build, the build tool sets `DESTDIR` to a temporary directory so files are collected there instead of being installed directly onto the system. Both `dpkg-buildpackage` and `rpmbuild` rely on this mechanism. For a detailed explanation of `PREFIX`, `DESTDIR`, and the `install` command, see [Make Install](https://github.com/ManiAm/primer-build-tools/blob/master/docs/15_make_install.md).


## Building a `.deb` Package

A Debian source package requires a `debian/` directory inside the source tree. The minimum files needed are:

```
hello-1.0/
├── hello.c
├── Makefile
└── debian/
    ├── changelog
    ├── control
    ├── rules
    └── source/
        └── format
```

### The `debian/changelog` File

Records the package version and release history. The format is strict — the spacing, punctuation, and indentation must be exact:

```
hello (1.0-1) noble; urgency=low

  * Initial release.

 -- Example User <user@example.com>  Fri, 20 Nov 2026 12:00:00 +0000
```

The version string (`1.0-1`) and the distribution name (`noble`) in this file determine the version and target release of the built package. `1.0` is the **upstream version** (the version of the original software), and `-1` is the **Debian revision** (incremented when the packaging changes but the upstream source does not). Because the format is strict, you can use the `dch` command (from the `devscripts` package) to create and update this file safely instead of editing it by hand.

### The `debian/control` File

The `control` file is the central piece of Debian packaging metadata. It describes both the **source package** and the **binary package(s)** it produces:

```
Source: hello
Section: utils
Priority: optional
Maintainer: Example User <user@example.com>
Build-Depends: debhelper-compat (= 13), gcc
Standards-Version: 4.6.2

Package: hello
Architecture: any
Depends: ${shlibs:Depends}, ${misc:Depends}
Description: Simple hello world program
 A minimal example package that prints "Hello, World!"
 to the terminal.
```

| Field           | Purpose |
|-----------------|---------|
| `Source`        | Name of the source package. |
| `Build-Depends` | Packages needed to compile the software. `debhelper-compat (= 13)` specifies the debhelper compatibility level. |
| `Package`       | Name of the binary package (what gets installed). |
| `Architecture`  | `any` means it is compiled for each architecture. Use `all` for architecture-independent packages (scripts, data). |
| `Depends`       | Runtime dependencies. `${shlibs:Depends}` is automatically filled in by `debhelper` based on the linked shared libraries. |
| `Description`   | A short summary on the first line, followed by a longer description indented with a space. |

### The `debian/rules` File

A Makefile that defines how to build the package. Unlike a regular Makefile, this file **must** have the executable permission bit set (`chmod +x debian/rules`) because `dpkg-buildpackage` invokes it directly. For most projects, `debhelper` handles everything:

```makefile
#!/usr/bin/make -f
%:
	dh $@
```

This three-line file delegates every build step — configure, compile, install, and package — to `debhelper`. When `debhelper` runs, it detects the `Makefile` in the source tree, calls `make` to compile and `make install` to stage files, and then packages the result. `debhelper` supports many build systems automatically, including Make, CMake, Autotools, Python setuptools, and Meson.

### The `debian/source/format` File

Specifies the source package format:

```
3.0 (native)
```

`3.0 (native)` is for software that exists only as a Debian package (no separate upstream project). `3.0 (quilt)` is the more common choice — it is used when the software originates from an upstream project and the Debian packaging may include patches applied on top.

### Building

Run the build from the source directory:

```
$ cd hello-1.0/
$ dpkg-buildpackage -us -uc -b
```

| Flag  | Purpose |
|-------|---------|
| `-us` | Do not sign the source package. |
| `-uc` | Do not sign the `.changes` file. |
| `-b`  | Build binary packages only (skip source package). |

The build output shows each step:

```
dpkg-buildpackage: info: source package hello
dpkg-buildpackage: info: source version 1.0-1
dpkg-buildpackage: info: host architecture amd64
 debian/rules clean
 debian/rules build
   dh_auto_build
	make -j36 "INSTALL=install --strip-program=true"
	gcc -o hello hello.c
 debian/rules binary
   dh_auto_install --destdir=debian/hello/
	make -j36 install DESTDIR=debian/hello
	install -D -m 755 hello debian/hello/usr/bin/hello
   dh_installdocs
   dh_installchangelogs
   dh_strip
   dh_shlibdeps
   dh_installdeb
   dh_gencontrol
   dh_md5sums
   dh_builddeb
dpkg-deb: building package 'hello' in '../hello_1.0-1_amd64.deb'.
```

The resulting `.deb` file is placed one directory above the source tree:

```
$ dpkg -c ../hello_1.0-1_amd64.deb
drwxr-xr-x root/root         0 2026-11-20 12:00 ./usr/
drwxr-xr-x root/root         0 2026-11-20 12:00 ./usr/bin/
-rwxr-xr-x root/root     14488 2026-11-20 12:00 ./usr/bin/hello
drwxr-xr-x root/root         0 2026-11-20 12:00 ./usr/share/
drwxr-xr-x root/root         0 2026-11-20 12:00 ./usr/share/doc/
drwxr-xr-x root/root         0 2026-11-20 12:00 ./usr/share/doc/hello/
-rw-r--r-- root/root       131 2026-11-20 12:00 ./usr/share/doc/hello/changelog.Debian.gz
```


## Building an `.rpm` Package

### Directory Structure

RPM expects source code and spec files in a fixed directory tree under `~/rpmbuild/`. Create this tree with:

```
$ rpmdev-setuptree
$ ls ~/rpmbuild/
BUILD  RPMS  SOURCES  SPECS  SRPMS
```

| Directory  | Purpose |
|------------|---------|
| `SPECS/`   | Contains `.spec` files (the build recipes). |
| `SOURCES/` | Contains source tarballs and patches. |
| `BUILD/`   | Temporary directory where sources are unpacked and compiled. |
| `RPMS/`    | Output directory for built binary RPMs. |
| `SRPMS/`   | Output directory for source RPMs (`.src.rpm`). |

Place the source code in a tarball:

```
$ tar czf ~/rpmbuild/SOURCES/hello-1.0.tar.gz hello-1.0/
```

### The `.spec` File

The `.spec` file is the single file that defines everything about an RPM package — metadata, build instructions, and the list of files to include. Unlike Debian, which spreads this information across several files in the `debian/` directory, RPM consolidates it all in one place. Create `~/rpmbuild/SPECS/hello.spec`:

```spec
%global debug_package %{nil}

Name:           hello
Version:        1.0
Release:        1%{?dist}
Summary:        Simple hello world program
License:        MIT
Source0:        hello-1.0.tar.gz

BuildRequires:  gcc

%description
A minimal example package that prints "Hello, World!" to the terminal.

%prep
%setup -q

%build
make

%install
make install DESTDIR=%{buildroot}

%files
%{_bindir}/hello

%changelog
* Fri Nov 20 2026 Example User <user@example.com> - 1.0-1
- Initial release
```

> **Note:** `%global debug_package %{nil}` disables automatic generation of `debuginfo` and `debugsource` sub-packages. Fedora's build system generates these by default, but our minimal Makefile does not produce debug symbols, so the build would fail without this line. Production packages typically use `%make_build` (which passes the distribution's standard compiler flags, including `-g` for debug info) and omit this line.

#### Spec File Sections

| Section | Purpose |
|---------|---------|
| **Header** (`Name`, `Version`, etc.) | Package metadata — name, version, license, source files, and build dependencies. |
| `%description` | Long description of the package (can be multi-line). |
| `%prep` | Prepares the source tree. `%setup -q` extracts the tarball from `SOURCES/` into `BUILD/` and changes into the extracted directory. The `-q` flag suppresses verbose output. |
| `%build` | Compiles the software. This is where you run `make`, `cmake`, `./configure`, or any other build command. |
| `%install` | Installs files into a temporary staging root (`%{buildroot}`) by running `make install` with `DESTDIR` set. Only files placed here will be included in the final RPM. |
| `%files` | Lists every file that should be included in the package. RPM macros like `%{_bindir}` expand to standard paths (e.g., `/usr/bin`). |
| `%changelog` | History of changes. Each entry follows a strict format: `* Day-of-week Month Day Year Author <email> - Version`. |

#### Common RPM Macros

| Macro            | Expands To               | Purpose |
|------------------|--------------------------|---------|
| `%{buildroot}`   | Temporary install root   | Where files are staged before packaging. |
| `%{_bindir}`     | `/usr/bin`               | Standard binary directory. |
| `%{_sbindir}`    | `/usr/sbin`              | System binary directory. |
| `%{_libdir}`     | `/usr/lib64` (on x86_64) | Library directory. |
| `%{_sysconfdir}` | `/etc`                   | Configuration directory. |
| `%{_datadir}`    | `/usr/share`             | Architecture-independent data. |
| `%{_mandir}`     | `/usr/share/man`         | Manual pages. |
| `%{?dist}`       | `.fc41`, `.el9`, etc.    | Distribution tag, appended to the release. |

### Building

Run the build from any directory by pointing `rpmbuild` at the spec file:

```
$ rpmbuild -ba ~/rpmbuild/SPECS/hello.spec
```

| Flag  | Purpose |
|-------|---------|
| `-ba` | Build **a**ll (both binary and source RPMs). |
| `-bb` | Build **b**inary RPM only. |
| `-bs` | Build **s**ource RPM only. |

With `-ba`, `rpmbuild` produces both a binary RPM (ready to install) and a source RPM (containing the spec file and tarball for reproducible builds). The key output lines:

```
Wrote: /root/rpmbuild/SRPMS/hello-1.0-1.fc41.src.rpm
Wrote: /root/rpmbuild/RPMS/x86_64/hello-1.0-1.fc41.x86_64.rpm
```

Inspect the result:

```
$ rpm -qpi ~/rpmbuild/RPMS/x86_64/hello-1.0-1.fc41.x86_64.rpm
Name        : hello
Version     : 1.0
Release     : 1.fc41
Architecture: x86_64
License     : MIT
Summary     : Simple hello world program
Description :
A minimal example package that prints "Hello, World!" to the terminal.
```


## Side-by-Side Comparison

| Step | `.deb` (Debian/Ubuntu) | `.rpm` (Fedora/RHEL) |
|------|------------------------|----------------------|
| **Install build tools** | `apt install build-essential debhelper` | `dnf install rpm-build rpmdevtools` |
| **Metadata** | `debian/` directory with `control`, `changelog`, `rules`, `source/format` | Single `.spec` file |
| **Package description** | `debian/control` | Header and `%description` in `.spec` |
| **Build dependencies** | `Build-Depends:` in `debian/control` | `BuildRequires:` in `.spec` |
| **Build rules** | `debian/rules` (Makefile using `dh`) | `%build` section in `.spec` |
| **File installation** | Automatic via `dh_auto_install` | `%install` section in `.spec` |
| **File listing** | Automatic (debhelper detects installed files) | Explicit `%files` section in `.spec` |
| **Build command** | `dpkg-buildpackage -us -uc -b` | `rpmbuild -ba hello.spec` |
| **Output location** | `../hello_1.0-1_amd64.deb` | `~/rpmbuild/RPMS/x86_64/hello-1.0-1.fc41.x86_64.rpm` |

### Key Differences

- **Metadata location** — Debian puts packaging metadata *inside* the source tree (the `debian/` directory). RPM keeps it *separate* — the `.spec` file and `~/rpmbuild/` tree live outside the source code.

- **Build automation** — Debian's `debhelper` (`dh`) automates most build steps: it detects the build system, runs the right compile and install commands, generates dependency information, and compresses documentation. RPM requires explicit build and install commands in the `.spec` file (e.g., `make` in `%build`, `make install` in `%install`), though macros like `%configure`, `%make_build`, and `%make_install` provide shorthand for common patterns.

- **File listing** — Debian automatically detects which files were installed and includes them in the package. RPM requires an explicit `%files` section listing every file — if a file is installed but not listed, the build fails.

- **Source packages** — `rpmbuild -ba` produces both a binary RPM and a **source RPM** (`.src.rpm`) in one step. The source RPM contains the spec file and source tarball, allowing anyone to reproduce the build. Debian has a similar concept (`.dsc` + `.orig.tar.gz` + `.debian.tar.xz`), but producing it requires a separate build step.


## Building in Clean Environments

Building packages on your everyday system can introduce **hidden dependencies** — the build might succeed only because you happen to have extra packages installed that are not listed as build dependencies. When someone else tries to build the same package on a minimal system, it fails because those unlisted dependencies are missing. To catch these problems early, both ecosystems provide tools that build packages inside a **clean environment** — a minimal, isolated system with only the declared build dependencies installed:

### Debian/Ubuntu

**`pbuilder`** creates a minimal chroot (an isolated copy of the base system), installs only the declared `Build-Depends`, and builds the package inside it:

```
# Create the clean build environment (once)
$ sudo pbuilder create --distribution noble

# Build a package in the clean environment
$ sudo pbuilder build ../hello_1.0-1.dsc
```

**`sbuild`** is a similar tool used by the official Debian build infrastructure. It can use chroots, containers, or virtual machines.

### Fedora/RHEL

**`mock`** works the same way for RPM packages — it creates a minimal chroot from a Fedora/RHEL repository, installs only the declared `BuildRequires`, and builds the package:

```
# Build a source RPM in a clean Fedora 41 environment
$ mock -r fedora-41-x86_64 ~/rpmbuild/SRPMS/hello-1.0-1.fc41.src.rpm
```

Both `pbuilder`/`sbuild` and `mock` ensure that the build dependencies declared in the package metadata are complete and accurate.

---

**Previous**: [← DNF](05_dnf.md)
