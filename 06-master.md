
# Master 노드 배포

Kubernetes master 노드는 다음 컴포넌트를 실행한다
- kube-apiserver
- kube-scheduler
- kube-controller-manager

kube-apiserver, kube-scheduler, kube-controller-manager 모두 multi-instance 모드로 실행
1. kube-scheduler와 kube-controller-manager는 자동으로 leader instance를 선출하고, 다른 instance는 blocking 상태가 된다. leader가 실패하면 새로운 leader를 선출해 서비스 가용성을 보장한다.
2. kube-apiserver는 stateless이므로 kube-nginx proxy를 통해 접근해서 서비스 가용성을 보장한다.

> 별도 명시가 없으면 모든 작업은 k8s-01 노드에서 실행한다.

---

## binary 파일 다운로드

CHANGELOG-1.33 페이지에서 binary tar 파일을 다운로드하고 압축 해제

```bash
cd /opt/k8s/work
wget https://dl.k8s.io/v1.33.3/kubernetes-server-linux-amd64.tar.gz
tar -xzvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes
tar -xzvf kubernetes-src.tar.gz
```

모든 master 노드에 binary 파일 복사:

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp kubernetes/server/bin/{apiextensions-apiserver,kube-apiserver,kube-controller-manager,kube-scheduler,kube-proxy,kubelet} root@${node_ip}:/opt/k8s/bin/
  ssh root@${node_ip} "chmod +x /opt/k8s/bin/*"
done
```

---

## kube-apiserver 배포

2 instance kube-apiserver 클러스터를 배포한다.

### kubernetes-master 인증서 및 private key 생성

CSR 파일 생성

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes-master",
  "hosts": [
    "127.0.0.1",
    "172.31.39.151",
    "172.31.32.72",
    "${CLUSTER_KUBERNETES_SVC_IP}",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local.",
    "kubernetes.default.svc.${CLUSTER_DNS_DOMAIN}."
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

- **hosts field**: 이 인증서 사용이 허가된 IP 및 domain name 목록. master 노드 IP, kubernetes service IP 및 domain name을 나열한다.

인증서 및 private key 생성

```bash
cd /opt/k8s/work
cfssl gencert -ca=/opt/k8s/work/ca.pem \
  -ca-key=/opt/k8s/work/ca-key.pem \
  -config=/opt/k8s/work/ca-config.json \
  -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
ls kubernetes*pem
```

모든 master 노드에 인증서 및 private key 복사

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "mkdir -p /etc/kubernetes/cert"
  scp kubernetes*.pem root@${node_ip}:/etc/kubernetes/cert/
done
```

### encryption config 파일 생성

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

master 노드의 `/etc/kubernetes` 디렉토리에 encryption config 파일 복사

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp encryption-config.yaml root@${node_ip}:/etc/kubernetes/
done
```

### audit policy 파일 생성

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > audit-policy.yaml <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
EOF
```

