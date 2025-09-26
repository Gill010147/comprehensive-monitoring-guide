# 완전 통합 모니터링 시스템 구축: 베어메탈과 쿠버네티스 환경

본 가이드는 기존의 Kubernetes 클러스터 애플리케이션 배포를 확장하여, 베어메탈 서버에 Prometheus/Grafana 모니터링 스택을 구축하고 이를 쿠버네티스 환경과 통합하는 종합적인 DevOps 파이프라인을 구현합니다. 실무에서 활용 가능한 하이브리드 모니터링 환경을 통해 멀티 플랫폼 인프라 관리 역량을 제시합니다.

## 📋 목차

- [프로젝트 개요](#프로젝트-개요)
- [기술 스택](#기술-스택)
- [베어메탈 모니터링 스택 구축](#베어메탈-모니터링-스택-구축)
- [Exporter 개념과 구현](#exporter-개념과-구현)
- [데이터베이스 모니터링 구축](#데이터베이스-모니터링-구축)
- [쿠버네티스 환경 연동](#쿠버네티스-환경-연동)
- [종합 시각화 대시보드](#종합-시각화-대시보드)
- [운영 모범 사례](#운영-모범-사례)

## 🎯 프로젝트 개요

### 하이브리드 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         통합 모니터링 아키텍처                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                            베어메탈 모니터링 계층                                │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │   Prometheus    │  │    Grafana      │  │  MySQL Server   │  │   Exporters  │ │
│  │     :9090       │  │     :3000       │  │     :3306       │  │   :9100      │ │
│  │  (바이너리 설치) │  │  (APT 패키지)   │  │                 │  │   :9104      │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  └──────────────┘ │
├─────────────────────────────────────────────────────────────────────────────────┤
│                          쿠버네티스 모니터링 계층                                │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │kube-prometheus  │  │  Spring Boot    │  │  Service        │  │ ServiceMonitor│ │
│  │     Stack       │  │   FlashGame     │  │   Monitoring    │  │              │ │
│  │   (Helm 배포)   │  │    :8080        │  │                 │  │              │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  └──────────────┘ │
├─────────────────────────────────────────────────────────────────────────────────┤
│                           통합 데이터 플로우                                     │
│  Node Exporter → Prometheus → Grafana ← kube-prometheus-stack                   │
│  MySQL Exporter ↗              ↑            ↗ Spring Boot Actuator              │
│  Application Metrics ──────────┘                                               │ │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 구현 목표

1. **베어메탈 모니터링**: 서버 리소스와 데이터베이스 성능을 실시간 모니터링
2. **쿠버네티스 통합**: 컨테이너 환경과 베어메탈 환경의 일원화된 모니터링
3. **다중 Exporter 활용**: Node, MySQL, Application 메트릭의 종합적 수집
4. **하이브리드 배포**: 바이너리, APT 패키지, Helm 차트의 최적 조합
5. **실시간 시각화**: 통합 대시보드를 통한 전체 인프라 현황 파악

## 🛠 기술 스택

### 모니터링 및 시각화
- **Prometheus (바이너리)**: 메트릭 수집 및 시계열 데이터베이스
- **Grafana (APT)**: 시각화 및 대시보드
- **kube-prometheus-stack (Helm)**: 쿠버네티스 통합 모니터링

### Exporter 생태계
- **Node Exporter**: 시스템 리소스 모니터링
- **MySQL Exporter**: 데이터베이스 성능 모니터링  
- **Spring Boot Actuator**: 애플리케이션 메트릭 노출

### 설치 방식별 특징

| 구성 요소 | 설치 방법 | 주요 이유 |
|-----------|----------|-----------|
| Prometheus | 바이너리 | 최신 버전 유지, 성능 최적화 |
| Grafana | APT 패키지 | 관리 편리, 자동 업데이트 가능 |
| Node Exporter | 바이너리 | 가벼운 실행, Prometheus에 최적화 |
| MySQL Exporter | 바이너리 | 유연한 설정, 커스터마이징 가능 |

## 📊 베어메탈 모니터링 스택 구축

### Prometheus 설치 및 구성

시계열 데이터베이스의 핵심인 Prometheus를 바이너리 방식으로 설치하여 최적의 성능을 확보합니다.

```bash
# 전용 사용자 계정 생성 (보안 강화)
sudo useradd --no-create-home --shell /bin/false prometheus

# 디렉터리 구조 생성
sudo mkdir /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus

# 최신 버전 다운로드 및 설치
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v3.2.1/prometheus-3.2.1.linux-amd64.tar.gz
tar xvf prometheus-3.2.1.linux-amd64.tar.gz

# 바이너리 및 설정 파일 배치
cd prometheus-*/
sudo mv prometheus promtool /usr/local/bin/
sudo mv prometheus.yml /etc/prometheus/
```

### 기본 설정 파일 구성

```yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s       # 메트릭 수집 주기

scrape_configs:
  - job_name: 'prometheus'   # Prometheus 자체 모니터링
    static_configs:
      - targets: ['localhost:9090']
      
  - job_name: 'node'         # 서버 리소스 모니터링
    static_configs:
      - targets: ['localhost:9100']
      
  - job_name: 'mysql-exporter'  # 데이터베이스 모니터링
    static_configs:
      - targets: ['localhost:9104']
```

> **스크레이핑(Scraping)이란?** Prometheus가 각 타겟의 /metrics 엔드포인트에서 메트릭 데이터를 주기적으로 가져오는 Pull 방식의 데이터 수집 방법입니다. 이는 Push 방식과 달리 중앙 집중식 관리와 서비스 디스커버리를 가능하게 합니다.

### systemd 서비스 등록

```bash
# Prometheus 서비스 파일 생성
sudo tee /etc/systemd/system/prometheus.service <<EOF
[Unit]
Description=Prometheus
Wants=network-online.target   
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.listen-address=0.0.0.0:9090

[Install]
WantedBy=multi-user.target
EOF

# 서비스 활성화 및 시작
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```

<img width="975" height="400" alt="image" src="https://github.com/user-attachments/assets/30da67c3-85be-4b94-93d4-7ec0cc27e879" />


### Grafana 설치 및 구성

관리 편의성을 위해 APT 패키지 방식으로 Grafana를 설치합니다.

```bash
# APT 저장소 추가 (최신 방식)
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

# 설치 및 서비스 시작
sudo apt update && sudo apt install -y grafana
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server
```

<img width="1003" height="244" alt="image" src="https://github.com/user-attachments/assets/31f7d112-d560-4975-acb2-1bde50982d33" />
<img width="1004" height="246" alt="image" src="https://github.com/user-attachments/assets/7333033c-51b2-445e-ba1c-17990879426a" />


## 📈 Exporter 개념과 구현

### Exporter 아키텍처 이해

Exporter는 Prometheus 생태계의 핵심 구성요소로, 다양한 시스템과 애플리케이션에서 메트릭을 추출하여 Prometheus 형식으로 변환하는 어댑터 역할을 수행합니다.

#### 동작 원리

1. **데이터 수집**: 대상 시스템에서 성능 지표 추출
2. **형식 변환**: Prometheus 메트릭 형식으로 변환
3. **HTTP 엔드포인트 노출**: `/metrics` 경로에서 데이터 제공
4. **Prometheus 스크레이핑**: 설정된 주기마다 데이터 수집

#### 메트릭 형식 예시

```prometheus
# HELP node_cpu_seconds_total Total CPU time
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="user"} 12345.6
node_cpu_seconds_total{cpu="0",mode="system"} 2345.6

# HELP mysql_global_status_connections Total connections
# TYPE mysql_global_status_connections counter
mysql_global_status_connections 1542
```

### Node Exporter 구현

서버의 하드웨어 및 OS 레벨 메트릭을 수집하는 기본 Exporter입니다.

```bash
# Node Exporter 다운로드 및 설치
cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
tar xvf node_exporter-1.9.1.linux-amd64.tar.gz
sudo mv node_exporter-*/node_exporter /usr/local/bin/

# systemd 서비스 등록
sudo tee /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

# 서비스 시작 및 확인
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter
```

### 주요 메트릭 수집 항목

| 카테고리 | 메트릭 예시 | 설명 |
|----------|-------------|------|
| CPU | `node_cpu_seconds_total` | CPU 사용 시간 (모드별) |
| 메모리 | `node_memory_MemTotal_bytes` | 총 메모리 용량 |
| 디스크 | `node_disk_io_time_seconds_total` | 디스크 I/O 시간 |
| 네트워크 | `node_network_receive_bytes_total` | 네트워크 수신 바이트 |
| 파일시스템 | `node_filesystem_avail_bytes` | 사용 가능한 디스크 공간 |

<img width="821" height="736" alt="image" src="https://github.com/user-attachments/assets/e2e9bc68-8b51-440b-9a5e-7ff6eee9349e" />


## 🗄️ 데이터베이스 모니터링 구축

### MySQL 환경 구성

애플리케이션의 백엔드 데이터베이스 성능을 모니터링하기 위한 MySQL 설정을 구성합니다.

```sql
-- 모니터링 전용 사용자 생성
CREATE USER 'monitoring'@'localhost' IDENTIFIED BY 'monitoring_password';

-- 필요 권한 부여
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'monitoring'@'localhost';
FLUSH PRIVILEGES;

-- 테스트 데이터베이스 및 테이블 생성
CREATE DATABASE IF NOT EXISTS fisa;
USE fisa;

CREATE TABLE metrics_test (
    id INT AUTO_INCREMENT PRIMARY KEY,
    metric_name VARCHAR(255),
    metric_value FLOAT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 샘플 데이터 삽입
INSERT INTO metrics_test (metric_name, metric_value) VALUES
('cpu_usage_percent', 15.3),
('memory_usage_percent', 45.6),
('disk_usage_percent', 78.2),
('active_connections', 12),
('queries_per_second', 156.7);
```

### MySQL Exporter 구축

```bash
# MySQL Exporter 다운로드 및 설치
cd /tmp
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.17.2/mysqld_exporter-0.17.2.linux-amd64.tar.gz
tar -xzf mysqld_exporter-0.17.2.linux-amd64.tar.gz
sudo mv mysqld_exporter-0.17.2.linux-amd64/mysqld_exporter /usr/local/bin/

# 버전 확인
mysqld_exporter --version

<img width="975" height="160" alt="image" src="https://github.com/user-attachments/assets/2fdc4c0c-5ee8-4083-8fb3-baff24d66821" />


# MySQL 접속 설정 파일 생성
sudo tee /etc/.my.cnf <<EOF
[client]
user=monitoring
password=monitoring_password
host=localhost
port=3306
EOF

# 보안을 위한 권한 설정
sudo chmod 600 /etc/.my.cnf
sudo chown prometheus:prometheus /etc/.my.cnf
```

### MySQL Exporter 서비스 등록

```bash
# systemd 서비스 파일 생성
sudo tee /etc/systemd/system/mysqld_exporter.service <<EOF
[Unit]
Description=Prometheus MySQL Exporter
After=network.target mysql.service

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/mysqld_exporter \
  --config.my-cnf=/etc/.my.cnf \
  --web.listen-address=:9104
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# 서비스 시작 및 활성화
sudo systemctl daemon-reload
sudo systemctl enable --now mysqld_exporter
sudo systemctl status mysqld_exporter

# 메트릭 노출 확인
curl http://localhost:9104/metrics | head -20
```

### 수집되는 MySQL 메트릭

| 메트릭 그룹 | 주요 메트릭 | 모니터링 목적 |
|-------------|-------------|---------------|
| 연결 관리 | `mysql_global_status_connections` | 총 연결 수 추적 |
| 쿼리 성능 | `mysql_global_status_queries` | 쿼리 실행 빈도 |
| 버퍼 풀 | `mysql_global_status_innodb_buffer_pool_pages_total` | 메모리 사용 효율성 |
| 복제 상태 | `mysql_slave_lag_seconds` | 마스터-슬레이브 지연 시간 |
| 테이블 락 | `mysql_global_status_table_locks_waited` | 테이블 잠금 대기 |

<img width="972" height="439" alt="image" src="https://github.com/user-attachments/assets/dae7f247-175e-4a91-834a-751782edf2d1" />


## ⚙️ 쿠버네티스 환경 연동

### Spring Boot 애플리케이션 메트릭 노출

기존에 배포된 Spring Boot 애플리케이션에서 Actuator를 통해 메트릭을 노출합니다.

```yaml
# application.yml 설정
management:
  endpoints:
    web:
      exposure:
        include: "prometheus,health,metrics,info"
  metrics:
    export:
      prometheus:
        enabled: true
  endpoint:
    prometheus:
      enabled: true
    health:
      show-details: always

server:
  port: 8080
```

### ServiceMonitor를 통한 자동 스크레이핑

```yaml
# spring-boot-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: spring-boot-servicemonitor
  namespace: monitoring
  labels:
    release: prometheus        # kube-prometheus-stack 릴리스와 일치
spec:
  namespaceSelector:
    matchNames:
      - default
  selector:
    matchLabels:
      app: spring-boot
  endpoints:
    - port: http               # Service의 포트 이름
      path: /actuator/prometheus
      interval: 15s
      scrapeTimeout: 10s
```

> **ServiceMonitor란?** Prometheus Operator가 제공하는 CRD로, 쿠버네티스 Service를 자동으로 Prometheus 스크레이핑 대상으로 등록합니다. 이를 통해 동적인 서비스 디스커버리와 설정 관리가 가능합니다.

### kube-prometheus-stack과의 통합

```bash
# ServiceMonitor 적용
kubectl apply -f spring-boot-servicemonitor.yaml

# 스크레이핑 대상 확인
kubectl -n monitoring port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090

# 브라우저에서 http://localhost:9090/targets 확인
```

### 베어메탈과 쿠버네티스 연동 설정

베어메탈의 Prometheus와 쿠버네티스의 kube-prometheus-stack 간 연동을 위한 설정입니다.

```yaml
# /etc/prometheus/prometheus.yml에 추가
scrape_configs:
  # 기존 설정들...
  
  - job_name: 'kubernetes-spring-app'
    kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names: ['default']
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        action: keep
        regex: spring-boot-service
      - source_labels: [__meta_kubernetes_endpoint_port_name]
        action: keep
        regex: http
      - target_label: __metrics_path__
        replacement: /actuator/prometheus
```

![통합 모니터링 아키텍처 동작 상태 스크린샷]

## 📊 종합 시각화 대시보드

### Grafana 데이터 소스 구성

```bash
# Grafana 접속: http://localhost:3000 (admin/admin)
# Data Sources > Add data source > Prometheus 선택
# URL: http://localhost:9090
# Save & Test 클릭
```

### 멀티 환경 통합 대시보드

#### 1. 시스템 리소스 대시보드 (Node Exporter)

**대시보드 ID: 1860 (Node Exporter Full)**

주요 패널 구성:
- CPU 사용률: `100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`
- 메모리 사용률: `(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100`
- 디스크 I/O: `rate(node_disk_read_bytes_total[5m]) + rate(node_disk_written_bytes_total[5m])`
- 네트워크 트래픽: `rate(node_network_receive_bytes_total[5m]) + rate(node_network_transmit_bytes_total[5m])`

#### 2. MySQL 성능 대시보드

**대시보드 ID: 7362 (MySQL Overview)**

핵심 메트릭:
- 연결 상태: `mysql_global_status_threads_connected`
- 쿼리 처리량: `rate(mysql_global_status_queries[5m])`
- 버퍼 풀 효율성: `mysql_global_status_innodb_buffer_pool_read_requests / mysql_global_status_innodb_buffer_pool_reads * 100`
- 슬로우 쿼리: `mysql_global_status_slow_queries`

#### 3. Spring Boot 애플리케이션 대시보드

**대시보드 ID: 10231 (Spring Boot Micrometer)**

애플리케이션 메트릭:
- HTTP 요청 처리량: `sum(rate(http_server_requests_seconds_count[5m])) by (uri, status)`
- 응답 시간 분포: `histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri))`
- JVM 메모리 사용량: `jvm_memory_used_bytes / jvm_memory_max_bytes * 100`
- Garbage Collection: `rate(jvm_gc_pause_seconds_count[5m])`

### 커스텀 통합 대시보드 구성

```json
{
  "dashboard": {
    "title": "Hybrid Infrastructure Monitoring",
    "panels": [
      {
        "title": "Infrastructure Health Overview",
        "targets": [
          {
            "expr": "up",
            "legendFormat": "{{job}} - {{instance}}"
          }
        ]
      },
      {
        "title": "Application Performance",
        "targets": [
          {
            "expr": "rate(http_server_requests_seconds_count[5m])",
            "legendFormat": "{{uri}} - {{status}}"
          }
        ]
      },
      {
        "title": "Database Connections",
        "targets": [
          {
            "expr": "mysql_global_status_threads_connected",
            "legendFormat": "Active Connections"
          }
        ]
      }
    ]
  }
}
```

![통합 대시보드 전체 화면 스크린샷]

## 🔧 Helm을 활용한 고급 배포 관리

### 커스텀 Helm 차트 생성

모니터링 스택의 일부를 Helm으로 관리하여 버전 관리와 배포 자동화를 구현합니다.

```bash
# 모니터링용 Helm 차트 생성
helm create monitoring-stack
cd monitoring-stack
```

### values.yaml 커스터마이징

```yaml
# values.yaml
prometheus:
  enabled: true
  image:
    repository: prom/prometheus
    tag: v2.45.0
  service:
    type: NodePort
    port: 9090
    nodePort: 30090
  
grafana:
  enabled: true
  adminPassword: "secure-admin-password"
  service:
    type: NodePort
    port: 3000
    nodePort: 30030
  datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus:9090
      isDefault: true

nodeExporter:
  enabled: true
  hostNetwork: true
  hostPID: true
  service:
    port: 9100

mysqlExporter:
  enabled: true
  config:
    host: mysql-service
    port: 3306
    user: monitoring
    password: monitoring_password
```

### 템플릿 구성

```yaml
# templates/prometheus-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "monitoring-stack.fullname" . }}-prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: "{{ .Values.prometheus.image.repository }}:{{ .Values.prometheus.image.tag }}"
        ports:
        - containerPort: {{ .Values.prometheus.service.port }}
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
        command:
        - /bin/prometheus
        - --config.file=/etc/prometheus/prometheus.yml
        - --storage.tsdb.path=/prometheus
        - --web.console.libraries=/etc/prometheus/console_libraries
        - --web.console.templates=/etc/prometheus/consoles
      volumes:
      - name: config
        configMap:
          name: {{ include "monitoring-stack.fullname" . }}-prometheus-config
```

### Helm 배포 및 관리

```bash
# 차트 린트 검사
helm lint ./monitoring-stack

# 드라이런으로 배포 시뮬레이션
helm install monitoring-stack ./monitoring-stack --dry-run --debug

# 실제 배포
helm install monitoring-stack ./monitoring-stack --namespace monitoring --create-namespace

# 업그레이드
helm upgrade monitoring-stack ./monitoring-stack --namespace monitoring

# 배포 상태 확인
helm status monitoring-stack --namespace monitoring

# 롤백 (필요 시)
helm rollback monitoring-stack 1 --namespace monitoring
```

![Helm으로 배포된 모니터링 스택 상태 스크린샷]

## 📈 고급 알림 및 자동화

### PrometheusRule을 통한 알림 규칙

```yaml
# alerting-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: infrastructure-alerts
  namespace: monitoring
spec:
  groups:
  - name: infrastructure.rules
    rules:
    - alert: HighCPUUsage
      expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage detected"
        description: "CPU usage is above 80% for {{ $labels.instance }}"
        
    - alert: MySQLDown
      expr: up{job="mysql-exporter"} == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "MySQL instance is down"
        description: "MySQL database is not responding"
        
    - alert: ApplicationResponseTime
      expr: histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri)) > 1
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "Application response time is high"
        description: "95th percentile response time is above 1 second"
```

### Alertmanager 구성

```yaml
# alertmanager-config.yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'monitoring@company.com'
  smtp_auth_username: 'monitoring@company.com'
  smtp_auth_password: 'app-password'

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'

receivers:
- name: 'web.hook'
  email_configs:
  - to: 'admin@company.com'
    subject: '[ALERT] {{ .GroupLabels.alertname }}'
    body: |
      {{ range .Alerts }}
      Alert: {{ .Annotations.summary }}
      Description: {{ .Annotations.description }}
      Instance: {{ .Labels.instance }}
      {{ end }}
  webhook_configs:
  - url: 'http://slack-webhook-url'
    send_resolved: true
```

## 🚀 성능 최적화 및 튜닝

### Prometheus 설정 최적화

```yaml
# /etc/prometheus/prometheus.yml - 운영 환경 설정
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'production'
    region: 'korea-central'

storage:
  tsdb:
    retention.time: 30d
    retention.size: 50GB
    
remote_write:
  - url: "https://prometheus-remote-storage/api/v1/write"
    queue_config:
      capacity: 2500
      max_samples_per_send: 1000
      batch_send_deadline: 5s
```

### 쿠버네티스 리소스 최적화

```yaml
# optimized-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app-optimized
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    spec:
      containers:
      - name: flashgame
        image: byeonggill/flashgame:latest
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
          failureThreshold: 3
        env:
        - name: JAVA_OPTS
          value: "-Xms512m -Xmx1024m -XX:+UseG1GC -XX:+UseStringDeduplication"
```

### 메트릭 수집 최적화

```bash
# Node Exporter 최적화 실행
sudo systemctl edit node_exporter
```

```ini
[Service]
ExecStart=
ExecStart=/usr/local/bin/node_exporter \
  --web.listen-address=":9100" \
  --collector.disable-defaults \
  --collector.cpu \
  --collector.memory \
  --collector.disk \
  --collector.network \
  --collector.filesystem \
  --collector.loadavg \
  --collector.systemd
```

## 💡 운영 모범 사례

### 보안 강화

#### 1. 인증 및 권한 관리

```yaml
# grafana 보안 설정
[security]
admin_user = admin
admin_password = $__env{GF_SECURITY_ADMIN_PASSWORD}
secret_key = $__env{GF_SECURITY_SECRET_KEY}
disable_gravatar = true
cookie_secure = true
cookie_samesite = strict

[auth.anonymous]
enabled = false

[auth.basic]
enabled = true

[users]
allow_sign_up = false
allow_org_create = false
```

#### 2. 네트워크 보안

```bash
# 방화벽 설정 (UFW)
sudo ufw allow from 10.0.0.0/8 to any port 9090 comment 'Prometheus'
sudo ufw allow from 10.0.0.0/8 to any port 3000 comment 'Grafana'
sudo ufw allow from 10.0.0.0/8 to any port 9100 comment 'Node Exporter'
sudo ufw deny 9104 comment 'MySQL Exporter - Internal Only'
```

#### 3. TLS 암호화

```yaml
# Prometheus TLS 설정
tls_server_config:
  cert_file: /etc/ssl/certs/prometheus.crt
  key_file: /etc/ssl/private/prometheus.key
  
scrape_configs:
  - job_name: 'secure-targets'
    scheme: https
    tls_config:
      ca_file: /etc/ssl/certs/ca.crt
      insecure_skip_verify: false
```

### 백업 및 복구

#### 데이터 백업 자동화

```bash
#!/bin/bash
# prometheus-backup.sh

BACKUP_DIR="/backup/prometheus"
DATA_DIR="/var/lib/prometheus"
DATE=$(date +%Y%m%d_%H%M%S)

# 백업 디렉토리 생성
mkdir -p $BACKUP_DIR

# Prometheus 스냅샷 생성
curl -XPOST http://localhost:9090/api/v1/admin/tsdb/snapshot

# 스냅샷 압축 백업
SNAPSHOT_DIR=$(ls -t $DATA_DIR/snapshots/ | head -n1)
tar -czf "$BACKUP_DIR/prometheus_$DATE.tar.gz" -C "$DATA_DIR/snapshots/" "$SNAPSHOT_DIR"

# 30일 이상 된 백업 파일 삭제
find $BACKUP_DIR -name "prometheus_*.tar.gz" -mtime +30 -delete

echo "Backup completed: prometheus_$DATE.tar.gz"
```

#### Grafana 대시보드 백업

```bash
#!/bin/bash
# grafana-backup.sh

GRAFANA_URL="http://localhost:3000"
API_KEY="your-grafana-api-key"
BACKUP_DIR="/backup/grafana"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# 모든 대시보드 백업
for dashboard_uid in $(curl -s -H "Authorization: Bearer $API_KEY" \
  "$GRAFANA_URL/api/search?type=dash-db" | jq -r '.[].uid'); do
  
  curl -s -H "Authorization: Bearer $API_KEY" \
    "$GRAFANA_URL/api/dashboards/uid/$dashboard_uid" | \
    jq '.dashboard' > "$BACKUP_DIR/dashboard_${dashboard_uid}_$DATE.json"
done

echo "Grafana dashboards backed up to $BACKUP_DIR"
```

### 모니터링 자동화

#### 헬스체크 및 자동 복구

```bash
#!/bin/bash
# monitoring-health-check.sh

SERVICES=("prometheus" "grafana-server" "node_exporter" "mysqld_exporter")
LOG_FILE="/var/log/monitoring-health.log"

check_service() {
    local service=$1
    if ! systemctl is-active --quiet $service; then
        echo "$(date): $service is down, attempting restart" | tee -a $LOG_FILE
        systemctl restart $service
        sleep 10
        if systemctl is-active --quiet $service; then
            echo "$(date): $service restarted successfully" | tee -a $LOG_FILE
        else
            echo "$(date): Failed to restart $service" | tee -a $LOG_FILE
            # 알림 발송 로직 추가
        fi
    fi
}

for service in "${SERVICES[@]}"; do
    check_service $service
done
```

## 🎉 결론

### 주요 성과

본 프로젝트를 통해 다음과 같은 종합적인 모니터링 인프라를 구축했습니다:

1. **하이브리드 모니터링 아키텍처**: 베어메탈과 쿠버네티스 환경의 완전 통합
2. **다층 메트릭 수집**: 시스템, 데이터베이스, 애플리케이션 레벨의 종합 모니터링
3. **확장 가능한 Exporter 생태계**: Node, MySQL, Spring Boot Actuator를 통한 다양한 메트릭 수집
4. **지능형 알림 시스템**: PrometheusRule과 Alertmanager를 통한 선제적 장애 대응
5. **시각화 및 대시보드**: 실시간 인프라 현황 파악과 성능 분석

### 기술적 역량 증명

#### DevOps 및 인프라 관리
- **다중 배포 방식 숙련도**: 바이너리, APT, Helm을 통한 최적화된 배포 전략
- **서비스 메시 구축**: Prometheus pull 방식 모니터링 아키텍처 설계
- **쿠버네티스 운영**: ServiceMonitor, CRD를 활용한 클라우드 네이티브 모니터링

#### 모니터링 및 관측 가능성 (Observability)
- **메트릭 엔지니어링**: 사용자 정의 메트릭 정의 및 수집 파이프라인 구축
- **시계열 데이터베이스 관리**: Prometheus TSDB 최적화 및 성능 튜닝
- **시각화 전문성**: Grafana를 통한 비즈니스 인텔리전스 대시보드 구성

#### 보안 및 운영
- **제로 트러스트 보안**: 최소 권한 원칙과 네트워크 세분화 적용
- **자동화 및 복구**: 장애 감지부터 자동 복구까지의 전체 파이프라인 구현
- **백업 및 재해복구**: 데이터 보존과 비즈니스 연속성 계획 수립

### 실무 적용 가능성

#### 엔터프라이즈 환경 적용
- **멀티 클라우드 모니터링**: AWS, GCP, Azure 등 다중 클라우드 환경 통합 관리
- **마이크로서비스 아키텍처**: 분산 서비스 간 의존성 추적 및 성능 모니터링
- **SRE/Platform Engineering**: 서비스 레벨 목표(SLO) 기반 운영 자동화

#### 확장 및 고도화 방안
- **AI/ML 연동**: 이상 탐지 및 예측 분석을 통한 지능형 모니터링
- **GitOps 통합**: ArgoCD와 연동한 선언적 인프라 관리
- **다중 지역 배포**: 글로벌 인프라 모니터링 및 재해복구 체계 구축

### 학습 성과 및 인사이트

1. **Prometheus 생태계 깊이 있는 이해**: Exporter 패턴과 Pull 기반 모니터링의 장점 체험
2. **쿠버네티스 운영 역량**: Operator 패턴과 CRD를 활용한 선언적 관리 방법 습득
3. **하이브리드 인프라 설계**: 온프레미스와 클라우드 환경의 효과적인 통합 방법론 개발
4. **관측 가능성 구현**: 메트릭, 로그, 트레이싱을 아우르는 통합 모니터링 철학 정립
5. **운영 자동화**: 장애 대응부터 용량 계획까지 전체 운영 생명주기 자동화 경험

이번 프로젝트는 단순한 모니터링 도구 설치를 넘어서, 현대적 인프라 운영의 핵심인 관측 가능성(Observability)을 구현하고 데이터 기반 의사결정을 가능하게 하는 완전한 플랫폼을 구축한 경험입니다. 실제 프로덕션 환경에서 요구되는 고가용성, 확장성, 보안성을 모두 고려한 엔터프라이즈급 솔루션으로 평가받을 수 있을 것입니다.

## 📚 참고 자료

- [Prometheus 공식 문서](https://prometheus.io/docs/)
- [Grafana 대시보드 갤러리](https://grafana.com/grafana/dashboards/)
- [Kubernetes Monitoring 가이드](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/)
- [Spring Boot Actuator 문서](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Prometheus Exporter 개발 가이드](https://prometheus.io/docs/instrumenting/writing_exporters/)
- [Helm 차트 개발 가이드](https://helm.sh/docs/chart_best_practices/)

---

**이전 Repository**: [k8s-monitoring-guide](https://github.com/Gill010147/k8s-monitoring-guide)

**Author**: Gill010147  
**Date**: 2025-09-26  
**Version**: 2.0.0
