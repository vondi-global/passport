# K3s Cluster Infrastructure Passport

> **Версия:** 1.0.0 | **Дата:** 2025-12-23 | **Статус:** PRODUCTION

---

## Краткое описание

Production K3s кластер для платформы Vondi с высокой доступностью, мониторингом и безопасностью.

**Архитектура:** 3-node HA cluster с embedded etcd
**Версия K3s:** v1.33.6+k3s1
**Готовность:** 95% (Phase 2 complete)

---

## Содержание

1. [Обзор кластера](#обзор-кластера)
2. [Ноды кластера](#ноды-кластера)
3. [Компоненты](#компоненты)
4. [Namespaces](#namespaces)
5. [Мониторинг](#мониторинг)
6. [Безопасность](#безопасность)
7. [Приложения](#приложения)
8. [Порты и доступ](#порты-и-доступ)
9. [Команды управления](#команды-управления)

---

## Обзор кластера

### Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                    K3s HA Cluster (3 nodes)                  │
│                     Embedded etcd consensus                   │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
   ┌────▼─────┐         ┌─────▼────┐         ┌─────▼────┐
   │ vondi-rs │         │ svetu-rs │         │  terra   │
   │  24GB    │         │  16GB    │         │  12GB    │
   │  8 vCPU  │         │  6 vCPU  │         │  6 vCPU  │
   └────┬─────┘         └─────┬────┘         └─────┬────┘
        │                     │                     │
        └─────────────────────┴─────────────────────┘
                              │
        ┌─────────────────────┴─────────────────────┐
        │                                           │
   ┌────▼────────┐                          ┌───────▼──────┐
   │   Traefik   │                          │  Monitoring  │
   │   Ingress   │                          │   Grafana    │
   │ v3.6.5      │                          │ VictoriaM    │
   └─────────────┘                          └──────────────┘
```

### Характеристики

| Параметр | Значение |
|----------|----------|
| **K3s версия** | v1.33.6+k3s1 |
| **Kubernetes API** | v1.33.6 |
| **Container Runtime** | containerd 2.1.5-k3s1.33 |
| **CNI** | Flannel (VXLAN) |
| **Ingress Controller** | Traefik v3.6.5 |
| **Cert Manager** | v1.19.2 (Let's Encrypt) |
| **Storage Class** | local-path (default) |
| **Etcd** | Embedded (HA mode) |

---

## Ноды кластера

### 1. vondi-rs (Master + Worker)

| Параметр | Значение |
|----------|----------|
| **Hostname** | vmi2881902 |
| **IP Address** | 62.169.20.78 |
| **RAM** | 24 GB |
| **CPU** | 8 vCPU |
| **Disk** | 400 GB SSD |
| **OS** | Ubuntu 24.04.3 LTS (kernel 6.8.0-88) |
| **Roles** | control-plane, etcd, worker |
| **Provider** | Contabo (voroshilovdo@gmail.com) |

**Основные workloads:**
- Backend (2 pods)
- Traefik Ingress (2 replicas)
- Harbor Registry
- Grafana

### 2. svetu-rs (Master + Worker)

| Параметр | Значение |
|----------|----------|
| **Hostname** | SveTu.rs |
| **IP Address** | 161.97.89.28 |
| **RAM** | 16 GB |
| **CPU** | 6 vCPU |
| **Disk** | 200 GB NVMe |
| **OS** | Ubuntu 24.04.3 LTS (kernel 6.8.0-60) |
| **Roles** | control-plane, etcd, worker |
| **Provider** | Contabo (voroshilovdo@gmail.com) |

**Основные workloads:**
- VictoriaMetrics (metrics storage)
- Loki (log aggregation)
- Redis StatefulSet
- kube-state-metrics

### 3. terra-vondi-rs (Master + Worker)

| Параметр | Значение |
|----------|----------|
| **Hostname** | vmi2583001 |
| **IP Address** | 207.180.197.172 |
| **RAM** | 12 GB |
| **CPU** | 6 vCPU |
| **Disk** | 300 GB SSD |
| **OS** | Ubuntu 24.04.3 LTS (kernel 6.8.0-90) |
| **Roles** | control-plane, etcd, worker |
| **Provider** | Contabo (info@svetu.rs) |

**Основные workloads:**
- Vondi Agro application
- Velero (backups)
- Kyverno (policy enforcement)
- vmagent (metrics collection)

### Cluster Resources

**Total Capacity:**
- RAM: 52 GB
- CPU: 20 vCPU
- Disk: 900 GB

**Current Usage:**
- RAM: 13.6 GB (26%)
- CPU: ~8% average
- Available: 38.4 GB RAM, 18.4 vCPU

---

## Компоненты

### Core Components

| Component | Version | Namespace | Replicas | Status |
|-----------|---------|-----------|----------|--------|
| **K3s Server** | v1.33.6+k3s1 | - | 3 | ✅ Running |
| **etcd** | embedded | - | 3 | ✅ Running |
| **Flannel CNI** | built-in | kube-system | 3 DaemonSet | ✅ Running |
| **CoreDNS** | built-in | kube-system | 2 | ✅ Running |
| **Local Path Provisioner** | built-in | kube-system | 1 | ✅ Running |

### Ingress & Networking

| Component | Version | Namespace | Replicas | Endpoints |
|-----------|---------|-----------|----------|-----------|
| **Traefik** | v3.6.5 | kube-system | 2 | *.vondi.rs, *.svetu.rs |
| **cert-manager** | v1.19.2 | cert-manager | 3 pods | Let's Encrypt automation |
| **MetalLB** | - | - | - | Not installed (using NodePort) |

### Registry

| Component | Version | Namespace | Endpoint | Storage |
|-----------|---------|-----------|----------|---------|
| **Harbor** | v2.14.1 | harbor | registry.vondi.rs | PVC 100GB |
| **Harbor Database** | PostgreSQL 15 | harbor | - | PVC 10GB |
| **Harbor Redis** | 7.x | harbor | - | PVC 5GB |

### Monitoring Stack

| Component | Version | Namespace | Purpose |
|-----------|---------|-----------|---------|
| **Grafana** | 11.x | monitoring | Visualization |
| **VictoriaMetrics** | latest | monitoring | Metrics storage (TSDB) |
| **vmagent** | latest | monitoring | Metrics collection |
| **Loki** | 3.0.0 | monitoring | Log aggregation |
| **Promtail** | 3.0.0 | monitoring | Log collection (3 pods) |
| **Node Exporter** | v1.8.2 | monitoring | System metrics (DaemonSet) |
| **kube-state-metrics** | v2.17.0 | monitoring | K8s objects metrics |

**Grafana URL:** https://grafana.vondi.rs/
**Credentials:** `/p/github.com/vondi-global/credentials/grafana-k3s.md`

### Security Stack

| Component | Version | Namespace | Coverage |
|-----------|---------|-----------|----------|
| **Trivy** | latest | harbor | Image scanning |
| **Falco** | v0.39.0 | falco-system | Runtime security (2/2 nodes) |
| **Kyverno** | v1.14.0 | kyverno | Admission control (4 pods) |
| **AIDE** | 0.18.6 | - | File integrity (3 nodes) |
| **rkhunter** | 1.4.6 | - | Rootkit detection |
| **fail2ban** | latest | - | SSH brute-force (154 IPs banned) |
| **UFW** | latest | - | Firewall (all nodes) |

### Backup Solutions

| Component | Version | Namespace | Schedule | Retention |
|-----------|---------|-----------|----------|-----------|
| **Velero** | latest | velero | On-demand | 30 days |
| **K3s etcd snapshots** | built-in | - | Every 12h | 5 snapshots |
| **PostgreSQL dumps** | cron | - | Daily | 7 days |

---

## Namespaces

| Namespace | Purpose | Pods | Status |
|-----------|---------|------|--------|
| **production** | Production applications | 2 | ✅ Active |
| **monitoring** | Monitoring stack | 10 | ✅ Active |
| **harbor** | Container registry | 6 | ✅ Active |
| **cert-manager** | Certificate automation | 3 | ✅ Active |
| **kyverno** | Policy enforcement | 4 | ✅ Active |
| **falco-system** | Runtime security | 2 | ✅ Active |
| **velero** | Backup solution | 1 | ✅ Active |
| **kube-system** | K3s core components | 12 | ✅ Active |
| **default** | Default namespace | 0 | ✅ Empty |

**Total namespaces:** 15

---

## Мониторинг

### Grafana Dashboards

**URL:** https://grafana.vondi.rs/

| Dashboard | UID | Назначение |
|-----------|-----|------------|
| **Node Exporter** | xfpJB9FGz | System metrics (CPU, RAM, Disk, Network) |
| **Kubernetes Monitoring** | msqzbWjWk | Pods & Containers (cAdvisor) |
| **K8S Dashboard** | 1a45d500-d723-40ff-8003-a43cc4c95ac9 | Full cluster overview |
| **kube-state-metrics-v2** | garysdevil-kube-state-metrics-v2 | Kubernetes objects |

### Metrics Collection

```bash
# VictoriaMetrics endpoint
kubectl port-forward -n monitoring victoria-metrics-victoria-metrics-single-server-0 8428:8428
curl 'http://localhost:8428/api/v1/query?query=up'

# Loki logs endpoint
kubectl port-forward -n monitoring loki-0 3100:3100
curl 'http://localhost:3100/loki/api/v1/labels'
```

### Key Metrics

**Node-level (Node Exporter):**
- CPU usage, load average
- Memory (total, used, cached)
- Disk I/O, space, IOPS
- Network traffic (eth0, flannel.1, wg0)

**Cluster-level (cAdvisor):**
- 51 pods
- 54 containers
- Aggregate CPU/Memory/Network

**Kubernetes objects (kube-state-metrics):**
- 3 nodes
- 27 deployments
- 15 namespaces
- StatefulSets, DaemonSets, PVCs

### Alert Rules

**Configured:** 4 alerts
- High memory usage (>80%)
- High CPU usage (>80%)
- Pod crash loops
- Node disk full (>90%)

---

## Безопасность

### Multi-Layer Defense (7 layers)

| Layer | Component | Function | Status |
|-------|-----------|----------|--------|
| **1** | Trivy | Image vulnerability scanning | ✅ 100% |
| **2** | Falco | Runtime threat detection | ✅ 2/2 nodes |
| **3** | Kyverno | Admission control & policies | ✅ 4/4 pods |
| **4** | AIDE | File integrity monitoring | ✅ 90% deployed |
| **5** | fail2ban | SSH brute-force protection | ✅ 154 IPs banned |
| **6** | UFW | Host firewall | ✅ All nodes |
| **7** | rkhunter | Rootkit detection | ✅ Installed |

**Security Score:** 9/10

### SSH Access

```bash
# vondi-rs (62.169.20.78)
ssh -o IdentitiesOnly=yes -i ~/.ssh/id_ed25519 root@vondi.rs

# svetu-rs (161.97.89.28)
ssh -o IdentitiesOnly=yes -i ~/.ssh/id_ed25519 root@svetu.rs

# terra-vondi-rs (207.180.197.172)
ssh root@terra.vondi.rs
```

**SSH Hardening:**
- PasswordAuthentication disabled
- Root login allowed (key-only)
- fail2ban active
- UFW firewall configured

---

## Приложения

### Production Namespace

| Application | Type | Replicas | Status | Endpoint |
|-------------|------|----------|--------|----------|
| **Backend Monolith** | Deployment | 2 | ✅ Running | backend.production.svc:3000 |
| **Redis** | StatefulSet | 1 | ✅ Running | redis.production.svc:6379 |
| **Frontend** | - | - | 📋 Planned | Week 4 migration |

### Current Deployment Status

**In K8s (production):**
- ✅ Backend Monolith (2 pods, ~180MB each, ~50m CPU)
- ✅ Redis (StatefulSet)

**Still in Docker (on hosts):**
- Frontend (port 3001)
- Auth Service (28086/20053)
- Listings Service (8084/50053)
- Delivery Service (8082/50052)
- WMS Service (8085/50055)

**Migration plan:** Weeks 4-6 (Frontend → Microservices)

---

## Порты и доступ

### External Access (Public)

| Service | Port | Protocol | URL |
|---------|------|----------|-----|
| **HTTP** | 80 | TCP | *.vondi.rs, *.svetu.rs |
| **HTTPS** | 443 | TCP | *.vondi.rs, *.svetu.rs |
| **SSH** | 22 | TCP | All nodes |

### Ingress Routes

| Host | Target | Namespace | Service |
|------|--------|-----------|---------|
| **grafana.vondi.rs** | Grafana | monitoring | grafana:80 |
| **registry.vondi.rs** | Harbor | harbor | harbor-portal:80 |
| **vondi.rs** | Frontend | - | External (Docker) |
| **dev.vondi.rs** | Dev Backend | - | External (Docker) |

### Internal Services

| Service | Port | Namespace | Type |
|---------|------|-----------|------|
| VictoriaMetrics | 8428 | monitoring | ClusterIP |
| Loki | 3100 | monitoring | ClusterIP |
| Grafana | 80 | monitoring | ClusterIP |
| Redis | 6379 | production | ClusterIP |
| Backend | 3000 | production | ClusterIP |

---

## Команды управления

### Cluster Info

```bash
# Node status
kubectl get nodes -o wide

# Cluster info
kubectl cluster-info

# Version info
kubectl version

# Resource usage
kubectl top nodes
kubectl top pods -A
```

### Deployment Management

```bash
# Production namespace
kubectl get pods -n production
kubectl logs -n production deployment/backend -f

# Restart deployment
kubectl rollout restart -n production deployment/backend

# Scale deployment
kubectl scale -n production deployment/backend --replicas=3
```

### Monitoring

```bash
# Monitoring namespace status
kubectl get pods -n monitoring

# Grafana logs
kubectl logs -n monitoring deployment/grafana -f

# VictoriaMetrics query
kubectl port-forward -n monitoring victoria-metrics-victoria-metrics-single-server-0 8428:8428 &
curl 'http://localhost:8428/api/v1/query?query=up'
```

### Security

```bash
# Falco alerts
kubectl logs -n falco-system daemonset/falco -f

# Kyverno policies
kubectl get clusterpolicy
kubectl get policyreport -A

# Trivy scan (via Harbor)
# See Harbor UI: https://registry.vondi.rs/
```

### Backup & Restore

```bash
# Velero backup
kubectl get backups -n velero

# Create on-demand backup
velero backup create manual-backup-$(date +%Y%m%d)

# etcd snapshots (automatic every 12h)
ls -lh /var/lib/rancher/k3s/server/db/snapshots/
```

---

## Storage

### PersistentVolumeClaims

| PVC | Size | Namespace | Used By | StorageClass |
|-----|------|-----------|---------|--------------|
| harbor-jobservice | 10GB | harbor | harbor-jobservice | local-path |
| harbor-registry | 100GB | harbor | harbor-registry | local-path |
| harbor-database | 10GB | harbor | harbor-database | local-path |
| harbor-redis | 5GB | harbor | harbor-redis | local-path |
| victoria-metrics | 50GB | monitoring | victoria-metrics | local-path |
| loki | 20GB | monitoring | loki | local-path |

**Total PVC usage:** ~195 GB / 900 GB available

---

## Network

### Flannel CNI

**Backend:** VXLAN
**Pod CIDR:** 10.42.0.0/16
**Service CIDR:** 10.43.0.0/16

**Node Pod CIDRs:**
- vondi-rs: 10.42.0.0/24
- svetu-rs: 10.42.1.0/24
- terra-vondi-rs: 10.42.2.0/24

### Service Discovery

**DNS:** CoreDNS (cluster.local)

```bash
# Internal service resolution
backend.production.svc.cluster.local:3000
redis.production.svc.cluster.local:6379
grafana.monitoring.svc.cluster.local:80
```

---

## Maintenance

### Updates

```bash
# Update K3s (на каждой ноде)
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.33.6+k3s1 sh -

# Drain node before maintenance
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Uncordon node after maintenance
kubectl uncordon <node-name>
```

### Monitoring Health

```bash
# Check all component health
kubectl get pods -A | grep -v Running

# Check node status
kubectl get nodes

# Check etcd health
kubectl -n kube-system exec -it etcd-<node> -- etcdctl endpoint health
```

---

## Status & Roadmap

### Current Status: 95% Complete

**Phase 1: Pre-Production** ✅ 100%
- K3s cluster installed
- Traefik, cert-manager, Harbor deployed
- SSH hardening, fail2ban, UFW configured

**Phase 2: Security & Monitoring** ✅ 95%
- Monitoring: 100% (Grafana + VictoriaMetrics + Loki)
- Security: 90% (7-layer defense, AIDE pending)
- Backups: 100% (Velero, etcd, PostgreSQL)

**Phase 3: Application Migration** 🔄 20%
- ✅ Backend migrated (Week 3)
- 📋 Frontend planned (Week 4)
- 📋 Microservices planned (Weeks 5-6)

**Phase 4: Advanced Features** 📋 0%
- HPA (Horizontal Pod Autoscaler)
- Network Policies
- Service Mesh (Istio/Linkerd)
- GitOps (ArgoCD)
- Canary Deployments

### Next Steps

1. **This week:** Frontend migration to K8s
2. **Weeks 5-6:** Microservices migration (Auth, Listings, Delivery, WMS, Payment, Notifications)
3. **Month 2:** Advanced features (HPA, Network Policies, ArgoCD)

---

## Troubleshooting

### Common Issues

**Pod not starting:**
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
```

**Node not ready:**
```bash
kubectl describe node <node-name>
systemctl status k3s  # On the node
journalctl -u k3s -f  # On the node
```

**Ingress not working:**
```bash
kubectl get ingress -A
kubectl logs -n kube-system deployment/traefik -f
kubectl describe ingress <ingress-name> -n <namespace>
```

**Certificate issues:**
```bash
kubectl get certificate -A
kubectl describe certificate <cert-name> -n <namespace>
kubectl logs -n cert-manager deployment/cert-manager -f
```

---

## Documentation

### Related Documents

- [CLAUDE.md](/p/github.com/vondi-global/CLAUDE.md) - Grafana quick access
- [K3S_CURRENT_STATUS.md](/tmp/K3S_CURRENT_STATUS_2025-12-23.md) - Detailed status report
- [Grafana Credentials](/p/github.com/vondi-global/credentials/grafana-k3s.md) - Access credentials

### External Links

- K3s Documentation: https://docs.k3s.io/
- Traefik: https://doc.traefik.io/traefik/
- Grafana: https://grafana.com/docs/
- VictoriaMetrics: https://docs.victoriametrics.com/

---

**Последнее обновление:** 2025-12-23
**Автор:** DevOps Team (K3s migration Phase 2)
**Версия документа:** 1.0.0