audit policy 파일 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp audit-policy.yaml root@${node_ip}:/etc/kubernetes/audit-policy.yaml
done
```

### metrics-server 또는 kube-prometheus 접근용 인증서 생성

CSR 파일 생성

```bash
cd /opt/k8s/work
cat > proxy-client-csr.json <<EOF
{
  "CN": "aggregator",
  "hosts": [],
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

- CN name은 kube-apiserver의 `--requestheader-allowed-names` parameter에 있어야 한다. 그렇지 않으면 나중에 metrics 접근 시 권한 부족 에러가 발생한다.

인증서 및 private key 생성

```bash
cd /opt/k8s/work
cfssl gencert -ca=/opt/k8s/work/ca.pem \
  -ca-key=/opt/k8s/work/ca-key.pem \
  -config=/opt/k8s/work/ca-config.json \
  -profile=kubernetes proxy-client-csr.json | cfssljson -bare proxy-client
ls proxy-client*.pem
```

모든 master 노드에 인증서 및 private key 복사

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp proxy-client*.pem root@${node_ip}:/etc/kubernetes/cert/
done
```

### /etc/kubernetes/pki/sa.key 생성

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
openssl genrsa -out sa.key 2048
openssl rsa -in sa.key -pubout -out sa.pub
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "mkdir -p /etc/kubernetes/pki/"
  scp sa.key sa.pub root@${node_ip}:/etc/kubernetes/pki/
done
```

### kube-apiserver systemd unit template 파일 생성

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > kube-apiserver.service.template <<EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=${K8S_DIR}/kube-apiserver
ExecStart=/opt/k8s/bin/kube-apiserver \\
  --advertise-address=##NODE_IP## \\
  --default-not-ready-toleration-seconds=360 \\
  --default-unreachable-toleration-seconds=360 \\
  --max-mutating-requests-inflight=2000 \\
  --max-requests-inflight=4000 \\
  --delete-collection-workers=2 \\
  --encryption-provider-config=/etc/kubernetes/encryption-config.yaml \\
  --etcd-cafile=/etc/kubernetes/cert/ca.pem \\
  --etcd-certfile=/etc/etcd/cert/etcd.pem \\
  --etcd-keyfile=/etc/etcd/cert/etcd-key.pem \\
  --etcd-servers=${ETCD_ENDPOINTS} \\
  --bind-address=##NODE_IP## \\
  --secure-port=6443 \\
  --tls-cert-file=/etc/kubernetes/cert/kubernetes.pem \\
  --tls-private-key-file=/etc/kubernetes/cert/kubernetes-key.pem \\
  --audit-log-maxage=15 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-truncate-enabled \\
  --audit-log-path=${K8S_DIR}/kube-apiserver/audit.log \\
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml \\
  --profiling \\
  --anonymous-auth=false \\
  --client-ca-file=/etc/kubernetes/cert/ca.pem \\
  --enable-bootstrap-token-auth \\
  --requestheader-allowed-names="aggregator" \\
  --requestheader-client-ca-file=/etc/kubernetes/cert/ca.pem \\
  --requestheader-extra-headers-prefix="X-Remote-Extra-" \\
  --requestheader-group-headers=X-Remote-Group \\
  --requestheader-username-headers=X-Remote-User \\
  --service-account-key-file=/etc/kubernetes/pki/sa.pub \\
  --service-account-signing-key-file=/etc/kubernetes/pki/sa.key \\
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \\
  --authorization-mode=Node,RBAC \\
  --runtime-config=api/all=true \\
  --enable-admission-plugins=NodeRestriction \\
  --allow-privileged=true \\
  --apiserver-count=2 \\
  --event-ttl=168h \\
  --kubelet-certificate-authority=/etc/kubernetes/cert/ca.pem \\
  --kubelet-client-certificate=/etc/kubernetes/cert/kubernetes.pem \\
  --kubelet-client-key=/etc/kubernetes/cert/kubernetes-key.pem \\
  --proxy-client-cert-file=/etc/kubernetes/cert/proxy-client.pem \\
  --proxy-client-key-file=/etc/kubernetes/cert/proxy-client-key.pem \\
  --service-cluster-ip-range=${SERVICE_CIDR} \\
  --service-node-port-range=${NODE_PORT_RANGE} \\
  --logtostderr=true \\
  --v=2
Restart=on-failure
RestartSec=10
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

주요 parameter 설명

- **--advertise-address**: apiserver가 외부에 알리는 IP (kubernetes service backend 노드 IP)
- **--default-*-toleration-seconds**: 노드 이상 관련 threshold 설정
- **--max-*-requests-inflight**: 요청 관련 최대 threshold
- **--etcd-***: etcd 접근용 인증서 및 etcd server 주소
- **--bind-address**: https가 listen하는 IP. 127.0.0.1일 수 없다. 그렇지 않으면 외부에서 secure port 6443에 접근 불가
- **--secure-port**: https listening port
- **--insecure-port=0**: http insecure port(8080) listening 비활성화
- **--tls-*-file**: apiserver가 사용하는 인증서, private key, CA 파일
- **--audit-***: audit policy 및 audit log 파일 관련 parameter
- **--client-ca-file**: client(kube-controller-manager, kube-scheduler, kubelet, kube-proxy 등) 요청에 포함된 인증서 검증
- **--enable-bootstrap-token-auth**: kubelet bootstrap token 인증 활성화
- **--requestheader-***: kube-apiserver aggregator layer 관련 설정. proxy-client & HPA에 필요
- **--requestheader-client-ca-file**: `--proxy-client-cert-file`과 `--proxy-client-key-file`로 지정한 인증서를 서명하는 데 사용. metric aggregator 활성화 시 필요
- **--requestheader-allowed-names**: 비어 있으면 안 된다. `--proxy-client-cert-file` 인증서의 CN name을 comma로 구분해서 지정. 여기서는 "aggregator"로 설정
- **--service-account-key-file**: ServiceAccount Token 서명용 public key 파일. kube-controller-manager의 `--service-account-private-key-file`로 지정한 private key 파일과 쌍을 이룬다
- **--runtime-config=api/all=true**: 모든 버전의 API 활성화 (예: autoscaling/v2alpha1)
- **--authorization-mode=Node,RBAC, --anonymous-auth=false**: Node와 RBAC authorization mode 활성화, 미인가 요청 거부
- **--enable-admission-plugins**: 기본적으로 비활성화된 일부 plugin 활성화
- **--allow-privileged**: privileged container 실행 허용
- **--apiserver-count=2**: apiserver instance 수 지정
- **--event-ttl**: event 보존 기간
- **--kubelet-***: 지정하면 https로 kubelet API에 접근. 인증서에 해당하는 user에 대한 RBAC rule을 정의해야 한다 (위의 kubernetes*.pem 인증서의 user는 kubernetes). 그렇지 않으면 kubelet API 접근 시 unauthorized 에러 발생
- **--proxy-client-***: apiserver가 metrics-server에 접근할 때 사용하는 인증서
- **--service-cluster-ip-range**: Service Cluster IP 주소 범위
- **--service-node-port-range**: NodePort port 범위

kube-apiserver 머신에서 kube-proxy가 실행되지 않는다면 `--enable-aggregator-routing=true` parameter도 추가해야 한다.

`--requestheader-XXX` 관련 parameter 참고:
- https://github.com/kubernetes-incubator/apiserver-builder/blob/master/docs/concepts/auth.md
- https://docs.bitnami.com/kubernetes/how-to/configure-autoscaling-custom-metrics/

> **주의:**
>
> 1. `--requestheader-client-ca-file`로 지정한 CA 인증서는 client auth와 server auth가 모두 있어야 한다.
> 2. `--requestheader-allowed-names`가 비어 있지 않은데 `--proxy-client-cert-file` 인증서의 CN name이 allowed-names에 없으면, 나중에 node나 pods metrics 조회 시 다음과 같은 에러가 발생한다:
>
> ```
> Error from server (Forbidden): nodes.metrics.k8s.io is forbidden: User "aggregator" cannot list resource "nodes" in API group "metrics.k8s.io" at the cluster scope
> ```

### 각 노드용 kube-apiserver systemd unit 파일 생성 및 배포

template 파일의 변수를 치환해서 각 노드용 systemd unit 파일 생성


```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for (( i=0; i < 2; i++ ))
do
  sed -e "s/##NODE_NAME##/${NODE_NAMES[i]}/" -e "s/##NODE_IP##/${NODE_IPS[i]}/" kube-apiserver.service.template > kube-apiserver-${NODE_IPS[i]}.service
done
ls kube-apiserver*.service
```

- NODE_NAMES와 NODE_IPS는 같은 길이의 bash array로, 각각 노드 이름과 해당 IP를 나타낸다.

생성된 systemd unit 파일 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp kube-apiserver-${node_ip}.service root@${node_ip}:/etc/systemd/system/kube-apiserver.service
done
```

### kube-apiserver 서비스 시작

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "mkdir -p ${K8S_DIR}/kube-apiserver"
  ssh root@${node_ip} "systemctl daemon-reload && systemctl enable kube-apiserver && systemctl restart kube-apiserver"
done
```

### kube-apiserver 실행 상태 확인

```bash
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "systemctl status kube-apiserver |grep 'Active:'"
done
```

상태가 `active (running)`인지 확인. 아니라면 로그를 확인한다

```bash
journalctl -u kube-apiserver
```

### 클러스터 상태 확인

```bash
$ kubectl cluster-info
Kubernetes control plane is running at https://172.31.39.151:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl get all --all-namespaces
NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.254.0.1   <none>        443/TCP   2m1s

$ kubectl get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                        ERROR
scheduler            Unhealthy   Get "https://127.0.0.1:10259/healthz": dial tcp 127.0.0.1:10259: connect: connection refused
controller-manager   Unhealthy   Get "https://127.0.0.1:10257/healthz": dial tcp 127.0.0.1:10257: connect: connection refused
etcd-0               Healthy     ok
etcd-1               Healthy     ok
```

kube-scheduler와 kube-controller-manager가 아직 설치되지 않아서 이 2개 컴포넌트의 상태가 Unhealthy로 나오는 건 정상이다.

### kube-apiserver listening port 확인

```bash
$ sudo netstat -lnpt|grep kube
tcp        0      0 172.31.39.151:6443      0.0.0.0:*               LISTEN      414344/kube-apiserv
```

- **6443**: https 요청을 받는 secure port. 모든 요청을 인증하고 인가한다.
- insecure port가 비활성화되어 있어서 8080 port는 listen하지 않는다.

---

## kube-controller-manager 배포

고가용성 kube-controller-manager 클러스터를 배포한다.

이 클러스터는 2개 노드로 구성된다. 시작 후 경쟁 선출을 통해 leader 노드가 생성되고, 다른 노드는 blocking 상태가 된다. leader 노드를 사용할 수 없게 되면 blocking된 노드가 다시 선출을 진행해 새로운 leader 노드를 생성하여 서비스 가용성을 보장한다.

보안 통신을 위해 먼저 x509 인증서와 private key를 생성한다. kube-controller-manager는 다음 두 가지 상황에서 이 인증서를 사용한다:

1. kube-apiserver의 secure port와 통신
2. secure port(https, 10257)에서 prometheus format metrics 출력

### kube-controller-manager 인증서 및 private key 생성

CSR 파일 생성

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
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
      "O": "system:kube-controller-manager",
      "OU": "bare-k8s"
    }
  ]
}
EOF
```

- **hosts list**: 모든 kube-controller-manager 노드 IP 포함
- **CN과 O**: 둘 다 `system:kube-controller-manager`. Kubernetes 기본 제공 ClusterRoleBindings `system:kube-controller-manager`가 kube-controller-manager에 필요한 권한을 부여한다.

인증서 및 private key 생성

```bash
cd /opt/k8s/work
cfssl gencert -ca=/opt/k8s/work/ca.pem \
  -ca-key=/opt/k8s/work/ca-key.pem \
  -config=/opt/k8s/work/ca-config.json \
  -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
