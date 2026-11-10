# Add User Ansible Role

## Overview

This role manages Linux user accounts and SSH public keys.

It supports:
- Creating Linux users.
- Creating user groups.
- Adding users to the sudo group.
- Configuring SSH public keys.
- Deleting users and their home directories.
- Compatibility with legacy Ansible 2.8.19.

## 1. Environment requirements

| Component | Version |
|---|---|
| Operating System | Ubuntu 24.04 LTS |
| Ansible Control Node | Ansible 2.8.19 |
| Control Node Python | Python 3.10 |
| Target System Python | Python 3.12 |
| Ansible Python | Python 3.11.18 |
| Python Environment | /opt/ansible-python311 |

**Why Python 3.11?**

Ansible 2.8.19 uses the Python `imp` module, which was removed in Python 3.12.

To maintain compatibility without upgrading Ansible or changing Ubuntu's default Python, we install Python 3.11 alongside Python 3.12 and configure a dedicated virtual environment.

## 2. Install Python 3.11 on Ubuntu 24.04

Ubuntu 24.04 does not provide Python 3.11 in its default repositories.

If approved by your organization's package policy, install it using the Deadsnakes PPA.

```bash
sudo apt update

sudo apt install software-properties-common -y

sudo add-apt-repository ppa:deadsnakes/ppa -y

sudo apt update

sudo apt install python3.11 python3.11-venv -y
```

Verify installation:

```bash
python3.11 --version
```

Expected:

```text
Python 3.11.18
```

**Important:** Do not change `/usr/bin/python3` or replace Ubuntu's default Python 3.12.

## 3. Create Python virtual environment

Create a dedicated virtual environment:

```bash
sudo python3.11 -m venv /opt/ansible-python311
```

Install pip dependencies:

```bash
sudo /opt/ansible-python311/bin/pip install --upgrade pip

sudo /opt/ansible-python311/bin/pip install \
    cffi \
    cryptography \
    pyOpenSSL \
    "urllib3<2"
```

Verify the virtual environment:

```bash
cat /opt/ansible-python311/pyvenv.cfg
```

Expected:

```text
include-system-site-packages = false
version = 3.11.18
```

Verify Python dependencies:

```bash
/opt/ansible-python311/bin/python -c "
import cffi
import cryptography
import OpenSSL
import urllib3

print('CFFI:', cffi.__version__)
print('Cryptography:', cryptography.__version__)
print('OpenSSL:', OpenSSL.__version__)
print('urllib3:', urllib3.__version__)
print('All dependencies loaded successfully')
"
```

## 4. Configure Ansible Python interpreter

Configure the Python interpreter in the inventory for managed Ubuntu 24.04 servers.

Example:

```ini
[all:vars]
ansible_python_interpreter=/opt/ansible-python311/bin/python
```

Alternatively, configure individual hosts:

```ini
10.175.4.47 ansible_python_interpreter=/opt/ansible-python311/bin/python
```

Ensure the virtual environment exists on every managed host using this configuration.

## 5. Verify Ansible connectivity

From the Jenkins control node:

```bash
ansible 10.175.4.47 \
-i inventory/env/hosts \
-m ping \
-v
```

Expected:

```text
10.175.4.47 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Verify the interpreter:

```bash
ansible 10.175.4.47 \
-i inventory/env/hosts \
-m setup \
-a 'filter=ansible_python*'
```

Expected interpreter:

```text
/opt/ansible-python311/bin/python
```

## 6. Add user execution

Example:

```bash
ansible-playbook \
-i inventory/env/hosts \
ssh.yml \
--tags add-user \
--extra-vars "remote=10.175.4.47 ssh_user=haridask public_key='YOUR_PUBLIC_KEY' ansible_python_interpreter=/opt/ansible-python311/bin/python" \
-v
```

If a password is required, supply a securely generated Linux password hash through the `password` variable.

Do not pass an empty password variable.

## 7. Delete user execution

```bash
ansible-playbook \
-i inventory/env/hosts \
ssh.yml \
--tags delete-user \
--extra-vars "remote=10.175.4.47 ssh_user=haridask ansible_python_interpreter=/opt/ansible-python311/bin/python" \
-v
```

**Warning:** The delete operation removes the user and their home directory.

## 8. Troubleshooting

### Error: No module named 'imp'

Cause: Ansible 2.8.19 is running with Python 3.12.

Solution: Configure the target to use Python 3.11.

### Error: No module named '_cffi_backend'

Cause: CFFI and Cryptography dependencies are missing or incompatible.

Solution:

```bash
/opt/ansible-python311/bin/pip install \
    cffi cryptography pyOpenSSL "urllib3<2"
```

### Error: Python interpreter not found

Cause: The configured virtual environment does not exist on the target server.

Solution:

```bash
sudo python3.11 -m venv /opt/ansible-python311
```

Then install the dependencies again.

## 9. Important notes

1. Do not upgrade or replace the existing Jenkins Ansible 2.8.19 installation as part of this setup.
2. Do not modify Ubuntu's default Python 3.12 installation.
3. Install the Python virtual environment on every applicable managed server.
4. Keep Python dependencies isolated inside `/opt/ansible-python311`.
5. Use password hashes instead of plaintext passwords.
6. Python 3.11 is a compatibility workaround, not a long-term supported solution for Ansible 2.8.19. Plan a controlled Ansible upgrade separately.

---

**Role:** add_user  
**Control Node:** Jenkins  
**Ansible Version:** 2.8.19  
**Target OS:** Ubuntu 24.04 LTS