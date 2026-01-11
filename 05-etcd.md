# etcd 클러스터 배포

etcd는 CoreOS에서 개발한 Raft 기반 분산 KV 저장소다. 서비스 디스커버리, 공유 설정, 동시성 제어(leader election, distributed lock 등)에 주로 사용된다. Kubernetes는 etcd 클러스터에 모든 API object와 runtime 데이터를 저장한다.

이 문서에서는 etcd 클러스터를 배포한다

- etcd binary 파일 다운로드 및 배포
- 각 etcd 클러스터 노드용 x509 인증서 생성 (client와 etcd 클러스터 간, etcd 노드 간 통신 암호화에 사용)
- etcd systemd unit 파일 생성 및 서비스 parameter 설정
- 클러스터 상태 확인

etcd 클러스터 노드 구성

| 노드명 | IP |
|--------|-----|
| k8s-01 | 172.31.39.151 |
| k8s-02 | 172.31.32.72 |

> **참고**
> 1. 별도 명시가 없으면 모든 작업은 k8s-01 노드에서 실행
> 2. Flanneld는 이 문서에서 설치하는 etcd v3.4.x와 호환되지 않는다. Flanneld를 설치하려면 (이 프로젝트에서는 Cilium 사용) etcd를 v3.3.x로 다운그레이드해야 한다.

---

## etcd binary 다운로드 및 배포

etcd release 페이지에서 최신 버전 다운로드

```bash
cd /opt/k8s/work
wget https://github.com/etcd-io/etcd/releases/download/v3.6.2/etcd-v3.6.2-linux-amd64.tar.gz
tar -xvf etcd-v3.6.2-linux-amd64.tar.gz
```

모든 클러스터 노드에 binary 파일 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp etcd-v3.6.2-linux-amd64/etcd* root@${node_ip}:/opt/k8s/bin
  ssh root@${node_ip} "chmod +x /opt/k8s/bin/*"
done
```

---

## etcd 인증서 및 private key 생성

CSR 파일 생성

```bash
cd /opt/k8s/work
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "172.31.39.151",
    "172.31.32.72"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "KR",
      "ST": "Seoul",
      "L": "Seoul",
      "O": "k8s",
      "OU": "bare-k8s"
    }
  ]
}
EOF
```

- **hosts**: 이 인증서 사용이 허가된 etcd 노드 IP 목록. 모든 etcd 클러스터 노드 IP를 나열해야 한다.

인증서 및 private key 생성

```bash
cd /opt/k8s/work
cfssl gencert -ca=/opt/k8s/work/ca.pem \
  -ca-key=/opt/k8s/work/ca-key.pem \
  -config=/opt/k8s/work/ca-config.json \
  -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
ls etcd*pem
```

각 etcd 노드에 인증서 및 private key 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "mkdir -p /etc/etcd/cert"
  scp etcd*.pem root@${node_ip}:/etc/etcd/cert/
done
```

---

## etcd systemd unit template 파일 생성

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > etcd.service.template <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=${ETCD_DATA_DIR}
ExecStart=/opt/k8s/bin/etcd \\
  --data-dir=${ETCD_DATA_DIR} \\
  --wal-dir=${ETCD_WAL_DIR} \\
  --name=##NODE_NAME## \\
  --cert-file=/etc/etcd/cert/etcd.pem \\
  --key-file=/etc/etcd/cert/etcd-key.pem \\
  --trusted-ca-file=/etc/kubernetes/cert/ca.pem \\
  --peer-cert-file=/etc/etcd/cert/etcd.pem \\
  --peer-key-file=/etc/etcd/cert/etcd-key.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/cert/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --listen-peer-urls=https://##NODE_IP##:2380 \\
  --initial-advertise-peer-urls=https://##NODE_IP##:2380 \\
  --listen-client-urls=https://##NODE_IP##:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls=https://##NODE_IP##:2379 \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=${ETCD_NODES} \\
  --initial-cluster-state=new \\
  --auto-compaction-mode=periodic \\
  --auto-compaction-retention=1 \\
  --max-request-bytes=33554432 \\
  --quota-backend-bytes=6442450944 \\
  --heartbeat-interval=250 \\
  --election-timeout=2000
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

- **WorkingDirectory, --data-dir**: 작업 디렉토리와 데이터 디렉토리를 `${ETCD_DATA_DIR}`로 지정. 서비스 시작 전에 이 디렉토리를 생성해야 한다.
- **--wal-dir**: WAL 디렉토리 지정. 성능 향상을 위해 SSD나 `--data-dir`과 다른 disk 사용 권장
- **--name**: 노드 이름 지정. `--initial-cluster-state` 값이 `new`일 때 `--name` 값은 반드시 `--initial-cluster` 목록에 있어야 한다.
- **--cert-file, --key-file**: etcd server와 client 간 통신에 사용하는 인증서 및 private key
- **--trusted-ca-file**: client 인증서를 서명한 CA 인증서. client 인증서 검증에 사용
- **--peer-cert-file, --peer-key-file**: etcd와 peer 간 통신에 사용하는 인증서 및 private key
- **--peer-trusted-ca-file**: peer 인증서를 서명한 CA 인증서. peer 인증서 검증에 사용

