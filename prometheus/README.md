# Prometheus

prometheus is an open source, metric based monitoring system.

**installation**

```bash
wget prometheus
tar -zxvf prometheus.tar.gz

mkdir /var/lib/prometheus
useradd -r -s /sbin/nologin -d /var/lib/prometheus
chown -R prometheus: /var/lib/prometheus

mkdir /etc/prometheus
cp consule* prometheus.yml /etc/prometheus
chown -R prometheus: /etc/prometheus

cp prometheus promtool /usr/local/bin

```

`/usr/lib/systemd/system/prometheus.service`

```tex
[Unit]
Description=Prometheus server
Wants=network-online.target
After=network-online.target
[Service]
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus \
--web.console.templates=/etc/prometheus/consoles
--web.console.libraries=/etc/prometheus/console_libraries
ExecReload=/usr/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=60s
[Install]
WantedBy=multi-user.target
```

```bash
ss -ntlp | grep prometheus
```

```bash
systemctl daemon-reload
systemctl start prometheus.service
systemctl enable prometheus.service
systemctl status prometheus.service

firewall-cmd --add-port=9090/tco --permanent
firewall-cmd --reload
firewall-cmd --list-all
```

```bash
journalctl -u prometheus -xe
```

# Node-Exporter

```bash
wget node-exporter
tar -zxvf node_exporter.tar.gz
useradd -r -s /sbin/nologin exporter
cp node_exporter /usr/local/bin
chwon -R exporter: /usr/local/bin/node_exporter
```

`/usr/lib/systemd/system/node_exporter.service`

```bash
[Unit]
Description=Node-Exporter
After=network.target
[Service]
User=exporter
Group=exporter
ExecStart=/usr/local/bin/node_exporter \
--collector.textfile.directory="/tmp/users"
[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl start node_exporter.service
systemctl enable node_exporter.service
systemctl status node_exporter
```

```bash
firewall-cmd --add-port=9100/tcp --permanent
firewall-cmd --reload
firewall-cmd --list-all
```

```bash
curl http://localhost:9100/metrics
```

`/etc/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 60s
scrape_configs:
  - job_name: node-exporter
    static_configs:
      - targets: ["192.168.1.2:9100"]
```

```bash
promtool check config /etc/prometheus/prometheus.yml
systemctl reload prometheus.service
```

```tex
metric_name{label1=value1, label2=value2,...} numerical_value
metric_name ~> __name__
```

metric_types:

- counter
- gauge
- summery
- histogram

**Prometheus options**

```bash
--web.enable-lifecycle
curl -X POST http://localhost:9090/-/reload
curl -X POST http://localhost:9090/-/quit

--web.page-title="Prometheus Server"
--web.cors.origin=".*"

--storage.tsdb.path="/data"
--storage.tsdb.retention.time=15d
--storage.tsdb.retention.size="512MB"

--rule.alert.for-outage-tolerance=1h # Max time to tolerate prometheus outage for restoring "for" state of alert
--rule.alert.for-grace-period=10m # Minimum duration between alert and restore "for" state
--rule.alert.resend-delay=1m # Minimum amount of time to wait before resending an alert to Alertmanager

--alertmanager.notification-queue-capacity=10000

--query.lookback-delta=5m
--query.timeout=2m
--query.max-concurrency=20 # Maximum number of queries executed.
```

**configuration file**

- global
- scrape_configs
- alerting
- rule_files
- remote_read
- remote_write

```yaml
global:
  scrape_inerval: 10s # scrape_interval < lookback_delta / 2
  evaluation_interval: 10s # equal with scrape_interval
  external_labels: # using for federation and tanos	
    dc: dc1
    prom: prom1
scrape_configs:
  - job_name: node-exporter
    scrape_interval: 60s
    scrape_timeout: 5s
    sample_limit: 1000
    static_configs:
      - targets: ["192.168.1.100:9100"]
    relabel_configs:
    metric_relabel_configs:
      - source_label: [__name__]
        regex: go.+
        action: drop
```

**Custom metric**

```bash
corntab -e
* * * * * echo active-users $(who | wc -l) > users.prom # per minitue
* * * * * for i in {1..6}; do echo active-users $(who | wc -l) > users.prom; sleep 10; done # per 10 second
node_exporter --collector.textfile.directory="/tmp/users"
```

# Blackbox-Exporter

