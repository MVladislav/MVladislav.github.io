---
layout: post
title: CIS - Docker (ansible)
date: 2023-05-20 19:55:00 +0200
categories:
  - CIS
  - Docker
tags:
  - security
  - hardening
  - linux
  - ubuntu
  - docker
  - cis
  - ansible
image: /assets/img/headers/docker.png
---

## \\\\ Description

This entry provides step-by-step instructions for installing Docker and preparing it to run the CIS Docker
benchmark, which applies CIS recommended hardening settings on the Docker service. By following these instructions,
developers and system administrators can install Docker and use it to automate the hardening process,
ensuring that the Docker service itself is configured according to best practices for security and compliance.
This documentation wiki blog is a useful resource for IT professionals who want to improve the security and
performance of their Docker service by installing Docker and applying the CIS benchmark for Docker automation.

---

## \\\\ Information - CIS & Docker

CIS (Center for Internet Security) documentations are guidelines developed by cybersecurity
experts to help organizations improve the security of their computer systems and networks.
They provide recommendations for configuring systems and devices to mitigate
cybersecurity risks and threats. By following these guidelines, organizations can
improve their security, ensure consistency, comply with regulatory requirements, and save costs.

Docker is an open-source platform that allows developers to build, ship, and run applications in containers.
Containers are isolated environments that include all the necessary components to run an application,
such as code, libraries, and system tools.

By setting up Docker and implementing CIS controls on client machines, organizations can achieve several benefits:

- Security: Docker containers provide a more secure way to run applications by isolating them from the host system.
  CIS controls help to harden and secure the client system against cyber threats.
- Portability: Docker containers are portable and can be easily moved between different environments,
  making it easier to deploy applications across multiple machines.
- Efficiency: Docker containers are lightweight and require fewer resources than traditional virtual machines,
  which can help to improve performance and efficiency.
- Consistency: By using Docker containers, developers can ensure that their application
  runs consistently across different environments, reducing the risk of errors and compatibility issues.

Overall, using Docker and implementing CIS controls can help organizations to improve
the security, portability, efficiency, and consistency of their applications and client systems.

---

> Before making any changes to software, systems, or devices,
> it's **important to thoroughly read and understand the configuration options**,
> and verify that the proposed changes align with your requirements.
> This can help avoid unintended consequences and ensure the software, system, or device operates as intended.
> {: .prompt-warning }

---

## \\\\ Prepare - Target Systems

- SSH access:
  - Ansible communicates with the target systems over **SSH**,
    so you need to have SSH access enabled on the target systems.
- Python:
  - Ansible requires Python to be installed on the target systems in order to execute its modules.
    Most Linux distributions come with Python pre-installed, but if Python is not installed on the target systems,
    you will need to install it before running Ansible.
  - Recommended is to have **python3** installed.
- Privilege escalation:
  - Ansible require **root privileges** to perform certain tasks.
    You will need to have a way to escalate privileges on the target systems, such as using `sudo` or `su`.

## \\\\ Prepare - Host System

install ansible on your main device, which you will use to setup devices:

```sh
$sudo apt install python3 python3-pip sshpass

# when python3 version >= "3.11" used
$python3 -m pip install --break-system-packages ansible

# when python3 version < "3.11" used
$python3 -m pip install ansible

# a quick fix for "ansible-galaxy collection install"
# when python3 version >= "3.11" used
$python3 -m pip install --break-system-packages -Iv "resolvelib<0.8.1"
# when python3 version < "3.11" used
$python3 -m pip install -Iv "resolvelib<0.8.1"
```

## \\\\ Prepare - Ansible on Host System

### // clone the project

```sh
$mkdir cis-docker && cd cis-docker
$git clone https://github.com/MVladislav/ansible-docker.git
```

### // prepare your inventory

```sh
$nano inventory.yml
```

> **change** the **host\*** variables to your needs

```yaml
all:
  children:
    clients:
      hosts:
        host1:
          ansible_user: groot
          ansible_host: 192.168.1.10
          ansible_ssh_private_key_file: ~/.ssh/id_ed25519
          pl_docker_cis_rule_2_1: true # to run install in rootless mode
          pl_docker_is_swarm_mode: true # allow run in swarm mode
        host2:
          ansible_user: groot
          ansible_host: 192.168.1.11
          ansible_ssh_private_key_file: ~/.ssh/id_ed25519
          pl_docker_cis_rule_2_1: false # to run install in rootless mode
          pl_docker_is_swarm_mode: false # allow run in swarm mode
  vars:
    ansible_python_interpreter: /usr/bin/python3
```

### // prepare your playbook

```sh
$nano playbook.yml
```

> **verify** the role **variable** are configured to your needs, or change them.\
> also you should **read** the **README** of the ansible role.

{% raw %}

```yaml
- name: DOCKER | CIS | install on clients
  become: true
  remote_user: "{{ ansible_user }}"
  hosts:
    - clients
  roles:
    - role: docker
      docker_user_shell: /bin/zsh
      docker_users_to_add_group:
        - name: "{{ ansible_user }}"
      docker_cis_rule_2_1: "{{ pl_docker_cis_rule_2_1 | default(true) | bool }}" # to run install in rootless mode
      docker_is_swarm_mode: "{{ pl_docker_is_swarm_mode | default(true) | bool }}" # allow run in swarm mode
      # -------------------------
```

{% endraw %}

### // run the role

run the role to setup your defined targets in the inventory
with the installation from docker and the CIS hardenings

> options:
>
> - `-k`: prompt for the SSH password required to connect to the target system
> - `--ask-become-pass`: prompt for the password required to elevate privileges on the target system

```sh
$ansible-playbook -i inventory.yml playbook.yml --ask-become-pass -k
```

---

## \\\\ Resources & More information's

- <https://github.com/MVladislav/ansible-docker>{:target="\_blank"}
- <https://downloads.cisecurity.org>{:target="\_blank"}