ls kube-controller-manager*pem
```

모든 master 노드에 인증서 및 private key 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp kube-controller-manager*.pem root@${node_ip}:/etc/kubernetes/cert/
done
```

### kubeconfig 파일 생성 및 배포

kube-controller-manager는 kubeconfig 파일로 apiserver에 접근한다. 이 파일에는 apiserver 주소, embed된 CA 인증서, kube-controller-manager 인증서가 포함된다:

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/k8s/work/ca.pem \
  --embed-certs=true \
  --server="https://##NODE_IP##:6443" \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.pem \
  --client-key=kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context system:kube-controller-manager \
  --cluster=kubernetes \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
```

- kube-controller-manager는 kube-apiserver와 같은 노드에 있으므로 노드 IP로 직접 kube-apiserver에 접근한다.

모든 master 노드에 kubeconfig 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for (( i=0; i < 2; i++ ))
do
  echo ">>> ${NODE_IPS[i]}"
  sed -e "s/##NODE_IP##/${NODE_IPS[i]}/" kube-controller-manager.kubeconfig > kube-controller-manager-${NODE_IPS[i]}.kubeconfig
  scp kube-controller-manager-${NODE_IPS[i]}.kubeconfig root@${NODE_IPS[i]}:/etc/kubernetes/kube-controller-manager.kubeconfig
done
```

