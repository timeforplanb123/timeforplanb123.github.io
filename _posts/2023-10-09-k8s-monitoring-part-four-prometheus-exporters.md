---
title: Monitoring with Prometheus, Loki, Grafana and Kubernetes. Part 4. Prometheus exporters
excerpt: "Exporting Prometheus metrics"
categories:
  - kubernetes
tags:
  - kubernetes
  - prometheus
  - snmp
toc: true
toc_label: "Getting Started"
---
## All pages

| Name                                        | Summary                                               |
| ------------------------------------------- | ----------------------------------------------------- |
| [Monitoring with Prometheus, Loki, Grafana and Kubernetes. Part 1. Kubernetes cluster][kubernetes-part1-post] | Here is about basic configuration of Kubernetes monitoring cluster |
| [Monitoring with Prometheus, Loki, Grafana and Kubernetes. Part 2. SNMP][kubernetes-part2-post] | Here is about SNMP O_O |
| [Monitoring with Prometheus, Loki, Grafana and Kubernetes. Part 3. Gitlab Agent][kubernetes-part3-post] | How to connect a Kubernetes cluster to Gitlab |
| [Monitoring with Prometheus, Loki, Grafana and Kubernetes. Part 4. Prometheus exporters][kubernetes-part4-post] | Exporting Prometheus metrics |

[kubernetes-part1-post]: {{ "" | relative_url }}{% post_url 2023-02-19-k8s-monitoring-part-one-k8s-cluster %}
[kubernetes-part2-post]: {{ "" | relative_url }}{% post_url 2023-02-25-k8s-monitoring-part-two-snmp %}
[kubernetes-part3-post]: {{ "" | relative_url }}{% post_url 2023-05-08-k8s-monitoring-part-three-gitlab-agent %}
[kubernetes-part4-post]: {{ "" | relative_url }}{% post_url 2023-10-09-k8s-monitoring-part-four-prometheus-exporters %}


## Prometheus exporters

