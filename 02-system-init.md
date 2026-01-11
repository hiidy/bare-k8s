# 클러스터 초기화 및 환경 설정

## 클러스터 구성

| 노드명 | IP |
| --------- | ------------- |
| k8s-01    | 172.31.39.151 |
| k8s-02    | 172.31.32.72  |

2대의 노드에 etcd, master, worker를 혼합 배포.

3노드 클러스터로 확장하려면:
- `environment.sh`의 `NODE_IPS`, `NODE_NAMES`, `ETCD_ENDPOINTS`, `ETCD_NODES` 변수에 새 노드 추가
- CSR 파일의 `hosts` 필드에 새 노드 IP 추가
- `for (( i=0; i < 2; i++ ))`에서 `2`를 `3`으로 변경
- 모든 노드에서 실행해야 하는 명령어는 새 노드에서도 실행

> 별도 명시가 없으면 초기화 작업은 모든 노드에서 실행해야 한다.

---

## hostname 설정

```bash
# 각 노드에서 실행
hostnamectl set-hostname k8s-01  # k8s-01 노드
hostnamectl set-hostname k8s-02  # k8s-02 노드
```

DNS가 hostname을 resolve하지 못하면 `/etc/hosts`에 추가:

```bash
cat >> /etc/hosts <<EOF
172.31.39.151 k8s-01
172.31.32.72 k8s-02
EOF
```

로그아웃 후 다시 로그인하면 hostname이 적용된다.

---

## 노드 간 SSH 신뢰 관계 설정

모든 노드에서 실행 아래 명령어를 실행

```bash
sed -i 's/PubkeyAuthentication no/PubkeyAuthentication yes/g' /etc/ssh/sshd_config
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
sed -i 's/^#\s*\(AuthorizedKeysFile\s\+.*\)/\1/' /etc/ssh/sshd_config
systemctl restart sshd
```

k8s-01 노드에서만 실행 (password 없이 모든 노드 접근 설정)

```bash
ssh-keygen -t rsa
ssh-copy-id root@k8s-01
ssh-copy-id root@k8s-02
```

설정 후 `ssh root@k8s-01`, `ssh root@k8s-02`, `ssh root@172.31.39.151`, `ssh root@172.31.32.72`로 접속 테스트해서 passwordless login이 되는지 확인하자. sshd 설정 문제로 실패할 수 있다.

---

## PATH 설정

```bash
echo 'export PATH=/opt/k8s/bin:$PATH' >> $HOME/.bashrc
source /root/.bashrc
```

`/opt/k8s/bin` 디렉토리에 설치 파일들을 저장한다.

---

## dependency 설치


```bash
apt install -y policycoreutils jq chrony conntrack ipvsadm ipset iptables curl sysstat wget socat git
```

- `ipvsadm`: kube-proxy ipvs 모드 관리 도구
- `chrony`: etcd 클러스터 시간 동기화용

---

## firewall 비활성화

firewall 비활성화하고, rule 초기화 후 default forwarding policy 설정

```bash
systemctl stop nftables
systemctl disable nftables
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat
iptables -P FORWARD ACCEPT
```

---

## swap 비활성화

swap이 켜져 있으면 kubelet이 시작에 실패한다. (`--fail-swap-on=false` 옵션으로 swap 체크를 비활성화할 수도 있다)

```bash
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

---

## SELinux 비활성화

SELinux가 켜져 있으면 kubelet이 디렉토리 mount 시 `Permission denied` 에러가 발생할 수 있다

```bash
setenforce 0
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

---

## kernel parameter 최적화

```bash
cat > kubernetes.conf <<EOF
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
net.ipv4.neigh.default.gc_thresh1=1024
net.ipv4.neigh.default.gc_thresh2=2048
net.ipv4.neigh.default.gc_thresh3=4096
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
EOF
cp kubernetes.conf /etc/sysctl.d/kubernetes.conf
sysctl -p /etc/sysctl.d/kubernetes.conf
```

- `tcp_tw_recycle=0`: NAT와 충돌해서 서비스 연결 문제가 생길 수 있으므로 꺼둔다.

---

## timezone 설정

```bash
timedatectl set-timezone Asia/Seoul
```

---

## 시간 동기화 설정

```bash
systemctl enable chrony
systemctl start chrony
```

동기화 상태 확인

```bash
timedatectl status
```

출력 예시