### kube-controller-manager systemd unit template 파일 생성

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > kube-controller-manager.service.template <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
WorkingDirectory=${K8S_DIR}/kube-controller-manager
ExecStart=/opt/k8s/bin/kube-controller-manager \\
  --profiling \\
  --cluster-name=kubernetes \\
  --controllers=*,bootstrapsigner,tokencleaner \\
  --kube-api-qps=1000 \\
  --kube-api-burst=2000 \\
  --leader-elect \\
  --use-service-account-credentials=true \\
  --concurrent-service-syncs=2 \\
  --bind-address=##NODE_IP## \\
  --secure-port=10257 \\
  --tls-cert-file=/etc/kubernetes/cert/kube-controller-manager.pem \\
  --tls-private-key-file=/etc/kubernetes/cert/kube-controller-manager-key.pem \\
  --authentication-kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \\
  --client-ca-file=/etc/kubernetes/cert/ca.pem \\
  --requestheader-allowed-names="aggregator" \\
  --requestheader-client-ca-file=/etc/kubernetes/cert/ca.pem \\
  --requestheader-extra-headers-prefix="X-Remote-Extra-" \\
  --requestheader-group-headers=X-Remote-Group \\
  --requestheader-username-headers=X-Remote-User \\
  --authorization-kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \\
  --cluster-signing-cert-file=/etc/kubernetes/cert/ca.pem \\
  --cluster-signing-key-file=/etc/kubernetes/cert/ca-key.pem \\
  --root-ca-file=/etc/kubernetes/cert/ca.pem \\
  --service-account-private-key-file=/etc/kubernetes/pki/sa.key \\
  --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \\
  --logtostderr=true \\
  --service-cluster-ip-range=${SERVICE_CIDR} \\
  --cluster-cidr=${CLUSTER_CIDR} \\
  --allocate-node-cidrs=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

