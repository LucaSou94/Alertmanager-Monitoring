# Alertmanager-Monitoring

# BlackBox-Monitoring

Descrizione

Questo progetto automatizza l'aggiunta di URL da un file JSON alla configurazione di Blackbox Exporter e Prometheus, consentendo il monitoraggio dinamico di endpoint HTTP.

## Prerequisiti

- Una macchina con Rocky Linux (o distribuzione equivalente)
- Permessi di root o sudo
- Connessione a Internet per scaricare i pacchetti necessari


## 1. Installazione di BlackBox_exporter

### 1) Installa BlackBox sulla VM Rocky Linux:
``` 
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.19.0/blackbox_exporter-0.19.0.linux-amd64.tar.gz
tar -xvzf blackbox_exporter-0.19.0.linux-amd64.tar.gz
mv blackbox_exporter-0.19.0.linux-amd64/blackbox_exporter /usr/local/bin/
mkdir /etc/blackbox_exporter/
mv blackbox_exporter-0.19.0.linux-amd64/blackbox.yml /etc/blackbox_exporter/
```
### 2) Creazione utente blackbox_exporter
```
useradd --no-create-home --shell /bin/false blackbox
```
### 3) Abiliatazione utente Blackbox
```
chown blackbox:blackbox /etc/blackbox_exporter/blackbox.yml
chown blackbox:blackbox /usr/local/bin/blackbox_exporter
```
### 4) Creazione del servizio blackbox_exporter
```
vim /etc/systemd/system/blackbox_exporter.service
```
```
[Unit]
Description=BlackBox Exporter
After=network.target

[Service]
User=blackbox
ExecStart=/usr/local/bin/blackbox_exporter --config.file=/etc/blackbox_exporter/blackbox.yml
Restart=on-failure

[Install]
WantedBy=multi-user.target

```

### 5) Avviare e Abilitare Blackbox_exporter:
```
sudo systemctl daemon-reload
sudo systemctl enable blackbox_exporter
sudo systemctl start blackbox_exporter
sudo systemctl status blackbox_exporter
```
## 2. Eseguire un Container con Podman sul ServerB

### 1) Avvio container nginx che ascolta sulla porta 8080

```
podman run -d --name nginx-container -p 8080:80 nginx
```

## 3. Installazione e Configurazione di Prometheus

### 1) Scarica e Installa Prometheus:
```   
wget https://github.com/prometheus/prometheus/releases/download/v2.31.1/prometheus-2.31.1.linux-amd64.tar.gz
tar -xvfz prometheus-2.31.1.linux-amd64.tar.gz
mv prometheus-2.31.1.linux-amd64/prometheus /usr/local/bin/
mv prometheus-2.31.1.linux-amd64/promtool   /usr/local/bin/
mkdir /etc/prometheus
mkdir /var/lib/prometheus
```
### 2) Creazione utente Prometheus

```
useradd --no-create-home --shell /bin/false prometheus
```
### 3) Abilitazione utente Prometheus
```
chown prometheus:prometheus /usr/local/bin/prometheus
chown prometheus:prometheus /usr/local/bin/promtool
chown prometheus:prometheus /etc/prometheus/prometheus.yml
chown prometheus:prometheus /var/lib/prometheus/
```
### 4) Controlliamo la configurazione di Prometheus prima di effettuare la modifica con il file json e script python:
```
cat /etc/prometheus/prometheus.yml
```
``` 
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 'localhost:9093'

rule_files:
  - "/etc/prometheus/alert.rules.yml"

scrape_configs:
- job_name: prometheus
  static_configs:
  - targets:
    - localhost:9090

- job_name: blackbox_http # Job per gli endpoint HTTP
  metrics_path: /probe
  params:
    module:
      - http_2xx
  static_configs:
    - targets:
        - ip-server:8080
        - ip-server
        - ip-server
        - ip-server
        - ip-server
        - ip-server
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: ip-server

- job_name: blackbox_tcp # Job per gli endpoint TCP
  metrics_path: /probe
  params:
    module:
      - tcp_connect
  static_configs:
    - targets:
        - ip-server
        - ip-server
        - ip-server
        - ip-server
        - ip-server
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: ip-server
```
### 5) Creazione del Servizio Systemd per Prometheus:
```
vim /etc/systemd/system/prometheus.service
```
``` 
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
--config.file=/etc/prometheus/prometheus.yml \
--storage.tsdb.path=/var/lib/prometheus/ \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```
### 6) Avviare e Abilitare Prometheus
 ```  
systemctl daemon-reload
systemctl start prometheus
systemctl enable prometheus
```

## 4. Installare e configurare Alertmanager  

### 1) Installazione Alertmanager

```
wget https://github.com/prometheus/alertmanager/releases/download/v0.24.0/alertmanager-0.24.0.linux-amd64.tar.gz
tar -xvf alertmanager-0.24.0.linux-amd64.tar.gz
mv alertmanager-0.24.0.linux-amd64/alertmanager /usr/local/bin/
mv alertmanager-0.24.0.linux-amd64/amtool /usr/local/bin/
mkdir /etc/alertmanager
mv alertmanager-0.24.0.linux-amd64/alertmanager.yml /etc/alertmanager/
mkdir /var/lib/alertmanager
```

### 2) Creazione utente Alertmanager

```
useradd --no-create-home --shell /bin/false alertmanager
```
### 3) Abilitazione utente Alertmanager

```
chown alertmanager:alertmanager /usr/local/bin/alertmanager
chown alertmanager:alertmanager /usr/local/bin/amtool
chown alertmanager:alertmanager /etc/prometheus/alertmanager.yml
chown alertmanager:alertmanager /var/lib/alertmanager/
```

### 4) Creazione servizio Alertmanager

```
vim /etc/systemd/system/alertmanager.service 
```

```
[Unit]
Description=Prometheus Alertmanager Service
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager --config.file=/etc/alertmanager/alertmanager.yml --storage.path=/var/lib/alertmanager/data
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### 5) Avviare e Abilitare Alertmanager
 ```  
systemctl daemon-reload
systemctl start alertmanager.service
systemctl enable alertmanager.service
```

## 5. Installare e configurare Postfix

 ```
sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix
 ```

## 6. Simulare il down su ServerB:


 ```
iptables -A INPUT -p tcp --dport 8080 -j DROP
 ```