```bash
wget blackbox_exporter
tar -xvf blackbox_exporter
cp blackbox_exporter /usr/local/bin
cp blackbox.yml /etc/blackbox

useradd -r -s /sbin/nologin exporter
chwon -R blackbox: /usr/local/bin/blackbox_exporter
chwon -R blackbox: /etc/blackbox/
```

`/usr/lib/systemd/system/blackbox_exporter.service`

```bash
[Unit]
Description=Blackbox-Exporter
After=network.target

[Service]
User=blackbox
Group=blackbox
Type=simple
ExecStart=usr/local/bin/blackbox_exporter \
--config.file /etc/blackbox/blackbox.yml

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl start blackbox_exporter.service
systemctl enable blackbox_exporter.service
systemctl status blackbox_exporter
```

```bash
firewall-cmd --add-port=9115/tcp --permanent
firewall-cmd --reload
firewall-cmd --list-all
```

```bash
curl "<blackbox-ip>/9115/probe?module=icmp&target=192.168.1.101"
```

```yaml
scrape_configs:
  - job_name: icmp-blackbox
    static_configs:
      - targets: ["192.168.1.100:9115"]
	metric_path: /probe
	params:
	  target: ["google.com"]
	  module: ["icmp"]
```

`/etc/blackbox/blackbox.yml`

```yaml
test_icmp:
  prober: icmp
  icmp:
    preffered_ip_protocol: ip4
```

```yaml
# job=job_name, instance="", __address__=google.com, __param_module=test_icmp
# job=job_name, instance=google.com, __address__=<blackbox-ip:9115>, __param_module=test_icmp, __param_target=google.com
scrape_configs:
  - job_name: icmp_blackbox
  	metric_path: /probe
    static_configs:
      - targets: ["google.com", "yahoo.com"]
    params:
      module: ["test_icmp"]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacment: <blackbox-ip>:9115
```

**DNS blackbox**

```yaml
dns_check:
  prober: dns
  preferred_ip_protocol: ip4
  query_name: prometheus.io
  query_type: A
  validate_answer_rrs:
    fail_if_not_matches_regexp:
      - ".+"
```

```bash
curl "<blackbox-ip>:9115/probe?module=dns_check&target=8.8.8.8" # resolve prometheus.io from 8.8.8.8
```

# PromQL

```bash
histogram_quantile(0.9,rate(prometheus_http_request_duration_seconds_bucket{handler="/metric"}[1m])) # عددی که ۹۰ درصد جامعه از آن کمتر است را میدهد

node_cpu_seconds_total{mode!=idel}[1m] # range vector
rate(node_cpu_seconds_total{mode!=idel}[1m]) # using range vector with rate
process_resident_memoty_bytes{job="node"} offset 1h

count without(cpu)(count without(mode) (node_cpu_seconds_total)) # count of cpu core
```

**Selector**

```bash
process_resident_memory_bytes{job=node} # job=node ~> matcher ~> = != !~ =~
sum without(cpu)(sum without(mode) (rate(node_cpu_seconds_total{mode!=idel}[1m]))) 
{foo=""} {foo=".*"} {foo!=""} # return error
{foo=".+"} {name="x", foo=""} # ok
```

**Grouping**

```bash
sum without(device,fstype,mountPoint) (node_file_system_free_bytes{device!="tempfs"})
sum by(instance) (node_file_system_free_bytes{device!="tempfs"})
sum by(job, instance, device) (node_filesystem_size_bytes)
sort_desc(count by (__name__) ({__name__=~".+"}))
```

**aggregation operator**

- sum
- count
- max
- min
- stddev # 68% and 95%
- stdvar # واریانس
- topk(n, query)
- bottomk(n, query)
- quantile(0.9, rate(node_cpu_seconds_total{mode!=idle}[1m])) # gauge
- histogram_quantile # histogram
- quantile_over_time
- count_value without(instance) ("version", software_version) # group_by value
- avg without(cpu) (rate(node_cpu_seconds_total[5m])) # sum /count

**Binary operator**

- arithmetic operator ===> * / - + % ^
- comparison operator ===> == != > < >= <=
- logical operator ===> and or unless

**bool modifier**

```bash
process_open_fds > bool 10
avg(process_open_fds > bool 10) * 100
avg (up) * 100
```

**vector matching**

- one to one
- one to many
- many to many

