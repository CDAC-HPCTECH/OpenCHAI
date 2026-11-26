# OpenCHAI Ansible and Python Upgrade Guide

**Purpose:** Record the controller-side Python/Ansible upgrade and the
EL10 compatibility fixes made while deploying xCAT through OpenCHAI.

**Environment validated in this update**

  Component                            Value
  ------------------------------------ ------------------------------
  Ansible controller                   `cn04`
  Existing controller Python           Python 3.9.21
  Existing Ansible Core                2.14.18
  New controller Python                Python 3.12.16
  New Ansible Core                     2.21.6
  Virtual environment                  `/opt/openchai/ansible/venv`
  EL10 managed nodes                   `master01`, `master02`
  EL10 managed-node Python             Python 3.12.13
  xCAT repository version in the run   2.19
  Repository OS path                   `rh10/x86_64`

> This guide records the configuration used in the reported deployment.
> It is not a claim that every historical RHEL release is supported.
> Validate each RHEL/Rocky/AlmaLinux major release and architecture
> against the chosen Ansible Core and xCAT release before declaring it
> supported.

------------------------------------------------------------------------

## 1. Why the upgrade was required

The original controller environment was:

``` text
Ansible Core: 2.14.18
Controller Python: 3.9.21
```

The EL10 managed nodes were running:

``` text
Python 3.12.13
```

The original `get_url` task failed with:

``` text
HTTPSConnection.__init__() got an unexpected keyword argument 'cert_file'
```

This is consistent with an incompatibility between the older Ansible
HTTP implementation and the Python 3.12 HTTP API used on the managed
node.

The solution was to use a newer Ansible Core in an isolated controller
virtual environment. The system Python and original system Ansible
installation were left untouched.

## 2. Controller: create the isolated Ansible environment

Run these commands on `cn04` as an administrator.

### 2.1 Check Python 3.12 availability

``` bash
python3.12 --version
```

If the interpreter and pip package are not installed, install them using
the controller's approved OS repositories. On a compatible RHEL/Rocky
host:

``` bash
dnf install -y python3.12 python3.12-pip
```

Package names can vary by OS release and enabled repositories. Confirm
availability before running the installation command.

### 2.2 Create the virtual environment

``` bash
mkdir -p /opt/openchai/automation/ansible

python3.12 -m venv /opt/openchai/automation/ansible/venv

source /opt/openchai/automation/ansible/venv/bin/activate

python -m pip install --upgrade pip setuptools wheel

python -m pip install "ansible-core==2.21.6"
```

The deployment reported the following working environment:

``` text
Ansible Core: 2.21.6
Python:       3.12.16
Venv:         /opt/openchai/automation/ansible/venv
```

### 2.3 Verify the environment

``` bash
source /opt/openchai/automation/ansible/venv/bin/activate

which ansible
which ansible-playbook

python --version
ansible --version
python -m pip show ansible-core
```

Expected executable path:

``` text
/opt/openchai/automation/ansible/venv/bin/ansible
/opt/openchai/automation/ansible/venv/bin/ansible-playbook
```

Always use the virtual-environment executable when running OpenCHAI
playbooks. Do not rely on whichever `ansible-playbook` happens to appear
first in the system `PATH`.

### 2.4 Preserve the old installation

Do not uninstall or overwrite the system-managed Ansible 2.14.18
installation during migration. Keeping it intact provides a rollback
path while OpenCHAI roles and collections are tested with the new
environment.

------------------------------------------------------------------------

## 3. Managed-node Python interpreter

Ansible modules execute using Python on the managed node. The
controller's Python version does not automatically become the
managed-node Python version.

The reported EL10 nodes use Python 3.12.13. Check the actual interpreter
paths:

``` bash
command -v python3
python3 --version
```

Check all hosts through Ansible:

``` bash
ansible all -i inventory_def.txt -m setup \
  -a 'filter=ansible_python*'
```

If the inventory has known interpreter paths, define them explicitly per
OS group. Example only:

``` ini
[el9:vars]
ansible_python_interpreter=/usr/bin/python3

[el10:vars]
ansible_python_interpreter=/usr/bin/python3
```

