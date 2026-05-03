# Ansible HA PostgreSQL Cluster

Triển khai cụm PostgreSQL High Availability sử dụng **Patroni + etcd + HAProxy** trên 3 nodes.

## Kiến trúc tổng thể

```
                     ┌─────────────────────────────────────┐
                     │          Control Node               │
                     │    (chạy ansible-playbook)          │
                     └──────────────┬──────────────────────┘
                                    │ SSH
               ┌────────────────────┼─────────────────────┐
               │                    │                     │
       ┌───────▼──────┐   ┌─────────▼────┐   ┌──────────▼─────┐
       │   node1       │   │   node2      │   │   node3        │
       │ 192.168.253.11│   │192.168.253.12│   │192.168.253.13  │
       ├───────────────┤   ├──────────────┤   ├────────────────┤
       │ etcd          │   │ etcd         │   │ etcd           │
       │ patroni       │   │ patroni      │   │ patroni        │
       │ postgresql    │   │ postgresql   │   │ postgresql     │
       │ HAProxy ◄─────┼───┼──────────────┼───┼── (LB)        │
       └───────────────┘   └──────────────┘   └────────────────┘
               │
               ▼
    ┌──────────────────────┐
    │   Client Application │
    │  Primary  → :5000    │
    │  Replicas → :5001    │
    │  Stats    → :7000    │
    └──────────────────────┘
```

## Cấu trúc project

```
ansible-ha-postgres/
├── ansible.cfg                     # Cấu hình Ansible
├── site.yml                        # Playbook chính
├── inventories/
│   └── hosts.ini                   # Danh sách hosts
├── group_vars/
│   └── all.yml                     # Biến toàn cục (mật khẩu, version, ports...)
├── roles/
│   ├── common/                     # Cấu hình OS cơ bản, NTP, sysctl
│   ├── etcd/                       # Cài đặt & cấu hình etcd cluster
│   ├── patroni/                    # Cài đặt PostgreSQL + Patroni
│   └── haproxy/                    # Cài đặt & cấu hình HAProxy
└── playbooks/
    ├── verify_cluster.yml          # Kiểm tra sức khỏe cluster
    ├── test_failover.yml           # Test failover thủ công
    └── reinit_replica.yml          # Reinit replica bị lag
```

## Yêu cầu

- Control node: Ansible >= 2.14
- Target nodes: Ubuntu 22.04 LTS
- SSH key từ control node đến cả 3 servers
- Python 3 trên tất cả nodes

## Cài đặt Ansible trên control node

```bash
sudo apt update && sudo apt install -y ansible python3-pip
pip3 install ansible
```

## Bước 1 — Cấu hình biến

Mở `group_vars/all.yml` và thay đổi các thông số quan trọng:

```yaml
# Mật khẩu PostgreSQL (BẮT BUỘC thay đổi trước khi deploy)
pg_superuser_password: "SuperSecurePassword123!"
pg_replication_password: "ReplicatorPassword123!"
pg_app_password: "AppPassword123!"

# HAProxy stats
haproxy_stats_password: "HaproxyAdmin123!"

# Phiên bản phần mềm (kiểm tra phiên bản mới nhất)
postgresql_version: "15"
patroni_version: "3.3.0"
etcd_version: "3.5.12"
```

## Bước 2 — Kiểm tra inventory

```bash
ansible-inventory -i inventories/hosts.ini --list
```

## Bước 3 — Test kết nối SSH

```bash
ansible postgres_ha -i inventories/hosts.ini -m ping
```

Kết quả mong đợi:
```
node1 | SUCCESS => { "ping": "pong" }
node2 | SUCCESS => { "ping": "pong" }
node3 | SUCCESS => { "ping": "pong" }
```

## Bước 4 — Deploy toàn bộ cluster

```bash
ansible-playbook site.yml
```

Hoặc deploy từng phần:

```bash
# Chỉ cài common
ansible-playbook site.yml --tags common

# Chỉ cài etcd
ansible-playbook site.yml --tags etcd

# Chỉ cài patroni/postgresql
ansible-playbook site.yml --tags patroni

# Chỉ cài haproxy
ansible-playbook site.yml --tags haproxy
```