주요 parameter 설명

- **--secure-port=10257, --bind-address=##NODE_IP##**: 모든 network interface의 port 10257에서 https /metrics 요청을 listen
- **--kubeconfig**: kubeconfig 파일 경로. kube-controller-manager가 kube-apiserver에 연결하고 검증하는 데 사용
- **--authentication-kubeconfig, --authorization-kubeconfig**: kube-controller-manager가 client 요청을 인증하고 인가하기 위해 apiserver에 연결하는 데 사용. kube-controller-manager는 더 이상 `--tls-ca-file`로 https metrics를 요청하는 client 인증서를 검증하지 않는다. 이 두 kubeconfig parameter를 설정하지 않으면 kube-controller-manager https port에 연결하는 client 요청이 거부된다 (권한 부족)
- **--cluster-signing-*-file**: TLS Bootstrap으로 생성된 인증서 서명
- **--root-ca-file**: container ServiceAccount에 배치되는 CA 인증서. kube-apiserver 인증서 검증에 사용
- **--service-account-private-key-file**: ServiceAccount의 token 서명용 private key 파일. kube-apiserver의 `--service-account-key-file`로 지정한 public key 파일과 쌍을 이뤄야 한다
- **--service-cluster-ip-range**: Service Cluster IP 범위. kube-apiserver의 동일 이름 parameter와 같아야 한다
- **--leader-elect=true**: 클러스터 실행 모드. 선출 기능 활성화. leader로 선출된 노드가 작업을 처리하고, 다른 노드는 blocking 상태
- **--controllers=*,bootstrapsigner,tokencleaner**: 활성화할 controller 목록. tokencleaner는 만료된 Bootstrap token을 자동으로 정리
- **--horizontal-pod-autoscaler-***: custom metrics 관련 parameter. autoscaling/v2alpha1 지원
- **--tls-cert-file, --tls-private-key-file**: https로 metrics 출력 시 사용하는 server 인증서 및 key
- **--use-service-account-credentials=true**: kube-controller-manager의 각 controller가 serviceaccount로 kube-apiserver에 접근