For EL8, set the interpreter to a supported installed Python version
that meets the managed-node requirements of the selected Ansible Core
release. Do not assume `/usr/bin/python3` has the same version on every
EL8 derivative.

**Important:** Ansible Core 2.21 has specific controller and
managed-node Python support requirements. Check the official Ansible
Core 2.21 support matrix before adding older RHEL releases or unusual
architectures. RHEL 7 and other legacy systems may require a separate
legacy automation environment.

------------------------------------------------------------------------

## 4. Ansible configuration and callback plugin

The first run with the new Ansible version stopped with:

``` text
Could not load 'yaml' callback plugin.
redirecting (type: callback) ansible.builtin.yaml
to community.general.yaml
```

The configuration was using the old `yaml` stdout callback name.

### 4.1 Recommended simple configuration

In `/etc/ansible/ansible.cfg`, use:

``` ini
[defaults]
stdout_callback = default
```

The default callback avoids a dependency on the YAML callback
collection.

If YAML-formatted output is required instead, install a compatible
`community.general` collection into the collection path used by this
virtual environment and configure the fully qualified callback name. Pin
and test the collection version as part of the OpenCHAI environment.

### 4.2 Inventory warnings

The run also reported:

``` text
Failed to load inventory plugin, skipping json
Failed to load inventory plugin, skipping txt
Found both group and host with same name: headnode
```

Investigate these separately; they are not the xCAT HTTPS or GPG errors.

Inspect inventory configuration:

``` bash
grep -nE 'inventory|enable_plugins|stdout_callback' \
  /etc/ansible/ansible.cfg
```

For a plain INI inventory, a suitable plugin list may include:

``` ini
[inventory]
enable_plugins = host_list, script, auto, yaml, ini
```

Do not remove a custom inventory script/plugin required by OpenCHAI.
Confirm the actual inventory source format before changing the plugin
list.

Inspect the parsed inventory:

``` bash
ansible-inventory -i inventory_def.txt --graph -vvv
```

Resolve host/group name collisions by using distinct names, for example:

``` ini
[headnodes]
headnode01 ansible_host=192.0.2.10
```

Use the actual inventory addresses and naming conventions in the
deployment.

------------------------------------------------------------------------

## 5. xCAT repository configuration for EL releases

The xCAT host setup role should derive its repository OS path from the
managed node's facts rather than hardcoding `rh8`.

For example:

``` yaml
- name: Configure xCAT repository variables
  ansible.builtin.set_fact:
    xcat_repo_version: "{{ xcat_version | default('2.19') }}"
    xcat_core_type: >-
      {{ 'core-snap' if (xcat_version | default('2.19')) == 'devel'
         else 'xcat-core' }}
    xcat_repo_os: "rh{{ ansible_distribution_major_version }}"
    xcat_repo_arch: "{{ ansible_architecture }}"
    xcat_repo_root: >-
      {{ xcat_reporoot | default('https://xcat.org/files/xcat/repos/yum') }}
```

This produces values such as:

``` text
EL8: rh8
EL9: rh9
EL10: rh10
Architecture: x86_64 (when reported by Ansible)
```

Do not assume every architecture or every xCAT version has a published
repository. Confirm that the exact repository URL exists before enabling
it.

### 5.1 Repository file downloads

The role uses `ansible.builtin.get_url` to download repository
configuration files. Keep TLS certificate validation enabled:

``` yaml
- name: Download xCAT core repo file
  ansible.builtin.get_url:
    url: "{{ xcat_repo_root }}/{{ xcat_repo_version }}/{{ xcat_core_type }}/xcat-core.repo"
    dest: /etc/yum.repos.d/xcat-core.repo
    owner: root
    group: root
    mode: "0644"
    force: true
    validate_certs: true
  register: xcat_core_repo
  retries: 3
  delay: 5
  until: xcat_core_repo is succeeded
```

The dependency repository URL used for EL10 x86_64 was:

``` text
https://xcat.org/files/xcat/repos/yum/2.19/xcat-dep/rh10/x86_64/xcat-dep.repo
```