## Bước 5 — Xác nhận cluster hoạt động

```bash
ansible-playbook playbooks/verify_cluster.yml
```

## Kết nối vào PostgreSQL

### Qua HAProxy (khuyến nghị)

```bash
# Kết nối primary (read-write)
psql -h 192.168.253.11 -p 5000 -U postgres -d postgres

# Kết nối replica (read-only)
psql -h 192.168.253.11 -p 5001 -U postgres -d postgres
```

### Từ application (connection string)

```
# Primary
postgresql://postgres:SuperSecurePassword123!@192.168.253.11:5000/appdb

# Replica
postgresql://postgres:SuperSecurePassword123!@192.168.253.11:5001/appdb
```

## Quản lý cluster

### Xem trạng thái cluster

```bash
# Trên bất kỳ node nào
patronictl -c /etc/patroni/patroni.yml list

# Output mẫu:
# + Cluster: postgres-ha ----+----+-----------+
# | Member | Host            | Role    | State   |
# +--------+-----------------+---------+---------+
# | node1  | 192.168.253.11  | Leader  | running |
# | node2  | 192.168.253.12  | Replica | running |
# | node3  | 192.168.253.13  | Replica | running |
```

### Thực hiện switchover (chủ động)

```bash
ansible-playbook playbooks/test_failover.yml
# Hoặc trực tiếp:
patronictl -c /etc/patroni/patroni.yml switchover postgres-ha --force
```

### Reload cấu hình Patroni (không restart)

```bash
patronictl -c /etc/patroni/patroni.yml reload postgres-ha
```

### Reinit replica bị lag

```bash
ansible-playbook playbooks/reinit_replica.yml -e target_node=node2
```

### Pause/Resume automatic failover

```bash
patronictl -c /etc/patroni/patroni.yml pause  postgres-ha
patronictl -c /etc/patroni/patroni.yml resume postgres-ha
```

## Kiểm tra etcd cluster

```bash
# Health check
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=http://192.168.253.11:2379,http://192.168.253.12:2379,http://192.168.253.13:2379

# Danh sách members
ETCDCTL_API=3 etcdctl member list \
  --endpoints=http://192.168.253.11:2379
```

## HAProxy Stats Dashboard

Mở trình duyệt: `http://192.168.253.11:7000/stats`
- Username: `admin`
- Password: (theo `haproxy_stats_password` trong group_vars)

## Ports sử dụng

| Port | Service | Mô tả |
|------|---------|-------|
| 5432 | PostgreSQL | Direct connection (bypass HAProxy) |
| 5000 | HAProxy | Primary (read-write) |
| 5001 | HAProxy | Replicas (read-only) |
| 7000 | HAProxy | Stats dashboard |
| 8008 | Patroni | REST API |
| 2379 | etcd | Client |
| 2380 | etcd | Peer |

## Logs

```bash
# Patroni
journalctl -u patroni -f
tail -f /var/log/patroni/*.log

# PostgreSQL
tail -f /var/log/postgresql/*.log

# etcd
journalctl -u etcd -f

# HAProxy
tail -f /var/log/haproxy/haproxy.log
```

## Troubleshooting

### Patroni không start

```bash
systemctl status patroni
journalctl -u patroni --no-pager -n 50
```

Kiểm tra etcd có accessible không:
```bash
ETCDCTL_API=3 etcdctl endpoint health --endpoints=http://192.168.253.11:2379
```

### Split-brain prevention

Patroni sử dụng etcd làm DCS (Distributed Configuration Store). Nếu một node mất kết nối etcd quá `ttl` (30s), nó tự động demote bản thân. Điều này đảm bảo luôn chỉ có **1 primary**.

### Replica bị lag quá nhiều

```bash
# Kiểm tra lag
psql -h 192.168.253.11 -p 5000 -U postgres -c \
  "SELECT client_addr, state, sent_lsn - write_lsn AS write_lag FROM pg_stat_replication;"

# Reinit nếu cần
patronictl -c /etc/patroni/patroni.yml reinit postgres-ha node2 --force
```