Some systems support metrics, and Prometheus can request metrics directly. For the rest of the systems, there are Prometheus exporters. Prometheus requests metrics from the exporter at a certain interval, the exporter collects metrics from the target system and returns a response with Prometheus format metrics. A list of useful exporters can be found in the [documentation](https://prometheus.io/docs/instrumenting/exporters/){:target="_blank"}.

In this part, I will collect Prometheus metrics and import Grafana dashboards for network devices (for example, Mikrotik and Huawei) and Linux machines. 


## SNMP Exporter and SNMP Generator

### SNMP Exporter

To convert SNMP OIDs values to Prometheus metrics, the official [Prometheus SNMP exporter](https://github.com/prometheus/snmp_exporter){:target="_blank"} is used. The hierarchical structure of SNMP allows you to map SNMP index instances to Prometheus labels ([mapping example](https://github.com/prometheus/snmp_exporter#mapping){:target="_blank"})). This principle underlies the work of SNMP exporter.
SNMP exporter is supplied with a [default configuration](https://github.com/prometheus/snmp_exporter/tree/main#configuration){:target="_blank"} that describes both the `IF-MIB` module and some vendor MIBs. Therefore, to collect basic metrics, it is enough to configure SNMP on network devices, install SNMP Exporter in Kubernetes cluster and [configure Prometheus](https://github.com/prometheus/snmp_exporter/tree/main#prometheus-configuration){:target="_blank"}. That's where I'll start.

1. Configuring SNMP on network devices
I have 2 network devices that I will use as an example, Mikrotik Router and old Huawei 23 series switch with configured `10.1.1.254` and `10.1.1.1` ip addresses respectively. On each device, I configured `SNMPv2c` with a `public` community. Usually it is not difficult to find an example of an SNMP configuration for a specific network device, so I will not give an example for my network devices.
2. Preparing manifests for snmp exporter
```yaml
# k8s/snmp-exporter/snmp-exporter-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: snmp-exporter-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: snmp-exporter 
  template:
    metadata:
      labels:
        component: snmp-exporter 
    spec:
      volumes:
        - name: snmp-exporter-config-volume
          configMap:
            name: snmp-exporter-config
      containers:
        - name: snmp-exporter 
          image: prom/snmp-exporter
          ports:
            - containerPort: 9116
          volumeMounts:
            - name: snmp-exporter-config-volume
              mountPath: /etc/snmp_exporter/
```

```yaml
# k8s/snmp-exporter/snmp-exporter-cluster-ip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: snmp-exporter-cluster-ip-service
spec:
  type: ClusterIP
  selector:
    component: snmp-exporter 
  ports:
    - port: 9116
      targetPort: 9116
```

```yaml
# k8s/snmp-exporter/snmp-exporter-config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: snmp-exporter-config
data:
  snmp.yml: |-
    # here is the contents of snmp.yml file
    # https://github.com/prometheus/snmp_exporter/blob/main/snmp.yml
```
3. Configuring Prometheus
I'll add new jobs to the Prometheus ConfigMap:
```yaml
# k8s/prometheus/prometheus-config-map.yaml
apiVersion: v1
# type of object 
kind: ConfigMap
metadata:
  # ConfigMap name
  # used in the Deployment object template
  name: prometheus-config
data:
  # there are no scrape targets and alerting rules
  prometheus.yml: |-
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    rule_files:
      - /etc/prometheus/prometheus_rules.yml
    scrape_configs:
      - job_name: 'snmp-exporter'
        # https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config
        scrape_interval: 2m
        scrape_timeout: 1m 
        static_configs:
          - targets:
            # huawei switch
            - '10.1.1.1'
            # mikrotik router
            - '10.1.1.254'
        metrics_path: '/snmp'
        params:
          module: ['if_mib']
        relabel_configs:
          - source_labels: ['__address__']
            target_label: '__param_target'
          - source_labels: ['__param_target']
            target_label: 'instance'
          - target_label: '__address__'
            replacement: 'snmp-exporter-cluster-ip-service.default.svc.cluster.local:9116'
      - job_name: 'snmp-exporter-mikrotik'
        # https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config
        scrape_interval: 1m
        scrape_timeout: 30s
        static_configs:
          - targets:
            # mikrotik router
            - '10.1.1.254'
        metrics_path: '/snmp'
        params:
          module: ['mikrotik']
        relabel_configs:
          - source_labels: ['__address__']
            target_label: '__param_target'
          - source_labels: ['__param_target']
            target_label: 'instance'
          - target_label: '__address__'
            replacement: 'snmp-exporter-cluster-ip-service.default.svc.cluster.local:9116'
  prometheus_rules.yml: |-
```

Pipeline from [Part 3](https://timeforplanb123.github.io/kubernetes/k8s-monitoring-part-three-gitlab-agent/#gitlab-pipeline){:target="_blank"} is used to update the monitoring cluster, so you need to `push` new configuration files to the repository. To update Prometheus ConfigMap, you need to manually apply a `rollout restart` for `prometheus-deployment` object after the pipeline completes its work:
```bash
# prometheus rollout restart
~$ kubectl rollout restart deployment prometheus-deployment
```

Now Prometheus collects the SNMP metrics described in the default configuration. You can use them for Grafana dashboards, for example, [SNMP Stats](https://grafana.com/grafana/dashboards/11169-snmp-stats/){:target="_blank"}. But what if you need OIDs that are missing from the SNMP Exporter configuration? Prepare your own configuration using the [SNMP Exporter Config Generator](https://github.com/prometheus/snmp_exporter/tree/main/generator){:target="_blank"}.

### SNMP Exporter Config Generator

Since I only have 2 network devices (Mikrotik Router and Huawei switch), then I would like:
- remove unused modules from the default configuration, save only `if_mib` and `mikrotik`
- add a module for huawei switch to the configuration. By default, there is no such module

The [SNMP Exporter Config Generator](https://github.com/prometheus/snmp_exporter/tree/main/generator){:target="_blank"} allows you to generate a configuration file for the SNMP Exporter. Based on the instructions from the `generator.yml` file, it parses MIBs from the `mibs` directory and generates the `snmp.yml` configuration file.

Preparing of the `generator.yml` file:
1. Clone a [SNMP Exporter repository](https://github.com/prometheus/snmp_exporter){:target="_blank"} to your machine:
```bash
~$ git clone https://github.com/prometheus/snmp_exporter.git
```
2. Open the `generator.yml` file and delete all modules, except `if_mib` and `mikrotik`. I use vim for this.
3. Just for an example, download `HUAWEI-MIB.mib` and `HUAWEI-ENTITY-EXTENT-MIB.mib` to `~/snmp_exporter/generator/mibs` directory:
```bash
~/snmp_exporter/generator/mibs$ curl http://www.circitor.fr/Mibs/Mib/H/HUAWEI-MIB.mib > HUAWEI-MIB.mib
~/snmp_exporter/generator/mibs$ curl http://www.circitor.fr/Mibs/Mib/H/HUAWEI-ENTITY-EXTENT-MIB.mib > HUAWEI-ENTITY-EXTENT-MIB.mib
```
4. Prepare the host to work with SNMP and find OIDs or SNMP object names. For example, `sysDescr`, `sysName`, `sysLocation`, `1.3.6.1.4.1.2011.5.25.31.1.1.1.1.5`, `1.3.6.1.4.1.2011.5.25.31.1.1.1.1.7`, `1.3.6.1.4.1.2011.5.25.31.1.1.1.1.11`. The process is described in [Part 2](https://timeforplanb123.github.io/kubernetes/k8s-monitoring-part-two-snmp/#how-to-search-snmp-oids-for-monitoring){:target="_blank"}. 
5. Add the found OID or SNMP object names to the `generator.yml` file according to the [instructions](https://github.com/prometheus/snmp_exporter/tree/main/generator#file-format){:target="_blank"}. You will have the following file:
```yaml
auths:
  auth_name:
    version: 2
    community: public
modules:
# Default IF-MIB interfaces table with ifIndex.
  if_mib:
    walk: [sysUpTime, interfaces, ifXTable]
    lookups:
      - source_indexes: [ifIndex]
        lookup: ifPhysAddress
      - source_indexes: [ifIndex]
        lookup: ifAlias
      - source_indexes: [ifIndex]
        # Uis OID to avoid conflict with PaloAlto PAN-COMMON-MIB.
        lookup: 1.3.6.1.2.1.2.2.1.2 # ifDescr
      - source_indexes: [ifIndex]
        # Use OID to avoid conflict with Netscaler NS-ROOT-MIB.
        lookup: 1.3.6.1.2.1.31.1.1.1.1 # ifName
    overrides:
      ifPhysAddress:
        # type: DisplayString
        ignore: true # Lookup metric
      ifAlias:
        ignore: true # Lookup metric
      ifDescr:
        ignore: true # Lookup metric
      ifName:
        ignore: true # Lookup metric
      ifType:
        type: EnumAsInfo

# Mikrotik Router
# http://download2.mikrotik.com/Mikrotik.mib
  mikrotik:
    walk:
      - interfaces
      - ifMIB
      - laIndex
      - sysUpTime
      - sysDescr
      - host
      - mikrotik
    lookups:
      - source_indexes: [ifIndex]
        lookup: ifName
      - source_indexes: [mtxrInterfaceStatsIndex]
        lookup: ifName
      - source_indexes: [hrStorageIndex]
        lookup: hrStorageDescr
      - source_indexes: [laIndex]
        lookup: laNames
        drop_source_indexes: true
    overrides:
      ifName:
        ignore: true # Lookup metric
      ifType:
        type: EnumAsInfo

# Huawei switch
  huawei_sw:
    walk: 
      - sysDescr
      # oid 1.3.6.1.2.1.1.1
      - sysName
      # oid 1.3.6.1.2.1.1.5
      - sysLocation
      # oid 1.3.6.1.2.1.1.6
      - 1.3.6.1.4.1.2011.5.25.31.1.1.1.1.5
      # hwEntityCpuUsage
      - 1.3.6.1.4.1.2011.5.25.31.1.1.1.1.7
      # hwEntityMemUsage
      - 1.3.6.1.4.1.2011.5.25.31.1.1.1.1.11
      # hwEntityTemperature
```

Now, based on the prepared file, I generate `snmp.yml`. The easiest way is to [run the generator in docker](https://github.com/prometheus/snmp_exporter/tree/main/generator#docker-users){:target="_blank"}:
```bash
make docker-generate
```
The generated configuration file for SNMP Exporter can be found in the generator root directory - `~/snmp_exporter/generator/snmp.yml`. Now, in order to make changes to the cluster, I need to update the configuration of SNMP Exporter and Prometheus:
1. Update SNMP Exporter ConfigMap:
```yaml
# k8s/snmp-exporter/snmp-exporter-config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: snmp-exporter-config
data:
  snmp.yml: |-
    # here is the contents of the new snmp.yml file generated by the snmp generator
```
2. Update Prometheus ConfigMap:
```yaml
# k8s/prometheus/prometheus-config-map.yaml
apiVersion: v1
# type of object 
kind: ConfigMap
metadata:
  # ConfigMap name
  # used in the Deployment object template
  name: prometheus-config
data:
  # there are no scrape targets and alerting rules
  prometheus.yml: |-
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    rule_files:
      - /etc/prometheus/prometheus_rules.yml
    scrape_configs:
      - job_name: 'snmp-exporter-if-mib'
        # https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config
        scrape_interval: 2m
        scrape_timeout: 1m 
        static_configs:
          - targets:
            # huawei switch
            - '10.1.1.1'
            # mikrotik router
            - '10.1.1.254`
        metrics_path: '/snmp'
        params:
          module: ['if_mib']
        relabel_configs:
          - source_labels: ['__address__']
            target_label: '__param_target'
          - source_labels: ['__param_target']
            target_label: 'instance'
          - target_label: '__address__'
            replacement: 'snmp-exporter-cluster-ip-service.default.svc.cluster.local:9116'
      - job_name: 'snmp-exporter-mikrotik-router'
        # https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config
        scrape_interval: 1m
        scrape_timeout: 30s
        static_configs:
          - targets:
            # mikrotik router
            - '10.1.1.254'
        metrics_path: '/snmp'
        params:
          module: ['mikrotik']
        relabel_configs:
          - source_labels: ['__address__']
            target_label: '__param_target'
          - source_labels: ['__param_target']
            target_label: 'instance'
          - target_label: '__address__'
            replacement: 'snmp-exporter-cluster-ip-service.default.svc.cluster.local:9116'
      - job_name: 'snmp-exporter-huawei-switch'
        # https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config
        scrape_interval: 1m
        scrape_timeout: 30s
        static_configs:
          - targets:
            # huawei switch
            - '10.1.1.1'
        metrics_path: '/snmp'
        params:
          module: ['huawei_sw']
        relabel_configs:
          - source_labels: ['__address__']
            target_label: '__param_target'
          - source_labels: ['__param_target']
            target_label: 'instance'
          - target_label: '__address__'
            replacement: 'snmp-exporter-cluster-ip-service.default.svc.cluster.local:9116'
  prometheus_rules.yml: |-
```
3. `push` new configuration files to the repository. Wait until pipeline from [Part 3](https://timeforplanb123.github.io/kubernetes/k8s-monitoring-part-three-gitlab-agent/#gitlab-pipeline){:target="_blank"} updates ConfigMap objects in the cluster. After that, in order for SNMP Exporter and Prometheus to reload the configuration, I need to manually apply a `rollout restart` for the corresponding `deployment` objects:
```bash
# prometheus rollout restart
~$ kubectl rollout restart deployment prometheus-deployment
# snmp exporter rollout restart
~$ kubectl rollout restart deployment snmp-exporter-deployment
```


## MKTXP

To collect metrics from Mikrotik, there is an awesome [MKTXP project](https://github.com/akpw/mktxp){:target="_blank"}. This is Prometheus Exporter, which uses the RouterOS API to collect metrics. For visualization, MKTXP provides an awesome [Grafana dashboard](https://grafana.com/grafana/dashboards/13679-mikrotik-mktxp-exporter/){:target="_blank"}, and the collected metrics can be viewed without Prometheus and Grafana, using the [MKTXP CLI](https://github.com/akpw/mktxp#a-check-on-reality){:target="_blank"}. Let's integrate MKTXP into the k8s cluster:
```yaml
# k8s/mktxp-exporter/mktxp-exporter-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mktxp-exporter-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      component: mktxp-exporter 
  template:
    metadata:
      labels:
        component: mktxp-exporter 
    spec:
      volumes:
        - name: mktxp-exporter-config-volume
          configMap:
            name: mktxp-exporter-config
      containers:
        - name: mktxp-exporter 
          image: ghcr.io/akpw/mktxp:latest 
          ports:
            - containerPort: 49090
          volumeMounts:
            - name: mktxp-exporter-config-volume
              mountPath: /home/mktxp/mktxp/
```

```yaml
# k8s/mktxp-exporter/mktxp-exporter-cluster-ip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mktxp-exporter-cluster-ip-service
spec:
  type: ClusterIP
  selector:
    component: mktxp-exporter 
  ports:
    - port: 49090
      targetPort: 49090
```

```yaml
# k8s/mktxp-exporter/mktxp-exporter-config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mktxp-exporter-config
data:
  mktxp.conf: |-
    ## Copyright (c) 2020 Arseniy Kuznetsov
    ##
    ## This program is free software; you can redistribute it and/or
    ## modify it under the terms of the GNU General Public License
    ## as published by the Free Software Foundation; either version 2
    ## of the License, or (at your option) any later version.
    ##
    ## This program is distributed in the hope that it will be useful,
    ## but WITHOUT ANY WARRANTY; without even the implied warranty of
    ## MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    ## GNU General Public License for more details.


    [msk01rb]
        enabled = True                  # turns metrics collection for this RouterOS device on / off

        hostname = 10.1.1.254           # RouterOS IP address
        port = 8728                     # RouterOS IP Port

        username = mktxp_user           # RouterOS user, needs to have 'read' and 'api' permissions
        password = mktxp_user_password 

        use_ssl = False                 # enables connection via API-SSL servis
        no_ssl_certificate = False      # enables API_SSL connect without router SSL certificate
        ssl_certificate_verify = False  # turns SSL certificate verification on / off

        installed_packages = True       # Installed packages
        dhcp = True                     # DHCP general metrics
        dhcp_lease = True               # DHCP lease metrics

        connections = True              # IP connections metrics
        connection_stats = True         # Open IP connections metrics

        pool = True                     # Pool metrics
        interface = True                # Interfaces traffic metrics

        firewall = True                 # IPv4 Firewall rules traffic metrics
        ipv6_firewall = False           # IPv6 Firewall rules traffic metrics
        ipv6_neighbor = False           # Reachable IPv6 Neighbors

        poe = True 
        monitor = True                  # Interface monitor metrics
        netwatch = True                 # Netwatch metrics
        public_ip = True                # Public IP metrics
        route = True                    # Routes metrics
        wireless = False 
        wireless_clients = False
        capsman = False 
        capsman_clients = False 

        user = True                     # Active Users metrics
        queue = True 

        remote_dhcp_entry = None        # An MKTXP entry for remote DHCP info resolution (capsman/wireless)

        use_comments_over_names = True  # when available, forces using comments over the interfaces names

  _mktxp.conf: |-
    ## Copyright (c) 2020 Arseniy Kuznetsov
    ##
    ## This program is free software; you can redistribute it and/or
    ## modify it under the terms of the GNU General Public License
    ## as published by the Free Software Foundation; either version 2
    ## of the License, or (at your option) any later version.
    ##
    ## This program is distributed in the hope that it will be useful,
    ## but WITHOUT ANY WARRANTY; without even the implied warranty of
    ## MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    ## GNU General Public License for more details.


    [MKTXP]
        port = 49090
        socket_timeout = 10

        initial_delay_on_failure = 120
        max_delay_on_failure = 900
        delay_inc_div = 5

        bandwidth = True                    # Turns metrics bandwidth metrics collection on / off
        bandwidth_test_interval = 420       # Interval for colllecting bandwidth metrics
        minimal_collect_interval = 10       # Minimal metric collection interval

        verbose_mode = False                # Set it on for troubleshooting

        fetch_routers_in_parallel = False   # Set to True if you want to fetch multiple routers parallel
        max_worker_threads = 5              # Max number of worker threads that can fetch routers (parallel fetch only)
        max_scrape_duration = 10            # Max duration of individual routers' metrics collection (parallel fetch only)
        total_max_scrape_duration = 30      # Max overall duration of all metrics collection (parallel fetch only)
```
Configure new target in Prometheus ConfigMap:
```yaml
# k8s/prometheus/prometheus-config-map.yaml
apiVersion: v1
# type of object 
kind: ConfigMap
metadata:
  # ConfigMap name
  # used in the Deployment object template
  name: prometheus-config
data:
  # there are no scrape targets and alerting rules
  prometheus.yml: |-
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    rule_files:
      - /etc/prometheus/prometheus_rules.yml
    scrape_configs:
      - job_name: 'snmp-exporter-if-mib'
        # https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config
        scrape_interval: 2m
        scrape_timeout: 1m 
        static_configs:
          - targets:
            # huawei switch
            - '10.1.1.1'
            # mikrotik router
            - '10.1.1.254`
        metrics_path: '/snmp'
        params:
          module: ['if_mib']
        relabel_configs:
          - source_labels: ['__address__']
            target_label: '__param_target'
          - source_labels: ['__param_target']
            target_label: 'instance'
          - target_label: '__address__'
            replacement: 'snmp-exporter-cluster-ip-service.default.svc.cluster.local:9116'
      - job_name: 'snmp-exporter-mikrotik-router'
        # https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config
        scrape_interval: 1m
        scrape_timeout: 30s
        static_configs:
          - targets:
            # mikrotik router
            - '10.1.1.254'
        metrics_path: '/snmp'
        params:
          module: ['mikrotik']
        relabel_configs:
          - source_labels: ['__address__']
            target_label: '__param_target'
          - source_labels: ['__param_target']
            target_label: 'instance'
          - target_label: '__address__'
            replacement: 'snmp-exporter-cluster-ip-service.default.svc.cluster.local:9116'
      - job_name: 'snmp-exporter-huawei-switch'
        # https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config
        scrape_interval: 1m
        scrape_timeout: 30s
        static_configs:
          - targets:
            # huawei switch
            - '10.1.1.1'
        metrics_path: '/snmp'
        params:
          module: ['huawei_sw']
        relabel_configs:
          - source_labels: ['__address__']
            target_label: '__param_target'
          - source_labels: ['__param_target']
            target_label: 'instance'
          - target_label: '__address__'
            replacement: 'snmp-exporter-cluster-ip-service.default.svc.cluster.local:9116'
      - job_name: 'mktxp-exporter-mikrotik-router'
        scrape_interval: 3m
        scrape_timeout: 30s
        static_configs:
          - targets:
            - 'mktxp-exporter-cluster-ip-service.default.svc.cluster.local:49090'
  prometheus_rules.yml: |-
```
Now, open Mikrotik terminal and configure new user (see [docs](https://github.com/akpw/mktxp#mikrotik-device-config){:target="_blank"}):
```bash
/user group add name=mktxp_group policy=api,read
/user add name=mktxp_user group=mktxp_group password=mktxp_user_password
```
`push` new configuration files to the repository. Wait until pipeline from [Part 3](https://timeforplanb123.github.io/kubernetes/k8s-monitoring-part-three-gitlab-agent/#gitlab-pipeline){:target="_blank"} to configure the MKTXP exporter in the cluster. Then import new [MKTXP dashboard](https://grafana.com/grafana/dashboards/13679-mikrotik-mktxp-exporter/){:target="_blank"} to Grafana and check the result.


## Node Exporter

[Node Exporter](https://github.com/prometheus/node_exporter){:target="_blank"} collects Hardware and OS metrics. For example, I will install Node Exporter on an Ubuntu machine with the `192.168.0.10` ip address, located outside the k8s cluster:
1. Download latest Node Exporter release (check it out [here](https://github.com/prometheus/node_exporter/releases){:target="_blank"}):
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar xvfz node_exporter-1.6.1.linux-amd64.tar.gz
```
2. Create systemd `node_exporter.service` file:
```bash
vi /etc/systemd/system/node_exporter.service
```
3. Add the following contents to the file:
```bash
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=root
Group=root
Type=simple
ExecStart=/home/user/node_exporter-1.6.1.linux-amd64/node_exporter

[Install]
WantedBy=multi-user.target
```
4. Run the node_exporter and add it to the startup:
```bash
systemctl start node_exporter
systemctl enable node_exporter
```
5. Check the node_exporter status:
```bash
systemctl status node_exporter
```

To collect metrics from an installed Node Exporter using Prometheus, I need:
1. Update Prometheus ConfigMap:
```yaml
# k8s/prometheus/prometheus-config-map.yaml
apiVersion: v1
# type of object 
kind: ConfigMap
metadata:
  # ConfigMap name
  # used in the Deployment object template
  name: prometheus-config
data:
  # there are no scrape targets and alerting rules
  prometheus.yml: |-
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    rule_files:
      - /etc/prometheus/prometheus_rules.yml
    scrape_configs:
      - job_name: 'snmp-exporter-if-mib'
        # https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config
        scrape_interval: 2m
        scrape_timeout: 1m 
        static_configs:
          - targets:
            # huawei switch
            - '10.1.1.1'
            # mikrotik router
            - '10.1.1.254`
        metrics_path: '/snmp'
        params:
          module: ['if_mib']
        relabel_configs:
          - source_labels: ['__address__']
            target_label: '__param_target'
          - source_labels: ['__param_target']
            target_label: 'instance'
          - target_label: '__address__'
            replacement: 'snmp-exporter-cluster-ip-service.default.svc.cluster.local:9116'
      - job_name: 'snmp-exporter-mikrotik-router'
        # https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config
        scrape_interval: 1m
        scrape_timeout: 30s
        static_configs:
          - targets:
            # mikrotik router
            - '10.1.1.254'
        metrics_path: '/snmp'
        params:
          module: ['mikrotik']
        relabel_configs:
          - source_labels: ['__address__']
            target_label: '__param_target'
          - source_labels: ['__param_target']
            target_label: 'instance'
          - target_label: '__address__'
            replacement: 'snmp-exporter-cluster-ip-service.default.svc.cluster.local:9116'
      - job_name: 'snmp-exporter-huawei-switch'
        # https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config
        scrape_interval: 1m
        scrape_timeout: 30s
        static_configs:
          - targets:
            # huawei switch
            - '10.1.1.1'
        metrics_path: '/snmp'
        params:
          module: ['huawei_sw']
        relabel_configs:
          - source_labels: ['__address__']
            target_label: '__param_target'
          - source_labels: ['__param_target']
            target_label: 'instance'
          - target_label: '__address__'
            replacement: 'snmp-exporter-cluster-ip-service.default.svc.cluster.local:9116'
      - job_name: 'mktxp-exporter-mikrotik-router'
        scrape_interval: 3m
        scrape_timeout: 30s
        static_configs:
          - targets:
            - 'mktxp-exporter-cluster-ip-service.default.svc.cluster.local:49090'
      - job_name: 'node-exporter'
        static_configs:
          # Ubuntu machine with node_exporter
          - targets: ['192.168.0.10:9100']
  prometheus_rules.yml: |-
```
2. `push` to the k8s cluster repository:
```bash
git add .
git commit -m "Add 192.168.0.10 node_exporter target to Prometheus ConfigMap"
git push origin main
```
3. On the k8s node, apply the `rollout restart` command for the `prometheus-deployment` object after the [pipeline](https://timeforplanb123.github.io/kubernetes/k8s-monitoring-part-three-gitlab-agent/#gitlab-pipeline){:target="_blank"} updates the k8s cluster:
```bash
kubectl rollout restart deployment prometheus-deployment
```
Now, let's add the Grafana Node Exporter dashboard. For example, [11074](https://grafana.com/grafana/dashboards/11074-node-exporter-for-prometheus-dashboard-en-v20201010/){:target="_blank"}