Use the corresponding derived OS and architecture variables for other
hosts.

The xCAT 2.19 repository layout and available repository files must be
checked against the official xCAT documentation/repository for the
selected release. Do not assume a `common` repository file or
signing-key URL exists unless verified for that release.

------------------------------------------------------------------------

## 6. xCAT GPG signing-key issue on EL10

After the Python/Ansible issue was resolved, DNF reached the xCAT
repositories but failed while validating the dependency repository
metadata:

``` text
Failed to download metadata for repo 'xcat-dep':
repomd.xml GPG signature verification error: Signing key not found
```

DNF also displayed an interactive import prompt:

``` text
Importing GPG key 0x8D818E69
Userid: xCAT Signing Key
Fingerprint: 6390 CEB7 8544 C1AF 4FD7 2DCC 64C8 2A86 8D81 8E69
Is this ok [y/N]:
```

An unattended Ansible task cannot reliably answer an interactive trust
prompt. The fix is to import the repository signing key explicitly
before refreshing repository metadata.

### 6.1 Verify the key fingerprint

The fingerprint reported by the repository in this deployment was:

``` text
6390 CEB7 8544 C1AF 4FD7 2DCC 64C8 2A86 8D81 8E69
```

Verify the key fingerprint using an independently trusted official xCAT
source or release documentation before accepting it. Do not trust a key
merely because it was downloaded from the same endpoint whose metadata
is being verified.

### 6.2 Import the dependency signing key

For the reported EL10 x86_64 repository, the key URL was:

``` text
https://xcat.org/files/xcat/repos/yum/2.19/xcat-dep/rh10/x86_64/repodata/repomd.xml.key
```

A manual import, after independently verifying the key, can be performed
with:

``` bash
rpm --import \
  https://xcat.org/files/xcat/repos/yum/2.19/xcat-dep/rh10/x86_64/repodata/repomd.xml.key
```

Verify the imported key:

``` bash
rpm -qa 'gpg-pubkey*'
```

### 6.3 Ansible task

Add the key-import task before `dnf makecache`:

``` yaml
- name: Import verified xCAT dependency signing key
  ansible.builtin.rpm_key:
    state: present
    key: "{{ xcat_repo_root }}/{{ xcat_repo_version }}/xcat-dep/{{ xcat_repo_os }}/{{ xcat_repo_arch }}/repodata/repomd.xml.key"
    fingerprint:
      - "6390CEB78544C1AF4FD72DCC64C82A868D818E69"
```

Only add core/common repository key-import tasks if those repositories
publish separate keys or require them. Verify their URLs and
fingerprints independently; do not assume all repositories use the same
key without checking.

Keep repository signature verification enabled. Do **not** use
`gpgcheck=0`, `repo_gpgcheck=0`, or `--nogpgcheck` as a workaround.

### 6.4 Refresh repository metadata non-interactively

``` yaml
- name: Refresh DNF repository metadata
  ansible.builtin.command:
    argv:
      - dnf
      - -y
      - makecache
      - --refresh
  changed_when: false
```

The `-y` flag handles package-manager confirmation prompts. It does not
replace explicit GPG key verification/import.

------------------------------------------------------------------------

## 7. Validate before deployment

Run checks in this order.

### 7.1 Confirm the controller environment

``` bash
source /opt/openchai/ansible/venv/bin/activate

ansible --version
python --version
```

### 7.2 Validate inventory parsing

``` bash
ansible-inventory -i inventory_def.txt --graph
```

### 7.3 Test connectivity

``` bash
ansible hpc_master -i inventory_def.txt -m ping
```

### 7.4 Check managed-node Python versions

``` bash
ansible hpc_master -i inventory_def.txt -m setup \
  -a 'filter=ansible_python_version'
```

### 7.5 Check playbook syntax

``` bash
ansible-playbook \
  phase05.2_provisioning_xcat_host_setup.yml \
  -i inventory_def.txt \
  -l hpc_master \
  --syntax-check
```

### 7.6 Run a check-mode review

``` bash
ansible-playbook \
  phase05.2_provisioning_xcat_host_setup.yml \
  -i inventory_def.txt \
  -l hpc_master \
  --check --diff
```