### 각 노드용 kube-controller-manager systemd unit 파일 생성 및 배포

template 파일의 변수를 치환해서 각 노드용 systemd unit 파일 생성

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for (( i=0; i < 2; i++ ))
do
  sed -e "s/##NODE_NAME##/${NODE_NAMES[i]}/" -e "s/##NODE_IP##/${NODE_IPS[i]}/" kube-controller-manager.service.template > kube-controller-manager-${NODE_IPS[i]}.service
done
ls kube-controller-manager*.service
```

모든 master 노드에 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp kube-controller-manager-${node_ip}.service root@${node_ip}:/etc/systemd/system/kube-controller-manager.service
done
```

### kube-controller-manager 서비스 시작

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "mkdir -p ${K8S_DIR}/kube-controller-manager"
  ssh root@${node_ip} "systemctl daemon-reload && systemctl enable kube-controller-manager && systemctl restart kube-controller-manager"
done
```

### 서비스 실행 상태 확인

```bash
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "systemctl status kube-controller-manager|grep Active"
done
```

상태가 `active (running)`인지 확인. 아니라면 로그를 확인한다

```bash
journalctl -u kube-controller-manager
```

kube-controller-manager는 port 10257에서 https 요청을 listen한다:

```bash
$ sudo netstat -lnpt | grep kube-cont
tcp        0      0 172.31.39.151:10257     0.0.0.0:*               LISTEN      423121/kube-control
```

### metrics 출력 확인

> kube-controller-manager 노드에서 실행한다.

```bash
curl -s --cacert /opt/k8s/work/ca.pem --cert /opt/k8s/work/admin.pem --key /opt/k8s/work/admin-key.pem https://172.31.39.151:10257/metrics | head
```

### 현재 leader 확인

```bash
$ kubectl -n kube-system get leases kube-controller-manager
NAME                      HOLDER                                         AGE
kube-controller-manager   k8s-01_850f4a9c-0c06-4f58-8a5f-d70f7ace5dff   44s
```

현재 Leader는 k8s-01 노드에서 실행 중인 kube-controller-manager instance다.

### kube-controller-manager 클러스터 고가용성 테스트

한 노드의 kube-controller-manager 서비스를 중지하고 다른 노드의 로그를 관찰해서 leader 권한을 획득하는지 확인한다.

k8s-01 노드의 kube-controller-manager 서비스를 중지하고 현재 Leader를 확인

```bash
$ sudo systemctl stop kube-controller-manager.service

$ kubectl -n kube-system get leases kube-controller-manager
NAME                      HOLDER                                         AGE
kube-controller-manager   k8s-02_18e9f0c0-db65-4bb6-abc5-c89f03fc3429   99s
```

kube-controller-manager Leader가 k8s-02 노드의 kube-controller-manager instance로 전환되었다.

### References

1. Controller 권한 및 use-service-account-credentials parameter: https://github.com/kubernetes/kubernetes/issues/48208
2. kubelet authentication and authorization: https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/

---

## kube-scheduler 배포

고가용성 kube-scheduler 클러스터를 배포한다.

이 클러스터는 2개 노드로 구성된다. 시작 후 경쟁 선출을 통해 leader 노드가 생성되고, 다른 노드는 blocking 상태가 된다. leader 노드를 사용할 수 없게 되면 나머지 노드가 다시 선출을 진행해 새로운 leader 노드를 생성하여 서비스 가용성을 보장한다.

보안 통신을 위해 먼저 x509 인증서와 private key를 생성한다. kube-scheduler는 다음 두 가지 상황에서 이 인증서를 사용한다

1. kube-apiserver의 secure port와 통신
2. secure port(https, 10259)에서 prometheus format metrics 출력

### kube-scheduler 인증서 및 private key 생성

CSR 파일 생성

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
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
      "O": "system:kube-scheduler",
      "OU": "bare-k8s"
    }
  ]
}
EOF
```

