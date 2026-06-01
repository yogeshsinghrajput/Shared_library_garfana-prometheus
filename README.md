# Assignment 5 - Monitoring Role (Prometheus & Grafana)

## Author

Rahul Parihar

## Description

This assignment creates a production-ready Ansible role called "monitoring_role" that automates the installation and configuration of Prometheus (metrics collection) and Grafana (visualization). The role is OS-independent, version-specific, and fully parameterized using Jinja2 templates for all configuration files.

## Use Case

In modern DevOps environments, monitoring is essential for observability. This role allows you to quickly deploy a complete monitoring stack across multiple servers with consistent configuration, making it ideal for infrastructure provisioning and containerized environments.

## Problem Statement

Create a monitoring role with:

- Version-specific installation capability
- OS-independent operation
- Configuration variablelization
- Jinja2 templating for all config files
- Separate handler files (not inline with tasks)
- Support for CentOS, Ubuntu, or both

## Implementation

### Step 1: Role Structure

Created a comprehensive production-ready role:

```
├── ansible.cfg
├── group_vars
│   ├── grafana.yml
│   └── prometheus.yml
├── inventory
│   └── hosts.ini
├── roles
│   ├── grafana
│   │   ├── defaults
│   │   │   └── main.yml
│   │   ├── files
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── meta
│   │   │   └── main.yml
│   │   ├── tasks
│   │   │   ├── configure.yml
│   │   │   ├── install_redhat.yml
│   │   │   ├── install_ubuntu.yml
│   │   │   └── main.yml
│   │   ├── templates
│   │   │   ├── datasource.yml.j2
│   │   │   └── grafana.ini.j2
│   │   └── vars
│   │       └── main.yml
│   └── prometheus
│       ├── defaults
│       │   └── main.yml
│       ├── files
│       ├── handlers
│       │   └── main.yml
│       ├── meta
│       │   └── main.yml
│       ├── tasks
│       │   ├── configure.yml
│       │   ├── install_redhat.yml
│       │   ├── install_ubuntu.yml
│       │   └── main.yml
│       ├── templates
│       │   └── prometheus.yml.j2
│       └── vars
│           └── main.yml
└── site.yml
```

Output:

<img width="517" height="1043" alt="image" src="https://github.com/user-attachments/assets/10de0a2e-b3d5-4920-a51d-664b51737c66" />



### Step 2: Default Variables

In `defaults/main.yml`:

```yaml
prometheus_port: 9090
prometheus_scrape_interval: "15s"

grafana_port: 3000
grafana_admin_user: "admin"
grafana_admin_password: "admin"

service_restart_policy: "always"
service_restart_delay: "5"
monitoring_directory_mode: "0755"
log_level: "info"
```

Output:


### Step 3: Role Variables

In `vars/main.yml`:

```yaml
tool_dir: "/opt/monitoring"

prometheus_version: "2.31.1"
prometheus_tar: "prometheus-{{ prometheus_version }}.linux-amd64.tar.gz"
prometheus_download_url: "https://github.com/prometheus/prometheus/releases/download/v{{ prometheus_version }}/{{ prometheus_tar }}"
prometheus_extracted_dir: "prometheus-{{ prometheus_version }}.linux-amd64"
prometheus_user: "prometheus"
prometheus_group: "prometheus"

grafana_version: "8.5.2"
grafana_tar: "grafana-{{ grafana_version }}.linux-amd64.tar.gz"
grafana_download_url: "https://dl.grafana.com/oss/release/{{ grafana_tar }}"
grafana_extracted_dir: "grafana-{{ grafana_version }}"
grafana_user: "grafana"
grafana_group: "grafana"

prometheus_grafana_dir: "{{ tool_dir }}"
```

Output:


### Step 4: Prerequisites Task

Handles OS-specific setup and user/group creation:

