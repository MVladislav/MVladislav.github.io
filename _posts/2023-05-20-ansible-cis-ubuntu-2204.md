---
layout: post
title: CIS - Ubuntu 22.04 (ansible)
date: 2023-05-20 20:00:00 +0200
categories:
  - CIS
  - Ubuntu
tags:
  - security
  - hardening
  - linux
  - ubuntu
  - cis
  - ansible
image: /assets/img/headers/ubuntu.png
---

## \\\\ Description

This entry provides step-by-step instructions for preparing Ansible to run the CIS role and apply CIS
recommended hardening settings on Ubuntu systems. By following these instructions, system administrators
can use Ansible to automate the hardening process and ensure that their Ubuntu systems are configured
according to best practices for security and compliance. This documentation wiki blog is a useful resource
for IT professionals who want to improve the security and performance of their Ubuntu systems using the
CIS benchmark and Ansible automation.

---

## \\\\ Information - CIS & Ubuntu

CIS (Center for Internet Security) documentations are guidelines developed by cybersecurity
experts to help organizations improve the security of their computer systems and networks.
They provide recommendations for configuring systems and devices to mitigate
cybersecurity risks and threats. By following these guidelines, organizations can
improve their security, ensure consistency, comply with regulatory requirements, and save costs.

The CIS Benchmarks provide detailed recommendations for configuring Ubuntu systems
to mitigate common cybersecurity risks and threats. These recommendations cover
a wide range of security topics, such as password policies, network settings,
file system permissions, and user accounts.

The CIS Benchmarks provide guidelines for hardening and securing Ubuntu operating systems.
By implementing these guidelines, organizations can reduce the risk of cyber attacks and
improve their security posture. This is important for protecting sensitive data and
complying with regulations.

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
$mkdir cis-hardening && cd cis-hardening
$git clone https://github.com/MVladislav/ansible-cis-ubuntu-2204.git
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
          pl_a_host_default_ntp: 192.168.1.1
          pl_a_host_fallback_ntp: 192.168.1.1
          pl_a_cis_setup: true
          pl_a_cis_setup_aide: true
          pl_a_cis_ipv6_required: true
        host2:
          ansible_user: groot
          ansible_host: 192.168.1.11
          ansible_ssh_private_key_file: ~/.ssh/id_ed25519
          pl_a_host_default_ntp: time.cloudflare.com
          pl_a_host_fallback_ntp: time.cloudflare.com
          pl_a_cis_setup: true
          pl_a_cis_setup_aide: false
          pl_a_cis_ipv6_required: false
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
- name: CIS | install on clients
  become: true
  remote_user: "{{ ansible_user }}"
  hosts:
    - clients
  roles:
    - role: ansible-cis-ubuntu-2204
      cis_ubuntu2204_section1: true
      cis_ubuntu2204_section2: true
      cis_ubuntu2204_section3: true
      cis_ubuntu2204_section4: true
      cis_ubuntu2204_section5: true
      cis_ubuntu2204_section6: true
      cis_ubuntu2204_section7: true
      # -------------------------
      cis_ubuntu2204_rule_5_1_24: true
      cis_ubuntu2204_rule_5_1_24_ssh_user: "{{ ansible_user }}"
      cis_ubuntu2204_rule_5_1_24_ssh_pub_key: "<ADD_PUB_KEY>"
      # -------------------------
      cis_ubuntu2204_rule_1_3_1_3: true # AppArmor complain mode
      cis_ubuntu2204_rule_1_3_1_4: false # AppArmor enforce mode
      # -------------------------
      cis_ubuntu2204_rule_1_4_1: false # bootloader password (disabled)
      cis_ubuntu2204_set_boot_pass: false # bootloader password (disabled)
      cis_ubuntu2204_disable_boot_pass: true # bootloader password (disabled)
      # -------------------------
      cis_ubuntu2204_rule_3_1_3: false # bluetooth service
      cis_ubuntu2204_rule_3_1_3_remove: false # bluetooth service
      # -------------------------
      cis_ubuntu2204_allow_gdm_gui: true
      cis_ubuntu2204_allow_autofs: true # Disable auto mount, set to true to allow it and not disable
      cis_ubuntu2204_rule_1_1_1_8: false # Disable USB Storage, set to false to not disable
      cis_ubuntu2204_time_synchronization_service: chrony # chrony | systemd-timesyncd
      cis_ubuntu2204_time_synchronization_time_server:
        - uri: time.cloudflare.com
          config: iburst
        - uri: ntp.ubuntu.com
          config: iburst
      cis_ubuntu2204_allow_cups: true
      # -------------------------
      cis_ubuntu2204_install_aide: "{{ cis_setup_aide | default(false) | bool }}"
      cis_ubuntu2204_config_aide: "{{ cis_setup_aide | default(false) | bool }}"
      cis_ubuntu2204_aide_cron:
        cron_user: root
        cron_file: aide
        aide_job: "/usr/bin/aide.wrapper --config /etc/aide/aide.conf --check"
        aide_minute: 0
        aide_hour: 5
        aide_day: "*"
        aide_month: "*"
        aide_weekday: "*"
      # -------------------------
      cis_ubuntu2204_journald_system_max_use: 4G
      cis_ubuntu2204_journald_system_keep_free: 8G
      cis_ubuntu2204_journald_runtime_max_use: 256M
      cis_ubuntu2204_journald_runtime_keep_free: 512M
      cis_ubuntu2204_journald_max_file_sec: 1month
      # -------------------------
      cis_ubuntu2204_required_ipv6: "{{ cis_ipv6_required | default(false) | bool }}"
      cis_ubuntu2204_firewall: ufw
      # -------------------------
      cis_ubuntu2204_cron_allow_users:
        - root
      cis_ubuntu2204_at_allow_users:
        - root
      # -------------------------
      cis_ubuntu2204_faillock_deny: 5
      cis_ubuntu2204_faillock_unlock_time: 900
      cis_ubuntu2204_faillock_minlen: 8
      cis_ubuntu2204_password_complexity:
        - key: "minclass"
          value: "3"
        - key: "dcredit"
          value: "-1"
        - key: "ucredit"
          value: "-1"
        - key: "ocredit"
          value: "-1"
        - key: "lcredit"
          value: "-1"
      # -------------------------
```

{% endraw %}

### // run the role

run the role to setup your defined targets in the inventory with the CIS hardenings:

> options:
>
> - `-k`: prompt for the SSH password required to connect to the target system
> - `--ask-become-pass`: prompt for the password required to elevate privileges on the target system

```sh
$ansible-playbook -i inventory.yml playbook.yml --ask-become-pass -k
```

---

## \\\\ Resources & More information's

- <https://downloads.cisecurity.org/>{:target="\_blank"}
- <https://github.com/MVladislav/ansible-cis-ubuntu-2204>{:target="\_blank"}
