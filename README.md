# DevOps Assignment

This repository contains my implementation of the DevOps take-home assignment.

The environment used for this assignment is an Ubuntu 22.04.4 LTS virtual machine.

The repository is public so the company can review the implementation without requiring access to a private repository.

> **Note:** This is a practice/test environment. The configurations may contain test credentials, passwords, self-signed certificates, and local lab settings where applicable. These values are included intentionally for demonstration and testing purposes and are **not intended for production use**.

---

## Project Structure

```text
.
├── README.md
├── task1/
│   └── nginx-docker/
│       ├── docker-compose.yml
│       ├── conf/
│       │   └── nginx.conf
│       ├── html/
│       │   └── local/
│       │       └── index.html
│       ├── ssl/
│       │   ├── nginx-task.crt
│       │   └── nginx-task.key
│       └── .htpasswd
└── task2/
    ├── docker-elk/
    ├── kafka/
    │   └── docker-compose.yml
    ├── rsyslog-config/
    │   └── devops-scenario.conf
    └── vsftp-config/
        └── vsftpd.conf
```

---

# Task 1 - Nginx

The first task contains a Dockerized Nginx setup.

The Nginx configuration provides:

- Local HTTPS endpoint
- Basic Authentication for the local endpoint
- Network endpoint with redirect
- TLS 1.2 and TLS 1.3
- Security-related HTTP headers
- Disabled Nginx version disclosure
- Self-signed TLS certificate for the local test environment

## Dockerized Nginx

The Docker configuration is located at:

```text
task1/nginx-docker/
```

The Docker Compose configuration runs Nginx using the official Nginx image.

### Endpoints

Local endpoint:

```text
https://192.168.36.129:8090/local
```

Network endpoint:

```text
https://192.168.36.129:8008/net
```

The `/local` endpoint is protected with HTTP Basic Authentication.

The `/net` endpoint redirects the request to:

```text
https://www.google.com
```

Because the environment uses a self-signed certificate, the browser may display a certificate warning.

---

# Task 2 - Logging Platform

The second task implements a complete logging pipeline using:

- Log generator
- systemd
- rsyslog
- Kafka
- AKHQ
- Logstash
- Elasticsearch
- Kibana
- Elasticsearch ILM
- FTP

## Architecture

```text
                         ┌─────────────────────┐
                         │   Log Generator     │
                         │   systemd service   │
                         └──────────┬──────────┘
                                    │
                                    │ UDP 127.0.0.1:5514
                                    ▼
                              ┌───────────┐
                              │  rsyslog  │
                              └─────┬─────┘
                                    │
                         ┌──────────┴──────────┐
                         │                     │
                         ▼                     ▼
                ┌─────────────────┐     ┌─────────────┐
                │ Local Log File  │     │    Kafka    │
                │ devops.log      │     │ devops-logs │
                └─────────────────┘     └──────┬──────┘
                                               │
                                               ├──────────────┐
                                               │              │
                                               ▼              ▼
                                         ┌──────────┐   ┌───────────┐
                                         │   AKHQ   │   │ Logstash  │
                                         └──────────┘   └─────┬─────┘
                                                              │
                                                              ▼
                                                       ┌─────────────┐
                                                       │Elasticsearch│
                                                       └──────┬──────┘
                                                              │
                                                              ▼
                                                         ┌─────────┐
                                                         │ Kibana  │
                                                         └─────────┘

FTP Client
    │
    │ FTP
    ▼
┌──────────────────────────┐
│ /var/log/devops-scenario │
│                          │
│ devops.log               │
└──────────────────────────┘
```

---

## 2.1 Log Generator

The log generator creates logs in the required format and sends them to the local rsyslog UDP listener.

The script is located at:

```text
/opt/log-generator/generate-log.sh
```

The logs are sent to:

```text
127.0.0.1:5514
```

The generator is managed by systemd:

```text
/etc/systemd/system/log-generator.service
```

The service continuously generates a new log approximately every 5 seconds.

### Generated log format

Example:

```text
myfirstlog : 2026-09-15T17:22:00+03:30 hello devops : {"country":"iran","city":"tehran","population":123456,"men":61728,"women":1234,"hOffset":246912,"vOffset":100,"weather":"sun1.city = (sun1.city / 100) * 90;"}
```

---

## 2.2 rsyslog

rsyslog listens for the generated UDP logs on:

```text
127.0.0.1:5514
```

The configuration is located at:

```text
task2/rsyslog-config/devops-scenario.conf
```

The configuration sends each log to two destinations:

1. Local log file
2. Kafka topic

Local log file:

```text
/var/log/devops-scenario/devops.log
```