```yaml
# prerequisites.yml

# Install prerequisite packages on Ubuntu
- name: Install prerequisite packages on Ubuntu machines
  apt:
    name:
      - wget
      - tar
    state: present
    update_cache: yes
  when: ansible_facts['os_family'] == "Debian"

# Install prerequisite packages on CentOS
- name: Install prerequisite packages on CentOS machines
  yum:
    name:
      - wget
      - tar
    state: present
  when: ansible_facts['os_family'] == "RedHat"

# Create prometheus and grafana users and groups
- name: Create prometheus group
  group:
    name: "{{ prometheus_group }}"
    state: present

- name: Create prometheus user
  user:
    name: "{{ prometheus_user }}"
    group: "{{ prometheus_group }}"
    create_home: no
    shell: /sbin/nologin
    system: yes

- name: Create grafana group
  group:
    name: "{{ grafana_group }}"
    state: present

- name: Create grafana user
  user:
    name: "{{ grafana_user }}"
    group: "{{ grafana_group }}"
    create_home: no
    shell: /sbin/nologin
    system: yes

# Create common installation directory
- name: Create directory for Prometheus and Grafana
  file:
    path: "{{ tool_dir }}"
    state: directory
    mode: "0755"
```

Output:


### Step 5: Prometheus Installation

Downloads and configures Prometheus:

```yaml
# prometheus.yml
- name: Download Prometheus package
  get_url:
    url: "{{ prometheus_download_url }}"
    dest: "/tmp/{{ prometheus_tar }}"

- name: Extract Prometheus package
  unarchive:
    src: "/tmp/{{ prometheus_tar }}"
    dest: "{{ prometheus_grafana_dir }}"
    remote_src: yes
    creates: "{{ prometheus_grafana_dir }}/{{ prometheus_extracted_dir }}"

- name: Set ownership for Prometheus directory
  file:
    path: "{{ prometheus_grafana_dir }}/{{ prometheus_extracted_dir }}"
    owner: "{{ prometheus_user }}"
    group: "{{ prometheus_group }}"
    recurse: yes

- name: Configure Prometheus
  template:
    src: "prometheus.yml.j2"
    dest: "{{ prometheus_grafana_dir }}/{{ prometheus_extracted_dir }}/prometheus.yml"
    owner: "{{ prometheus_user }}"
    group: "{{ prometheus_group }}"
    mode: "0644"
  notify:
    - Restart Prometheus
```

Output:


### Step 6: Grafana Installation

Downloads and configures Grafana:

```yaml
# grafana.yml
- name: Download Grafana package
  get_url:
    url: "{{ grafana_download_url }}"
    dest: "/tmp/{{ grafana_tar }}"

- name: Extract Grafana package
  unarchive:
    src: "/tmp/{{ grafana_tar }}"
    dest: "{{ prometheus_grafana_dir }}"
    remote_src: yes
    creates: "{{ prometheus_grafana_dir }}/{{ grafana_extracted_dir }}"

- name: Set ownership for Grafana directory
  file:
    path: "{{ prometheus_grafana_dir }}/{{ grafana_extracted_dir }}"
    owner: "{{ grafana_user }}"
    group: "{{ grafana_group }}"
    recurse: yes

- name: Configure Grafana
  template:
    src: "grafana.ini.j2"
    dest: "{{ prometheus_grafana_dir }}/{{ grafana_extracted_dir }}/conf/grafana.ini"
    owner: "{{ grafana_user }}"
    group: "{{ grafana_group }}"
    mode: "0644"
  notify:
    - Restart Grafana
```

Output:


### Step 7: Prometheus Configuration Template

Jinja2 template for prometheus.yml:

```jinja2
# templates/prometheus.yml.j2
global:
  scrape_interval: {{ prometheus_scrape_interval }}
  evaluation_interval: {{ prometheus_scrape_interval }}

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['{{ prometheus_target }}']
```

Output:


### Step 8: Grafana Configuration Template

Jinja2 template for grafana.ini:

```jinja2
# templates/grafana.ini.j2
[server]
protocol = http
http_addr = 0.0.0.0
http_port = {{ grafana_port }}

[database]
type = sqlite3

[security]
admin_user = {{ grafana_admin_user }}
admin_password = {{ grafana_admin_password }}

[users]
allow_sign_up = false

[auth.anonymous]
enabled = false
```

Output:


### Step 9: Systemd Service Templates

```jinja2
# templates/prometheus.service.j2
[Unit]
Description=Prometheus Monitoring
After=network.target

[Service]
Type=simple
User={{ prometheus_user }}
Group={{ prometheus_group }}
ExecStart={{ prometheus_grafana_dir }}/{{ prometheus_extracted_dir }}/prometheus --config.file={{ prometheus_grafana_dir }}/{{ prometheus_extracted_dir }}/prometheus.yml
Restart=always

[Install]
WantedBy=multi-user.target
```