```bash
node_network_receive_bytes_total < node_network_transmit_bytes_total # one to one

#ignoring
sum without(cpu) (rate(node_cpu_seconds_total{mode=idel}[5m])) / ignoring (mode) sum without(mode,cpu) (rate(node_cpu_seconds_total[5m]))
{instance="192.168.1.100",job="node", mode="idle"} / ignoring (mode) {instance="192.168.1.100", job="node"}

# on
sum without(cpu) (rate(node_cpu_seconds_total{mode=idel}[5m])) / on (instance) sum without(mode,cpu) (rate(node_cpu_seconds_total[5m]))

# or یا سمت چپ یا سمت راست
node_network_transmit_bytes_total < node_network_receive_bytes_total or ignoring(device) sum without(mode,cpu) (rate(node_cpu_seconds_total{mode!=idel}[5m]))
# and
اگر شرط سمت راست برقرار بود سمت چپ را نشان بده
# unless
اگر شرط سمت راست برقرار بود سمت چپ را نشان نده
year(node_boot_time_second) == scalar(year())
year(node_boot_time_second) == on group_left year()
month(node_boot_time_second) == scalar(month()) and year(node_boot_time_second) == scalar(year())
```

**Functions**

- abs
- ln, log2, log10
- exp(vector(1))
- sqrt(vector(9))
- ceil(vector(0.1)) ~> 1
- floor(vector(0.9)) ~> 0
- round(vector(5.5)) ~> 6 round(vector(5.49)) ~> 5
- clamp_max ~> grather than n return n
- clamp_min(process_open_fds, 10) ~> less than 10 return 10
- hour()
- time() ~> return scalar -- (time() - node_boot_time_seconds) / 3600
- minute, hour, day_of_week, day_of_month, day_in_month, month, year
- timestamp
- label_replace(node_disk_read_bytes_total, "dev", "${1}", "device", "(.*)") ~> create dev label
- label_join(node_cpu_seconds_total, "my_new_label","seperator", "mode","cpu") ~> join labels and create new label
- absent(up) ~> check up metric exists
- sort, sort_desc
- hsitogram_quantile
- rate
- increase
- irate ~> last two sample
- resets
- changes(process_start_time_seconds[1h]) ~> gauge
- deriv ~> like rate
- predict_linear(node_filesystem_free_bytes{job="node"}[1h], 4 * 3600)
- delta ~> x - x offset 1h
- idelta ~>  last two sample
- holt_winters(process_resident_memory_bytes, smoothing factor 0.1,trend factor 0.5)
- avg_over_time
- quantile_over_time ~> doesn't work with rate

# Service Discovery

**File Service Discovery**

```json
# /etc/promethesu/servers.json
[
    {
        "targets": ["192.168.1.100:9100", "192.168.1.101:9100"],
        "labels": {
            "team": "infra",
            "job": "node" # required
        }
    },
    {
        "targets": ["192.168.1.102:9100"],
        "labels": {
            "team": "monitoring",
            "job": "node"
        }
    }
]
```

```yaml
# /etc/prometheus/prometheus.yml
scrape_configs:
  - job_name: file
    metric_path: /admin/metrics
    scheme: https
    tls_config:
      inscure_skip_verify: true
    file_sd_configs:
      - files:
          - "*.josn"
    relabel_configs: # before scrape
      - source_labels: [team]
        regex: infra|monitoring
        action: keep
      - source_labels: [team,city]
        regex: infra;tehran
        action: drop
      - source_labels: [team]
        regex: monitoring
        replacement: monitor
        target_label: team
        action: replace
      - source_labels: [team]
        regex: (.*)ing
        replacement: ${1}
        target_label: team
        action: replace
      - source_labels: [] # delete team label and don't store to tsdb
        target_label: team
      - regex: __meta_ec2_public_tag_monitor_(.*)
        replacement: ${1}
        action: lablemap
    metric_relabel_configs: # after scrape
      - source_labels: [__name__]
        regex: http_request_size_bytes
        action: drop
      - regex: 'node_.*'
        action: labeldrop #labelkeep
```

# Docker

**cadvisor**

```bash
docker run --volume=/:/rootfs:ro --volume=/var/run:var/run:rw --volume=/sys:/sys:ro --volume=/var/lib/docker/:/var/lib/docker:ro --volume=/dev/disk:/dev/disk:ro --publish=8080:8080 --detach=true --name=cadvisor gcr.io/cadvisor/cadvisor
```

