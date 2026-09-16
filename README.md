DevOps Take-Home Assignment

This repository contains my implementation of the DevOps take-home assignment.

The project was implemented and tested on an Ubuntu 22.04.4 LTS virtual machine.

Note: This repository is public so the assignment can be reviewed without requiring GitHub account access or repository permissions. The configurations contain practice/test credentials and local lab settings where applicable.

Project Structure
.
├── task1/
│   └── nginx-docker/
│
├── task2/
│   ├── docker-elk/
│   ├── kafka/
│   ├── rsyslog-config/
│   └── vsftp-config/
│
└── README.md
Task 1 - Nginx

The task1/nginx-docker directory contains the Dockerized Nginx configuration.

Implemented:

Nginx running with Docker Compose
HTTP/HTTPS server configuration
TLS configuration
TLS 1.2 and TLS 1.3
Basic Authentication
Security headers
server_tokens off
Local endpoint
External/network endpoint
HTTP redirects
Nginx configuration validation
Task 2 - Logging Platform

The task2 directory contains the logging pipeline and supporting services.

The overall architecture is:

                    ┌──────────────┐
                    │ Log Generator│
                    └──────┬───────┘
                           │ UDP
                           │ :5514
                           ▼
                    ┌──────────────┐
                    │    rsyslog   │
                    └──────┬───────┘
                           │
                    ┌──────┴───────┐
                    │              │
                    ▼              ▼
              Local Log File     Kafka
                                   │
                         ┌─────────┴─────────┐
                         │                   │
                         ▼                   ▼
                       AKHQ               Logstash
                                             │
                                             ▼
                                      Elasticsearch
                                             │
                                             ▼
                                          Kibana

                    FTP
                     │
                     ▼
             Generated Log Files
2.1 Kafka and AKHQ

Directory:

task2/kafka/

Kafka is used as the message/event streaming layer.

Implemented:

Apache Kafka running with Docker Compose
Single-node KRaft configuration
Internal and external Kafka listeners
devops-logs topic
Topic retention configured to 3 hours
AKHQ for Kafka monitoring and management

Kafka topic:

devops-logs

Retention:

3 hours
2.2 rsyslog

Directory:

task2/rsyslog-config/

rsyslog receives UDP logs on:

127.0.0.1:5514

The logs are then sent to:

A local log file
Kafka

Local log directory:

/var/log/devops-scenario/

Main log file:

/var/log/devops-scenario/devops.log

The rsyslog configuration uses:

imudp
omfile
omkafka
A dedicated ruleset for the assignment logs
2.3 Log Generator

The log generator produces logs in the required format and sends them to rsyslog over UDP.

Example:

myfirstlog : 2026-09-15T17:22:00+03:30 hello devops : {"country":"iran","city":"tehran","population":...}

The generator is configured as a systemd service.

Example:

sudo systemctl status log-generator
2.4 ELK Stack

Directory:

task2/docker-elk/

The ELK stack is based on the Dockerized Elasticsearch, Logstash and Kibana stack.

Implemented:

Elasticsearch
Logstash
Kibana
Kafka input for Logstash
Grok parsing
JSON parsing
Timestamp parsing
Elasticsearch output

The Logstash pipeline consumes:

devops-logs

from Kafka and sends the parsed events to Elasticsearch.

2.5 Elasticsearch ILM

Elasticsearch Index Lifecycle Management is configured manually.

The lifecycle includes:

Hot phase
Rollover based on index size/age
Delete phase

The Logstash output writes to a stable Elasticsearch alias:

devops-logs

The alias points to rollover-managed indices such as:

logstash-devops-000001
logstash-devops-000002
logstash-devops-000003

This allows Logstash to continue writing to the same alias while Elasticsearch manages index rollover.

2.6 FTP

Directory:

task2/vsftp-config/

vsftpd is configured to provide read-only external access to the generated log directory.

Implemented:

Local FTP authentication
FTP-only user access
Read-only FTP configuration
Passive FTP
Dedicated passive port range
Access to:
/var/log/devops-scenario/

The FTP service is intended for accessing the generated logs from outside the VM.

Configuration Notes

This project was built as a practical lab/take-home assignment.

Some configuration values are intentionally specific to the test environment, including:

VM IP addresses
Docker service names
Test usernames
Test passwords
Local filesystem paths
Development/test TLS certificates
Validation

Examples of commands used during testing:

Nginx
nginx -t
rsyslog
sudo rsyslogd -N1
Kafka
docker exec devops-kafka \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:29092 \
  --list
Elasticsearch
curl http://localhost:9200/_cluster/health?pretty
Docker
docker compose ps
Environment

Test environment:

OS: Ubuntu 22.04.4 LTS
Container runtime: Docker
Orchestration: Docker Compose
