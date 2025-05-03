
# AWX building custom execution environment

An **Execution Environment (EE)** in Ansible AWX is a standardized, containerized environment that encapsulates all dependencies needed to run Ansible Playbooks, including Ansible itself, Python libraries, system tools, and Ansible Collections. Unlike traditional setups where dependencies are installed on a host, EEs provide a consistent, isolated, and portable environment, ensuring that Playbooks execute reliably across different systems. This container-based approach aligns with modern DevOps practices, leveraging tools like Docker or Podman to package and distribute these environments.

The need for Execution Environments arises from the challenges of managing complex dependency chains and ensuring reproducibility in automation workflows. Without EEs, differences in host environments (e.g., Python versions, installed packages) can lead to inconsistent Playbook execution or failures. EEs solve this by allowing users to define custom environments tailored to specific projects, ensuring that all required tools and configurations are bundled into a single, versioned image. This also simplifies scaling and deployment in AWX, as jobs can run in isolated containers without affecting the host system.

## 1. Create a new Python virtual environment


```bash
cd /home
mkdir awx && cd awx
python3 -m venv ansible-builder/
source /home/awx/ansible-builder/bin/activate
```

## 2. Create a working directory for your custom EE

```bash
mkdir /home/awx/ee && cd /home/awx/ee
```

Add the `execution-environment.yml`, `requirements.txt`, `requirements.yml` and `bindep.txt`


- **execution-environment.yml**: The primary file that orchestrates the build process and references requirements.txt and requirements.yml for dependency installation.
- **requirements.txt**: Focuses on Python packages (e.g., libraries for specific modules).
- **requirements.yml**: Focuses on Ansible-specific content (Collections and Roles).

### execution-environment.yml

- **Purpose**: The main configuration file that defines the structure and settings of the Execution Environment image.
- **Content**:
    - • **Base Image**: Specifies the base image (e.g., quay.io/ansible/awx-ee) to build upon.
    - • **Dependencies**: References other files (e.g., requirements.yml, requirements.txt, bindep.txt) for installing packages and modules.
    - • **Additional Build Steps**: Allows custom commands (e.g., installing packages or applying settings) before or after the build process.

```bash
---
version: 3
images:
  base_image:
    name: quay.io/centos/centos:stream9
dependencies:
  ansible_core:
    package_pip: ansible-core>=2.15.8
  ansible_runner:
    package_pip: ansible-runner
  galaxy: requirements.yml
  python: requirements.txt
additional_build_steps:
  append_base:
    - RUN yum upgrade -y
    - RUN yum install -y python3
    - RUN yum install -y python3-pip
    - RUN yum install -y krb5-devel
    - RUN yum install -y krb5-libs
    - RUN yum install -y krb5-workstation
    - RUN yum install -y python3-devel
    - RUN yum install -y gcc
    - RUN yum install -y epel-release
    - RUN python3 -m pip install --upgrade --force pip
    - RUN pip3 install pypsrp[kerberos]
    - RUN pip3 install pyVim PyVmomi
    - COPY --from=quay.io/project-receptor/receptor:latest /usr/bin/receptor /usr/bin/receptor
    - RUN mkdir -p /var/run/receptor
```    


### requirements.txt

- **Purpose**: Defines the **Python packages** required in the Execution Environment.
- **Content**: A list of Python packages (and optional specific versions) to be installed via pip.
- **Example**:

```bash
dnspython
pykerberos
pywinrm
awxkit==21.6.0
urllib3
```

### requirements.yml

- **Purpose**: Specifies **Ansible Collections** or **Roles** to be installed in the Execution Environment.
- **Content**: A list of Ansible Collections or Roles to be fetched from **Ansible Galaxy** or other repositories (e.g., Git).

- **Example**:


```bash
collections:
  - name: ansible.netcommon
  - name: ansible.utils
  - name: ansible.windows
  - name: community.crypto
  - name: community.dns
  - name: community.docker
  - name: community.general
  - name: community.grafana
  - name: community.network
  - name: community.windows
  - name: community.mysql
  - name: community.libvirt
  - name: microsoft.ad
```  

### bindep.txt

- **Purpose**: The bindep.txt file is used to define **system-level dependencies** (binary packages) required in the Execution Environment for Ansible AWX.
- **Content**: A list of system packages (e.g., RPM or DEB packages) needed for the environment, typically installed using the system’s package manager (e.g., yum, apt). These are dependencies that cannot be installed via pip or Ansible Galaxy.

- **Example**:

```bash
git [platform:rpm]
iputils [platform:rpm]
```

## 3. Build and Push Image to Container Registry

```bash
ansible-builder build --tag quay.io/ansible_awx/custom_env:v1.0.1 -v 3
```

### Add extra tags
```bash
docker image ls
docker tag quay.io/ansible_awx/custom_env:v1.0.1 quay.io/ansible_awx/custom_env:latest
```

### Push image to registry
```bash
docker push quay.io/ansible_awx/custom_env:latest
```