# k3s deployment on Raspberry Pis

General note: this project is meant as a starting point to customize your own setup rather than a complete solution.
I noted down the most important steps I did to get things set up and running and provided a stripped down/simplified version of.

# Hardware setup

For my k3s cluster I am using a Raspberry Pi 4 (4GB) as master and two Raspberry Pi 4 (8GB) as workers.
As OS of choice I went with Fedora CoreOS installed on SD cards, with a more durable 64GB USB stick for storage.
The log directory as well the persistent volumes are mounted on the USB stick to reduce writes to the SD card.

Besides that I am using the k3s cluster to monitor Proxmox and TrueNAS servers as well as pfSense firewalls.

# Preparation
Enter the appropriate IPs in `inventory/hosts.ini` and `inventory/group_vars/all.yml`.
Add the server certificate and private key in `role/traefik/files` as `k3s.crt` and `k3s.key` for https.
Also search for `example.local` and replace all entries with your preferred host names/domains.

## Install k3s

## For Fedora CoreOS prepare k3s SELinux

Login to master/workers.
```bash
sudo rpm-ostree install python3
sudo rpm-ostree install https://github.com/k3s-io/k3s-selinux/releases/download/v1.2.stable.2/k3s-selinux-1.2-2.el8.noarch.rpm
sudo systemctl reboot
```

## Install k3s

In [k3s-ansible](https://github.com/k3s-io/k3s-ansible) (follow instructions from that repo) run
```bash
ansible-playbook site.yml -i inventory/hosts.ini
```
and tag worker nodes
```bash
kubectl label node k3s-worker-0 kubernetes.io/role=worker
kubectl label node k3s-worker-0 node-type=worker
```

## Upgrade k3s
In k3s-ansible update version in inventory/group_vars/all.yml and then run again
```bash
ansible-playbook site.yml -i inventory/hosts.ini
```

# Prepare cluster basics
Setup local storage etc

```bash
ansible-playbook cluster_setup.yml
```

# Set up pv
```bash
ansible-playbook support/create-local-storage.yml --limit node[0] -e local_volume_size=10G -e local_volume_name="vol_10_0"
```

## Monitoring Stack

The monitoring playbook does set up the monitoring stack for k3s itself as well as components to monitor external services: Proxmox, TrueNAS and pfSense.

The basic monitoring stack consists of:
* Prometheus for metrics and scraping
* Loki for logs
* Node Exporter to collect metrics
* Promtail to collect logs
* Grafana for visualization of metrics and logs

I used nodeSelectors on prometheus and loki to ensure they are deployed on separate workers to distribute the load as they by far use the most resources in my cluster.

Additionally, the following components are set up to collect logs and metrics of external services
* rsyslog/promtail to receive external syslogs (rsyslog is added before promtail to ensure external logs can be read by promtail)
* Graphite Exporter to collect metrics of TrueNAS

To set everything up just run:
```bash
ansible-playbook monitoring.yml
```

## Set up TrueNAS
In TrueNAS under System / Reporting the remote graphite server can be set.
I set up port forwarding on the firewall from 2003 to 30109 as TrueNAS per default reports to port 2003 but k3s uses the range 30000-32000 for node ports. 
The setting can be overridden in /etc/local/collectd.conf but each reboot resets back to default so using port forwarding was the simplest solution.

Under System / Advanced the master node can be set as syslog server to report logs to.

## Set up Proxmox 
On Proxmox set up node_exporter and [pve_exporter](https://github.com/prometheus-pve/prometheus-pve-exporter), Prometheus is configured to scrape it.

For syslog forwarding just configure it on Proxmox (replace with your master node):
```bash
echo "*.* @k3s.example.local:30511" >> /etc/rsyslog.d/remote-logging.conf
```

## Set up pfSense
For metrics install the Telegraf plugin and set it up to provide a Prometheus endpoint following [README.md](roles/monitoring/files/README.md)

For syslog forwarding configure it under Status / System Logs / Settings. 
Set the Log Message Format to syslog and enable remote logging with the master node (port 30510) as remote log server.


## Application Stack
The application playbook sets up the homer dashboard as well as the Servarr stack. Transmission is expected as an external service.
Just replace the appropriate host names/domain and run:

```bash
ansible-playbook applications.yml
```