```yaml
scrape_configs:
  - job: cadvisor
    static_configs:
      - targets: [192.168.1.100:8080]
    metric_relabel_configs:
      - source_labels: [__name__,image]
        regex: container.*;()
        action: drop
      - regex: id
        action: labeldrop
        
```

**docker service discovery**

```bash
# /usr/lib/systemd/system
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -containerd=/run/containerdcontainerd.sock
# OR
# /etc/docker/daemon.json
{
	hosts: ["0.0.0.0:2375"]
}
```

```bash
systemctl daemon-reload
systemctl restart docker
ss -pentaul
```

```yaml
static_configs:
  - job: docker_service_discovery
    docker_sd_configs:
      - host: tcp://192.168.1.100:2375
    relabel_configs:
      - source_labels: [__meta_docker_port_private]
        regex: ()
        action: drop
      - source_labels: [__addres__]
        regex: .+:([0-9].+)
        action: replace
        replacement: 192.168.1.100:${1}
        target_label: __adddress__
      - regex: __meta_docker_container_(name)
        replacement: ${1}
        action: labelmap
      - source_labels: [name]
        regex: /(.+)
        replacement: ${1}
        target_label: name
      - regex: container_label_maintainer|id
        action: labeldrop
```

# Kubernetes

```yaml
scrape_configs:
  - job_name: kubernetes-node
    scheme: https
    kubernetes_sd_configs:
      - role: node
    tls_config:
      insecure_skip_verify: yes
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    authorization:
      credentials_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  - job_name: kubernetes-cadvisor
    scheme: https
    kubernetes_sd_configs:
      - role: node
    tls_config:
      insecure_skip_verify: yes
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    authorization:
      credentials_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    
```

```bash
# find ca path
kubectl exec -it <prometheus-pod> sh
find / -name ca.crt 2>/dev/null
```

# Alerting

`/etc/prometheus/rules.yml`

```yaml
groups:
  - name: node_exporter_rules
    rules:
      - record: cpu:node_cpu_seconds_total:utilization
        expr: sum without(mode)(rate(node_cpu_seconds_total{mode!=idel}[1m])) / sum without(mode)(rate(node_cpu_seconds_total[1m])) * 100
      - record: job:up:avg
        expr: avg(up{job=~.*}) * 100
      - record: instance:up
        expr: up
        labels:
          severity: high
      - alert: some instances are down
        expr: job:up:avg > 70 < 100 and on() hour() > 4 < 5
        labels:
          severity: avarage
          vendor: tata
      - alert: not connected instance
        expr: instance:up == 0
        for: 1m
        labels:
          severity: critical
          team: monitoring
        annotaions:
          summery: Target are not responding
          Description: Severity {{ $labels.severity }}
```

`/etc/prometheus/prometheus.yml`

```yaml
global:
  external_labels:
    - dc: tehran
rule_files:
  - rules.yml
```

# Alertmanager

```bash
cp alertmanager /usr/local/bin
cp amtool /usr/local/bin
cp alertmanager.yml /etc/alertmanager/
useradd -rs /sbin/nologin -M alertmanager
chown -R alertmanger: /etc/alertmanger
mkdir /var/lib/alertmanager/data
chown -R alertmanger: /var/lib/alertmanager
```

```bash
[Unit]
Description=Alert Manage
After=prometheus.service network.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/bin/alertmanager \
--config-file=/etc/alertmanager/alertmanager.yml \
--storage.path=/var/lib/alertmanager/data

[Install]
WantedBy=multi-user.target
```

```yaml
global:
  smtp_smarthost: 192.168.1.101:25
  smtp_from: tata@prometheus.com
  smtp_require_tls: false
  
route:
  receiver: default
  routes:
    - matchers:
        - severity: high
      receiver: hamid
    - matchers:
        - team =~ "tehran|tabriz"
      receiver: manager
receivers:
  - name: default
    email_configs:
      - to: noc@prometheus.tata
        require_tls: flase
  - nema: hamid
    email_configs:
      - to: darani.h@tiddev.com
    telegram_configs:
      - bot_token: ...
        chat_id: ...
```

`/etc/prometheus/prometheus.yml`

```yml
alerting:
  alertmanager:
    - static_configs:
        - targets:
            - localhost:9093
```

# Thanos