Kafka topic:

```text
devops-logs
```

The Kafka output is implemented using the rsyslog Kafka output module.

### Validation

Check the rsyslog configuration:

```bash
sudo rsyslogd -N1
```

Check the UDP listener:

```bash
sudo ss -lunp | grep 5514
```

Restart rsyslog:

```bash
sudo systemctl restart rsyslog
```

---

# 2.3 Kafka

Kafka is deployed using Docker Compose.

Configuration:

```text
task2/kafka/docker-compose.yml
```

The setup contains:

- Kafka broker
- AKHQ

Kafka uses KRaft mode and does not require ZooKeeper.

The Kafka topic used by the logging pipeline is:

```text
devops-logs
```

The topic is configured with a retention period of:

```text
3 hours
```

or:

```text
10800000 ms
```

### Kafka listeners

Kafka provides:

```text
Internal Docker listener:
kafka:29092

External listener:
192.168.36.129:9092
```

The internal listener is used by containers such as AKHQ.

The external listener allows services running directly on the Ubuntu VM, such as rsyslog, to connect to Kafka.

### Topic retention

The retention configuration is:

```text
retention.ms=10800000
```

---

# 2.4 AKHQ

AKHQ is used to monitor Kafka.

AKHQ is available at:

```text
http://192.168.36.129:8080
```

It can be used to inspect:

- Kafka brokers
- Topics
- Partitions
- Messages
- Consumer groups
- Topic configuration

The configured Kafka topic is:

```text
devops-logs
```

---

# 2.5 Logstash

Logstash consumes messages from Kafka and sends them to Elasticsearch.

The Logstash pipeline performs the following steps:

```text
Kafka
  ↓
Grok
  ↓
JSON parsing
  ↓
Date parsing
  ↓
Elasticsearch
```

The pipeline:

1. Reads messages from the `devops-logs` Kafka topic.
2. Uses a Kafka consumer group.
3. Parses the custom log prefix using Grok.
4. Extracts the JSON payload.
5. Parses the JSON into Elasticsearch fields.
6. Converts the log timestamp into Elasticsearch `@timestamp`.
7. Sends the resulting event to Elasticsearch.

The Kafka consumer group is:

```text
logstash-devops
```

---

# 2.6 Elasticsearch

Elasticsearch is part of the ELK stack.

The ELK stack is based on the official:

```text
deviantony/docker-elk
```

project.

The Elasticsearch container provides the storage and search layer for the logs.

The logs are written through the Elasticsearch alias:

```text
devops-logs
```

This allows Elasticsearch ILM to manage index rollover while Logstash continues writing to the same alias.

---

# 2.7 Kibana

Kibana provides the visualization layer for Elasticsearch.

It can be used to:

- Search logs
- Inspect log fields
- Create visualizations
- Create dashboards
- Analyze log volume and fields

Example fields available from the generated JSON include:

```text
country
city
population
men
women
hOffset
vOffset
weather
```

The `@timestamp` field is generated from the timestamp contained in the original log message.

---

# 2.8 Elasticsearch ILM

Elasticsearch Index Lifecycle Management is configured manually for the generated logs.

The purpose of the configuration is to demonstrate index rollover and lifecycle management.

The architecture is:

```text
Logstash
    │
    ▼
devops-logs alias
    │
    ▼
logstash-devops-000001
    │
    │ rollover
    ▼
logstash-devops-000002
    │
    │ rollover
    ▼
logstash-devops-000003
```

The ILM policy contains:

- Hot phase
- Rollover condition
- Delete phase

The rollover policy uses:

```text
max_primary_shard_size
max_age
```

The retention/deletion phase removes old indices after the configured lifecycle period.

### Important concept

Logstash does not need to know the current physical index name.

It writes to:

```text
devops-logs
```

Elasticsearch ILM manages the physical indices behind the alias.

For example:

```text
devops-logs
      │
      ▼
logstash-devops-000002
```

After rollover:

```text
devops-logs
      │
      ▼
logstash-devops-000003
```

The alias remains stable while the physical index changes.

### ILM validation

Check ILM status:

```bash
curl -s -u elastic:"$ELASTIC_PASSWORD" \
  "http://localhost:9200/_ilm/status?pretty"
```

Check the lifecycle state of an index:

```bash
curl -s -u elastic:"$ELASTIC_PASSWORD" \
  "http://localhost:9200/logstash-devops-000001/_ilm/explain?pretty"
```

Check the alias:

```bash
curl -s -u elastic:"$ELASTIC_PASSWORD" \
  "http://localhost:9200/_alias/devops-logs?pretty"
```

