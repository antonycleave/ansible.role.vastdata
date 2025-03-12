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

with the default options it will grab the latest version from the VAST metadata service and get the the latest version. It will patch the makfile to allow the default C compiler to be set via and env var to fix an annoying bug in Ubuntu when using the HWE kernels and then build it with the appropriate compiler in /tmp. Once built it will copy the build artifacts to the default publish dir (/opt/vast/). Finally it will install it and on RHEL system exclude the kernel from further updates.

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
  vars:
    ANSIBLE_DEBUG: True # remove this to skip extra debug output, useful for dev
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
    state: latest
```

this means that if you provide a list of  vastdata_packages in group vars they will get added to the defaults. . . just writing this has made me change the implementation to  use an override like the earlier example but you *could* use this if you wanted dictionaries are probably safer to combine
