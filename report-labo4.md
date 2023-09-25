# Lab Report: Monitoring

## Laurens De Vos

## Learning goals

- Installing Prometheus and Node Exporter
- Configure basic system monitoring (CPU, memory, etc.)
- Monitor services
- Setting up a monitoring dashboard with Grafana

## Acceptance criteria

- Demonstrate the working Prometheus server:
  - Show targets in the webinterface
  - Run a query for a metric
- Show the Grafana dashboard you created
- Demonstrate any extensions to the assignment that you have implemented
- Your lab report is detailed and complete

## 2.1. Set up the monitoring server

Add a new server, called `srv004` to the `vmlab` environment and assign IP-address 172.16.128.4. Install Prometheus using the `cloudalchemy.prometheus` role.

`in vagrant-hosts.yml plaatsen we een nieuwe vm genaamd srv004`

```yaml
# vagrant_hosts.yml
#
# List of hosts to be created by Vagrant. For more information about the
# possible settings, see the documentation at
# <https://github.com/bertvv/ansible-skeleton>
---
- name: srv010
  ip: 172.16.128.10
  netmask: 255.255.0.0
  box: bento/almalinux-9

- name: srv001
  ip: 172.16.128.2
  netmask: 255.255.0.0
  box: bento/almalinux-9

- name: srv003
  ip: 172.16.128.3
  netmask: 255.255.0.0
  box: bento/almalinux-9

- name: srv004
  ip: 172.16.128.4
  netmask: 255.255.0.0
  box: bento/almalinux-9
# Example of a more elaborate host definition
# - name: srv002
#   box: bento/fedora-28
#   memory: 2048
#   cpus: 2
#   ip: 172.20.0.10
#   netmask: 255.255.0.0
#   mac: '13:37:de:ad:be:ef'
#   playbook: srv002.yml
#   forwarded_ports:
#     - host: 8080
#       guest: 80
#     - host: 8443
#       guest: 443
#   synced_folders:
#     - src: test
#       dest: /tmp/test
#     - src: www
#       dest: /var/www/html
#       options:
#         :create: true
#         :owner: root
#         :group: root
#         :mount_options: ['dmode=0755', 'fmode=0644']
```

`next we install the cloudalchemy.prometheus role by adding the role in the site.yml file and running the role-deps.sh script`

```yaml
# site.yml
---
- hosts: all
  roles:
    - bertvv.rh-base
- hosts: srv010
  roles:
    - role: geerlingguy.mysql
      become: yes
    - bertvv.wordpress
    - bertvv.httpd
    - geerlingguy.php
- hosts: srv001
  roles:
    - bertvv.bind
- hosts: srv003
  roles:
    - bertvv.dhcp
- hosts: srv004
  roles:
    - cloudalchemy.prometheus
```

![""](./img_lab4/script.png)

Set the variables `prometheus_targets` and `prometheus_scrape_configs`.

```yaml
prometheus_scrape_jobs:
  - job_name: "prometheus" # Custom scrape job, here using static_config
    metrics_path: "/metrics"
    static_configs:
      - targets:
          - "localhost:9090"
  - job_name: "nodes"
    metrics_path: /metrics
    static_configs:
      - targets:
          - srv010.infra.lan:9100
          - srv001.infra.lan:9100
          - srv003.infra.lan:9100
```

If your Prometheus server has to access all monitored VMs through their IP-address, the metrics they emit will become harder to interpret. We have set up a DNS server, so let's make use of it. In `site.yml`, add a `pre_task:` to the playbook section for `srv004` where you set your DNS server as the resolver to be used by your monitoring server. Remember that on EL, this is done by editing the `/etc/resolv.conf` file.

```yaml
---
- hosts: all
  roles:
    - bertvv.rh-base
- hosts: srv010
  roles:
    - role: geerlingguy.mysql
      become: yes
    - bertvv.wordpress
    - bertvv.httpd
    - geerlingguy.php
- hosts: srv001
  roles:
    - bertvv.bind
- hosts: srv003
  roles:
    - bertvv.dhcp
- hosts: srv004
  pre_tasks:
    - name: add DNS server
      lineinfile:
        path: /etc/resolv.conf
        regexp: "^(.*)nameserver(.*)$"
        line: "nameserver 172.16.128.2"
  roles:
    - cloudalchemy.prometheus
```

> If the Prometheus installation complains about `libselinux-python` rename `redhat.yml` to `redhat-7.yml` and `redhat-8.yml` to `redhat.yml` in the roles' `vars` folder.

![""](./img_lab4/vars.png)

## 2.2. Install Node Exporter

The Prometheus server polls all monitored systems for any metrics that you expose. Basic system metrics can be provided by Prometheus' Node Exporter.

Install the `cloudalchemy.node_exporter` role on all VMs in your setup. Update the `prometheus_targets` and/or `prometheus_scrape_configs` as needed.

```yaml
---
- hosts: all
  roles:
    - bertvv.rh-base
    - cloudalchemy.node-exporter
- hosts: srv010
  roles:
    - role: geerlingguy.mysql
      become: yes
    - bertvv.wordpress
    - bertvv.httpd
    - geerlingguy.php
- hosts: srv001
  roles:
    - bertvv.bind
- hosts: srv003
  roles:
    - bertvv.dhcp
- hosts: srv004
  pre_tasks:
    - name: add DNS server
      lineinfile:
        path: /etc/resolv.conf
        regexp: "^(.*)nameserver(.*)$"
        line: "nameserver 172.16.128.2"
  roles:
    - cloudalchemy.prometheus
```

nu moeten we opnieuw de rol installeren door het script in `/ansible/scrips` laten runnen met `bash ./scripts/role-deps.sh`

Restart Prometheus and ensure that it can access metrics for all the VMs.