Check indices:

```bash
curl -s -u elastic:"$ELASTIC_PASSWORD" \
  "http://localhost:9200/_cat/indices?v"
```

Check cluster health:

```bash
curl -s -u elastic:"$ELASTIC_PASSWORD" \
  "http://localhost:9200/_cluster/health?pretty"
```

---

# 2.9 FTP

FTP is configured so that a dedicated user can access the directory containing the rsyslog-generated logs from outside the VM.

The FTP server is:

```text
vsftpd
```

The configuration file is:

```text
task2/vsftp-config/vsftpd.conf
```

The FTP user can access:

```text
/var/log/devops-scenario
```

The FTP configuration is read-only:

```text
write_enable=NO
```

Anonymous FTP access is disabled:

```text
anonymous_enable=NO
```

The FTP server uses passive mode with the configured port range:

```text
40000-40010
```

FTP control connection:

```text
TCP 21
```

### FTP access

The FTP user can connect from outside the VM using:

```text
ftp 192.168.36.129
```

After authentication, the user can access the generated log:

```text
devops.log
```

Example FTP commands:

```text
pwd
ls
get devops.log
bye
```

The FTP service is intended for this assignment and lab environment.

For production environments, encrypted alternatives such as SFTP or FTPS would normally be preferred.

---

# Configuration Notes

This repository is intended for a DevOps assignment and lab demonstration.

Therefore, some files contain:

- Test credentials
- Test passwords
- Local IP addresses
- Local usernames
- Self-signed certificates
- Development/test TLS configuration
- Local Docker configuration

These settings are intentionally included so the complete environment can be reviewed and reproduced.

They should **not be reused as production credentials or production security configuration**.

---

# Validation

## Nginx

Check the Nginx configuration:

```bash
sudo nginx -t
```

Check running containers:

```bash
docker ps
```

Test the local endpoint:

```bash
curl -k -I https://192.168.36.129:8090/local
```

Test the redirect endpoint:

```bash
curl -k -I https://192.168.36.129:8008/net
```

---

## Log Generator

Check service status:

```bash
sudo systemctl status log-generator.service
```

Check service logs:

```bash
sudo journalctl -u log-generator.service
```

---

## rsyslog

Validate configuration:

```bash
sudo rsyslogd -N1
```

Check UDP listener:

```bash
sudo ss -lunp | grep 5514
```

Check generated log file:

```bash
sudo tail -f /var/log/devops-scenario/devops.log
```

---

## Kafka

Check containers:

```bash
docker ps
```

List topics:

```bash
docker exec devops-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:29092 \
  --list
```

Describe the logging topic:

```bash
docker exec devops-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:29092 \
  --describe \
  --topic devops-logs
```

Check topic configuration:

```bash
docker exec devops-kafka \
  /opt/kafka/bin/kafka-configs.sh \
  --bootstrap-server localhost:29092 \
  --entity-type topics \
  --entity-name devops-logs \
  --describe
```

---

## Elasticsearch

Check cluster health:

```bash
curl -s -u elastic:"$ELASTIC_PASSWORD" \
  "http://localhost:9200/_cluster/health?pretty"
```

List indices:

```bash
curl -s -u elastic:"$ELASTIC_PASSWORD" \
  "http://localhost:9200/_cat/indices?v"
```

Check the logging alias:

```bash
curl -s -u elastic:"$ELASTIC_PASSWORD" \
  "http://localhost:9200/_alias/devops-logs?pretty"
```

---

## FTP

Check FTP service:

```bash
sudo systemctl status vsftpd
```

Check listening port:

```bash
sudo ss -lntp | grep :21
```

Check the log directory:

```bash
ls -lah /var/log/devops-scenario/
```

---

# Environment

The assignment was implemented and tested in:

```text
OS: Ubuntu 22.04.4 LTS
Virtualization: VMware
Architecture: x86_64
```

Example VM network address:

```text
192.168.36.129
```

Main components:

```text
Nginx
Docker
Docker Compose
rsyslog
Kafka
AKHQ
Logstash
Elasticsearch
Kibana
Elasticsearch ILM
vsftpd
systemd
```

---

# Summary

The final logging pipeline is:

```text
Log Generator
      │
      │ UDP :5514
      ▼
   rsyslog
    │    │
    │    └──────────────► Local Log File
    │
    ▼
   Kafka
    │
    ├──────────────► AKHQ
    │
    ▼
 Logstash
    │
    ▼
Elasticsearch
    │
    ▼
  Kibana
```

The generated log files are also made available through the configured FTP service.

The Elasticsearch indices are managed using manually configured ILM policies with rollover and retention.