---

## 각 노드용 etcd systemd unit 파일 생성 및 배포

template 파일의 변수를 치환해서 각 노드용 systemd unit 파일 생성

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for (( i=0; i < 2; i++ ))
do
  sed -e "s/##NODE_NAME##/${NODE_NAMES[i]}/" -e "s/##NODE_IP##/${NODE_IPS[i]}/" etcd.service.template > etcd-${NODE_IPS[i]}.service
done
ls *.service
```

- NODE_NAMES와 NODE_IPS는 같은 길이의 bash array로, 각각 노드 이름과 해당 IP를 나타낸다.
- etcd를 2개 노드에 배포하므로 for loop는 2번 반복

생성된 systemd unit 파일 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp etcd-${node_ip}.service root@${node_ip}:/etc/systemd/system/etcd.service
done
```

---

## etcd 서비스 시작

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "mkdir -p ${ETCD_DATA_DIR} ${ETCD_WAL_DIR}"
  ssh root@${node_ip} "systemctl daemon-reload && systemctl enable etcd && systemctl restart etcd " &
done
```

- etcd 데이터 디렉토리와 작업 디렉토리를 먼저 생성해야 한다.
- etcd 프로세스가 처음 시작될 때 다른 노드의 etcd가 클러스터에 join하기를 기다린다. `systemctl start etcd` 명령이 잠시 hang되는데, 정상 동작이다.

---

## etcd 시작 결과 확인

```bash
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "systemctl status etcd|grep Active"
done
```

상태가 `active (running)`인지 확인. 아니라면 로그를 확인한다:

```bash
journalctl -u etcd
```

---

## 서비스 상태 검증

etcd 클러스터 배포 후, 아무 etcd 노드에서 다음 명령 실행

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  /opt/k8s/bin/etcdctl \
    --endpoints=https://${node_ip}:2379 \
    --cacert=/etc/kubernetes/cert/ca.pem \
    --cert=/etc/etcd/cert/etcd.pem \
    --key=/etc/etcd/cert/etcd-key.pem endpoint health
done
```

- etcd/etcdctl v3.5.16부터 V3 API가 기본 활성화되어 있어서 etcdctl 명령 실행 시 `ETCDCTL_API=3` 환경 변수를 지정할 필요 없다.
- Kubernetes 1.13부터 etcd v2 버전은 더 이상 지원되지 않는다.

예상 출력:

```
>>> 172.31.39.151
https://172.31.39.151:2379 is healthy: successfully committed proposal: took = 2.192932ms
>>> 172.31.32.72
https://172.31.32.72:2379 is healthy: successfully committed proposal: took = 2.034181ms
```

모든 노드가 `healthy`로 출력되면 클러스터 서비스가 정상이다.

---

## 현재 leader 확인

```bash
source /opt/k8s/bin/environment.sh
/opt/k8s/bin/etcdctl \
  -w table --cacert=/etc/kubernetes/cert/ca.pem \
  --cert=/etc/etcd/cert/etcd.pem \
  --key=/etc/etcd/cert/etcd-key.pem \
  --endpoints=${ETCD_ENDPOINTS} endpoint status
```

출력 예시:

```
+----------------------------+------------------+---------+-----------------+---------+--------+-----------------------+--------+-----------+------------+-----------+------------+--------------------+--------+--------------------------+-------------------+
|          ENDPOINT          |        ID        | VERSION | STORAGE VERSION | DB SIZE | IN USE | PERCENTAGE NOT IN USE | QUOTA  | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS | DOWNGRADE TARGET VERSION | DOWNGRADE ENABLED |
+----------------------------+------------------+---------+-----------------+---------+--------+-----------------------+--------+-----------+------------+-----------+------------+--------------------+--------+--------------------------+-------------------+
| https://172.31.39.151:2379 | fbaa86d83a229b6d |   3.6.2 |           3.6.0 |  8.6 MB | 5.8 MB |                   33% | 6.4 GB |     false |      false |         5 |     525030 |             525030 |        |                          |             false |
|  https://172.31.32.72:2379 | 5fbde31cc09a9bac |   3.6.2 |           3.6.0 |  9.4 MB | 5.8 MB |                   38% | 6.4 GB |      true |      false |         5 |     525030 |             525030 |        |                          |             false |
+----------------------------+------------------+---------+-----------------+---------+--------+-----------------------+--------+-----------+------------+-----------+------------+--------------------+--------+--------------------------+-------------------+
```

위 결과에서 현재 leader는 172.31.32.72다.