```jinja2
# templates/grafana.service.j2
[Unit]
Description=Grafana Monitoring Visualization
After=network.target

[Service]
Type=simple
User={{ grafana_user }}
Group={{ grafana_group }}
ExecStart={{ prometheus_grafana_dir }}/{{ grafana_extracted_dir }}/bin/grafana-server
Restart=always

[Install]
WantedBy=multi-user.target
```

Output:


### Step 10: Handlers

```yaml
# handlers/main.yml
- name: Reload Systemd
  systemd:
    daemon_reload: yes

- name: Restart Prometheus
  systemd:
    name: prometheus
    state: restarted

- name: Restart Grafana
  systemd:
    name: grafana
    state: restarted
```

Output:


### Step 11: Main Playbook

```yaml
# play.yml
- name: Setup Monitoring Stack
  hosts: all_servers
  become: yes
  roles:
    - monitoring_role
```

Output:


### Step 12: Run the Playbook

```bash
cd ansible-code
ansible-playbook -i inventory/hosts play.yml
```

Output:
<img width="1894" height="1043" alt="image" src="https://github.com/user-attachments/assets/eeaaee9b-2177-4d6e-a6de-270447a12828" />
<img width="1891" height="1046" alt="image" src="https://github.com/user-attachments/assets/64ffb982-f2b4-4cad-ad8a-fa5cc03912e8" />
<img width="1894" height="767" alt="image" src="https://github.com/user-attachments/assets/5f5d19ef-11b7-4905-bb16-a82aa92c3c9a" />



## Checkpoint / Verification

Verify the installation:

```bash
# Check Prometheus user and group
id prometheus

# Check Grafana user and group
id grafana

# Check monitoring directory
ls -la /opt/monitoring/

# Check Prometheus binary
/opt/monitoring/prometheus-2.31.1.linux-amd64/prometheus --version

# Check Grafana binary
/opt/monitoring/grafana-8.5.2/bin/grafana-server --version

# Check systemd services
systemctl status prometheus
systemctl status grafana

# Test Prometheus
curl http://<SERVER IP>:9090/-/healthy

# Test Grafana
curl http://<SERVER IP>:3000/api/health

# Check configuration
cat /opt/monitoring/prometheus-2.31.1.linux-amd64/prometheus.yml
```

Output:

<img width="1499" height="468" alt="image" src="https://github.com/user-attachments/assets/5c0e8201-14b3-44e7-91d1-0f1b7ad4a654" />

<img width="1901" height="501" alt="image" src="https://github.com/user-attachments/assets/5ac0477e-4dbd-4bde-8491-acc026b16add" />




### Prometheus Dashboard

<img width="1919" height="700" alt="image" src="https://github.com/user-attachments/assets/dc0522af-05af-41da-ab94-288e7f084a9d" />
<img width="1919" height="1048" alt="image" src="https://github.com/user-attachments/assets/e752bb42-1bb4-414e-a514-bf1a08296506" />


### Grafana Dashboard

<img width="1919" height="1064" alt="image" src="https://github.com/user-attachments/assets/03fb9afd-12ad-4ac5-8e9c-bafa222a33f7" />
<img width="1917" height="1064" alt="image" src="https://github.com/user-attachments/assets/71b96e69-df57-40cf-b6f1-20e8bd70d24c" />



<img width="1919" height="1059" alt="image" src="https://github.com/user-attachments/assets/c45be4c3-e72f-4785-aa2d-c01e0dd9622b" />
<img width="1919" height="1056" alt="image" src="https://github.com/user-attachments/assets/c8b74f4a-142e-434a-b4df-bccf1e752702" />


## What I Learned

This assignment pushed me to create a production-ready Ansible role. I focused on making it work across different Linux distributions by using conditionals and environment variables. Version-specific downloads are important because you might need a specific version for compatibility. Using Jinja2 templates for all configuration files means users can customize everything through variables without touching the actual configs. The role metadata and proper directory structure prepare it for sharing on Ansible Galaxy. I also learned about SPDX licenses and why they're important for open source projects. Handlers should always be separate from tasks - it makes the role cleaner and follows best practices.