Check mode cannot fully validate every download, package installation,
repository metadata operation, or external side effect. Follow it with a
real run on a test node.

### 7.7 Execute the playbook

``` bash
ansible-playbook \
  phase05.2_provisioning_xcat_host_setup.yml \
  -i inventory_def.txt \
  -l hpc_master \
  -vv
```

------------------------------------------------------------------------

## 8. Operational recommendations

1.  **Pin versions:** Keep the Ansible Core version and required
    collection versions in a requirements file. Upgrade them only after
    testing OpenCHAI roles.
2.  **Keep environments isolated:** Do not replace system Python or
    system Ansible just to update OpenCHAI.
3.  **Use explicit interpreters where necessary:** Ensure each managed
    OS has a supported Python interpreter.
4.  **Keep TLS and GPG verification enabled:** Do not disable
    certificate or package-signature checks to get a successful run.
5.  **Validate repository paths:** Check xCAT repository availability
    for each OS major version, architecture, and xCAT release.
6.  **Test DNF behavior:** EL10 uses a newer DNF stack; validate
    repository-management and package tasks on EL10 as well as EL8/EL9.
7.  **Preserve rollback:** Keep the old controller environment until the
    new environment has passed the OpenCHAI regression tests.
8.  **Record actual test results:** Mark an OS/version combination
    supported only after the full installation workflow succeeds on that
    combination.

------------------------------------------------------------------------

## 9. Troubleshooting quick reference

  ----------------------------------------------------------------------------------------
  Symptom                             Likely area               First check
  ----------------------------------- ------------------------- --------------------------
  `HTTPSConnection ... cert_file`     Old Ansible / managed     Controller Ansible version
                                      Python compatibility      and managed-node Python

  `Could not load 'yaml' callback`    Callback configuration or `stdout_callback` in
                                      missing collection        `ansible.cfg`

  `Failed to load inventory plugin`   Inventory source/plugin   `ansible-inventory -vvv`
                                      configuration             

  Host and group share a name         Inventory naming          Inventory group/host
                                      collision                 definitions

  `Signing key not found`             Repository GPG key not    Verify fingerprint, import
                                      imported/trusted          key, retry metadata

  `No match for argument`             Repository/package        Enabled repos and
                                      availability              EL-specific package names

  Repository URL returns 404          Incorrect                 Verify the exact xCAT
                                      OS/architecture/release   repository URL
                                      path                      
  ----------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 10. Change record

  ------------------------------------------------------------------------------
  Area                    Previous state          Updated state
  ----------------------- ----------------------- ------------------------------
  Controller Python       3.9.21                  3.12.16 in dedicated venv

  Ansible Core            2.14.18                 2.21.6 in dedicated venv

  System Ansible          Existing installation   Preserved for rollback

  Managed EL10 Python     3.12.13                 Unchanged

  YAML callback           Legacy `yaml` name      Use `default`, or
                                                  install/configure compatible
                                                  collection

  xCAT OS repository path Previously              Derived from OS major version
                          hardcoded/EL-specific   

  xCAT dependency GPG key Interactive DNF import  Explicit fingerprint-validated
                                                  import

  Metadata refresh        Interactive             `dnf -y makecache --refresh`
                          confirmation possible   
  ------------------------------------------------------------------------------

**Completion note:** The reported deployment progressed past the
Python/Ansible compatibility issue and the xCAT repository GPG issue
after the corresponding changes. Re-run the validation steps after
future changes to Ansible, Python, xCAT repositories, or OS versions.

## 11. Official references

-   Ansible Core documentation: https://docs.ansible.com/ansible/latest/
-   Ansible Core support matrix:
    https://docs.ansible.com/ansible/latest/reference_appendices/release_and_maintenance.html
-   Ansible `get_url` module:
    https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html
-   Ansible `rpm_key` module:
    https://docs.ansible.com/ansible/latest/collections/ansible/builtin/rpm_key_module.html
-   xCAT documentation: https://xcat-docs.readthedocs.io/
-   xCAT repository: https://xcat.org/files/xcat/repos/yum/
