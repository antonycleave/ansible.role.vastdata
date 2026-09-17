# Ansible role to download vast, build and install

## What it does

This will download build and optionally install the vast NFS driver

it has been tested on:

- Rocky 8.10
  - mlnx ofed - driver builds but does not function
  - no ofed
- Rocky 9.5
  - mlnx ofed
  - no ofed
- Ubuntu 22.04 (6.5.0 kernel)
  - mlnx ofed
  - no ofed
- Ubuntu 22.04 (5.5.0 kernel)
  - no ofed
- Ubuntu 24.04 (6.17.0 kernel, aarch64/GB300)
  - mlnx ofed (DOCA 3.3.0, OFED-internal-26.01-1.0.0), vastnfs 4.5.9

with the default options it downloads the **pinned** version, verifies it
against a known sha256, patches the vendor makefile so the compiler the kernel
was built with survives, and builds in /tmp. Once built it copies the build
artifacts to the default publish dir (/opt/vast/). Finally it installs the
package and, on RHEL systems, excludes the kernel from further updates.

### Pinned by default

`vastdata_version` defaults to a specific release, with its sha256 in
`vastdata_known_checksums`. An unpinned build is not reproducible — two runs a
week apart can ship different drivers. Set `vastdata_version: ""` to opt back
in to the metadata-service lookup, and add a checksum entry when pinning
something new or the download goes unverified.

### The makefile patch is load-bearing

`files/makefile.patch` wraps vastnfs's unconditional `CC = $(CROSS_COMPILE)gcc`
in an `ifndef CC` guard. vastnfs's own DKMS runner detects the compiler the
target kernel was built with and passes it in; without the guard the makefile
overwrites that with plain `gcc`. On aarch64/6.17 a patched build logs
`CC='aarch64-linux-gnu-gcc-13'` and an unpatched one logs `CC='gcc'` — the same
compiler *there*, but not on an Ubuntu HWE kernel. The role no longer sets `CC`
itself: the old `vastdata_kernel_cc` default was x86-only and was applied to
the packaging step rather than the compile, so it never took effect on a DKMS
build.

### OFED mode is mostly not yours to choose