- **hosts list**: 모든 kube-scheduler 노드 IP 포함
- **CN과 O**: 둘 다 `system:kube-scheduler`. Kubernetes 기본 제공 ClusterRoleBindings `system:kube-scheduler`가 kube-scheduler에 필요한 권한을 부여한다.

인증서 및 private key 생성:

```bash
cd /opt/k8s/work
cfssl gencert -ca=/opt/k8s/work/ca.pem \
  -ca-key=/opt/k8s/work/ca-key.pem \
  -config=/opt/k8s/work/ca-config.json \
  -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
ls kube-scheduler*pem
```

모든 master 노드에 인증서 및 private key 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp kube-scheduler*.pem root@${node_ip}:/etc/kubernetes/cert/
done
```

### kubeconfig 파일 생성 및 배포

kube-scheduler는 kubeconfig 파일로 apiserver에 접근한다. 이 파일에는 apiserver 주소, embed된 CA 인증서, kube-scheduler 인증서가 포함된다

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/k8s/work/ca.pem \
  --embed-certs=true \
  --server="https://##NODE_IP##:6443" \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.pem \
  --client-key=kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context system:kube-scheduler \
  --cluster=kubernetes \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
```

모든 master 노드에 kubeconfig 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for (( i=0; i < 2; i++ ))
do
  echo ">>> ${NODE_IPS[i]}"
  sed -e "s/##NODE_IP##/${NODE_IPS[i]}/" kube-scheduler.kubeconfig > kube-scheduler-${NODE_IPS[i]}.kubeconfig
  scp kube-scheduler-${NODE_IPS[i]}.kubeconfig root@${NODE_IPS[i]}:/etc/kubernetes/kube-scheduler.kubeconfig
done
```

### kube-scheduler config 파일 생성

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > kube-scheduler.yaml.template <<EOF
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  burst: 200
  kubeconfig: "/etc/kubernetes/kube-scheduler.kubeconfig"
  qps: 100
enableContentionProfiling: false
enableProfiling: true
leaderElection:
  leaderElect: true
EOF
```

- **--kubeconfig**: kubeconfig 파일 경로. kube-scheduler가 kube-apiserver에 연결하고 검증하는 데 사용
- **--leader-elect=true**: 클러스터 실행 모드. 선출 기능 활성화. leader로 선출된 노드가 작업을 처리하고, 다른 노드는 blocking 상태

template 파일의 변수 치환

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for (( i=0; i < 2; i++ ))
do
  sed -e "s/##NODE_NAME##/${NODE_NAMES[i]}/" -e "s/##NODE_IP##/${NODE_IPS[i]}/" kube-scheduler.yaml.template > kube-scheduler-${NODE_IPS[i]}.yaml
done
ls kube-scheduler*.yaml
```

- NODE_NAMES와 NODE_IPS는 같은 길이의 bash array로, 각각 노드 이름과 해당 IP를 나타낸다.

모든 master 노드에 kube-scheduler config 파일 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp kube-scheduler-${node_ip}.yaml root@${node_ip}:/etc/kubernetes/kube-scheduler.yaml
done
```

- kube-scheduler.yaml로 rename됨

### kube-scheduler systemd unit template 파일 생성

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > kube-scheduler.service.template <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
WorkingDirectory=${K8S_DIR}/kube-scheduler
ExecStart=/opt/k8s/bin/kube-scheduler \\
  --config=/etc/kubernetes/kube-scheduler.yaml \\
  --bind-address=##NODE_IP## \\
  --secure-port=10259 \\
  --tls-cert-file=/etc/kubernetes/cert/kube-scheduler.pem \\
  --tls-private-key-file=/etc/kubernetes/cert/kube-scheduler-key.pem \\
  --authentication-kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \\
  --client-ca-file=/etc/kubernetes/cert/ca.pem \\
  --requestheader-allowed-names="aggregator" \\
  --requestheader-client-ca-file=/etc/kubernetes/cert/ca.pem \\
  --requestheader-extra-headers-prefix="X-Remote-Extra-" \\
  --requestheader-group-headers=X-Remote-Group \\
  --requestheader-username-headers=X-Remote-User \\
  --authorization-kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \\
  --v=2