```
               Local time: Sun 2026-01-11 23:16:21 KST
           Universal time: Sun 2026-01-11 14:16:21 UTC
                 RTC time: Sun 2026-01-11 14:16:21
                Time zone: Asia/Seoul (KST, +0900)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

- `System clock synchronized: yes`: 시간 동기화 완료
- `NTP service: active`: 시간 동기화 서비스 활성화됨

UTC 시간을 hardware clock에 저장

```bash
# UTC 시간을 hardware clock에 저장
timedatectl set-local-rtc 0

# 시스템 시간에 의존하는 서비스 재시작
systemctl restart rsyslog 
systemctl restart chrony
```

---

## 불필요한 서비스 비활성화

활성화되어 있으면 비활성화, 아니면 무시

```bash
systemctl stop postfix && systemctl disable postfix
```

---

## 설치 디렉토리 생성

```bash
mkdir -p /opt/k8s/{bin,work} /etc/{kubernetes,etcd}/cert
```

---

## environment 스크립트 배포

`environment.sh`에 클러스터 설정값을 정의한다. 본인 노드 정보에 맞게 수정 후 모든 노드에 배포.

```bash
#!/usr/bin/bash

# EncryptionConfig용 encryption key 생성
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

# 클러스터 노드 IP array
export NODE_IPS=(172.31.39.151 172.31.32.72)

# 클러스터 노드 hostname array
export NODE_NAMES=(k8s-01 k8s-02)

# etcd 클러스터 service address list
export ETCD_ENDPOINTS="https://172.31.39.151:2379,https://172.31.32.72:2379"

# etcd 클러스터 간 통신 IP 및 port
export ETCD_NODES="k8s-01=https://172.31.39.151:2380,k8s-02=https://172.31.32.72:2380"

# kube-apiserver reverse proxy (kube-nginx) 주소
export KUBE_APISERVER="https://127.0.0.1:8443"

# 노드 간 통신 network interface
export IFACE="eth0"

# etcd data directory
export ETCD_DATA_DIR="/data/k8s/etcd/data"

# etcd WAL directory (SSD 권장, 또는 ETCD_DATA_DIR과 다른 disk partition)
export ETCD_WAL_DIR="/data/k8s/etcd/wal"

# k8s component data directory
export K8S_DIR="/data/k8s/k8s"

## DOCKER_DIR과 CONTAINERD_DIR 중 하나 선택
# docker data directory
export DOCKER_DIR="/data/k8s/docker"

# containerd data directory
export CONTAINERD_DIR="/data/k8s/containerd"

## 아래 parameter는 일반적으로 수정할 필요 없음

# TLS Bootstrapping용 token
# 생성 명령어: head -c 16 /dev/urandom | od -An -t x | tr -d ' '
BOOTSTRAP_TOKEN="41f7e4ba8b7be874fcff18bf5cf41a7c"

# 현재 사용하지 않는 network segment로 Service와 Pod 대역 정의

# Service CIDR (배포 전에는 routing 불가, 배포 후 kube-proxy가 클러스터 내 routing 보장)
SERVICE_CIDR="10.254.0.0/16"

# Pod CIDR (/16 권장, 배포 전에는 routing 불가, 배포 후 CNI가 클러스터 내 routing 보장)
CLUSTER_CIDR="172.30.0.0/16"

# Service port range (NodePort Range)
export NODE_PORT_RANGE="30000-32767"

# kubernetes service IP (보통 SERVICE_CIDR의 첫 번째 IP)
export CLUSTER_KUBERNETES_SVC_IP="10.254.0.1"

# Cluster DNS service IP (SERVICE_CIDR에서 미리 할당)
export CLUSTER_DNS_SVC_IP="10.254.0.2"

# Cluster DNS domain (trailing dot 없이)
export CLUSTER_DNS_DOMAIN="cluster.local"

# binary directory를 PATH에 추가
export PATH=/opt/k8s/bin:$PATH
```

`NODE_IPS`, `NODE_NAMES`, `ETCD_ENDPOINTS`, `ETCD_NODES`는 본인 노드 정보에 맞게 수정해야 한다.

스크립트 배포

```bash
source environment.sh  # 먼저 수정 후 실행
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    scp environment.sh root@${node_ip}:/opt/k8s/bin/
    ssh root@${node_ip} "chmod +x /opt/k8s/bin/*"
  done
```