Both vendor build scripts read `ofed_info -s` and then pick DKMS or prebuilt
modules from the installed `mlnx-ofed-kernel-*` / `mlnx-ofa_kernel-*` package,
**ignoring `--dkms`**. The role therefore asserts up front that an OFED build
has an OFED kernel package to detect (otherwise the script exits -1 with "OFED
setup type not detected"), and asserts afterwards that the package it asked for
is the one that got built. Set `vastdata_ofed_mode: none` to pass `--no-ofed`
and build against the distro kernel only.

### Re-runs

The role skips download, build and install when the package this build would
produce is already installed, so a second run is a no-op instead of a rebuild.
The comparison reconstructs the full package version, including the OFED
release the vendor scripts bake into it
(`4.5.9-vastdata-OFED-internal-26.01-1.0.0`), so an OFED/DOCA bump looks like a
different version and rebuilds without anyone having to remember a flag.
`vastdata_force_rebuild: true` is the escape hatch for the rest — a changed
kernel, a partial install, or just wanting the build re-run.

Note that installing a rebuild of an *identical* version is an apt/dnf no-op,
so the task reports `ok` and the handlers do not fire. That is correct: there
is nothing to change. Use `vastdata_force_rebuild` with a genuinely different
version (or a purge) if you need the module physically rebuilt.

## Dependencies

This ansible is fairly generic but it has only been tested using ansible-playbook [core 2.11.12]

https://github.com/antonycleave/ansible.role.yumexclude.git

it has been tested with ansible.role.yumexclude.git v 0.1.0 only

## Example Run

**configure Ansible to put galaxy roles where we can see them**
```
cat <<EOF >ansible.cfg
[defaults]
roles_path = roles.galaxy:roles
EOF
```

**create ansible galaxy requirements file**
```
cat <<EOF >requirements.yml
---
roles:
  - name: yumexclude
    src: https://github.com/antonycleave/ansible.role.yumexclude.git
    version: v0.1.0

  - name: vastdata
    src: https://github.com/antonycleave/ansible.role.vastdata.git
    version: v0.1.0
EOF
```

**install roles using ansible galaxy**
```
ansible-galaxy install -r requirements.yml
```

**Prep inventory.**
This is mine with a full testing suite of VMs make your own as appropriate
```
cat inventory
[all:vars]
ansible_user=ubuntu

[test:children]
test_ubuntu
test_rocky

[test_ubuntu]
test-ubuntu-22-stock ansible_host=192.168.7.248
test-ubuntu-22-bcom ansible_host=192.168.7.211
test-ubuntu-22-mlnx ansible_host=192.168.7.50

[test_rocky]
test-rocky-8-bcom ansible_host=192.168.7.54
test-rocky-9-bcom ansible_host=192.168.7.150
test-rocky-8-mlnx ansible_host=192.168.7.178
test-rocky-9-mlnx ansible_host=192.168.7.147
[test_rocky:vars]
ansible_user=rocky
```

**create example playbook**
this one targets a group called test in the inventory I just showed you

```
cat <<EOF >site.yml
---
- hosts: test
  become: true
  roles:
    - vastdata
EOF
```

**run ansible-playbook**

```
ansible-playbook site.yml  -i inventory
```

hopefully you see this at the end:

```
PLAY RECAP *******************************************************************************************************
test-rocky-8-bcom          : ok=22   changed=5    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0
test-rocky-8-mlnx          : ok=22   changed=5    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0
test-rocky-9-bcom          : ok=22   changed=5    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0
test-rocky-9-mlnx          : ok=22   changed=5    unreachable=0    failed=0    skipped=4    rescued=0    ignored=0
test-ubuntu-22-bcom        : ok=17   changed=7    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
test-ubuntu-22-mlnx        : ok=17   changed=7    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
test-ubuntu-22-stock       : ok=16   changed=8    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
```

## Advanced Usage

for most cases the defaults are sane. But if you want a specific version of the package hosted on an internal repo you'll need to change:

- `vastdata_baseurl` to point to the internal repo location e.g. `http://myrepo.example.com/vast/source` or `http://myrepo.example.com/vast/{{vastdata_version}}/source`
- `vastdata_version` which is used to build  the extract paths of the default tar files and in the default base URL
- you might be insane and not want a dkms build where it's supported. . . if so set `vastdata_build_dkms` to `False`

If you or vast change the tar internal layout you may also need to override the `vastdata_tar_extracted_dir` variable as it assumes that the tar file creates a `vastnfs-{{ vastdata_version }}` subdir. On this note if you want the extracted source path to persist reboots then change where it goes using `vastdata_tar_extract_dir`

### Variable precedence caveats

I don't like the way inventory (group and host) vars are so low on the ansible precedence list if I make the effort of setting a  group var I want to use it!

To work around this I read OS specific defaults into a dict called `<rolename>_os_defaults` so that it is possible to override these with inventory vars I use this a lot in my roles for os_dependent overridable defaults

```
- name: Load a variable file based on the OS type into the vastdata_os_defaults dict
  ansible.builtin.include_vars:
    file: "{{ item }}"
    name: vastdata_os_defaults
  with_first_found:
    - "{{ ansible_distribution }}-{{ ansible_distribution_major_version }}.yml"
    - "{{ ansible_os_family }}-{{ ansible_distribution_major_version }}.yml"
    - "{{ ansible_distribution }}.yml"
    - "{{ ansible_os_family }}.yml"
```

#### Override

for vars that I want to be overridden I  do this:

```
- name: rebuild_initramfs
  ansible.builtin.command: "{{ vastdata_update_initramfs_cmd | default( vastdata_os_defaults['vastdata_update_initramfs_cmd'] ) }}"
```

this will use `vastdata_update_initramfs_cmd` from and inventory host/group var if it exists and fall back to the var with the same name from a file included from `vars/Debain.yml`, `vars/Ubuntu-22.yml`, `vars/Ubuntu-24.yml` or similar

#### Never Override

sometimes you want the default ansible behavior and in this case you can simply use the one in `vastdata_os_defaults`  like `vastdata_os_defaults['vastdata_pkg_type']` in the following

```
- name: copy build output to permanent location
  ansible.builtin.copy:
    remote_src: True
    src: "{{vastdata_tar_extract_dir}}/vastnfs-{{ vastdata_version }}/{{ vastdata_os_defaults['vastdata_pkg_type'] }}-dist/"
    dest: "{{ vastdata_publish_dir }}"
```

#### Merge?

sometimes we want to combine lists to use both. . . this is almost certainly a bad idea

```
- name: install package deps
  ansible.builtin.package:
    name: "{{ vastdata_packages | default( [] ) + vastdata_os_defaults['vastdata_packages'] | default( [] ) }}"
    state: present
```

this means that if you provide a list of  vastdata_packages in group vars they will get added to the defaults. . . just writing this has made me change the implementation to  use an override like the earlier example but you *could* use this if you wanted dictionaries are probably safer to combine