Restart=always
RestartSec=5
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
EOF
```

### 각 노드용 kube-scheduler systemd unit 파일 생성 및 배포

template 파일의 변수를 치환해서 각 노드용 systemd unit 파일 생성

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for (( i=0; i < 2; i++ ))
do
  sed -e "s/##NODE_NAME##/${NODE_NAMES[i]}/" -e "s/##NODE_IP##/${NODE_IPS[i]}/" kube-scheduler.service.template > kube-scheduler-${NODE_IPS[i]}.service
done
ls kube-scheduler*.service
```

모든 master 노드에 systemd unit 파일 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp kube-scheduler-${node_ip}.service root@${node_ip}:/etc/systemd/system/kube-scheduler.service
done
```

### kube-scheduler 서비스 시작

```bash
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "mkdir -p ${K8S_DIR}/kube-scheduler"
  ssh root@${node_ip} "systemctl daemon-reload && systemctl enable kube-scheduler && systemctl restart kube-scheduler"
done
```

### 서비스 실행 상태 확인

```bash
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "systemctl status kube-scheduler|grep Active"
done
```

상태가 `active (running)`인지 확인. 아니라면 로그를 확인한다:

```bash
journalctl -u kube-scheduler
```

### metrics 출력 확인

> kube-scheduler 노드에서 실행한다.

kube-scheduler는 port 10259에서 https 요청을 listen한다. 이 interface는 외부에 `/metrics`와 `/healthz`를 제공한다:

```bash
$ sudo netstat -lnpt |grep kube-sch
tcp        0      0 172.31.39.151:10259     0.0.0.0:*               LISTEN      433307/kube-schedul
```

`/metrics` interface 접근:

```bash
curl -s --cacert /opt/k8s/work/ca.pem --cert /opt/k8s/work/admin.pem --key /opt/k8s/work/admin-key.pem https://172.31.39.151:10259/metrics |head
# HELP aggregator_discovery_aggregation_count_total [ALPHA] Counter of number of times discovery was aggregated
# TYPE aggregator_discovery_aggregation_count_total counter
aggregator_discovery_aggregation_count_total 0
# HELP apiserver_audit_event_total [ALPHA] Counter of audit events generated and sent to the audit backend.
# TYPE apiserver_audit_event_total counter
apiserver_audit_event_total 0
# HELP apiserver_audit_requests_rejected_total [ALPHA] Counter of apiserver requests rejected due to an error in audit logging backend.
# TYPE apiserver_audit_requests_rejected_total counter
apiserver_audit_requests_rejected_total 0
# HELP apiserver_client_certificate_expiration_seconds [ALPHA] Distribution of the remaining lifetime on the certificate used to authenticate a request.
```

### 현재 leader 확인

```bash
$ kubectl -n kube-system get leases kube-scheduler
NAME             HOLDER                                        AGE
kube-scheduler   k8s-01_00dc5c29-22d9-4354-bd9d-025bae5ebc11   6m53s
```

현재 Leader는 k8s-01 노드에서 실행 중인 kube-scheduler instance다.

### kube-scheduler 클러스터 고가용성 테스트

한 노드의 kube-scheduler 서비스를 중지하고 다른 노드의 로그를 관찰해서 leader 권한을 획득하는지 확인한다.

k8s-01 노드의 kube-scheduler 서비스를 중지하고 현재 Leader를 확인

```bash
$ sudo systemctl stop kube-scheduler.service

$ kubectl -n kube-system get leases kube-scheduler
NAME             HOLDER                                        AGE
kube-scheduler   k8s-02_42ee9af7-637e-4d17-9285-5da67483a916   8m3s
```

kube-scheduler Leader가 k8s-02 노드의 kube-scheduler instance로 전환되었다.