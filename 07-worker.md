# Worker 노드 배포

Kubernetes 워커 노드에서는 다음 구성 요소들이 실행됩니다.

* **kube-nginx**
* **containerd**
* **kubelet**
* **kube-proxy**
* **cilium**

---

## kube-apiserver 고가용성 구성 요소 배포

이 섹션에서는 nginx의 Layer 4 투명 프록시 기능을 활용하여 Kubernetes 워커 노드 구성 요소가 kube-apiserver 클러스터에 고가용성으로 접근할 수 있도록 설정하는 방법을 설명합니다.

> **참고:** 특별한 언급이 없으면 이 섹션의 모든 작업은 `k8s-01` 노드에서 진행합니다.

### nginx 프록시 기반 kube-apiserver 고가용성 구성

* 컨트롤 플레인의 `kube-controller-manager`와 `kube-scheduler`는 각각 여러 인스턴스로 배포되어 로컬 `kube-apiserver`에 연결됩니다. 따라서 인스턴스 하나만 정상 동작해도 고가용성이 유지됩니다.
* 클러스터 내부의 Pod는 Kubernetes 서비스 도메인인 `kubernetes`를 통해 `kube-apiserver`에 접근합니다. `kube-dns`가 여러 `kube-apiserver` 노드 IP로 자동 해석해주므로 이 역시 고가용성이 보장됩니다.
* 각 노드에서 nginx 프로세스를 실행하고, 백엔드에 여러 `apiserver` 인스턴스를 연결합니다. nginx가 헬스 체크와 로드 밸런싱을 담당합니다.
* `kubelet`과 `kube-proxy`는 로컬 nginx(`127.0.0.1`에서 리스닝)를 통해 `kube-apiserver`에 접근하여 고가용성을 확보합니다.

### Nginx 다운로드 및 컴파일

소스 코드를 다운로드합니다.

```bash
cd /opt/k8s/work
wget http://nginx.org/download/nginx-1.29.0.tar.gz
tar -xzvf nginx-1.29.0.tar.gz
```

컴파일 옵션을 설정합니다.

```bash
cd /opt/k8s/work/nginx-1.29.0
mkdir nginx-prefix
./configure --with-stream --without-http --prefix=$(pwd)/nginx-prefix --without-http_uwsgi_module --without-http_scgi_module --without-http_fastcgi_module
```

* **--with-stream:** Layer 4 투명 포워딩(TCP 프록시) 기능을 활성화합니다.
* **--without-xxx:** 불필요한 기능을 비활성화하여 바이너리의 의존성을 최소화합니다.

출력 결과는 다음과 같습니다.

```
Configuration summary
  + PCRE library is not used
  + OpenSSL library is not used
  + zlib library is not used

  nginx path prefix: "/opt/k8s/work/nginx-1.29.0/nginx-prefix"
  nginx binary file: "/opt/k8s/work/nginx-1.29.0/nginx-prefix/sbin/nginx"
  nginx modules path: "/opt/k8s/work/nginx-1.29.0/nginx-prefix/modules"
  nginx configuration prefix: "/opt/k8s/work/nginx-1.29.0/nginx-prefix/conf"
  nginx configuration file: "/opt/k8s/work/nginx-1.29.0/nginx-prefix/conf/nginx.conf"
  nginx pid file: "/opt/k8s/work/nginx-1.29.0/nginx-prefix/logs/nginx.pid"
  nginx error log file: "/opt/k8s/work/nginx-1.29.0/nginx-prefix/logs/error.log"
  nginx http access log file: "/opt/k8s/work/nginx-1.29.0/nginx-prefix/logs/access.log"
  nginx http client request body temporary files: "client_body_temp"
  nginx http proxy temporary files: "proxy_temp"
```

컴파일하고 설치합니다.

```bash
make && make install
```

### 컴파일된 Nginx 확인

```bash
$ ./nginx-prefix/sbin/nginx -v
nginx version: nginx/1.29.0

$ ldd ./nginx-prefix/sbin/nginx
    linux-vdso.so.1 (0x00007ffcdb9e5000)
    libcrypt.so.1 => /lib/x86_64-linux-gnu/libcrypt.so.1 (0x00007f3888a74000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f3888893000)
    /lib64/ld-linux-x86-64.so.2 (0x00007f3888cc2000)
```

### Nginx 설치 및 배포

디렉토리 구조를 생성합니다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "mkdir -p /opt/k8s/kube-nginx/{conf,logs,sbin}"
done
```

바이너리를 복사합니다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp /opt/k8s/work/nginx-1.29.0/nginx-prefix/sbin/nginx root@${node_ip}:/opt/k8s/kube-nginx/sbin/kube-nginx
  ssh root@${node_ip} "chmod a+x /opt/k8s/kube-nginx/sbin/*"
done
```

* 바이너리 파일명을 `kube-nginx`로 변경했습니다.

Layer 4 투명 포워딩용 nginx 설정 파일을 작성합니다.

```bash
cd /opt/k8s/work
cat > kube-nginx.conf << \EOF
worker_processes 1;

events {
    worker_connections  1024;
}

stream {
    upstream backend {
        hash $remote_addr consistent;
        server 172.31.39.151:6443        max_fails=3 fail_timeout=30s;
        server 172.31.32.72:6443        max_fails=3 fail_timeout=30s;
    }

    server {
        listen 127.0.0.1:8443;
        proxy_connect_timeout 1s;
        proxy_pass backend;
    }
}
EOF
```

* `upstream backend`의 서버 목록에는 클러스터의 `kube-apiserver` 노드 IP를 입력합니다. 본인의 환경에 맞게 수정하세요.

설정 파일을 배포합니다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp kube-nginx.conf root@${node_ip}:/opt/k8s/kube-nginx/conf/kube-nginx.conf
done
```

### systemd 유닛 파일 구성 및 서비스 시작

`kube-nginx` systemd 유닛 파일을 작성합니다.

```bash
cd /opt/k8s/work
cat > kube-nginx.service <<EOF
[Unit]
Description=kube-apiserver nginx proxy
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/opt/k8s/kube-nginx/logs/kube-nginx.pid
ExecStartPre=/opt/k8s/kube-nginx/sbin/kube-nginx -c /opt/k8s/kube-nginx/conf/kube-nginx.conf -p /opt/k8s/kube-nginx -t
ExecStart=/opt/k8s/kube-nginx/sbin/kube-nginx -c /opt/k8s/kube-nginx/conf/kube-nginx.conf -p /opt/k8s/kube-nginx
ExecReload=/opt/k8s/kube-nginx/sbin/kube-nginx -c /opt/k8s/kube-nginx/conf/kube-nginx.conf -p /opt/k8s/kube-nginx -s reload
PrivateTmp=true
Restart=always
RestartSec=5
StartLimitInterval=0
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

systemd 유닛 파일을 배포합니다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp kube-nginx.service root@${node_ip}:/etc/systemd/system/
done
```

`kube-nginx` 서비스를 시작합니다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "systemctl daemon-reload && systemctl enable kube-nginx && systemctl restart kube-nginx"
done
```

### kube-nginx 서비스 상태 확인

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "systemctl status kube-nginx |grep 'Active:'"
done
```

`active (running)` 상태인지 확인합니다. 문제가 있으면 다음 명령어로 로그를 확인하세요.

```bash
journalctl -u kube-nginx
```

---

## containerd 구성 요소 배포

`containerd`는 Kubernetes CRI(Container Runtime Interface)를 구현하여 이미지 관리, 컨테이너 관리 등 핵심 컨테이너 런타임 기능을 제공합니다. `dockerd`보다 가볍고 안정적이며 이식성이 뛰어납니다. 현재 기업 환경에서 가장 널리 사용되는 컨테이너 런타임입니다.

> **참고:**
> 1. 특별한 언급이 없으면 이 섹션의 모든 작업은 `k8s-01` 노드에서 진행합니다.
> 2. Docker를 사용하려면 부록 B: Docker 배포를 참조하세요.
> 3. Docker는 flannel과 함께 사용해야 하며, flannel을 먼저 설치해야 합니다.


### 바이너리 파일 다운로드 및 배포

바이너리 파일을 다운로드합니다.

```bash
cd /opt/k8s/work
wget https://github.com/containerd/containerd/releases/download/v2.1.3/containerd-2.1.3-linux-amd64.tar.gz
wget https://github.com/opencontainers/runc/releases/download/v1.3.0/runc.amd64
wget https://github.com/containerd/nerdctl/releases/download/v2.1.3/nerdctl-2.1.3-linux-amd64.tar.gz
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.33.0/crictl-v1.33.0-linux-amd64.tar.gz
```

압축을 해제합니다.

```bash
cd /opt/k8s/work
tar -xvf containerd-2.1.3-linux-amd64.tar.gz
tar -xvf crictl-v1.33.0-linux-amd64.tar.gz
tar -xvf nerdctl-2.1.3-linux-amd64.tar.gz
```

모든 워커 노드에 바이너리를 배포합니다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp bin/* root@${node_ip}:/opt/k8s/bin
  scp runc.amd64 root@${node_ip}:/opt/k8s/bin/runc
  scp crictl root@${node_ip}:/opt/k8s/bin
  scp nerdctl root@${node_ip}:/opt/k8s/bin
  ssh root@${node_ip} "chmod a+x /opt/k8s/bin/* && mkdir -p /etc/containerd"
done
```

### containerd 설정 파일 생성 및 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat << EOF | sudo tee containerd-config.toml
version = 3
root = "${CONTAINERD_DIR}/root"
state = "${CONTAINERD_DIR}/state"
oom_score = -999

[grpc]
  max_recv_message_size = 16777216
  max_send_message_size = 16777216

[debug]
  level = "info"

[metrics]
  address = ""
  grpc_histogram = false

[plugins]
  [plugins."io.containerd.cri.v1.runtime"]
    [plugins."io.containerd.cri.v1.runtime".containerd]
      default_runtime_name = "runc"
      [plugins."io.containerd.cri.v1.runtime".containerd.runtimes]
        [plugins."io.containerd.cri.v1.runtime".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.cri.v1.runtime".containerd.runtimes.runc.options]
            BinaryName = "/opt/k8s/bin/runc"
            SystemdCgroup = true
    [plugins."io.containerd.cri.v1.runtime".cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
  [plugins."io.containerd.cri.v1.images"]
    [plugins."io.containerd.cri.v1.images".registry]
      config_path = "/etc/containerd/certs.d"
EOF
```

설정 파일을 배포합니다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "mkdir -p ${CONTAINERD_DIR}/{root,state}"
  scp containerd-config.toml root@${node_ip}:/etc/containerd/config.toml
done
```

### containerd systemd 유닛 파일 생성

```bash
cd /opt/k8s/work
cat <<EOF | sudo tee containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target dbus.service

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/opt/k8s/bin/containerd --config /etc/containerd/config.toml
Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF
```

### systemd 유닛 파일 배포 및 containerd 서비스 시작

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp containerd.service root@${node_ip}:/etc/systemd/system/
  ssh root@${node_ip} "systemctl daemon-reload && systemctl enable containerd && systemctl restart containerd"
done
```

### crictl 설정 파일 생성 및 배포

`crictl`은 CRI 호환 컨테이너 런타임용 CLI 도구로, docker 명령과 비슷한 기능을 제공합니다. 자세한 내용은 공식 문서를 참조하세요.

```bash
cd /opt/k8s/work
cat << EOF | sudo tee crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

모든 워커 노드에 배포합니다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp crictl.yaml root@${node_ip}:/etc/crictl.yaml
done
```

### nerdctl 도구 설치

`containerd`를 설치하면 `crictl`과 `ctr` 명령이 함께 설치됩니다. 하지만 이 도구들은 docker 명령과 사용법이 다소 다릅니다. 컨테이너 이미지 작업에는 `nerdctl` 사용을 권장합니다. `nerdctl`은 containerd 프로젝트의 하위 프로젝트로, Docker CLI와 호환되도록 설계되었습니다. docker 명령 대신 사용할 수 있으며, containerd 런타임만 지원합니다.

`nerdctl` 설정 디렉토리를 생성합니다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "mkdir -p /etc/nerdctl && touch /etc/nerdctl/nerdctl.toml"
done
```



## containerd 이미지 미러 구성


```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "mkdir -p /etc/containerd/certs.d/docker.io"
  ssh root@${node_ip} 'cat > /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.io"
[host."https://mirror.gcr.io"]
  capabilities = ["pull", "resolve"]
EOF'
done
```

---

## containerd 이미지 미러 구성 상세 설명 (참고용, 실행 불필요)

앞서 `containerd` 설치 과정에서 이미지 미러를 구성했습니다. 여기서는 구성 방법을 자세히 설명합니다.

### 이미지 미러 구성 권장 사항

Kubernetes에서 Pod를 생성할 때 해외 이미지 레지스트리에서 이미지를 다운로드해야 할 수 있습니다. 네트워크 상황에 따라 속도가 느릴 수 있으며, 이런 경우 이미지 미러를 구성하면 다운로드 속도를 개선할 수 있습니다.

인터넷에서 `containerd` 이미지 미러 구성 방법을 검색하면 대부분 `/etc/containerd/config.toml` 파일을 직접 수정하는 방식을 안내합니다. 이 방식은 최신 버전의 `containerd`에서 deprecated되었고 향후 제거될 예정입니다(현재는 아직 동작합니다). 또한 설정을 변경할 때마다 `systemctl restart containerd.service`로 재시작해야 하는 단점이 있습니다.

최신 버전의 `containerd`에서는 이미지 레지스트리 설정을 별도 디렉토리에 두는 방식을 권장합니다. `/etc/containerd/config.toml`에서 `config_path`를 설정하고 레지스트리 설정 디렉토리를 지정합니다. 이 방식은 처음 `config_path`를 활성화할 때만 재시작이 필요하고, 이후 레지스트리 설정을 추가할 때는 재시작 없이 바로 적용되어 편리합니다. `config_path`가 포함된 `containerd` 설정 예시:

```toml
[plugins."io.containerd.cri.v1.images".registry]
  config_path = "/etc/containerd/certs.d"
```

`/etc/containerd/config.toml`에 `config_path = /etc/containerd/certs.d`를 지정하면, `containerd` 이미지 레지스트리 설정은 다음과 같은 구조를 갖습니다.

```
$ tree /etc/containerd/certs.d
/etc/containerd/certs.d
├── docker.io
│    └── hosts.toml
├── k8s.gcr.io
│    └── hosts.toml
└── registry.k8s.io
     └── hosts.toml
```

위 구조에서 첫 번째 레벨 디렉토리는 이미지 레지스트리의 도메인 또는 IP 주소이고, 두 번째 레벨은 `hosts.toml` 파일입니다. `hosts.toml`에서 지원하는 항목은 `server`, `capabilities`, `ca`, `client`, `skip_verify`, `[header]`, `override_path`입니다. `hosts.toml` 파일 예시:

```toml
server = "https://docker.io"
[host."https://mirror.gcr.io"]
  capabilities = ["pull", "resolve"]
```

`hosts.toml`에 여러 이미지 레지스트리를 설정할 수 있습니다. `containerd`는 이미지를 다운로드할 때 설정된 순서대로 레지스트리를 시도하며, 앞쪽 레지스트리가 실패하면 다음 레지스트리를 사용합니다. 따라서 '다운로드 속도가 빠른 레지스트리를 앞쪽에 배치'하는 것이 좋습니다.

### 이미지 미러 구성 (선택 사항)

아래는 Google Mirror를 사용한 이미지 미러 구성 예시입니다:

```bash
mkdir -p /etc/containerd/certs.d/docker.io

# docker.io mirrors (Google Mirror 사용)
cat > /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.io"
[host."https://mirror.gcr.io"]
  capabilities = ["pull", "resolve"]
EOF
```

> **참고:** 한국에서는 대부분의 이미지 레지스트리에 직접 접속이 가능하므로 미러 설정 없이 사용해도 됩니다. 속도 문제가 발생할 때만 미러를 구성하세요.

### 이미지 레지스트리 미러 동작 확인

이미지 미러를 구성했다면 다음 방법으로 정상 동작 여부를 확인할 수 있습니다.

`nerdctl` 명령은 `/etc/containerd/certs.d` 디렉토리의 설정을 자동으로 사용합니다. 반면 `ctr` 명령은 `--hosts-dir=/etc/containerd/certs.d` 옵션을 명시해야 합니다. 예: `ctr -n k8s.io i pull --hosts-dir=/etc/containerd/certs.d registry.k8s.io/sig-storage/csi-provisioner:v3.5.0`. 실제로 이미지 미러가 사용되는지 확인하려면 `--debug=true` 옵션을 추가하세요.

`registry.k8s.io` 이미지 레지스트리 확인:

```bash
ctr -n k8s.io i pull --hosts-dir=/etc/containerd/certs.d registry.k8s.io/sig-storage/csi-provisioner:v3.5.0
```

`docker.io` 이미지 레지스트리 확인:

```bash
ctr -n k8s.io i pull --hosts-dir=/etc/containerd/certs.d docker.io/library/nginx:1.25.3
```



## kubelet 구성 요소 배포

`kubelet`은 각 워커 노드에서 실행되며, `kube-apiserver`로부터 요청을 받아 Pod 컨테이너를 관리하고 `exec`, `run`, `logs` 등의 명령을 처리합니다.

`kubelet`은 시작 시 노드 정보를 `kube-apiserver`에 자동 등록합니다. 내장된 `cadvisor`가 노드의 리소스 사용량을 수집하고 모니터링합니다.

보안을 위해 배포 시 `kubelet`의 비보안 HTTP 포트는 비활성화합니다. 요청은 인증 및 인가 과정을 거치며, 권한이 없는 접근(예: `apiserver`, `heapster`의 요청)은 거부됩니다.

> **참고:** 특별한 언급이 없으면 이 섹션의 모든 작업은 `k8s-01` 노드에서 진행합니다.

### kubelet 바이너리 파일 다운로드 및 배포

'마스터 노드 구성 요소 배포' 섹션에서 이미 `kubelet` 파일을 다운로드하고 배포했습니다.

### kubelet bootstrap kubeconfig 파일 생성

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_name in ${NODE_NAMES[@]}
do
  echo ">>> ${node_name}"

  # 토큰 생성
  export BOOTSTRAP_TOKEN=$(kubeadm token create \
    --description kubelet-bootstrap-token \
    --groups system:bootstrappers:${node_name} \
    --kubeconfig ~/.kube/config)

  # 클러스터 정보 설정
  kubectl config set-cluster kubernetes \
    --certificate-authority=/opt/k8s/work/ca.pem \
    --embed-certs=true \
    --server=${KUBE_APISERVER} \
    --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig

  # 클라이언트 인증 정보 설정
  kubectl config set-credentials kubelet-bootstrap \
    --token=${BOOTSTRAP_TOKEN} \
    --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig

  # 컨텍스트 설정
  kubectl config set-context default \
    --cluster=kubernetes \
    --user=kubelet-bootstrap \
    --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig

  # 기본 컨텍스트 지정
  kubectl config use-context default --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig
done
```

또는 수동으로 토큰을 생성하는 방식:

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh

for node_name in ${NODE_NAMES[@]}
do
  echo ">>> ${node_name}"
  
  # 토큰 생성 (형식: [a-z0-9]{6}.[a-z0-9]{16})
  TOKEN_ID=$(openssl rand -hex 3)
  TOKEN_SECRET=$(openssl rand -hex 8)
  BOOTSTRAP_TOKEN="${TOKEN_ID}.${TOKEN_SECRET}"
  
  echo "Token for ${node_name}: ${BOOTSTRAP_TOKEN}"
  
  # Bootstrap token Secret 생성
  kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-${TOKEN_ID}
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  token-id: "${TOKEN_ID}"
  token-secret: "${TOKEN_SECRET}"
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"
  auth-extra-groups: "system:bootstrappers:${node_name}"
  expiration: "$(date -d '+24 hours' -u +%Y-%m-%dT%H:%M:%SZ)"
EOF

  # kubeconfig 생성
  kubectl config set-cluster kubernetes \
    --certificate-authority=/opt/k8s/work/ca.pem \
    --embed-certs=true \
    --server=${KUBE_APISERVER} \
    --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig

  kubectl config set-credentials kubelet-bootstrap \
    --token=${BOOTSTRAP_TOKEN} \
    --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes \
    --user=kubelet-bootstrap \
    --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig

  kubectl config use-context default --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig
done
```

* 토큰은 `kubeconfig`에 저장됩니다. 부트스트랩이 완료되면 `kube-controller-manager`가 `kubelet`용 클라이언트/서버 인증서를 생성합니다.

각 노드에서 `kubeadm`으로 생성한 토큰을 확인합니다:

```bash
$ kubeadm token list --kubeconfig ~/.kube/config
TOKEN                    TTL           EXPIRES                USAGES                    DESCRIPTION                                                EXTRA GROUPS
mpn28j.nt3n72wfps53h6ls   23h           2024-10-12T11:49:03Z   authentication,signing   kubelet-bootstrap-token                                    system:bootstrappers:k8s-01
zq06na.t7jux35chqpkfcdl   23h           2024-10-12T11:49:04Z   authentication,signing   kubelet-bootstrap-token                                    system:bootstrappers:k8s-02
```

* 토큰은 1일 동안 유효합니다. 만료 후에는 `kubelet` 부트스트랩에 사용할 수 없으며, `kube-controller-manager`의 `tokencleaner`가 정리합니다.
* `kube-apiserver`는 `kubelet`의 부트스트랩 토큰을 받으면 요청의 사용자를 `system:bootstrap:<Token ID>`로, 그룹을 `system:bootstrappers`로 설정합니다. 이 그룹에 대해 나중에 `ClusterRoleBinding`을 설정합니다.

### 모든 워커 노드에 bootstrap kubeconfig 파일 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_name in ${NODE_NAMES[@]}
do
  echo ">>> ${node_name}"
  scp kubelet-bootstrap-${node_name}.kubeconfig root@${node_name}:/etc/kubernetes/kubelet-bootstrap.kubeconfig
done
```

### kubelet 설정 파일 생성 및 배포

v1.10부터 일부 `kubelet` 옵션은 설정 파일로 지정해야 합니다. `kubelet --help`를 실행하면 다음 메시지가 표시됩니다.

```
DEPRECATED: This parameter should be set via the config file specified by the Kubelet's --config flag
```

`kubelet` 설정 파일 템플릿을 생성합니다 (설정 가능한 항목은 `KubeletConfiguration` 참조):

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > kubelet-config.yaml.template <<EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: "##NODE_IP##"
staticPodPath: ""
syncFrequency: 1m
fileCheckFrequency: 20s
httpCheckFrequency: 20s
staticPodURL: ""
port: 10250
readOnlyPort: 0
rotateCertificates: true
serverTLSBootstrap: true
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/etc/kubernetes/cert/ca.pem"
authorization:
  mode: Webhook
registryPullQPS: 0
registryBurst: 20
eventRecordQPS: 0
eventBurst: 20
enableDebuggingHandlers: true
enableContentionProfiling: true
healthzPort: 10248
healthzBindAddress: "##NODE_IP##"
clusterDomain: "${CLUSTER_DNS_DOMAIN}"
clusterDNS:
  - "${CLUSTER_DNS_SVC_IP}"
nodeStatusUpdateFrequency: 10s
nodeStatusReportFrequency: 1m
imageMinimumGCAge: 2m
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
volumeStatsAggPeriod: 1m
kubeletCgroups: ""
systemCgroups: ""
cgroupRoot: ""
cgroupsPerQOS: true
cgroupDriver: systemd
runtimeRequestTimeout: 10m
hairpinMode: promiscuous-bridge
maxPods: 220
podCIDR: "${CLUSTER_CIDR}"
podPidsLimit: -1
resolvConf: /etc/resolv.conf
maxOpenFiles: 1000000
kubeAPIQPS: 1000
kubeAPIBurst: 2000
serializeImagePulls: false
evictionHard:
  memory.available:  "100Mi"
  nodefs.available:  "10%"
  nodefs.inodesFree: "5%"
  imagefs.available: "15%"
evictionSoft: {}
enableControllerAttachDetach: true
failSwapOn: true
containerLogMaxSize: 20Mi
containerLogMaxFiles: 10
systemReserved: {}
kubeReserved: {}
systemReservedCgroup: ""
kubeReservedCgroup: ""
enforceNodeAllocatable: ["pods"]
EOF
```

* **address:** `kubelet` 보안 포트(https, 10250)가 바인딩할 주소입니다. `127.0.0.1`로 설정하면 `kube-apiserver`, `heapster` 등이 `kubelet` API를 호출할 수 없습니다.
* **readOnlyPort=0:** 읽기 전용 포트(기본값 10255)를 비활성화합니다.
* **authentication.anonymous.enabled:** false로 설정하여 10250 포트에 대한 익명 접근을 차단합니다.
* **authentication.x509.clientCAFile:** 클라이언트 인증서에 서명한 CA 인증서를 지정하여 HTTPS 인증서 인증을 활성화합니다.
* **authentication.webhook.enabled=true:** HTTPS bearer 토큰 인증을 활성화합니다.
* x509 인증서나 웹훅으로 인증되지 않은 요청(`kube-apiserver` 등)은 `Unauthorized`로 거부됩니다.
* **authorization.mode=Webhook:** `kubelet`이 `SubjectAccessReview` API를 사용하여 `kube-apiserver`에 해당 사용자/그룹의 리소스 조작 권한을 확인합니다(RBAC).
* **featureGates.RotateKubeletClientCertificate, featureGates.RotateKubeletServerCertificate:** 인증서 자동 갱신을 활성화합니다. 인증서 유효 기간은 `kube-controller-manager`의 `--experimental-cluster-signing-duration` 옵션으로 설정합니다.
* root 권한으로 실행해야 합니다.

각 노드별 `kubelet` 설정 파일을 생성하고 배포합니다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for (( i=0; i < 2; i++ ))
do
  echo ">>> ${NODE_NAMES[i]}"
  sed -e "s/##NODE_IP##/${NODE_IPS[i]}/" kubelet-config.yaml.template > kubelet-config-${NODE_NAMES[i]}.yaml
  scp kubelet-config-${NODE_NAMES[i]}.yaml root@${NODE_NAMES[i]}:/etc/kubernetes/kubelet-config.yaml
done
```

### kubelet systemd 유닛 파일 생성 및 배포

`kubelet` systemd 유닛 파일 템플릿을 생성합니다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > kubelet.service.template <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
WorkingDirectory=${K8S_DIR}/kubelet
ExecStart=/opt/k8s/bin/kubelet \\
  --bootstrap-kubeconfig=/etc/kubernetes/kubelet-bootstrap.kubeconfig \\
  --cert-dir=/etc/kubernetes/cert \\
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
  --config=/etc/kubernetes/kubelet-config.yaml \\
  --hostname-override=##NODE_NAME## \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --root-dir=${K8S_DIR}/kubelet \\
  --v=2
Restart=always
RestartSec=5
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
EOF
```

* `--hostname-override` 옵션을 설정하면 `kube-proxy`에도 동일하게 설정해야 합니다. 그렇지 않으면 노드를 찾지 못합니다.
* **--bootstrap-kubeconfig:** 부트스트랩용 `kubeconfig` 파일 경로입니다. `kubelet`은 이 파일의 사용자 이름과 토큰으로 `kube-apiserver`에 TLS 부트스트랩 요청을 보냅니다.
* K8S가 `kubelet`의 CSR 요청을 승인하면 `--cert-dir` 디렉토리에 인증서와 개인 키가 생성되고, `--kubeconfig` 파일에 기록됩니다.
* **--pod-infra-container-image:** redhat의 `pod-infrastructure:latest` 이미지는 사용하지 않습니다. 컨테이너 좀비 프로세스를 회수하지 못하는 문제가 있기 때문입니다.

각 노드별 `kubelet` systemd 유닛 파일을 생성하고 배포합니다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_name in ${NODE_NAMES[@]}
do
  echo ">>> ${node_name}"
  sed -e "s/##NODE_NAME##/${node_name}/" kubelet.service.template > kubelet-${node_name}.service
  scp kubelet-${node_name}.service root@${node_name}:/etc/systemd/system/kubelet.service
done
```

### kube-apiserver에 kubelet API 접근 권한 부여

`kubectl exec`, `run`, `logs` 등의 명령을 실행하면 `apiserver`가 요청을 `kubelet`의 https 포트로 전달합니다. 여기서는 `apiserver`가 사용하는 인증서(`kubernetes.pem`, 사용자 이름 CN: `kubernetes-master`)가 `kubelet` API에 접근할 수 있도록 RBAC 규칙을 정의합니다.

```bash
kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes-master
```

### 부트스트랩 토큰 인증 및 권한 부여

`kubelet`이 시작될 때 `--config`에 지정된 파일을 찾습니다. 파일이 없으면 `--bootstrap-kubeconfig`에 지정된 `kubeconfig` 파일을 사용하여 `kube-apiserver`에 CSR(인증서 서명 요청)을 보냅니다.

`kube-apiserver`는 CSR 요청을 받으면 포함된 토큰을 검증합니다. 검증 후 요청 사용자를 `system:bootstrap:<Token ID>`로, 그룹을 `system:bootstrappers`로 설정합니다. 이 과정을 **부트스트랩 토큰 인증(Bootstrap Token Auth)**이라고 합니다.

기본적으로 이 사용자와 그룹은 CSR을 생성할 권한이 없습니다. 따라서 `kubelet` 시작 시 다음과 같은 에러가 발생합니다.

```
failed to run Kubelet: cannot create certificate signing request: certificatesigningrequests.certificates.k8s.io is forbidden: User "system:bootstrap:mpn28j" cannot create resource "certificatesigningrequests" in API group "certificates.k8s.io" at the cluster scope
```

해결 방법은 `system:bootstrappers` 그룹을 `system:node-bootstrapper` ClusterRole에 바인딩하는 `ClusterRoleBinding`을 생성하는 것입니다.

```bash
kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --group=system:bootstrappers
```

### CSR 요청 자동 승인 및 kubelet 클라이언트 인증서 생성

`kubelet`이 CSR 요청을 생성하면 이를 승인해야 합니다. 두 가지 방법이 있습니다.

1. `kube-controller-manager`가 자동으로 승인
2. `kubectl certificate approve` 명령으로 수동 승인

CSR이 승인되면 `kubelet`은 `kube-controller-manager`에게 클라이언트 인증서 생성을 요청합니다. `kube-controller-manager`의 `csrapproving` 컨트롤러는 `SubjectAccessReview` API를 사용하여 `kubelet` 요청(그룹: `system:bootstrappers`)에 해당 권한이 있는지 확인합니다.

`system:bootstrappers` 그룹과 `system:nodes` 그룹에 클라이언트 승인, 클라이언트 갱신, 서버 인증서 갱신 권한을 부여하는 3개의 `ClusterRoleBinding`을 생성합니다(서버 CSR은 아래에서 수동 승인).

```bash
cd /opt/k8s/work
cat > csr-crb.yaml <<EOF
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: auto-approve-csrs-for-group
subjects:
  - kind: Group
    name: system:bootstrappers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
  apiGroup: rbac.authorization.k8s.io
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: node-client-cert-renewal
subjects:
  - kind: Group
    name: system:nodes
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
  apiGroup: rbac.authorization.k8s.io
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: node-server-cert-renewal
subjects:
  - kind: Group
    name: system:nodes
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeserver
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f csr-crb.yaml
```

* **auto-approve-csrs-for-group:** 노드의 첫 번째 CSR을 자동 승인합니다. 첫 번째 CSR의 요청 그룹은 `system:bootstrappers`입니다.
* **node-client-cert-renewal:** 만료된 노드의 후속 클라이언트 인증서를 자동 승인합니다. 자동 생성된 인증서 그룹은 `system:nodes`입니다.
* **node-server-cert-renewal:** 만료된 노드의 후속 서버 인증서를 자동 승인합니다. 자동 생성된 인증서 그룹은 `system:nodes`입니다.

### kubelet 서비스 시작

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_name in ${NODE_NAMES[@]}
do
  echo ">>> ${node_name}"
  ssh root@${node_name} "mkdir -p ${K8S_DIR}/kubelet/kubelet-plugins/volume/exec/"
  ssh root@${node_name} "systemctl daemon-reload && systemctl enable kubelet && systemctl restart kubelet"
done
```

* 서비스 시작 전에 작업 디렉토리를 생성해야 합니다.
* **Swap 파티션을 반드시 비활성화**해야 합니다. 그렇지 않으면 `kubelet`이 시작되지 않습니다.

`kubelet`이 시작되면 `--bootstrap-kubeconfig`를 사용하여 `kube-apiserver`에 CSR 요청을 보냅니다. CSR이 승인되면 `kube-controller-manager`가 `kubelet`용 TLS 클라이언트 인증서, 개인 키, `--kubeletconfig` 파일을 생성합니다.

참고: `kube-controller-manager`는 TLS 부트스트랩용 인증서와 개인 키를 생성하기 위해 `--cluster-signing-cert-file` 및 `--cluster-signing-key-file` 옵션을 설정해야 합니다.

### kubelet 상태 확인

잠시 기다리면 모든 노드에서 CSR이 자동으로 승인됩니다.

```bash
$ kubectl get csr
NAME        AGE     SIGNERNAME                                    REQUESTOR                 REQUESTEDDURATION   CONDITION
csr-dkfxc   22m     kubernetes.io/kube-apiserver-client-kubelet   system:bootstrap:696eb4   <none>              Approved,Issued
csr-kfw7l   22m     kubernetes.io/kubelet-serving                 system:node:k8s-01        <none>              Pending
csr-wgdvq   22m     kubernetes.io/kubelet-serving                 system:node:k8s-02        <none>              Pending
csr-zjxhc   22m     kubernetes.io/kube-apiserver-client-kubelet   system:bootstrap:9767ca   <none>              Approved,Issued
```

* `Pending` 상태의 CSR은 `kubelet` 서버 인증서용으로, 수동 승인이 필요합니다. (아래 참조)

모든 노드가 등록되었습니다 (`NotReady` 상태는 정상이며, 네트워크 플러그인 설치 후 해결됩니다).

```bash
$ kubectl get node
NAME    STATUS    ROLES    AGE     VERSION
k8s-01   NotReady   <none>   4m54s   v1.33.3
k8s-02   NotReady   <none>   4m54s   v1.33.3

```

`kube-controller-manager`가 각 노드에 `kubeconfig` 파일과 공개/개인 키를 생성했습니다.

```bash
$ ls -l /etc/kubernetes/kubelet.kubeconfig
-rw------- 1 root root 2310 Oct 11 19:56 /etc/kubernetes/kubelet.kubeconfig

$ ls -l /etc/kubernetes/cert/|grep kubelet
-rw------- 1 root root 1277 Oct 11 19:56 kubelet-client-2024-10-11-19-56-37.pem
lrwxrwxrwx 1 root root   59 Oct 11 19:56 kubelet-client-current.pem -> /etc/kubernetes/cert/kubelet-client-2024-10-11-19-56-37.pem
```

* `kubelet` 서버 인증서는 아직 자동 생성되지 않았습니다.

### 서버 인증서 CSR 수동 승인

보안상의 이유로 CSR 승인 컨트롤러는 `kubelet` 서버 인증서 서명 요청을 자동으로 승인하지 않습니다. 수동으로 승인해야 합니다.

```bash
$ kubectl get csr | grep Pending | awk '{print $1}' | xargs kubectl certificate approve
certificatesigningrequest.certificates.k8s.io/csr-7gn8v approved
certificatesigningrequest.certificates.k8s.io/csr-8fmc4 approved
```

### kubelet API 인증 및 인가

`kubelet`은 다음 인증 옵션으로 구성되었습니다.

* **authentication.anonymous.enabled:** false로 설정하여 10250 포트에 대한 익명 접근을 차단합니다.
* **authentication.x509.clientCAFile:** 클라이언트 인증서에 서명한 CA 인증서를 지정하여 HTTPS 인증서 인증을 활성화합니다.
* **authentication.webhook.enabled=true:** HTTPS bearer 토큰 인증을 활성화합니다.

또한 다음 인가 옵션으로 구성되었습니다.

* **authorization.mode=Webhook:** RBAC 인가를 활성화합니다.

`kubelet`이 요청을 받으면 `clientCAFile`로 인증서 서명을 검증하거나 bearer 토큰의 유효성을 확인합니다. 둘 다 통과하지 못하면 `401 Unauthorized`로 거부됩니다.

인증 후 `kubelet`은 `SubjectAccessReview` API로 `kube-apiserver`에 요청을 보내, 해당 인증서나 토큰에 대응하는 사용자/그룹이 리소스를 조작할 권한이 있는지 확인합니다(RBAC).

### 인증서 인증 및 인가

```bash
curl -s --cacert /opt/k8s/work/ca.pem --cert /opt/k8s/work/admin.pem --key /opt/k8s/work/admin-key.pem https://172.31.39.151:10250/metrics | head
```

* `--cacert`, `--cert`, `--key` 옵션에는 반드시 `/opt/k8s/work/admin.pem`과 같은 파일 경로를 지정해야 합니다. 그렇지 않으면 `401 Unauthorized`가 반환됩니다.

### Bearer 토큰 인증 및 인가

`ServiceAccount`를 생성하고 `system:kubelet-api-admin` ClusterRole에 바인딩하여 `kubelet` API 호출 권한을 부여합니다.

```bash
kubectl create sa kubelet-api-test
kubectl create clusterrolebinding kubelet-api-test --clusterrole=system:kubelet-api-admin --serviceaccount=default:kubelet-api-test
SECRET=$(kubectl get secrets | grep kubelet-api-test | awk '{print $1}')
TOKEN=$(kubectl describe secret ${SECRET} | grep -E '^token' | awk '{print $2}')
echo ${TOKEN}
```

토큰으로 `/metrics`에 접근합니다:

```bash
curl -s --cacert /opt/k8s/work/ca.pem -H "Authorization: Bearer ${TOKEN}" https://172.31.39.151:10250/metrics | head
```

### cadvisor 및 metrics

`cadvisor`는 `kubelet` 바이너리에 내장되어 있으며, 노드의 각 컨테이너에 대한 리소스(CPU, 메모리, 디스크, 네트워크) 사용량을 수집합니다.

브라우저에서 `https://10.37.91.93:10250/metrics` 및 `https://10.37.91.93:10250/metrics/cadvisor`에 접속하면 각각 `kubelet`과 `cadvisor` 메트릭이 반환됩니다.

> **참고:**
> * `kubelet.config.json`에서 `authentication.anonymous.enabled`를 false로 설정하여 10250 포트의 https 서비스에 대한 익명 접근을 차단했습니다.
> * '부록 C: 브라우저에서 kube-apiserver 보안 포트 접속'을 참조하여 인증서를 생성하고 가져온 후 10250 포트에 접속하세요.

### 참고 자료

1. kubelet 인증 및 인가: [https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-authentication-authorization/](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-authentication-authorization/)

---

## kube-proxy 구성 요소 배포

`kube-proxy`는 모든 워커 노드에서 실행됩니다. `apiserver`로부터 서비스 및 엔드포인트 변경 사항을 감시하고, 라우팅 규칙을 생성하여 서비스 IP와 로드 밸런싱 기능을 제공합니다.

이 문서에서는 `ipvs` 모드로 `kube-proxy`를 배포합니다.

> **참고:** 특별한 언급이 없으면 이 섹션의 모든 작업은 `k8s-01` 노드에서 진행한 후 원격으로 파일을 배포하고 명령을 실행합니다.

### kube-proxy 바이너리 파일 다운로드 및 배포

'마스터 노드 구성 요소 배포' 섹션에서 이미 `kube-proxy` 파일을 다운로드하고 배포했습니다.

### kube-proxy 인증서 생성

CSR(인증서 서명 요청)을 생성합니다.

```bash
cd /opt/k8s/work
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
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

* **CN:** 인증서 사용자를 `system:kube-proxy`로 지정합니다.
* 사전 정의된 RoleBinding인 `system:node-proxier`가 `system:kube-proxy` 사용자를 `system:node-proxier` Role에 바인딩하며, `kube-apiserver` Proxy 관련 API를 호출할 권한을 부여합니다.
* 이 인증서는 `kube-proxy`에서 클라이언트 인증서로만 사용되므로 `hosts` 필드는 비워둡니다.

인증서와 개인 키를 생성합니다.

```bash
cd /opt/k8s/work
cfssl gencert -ca=/opt/k8s/work/ca.pem \
  -ca-key=/opt/k8s/work/ca-key.pem \
  -config=/opt/k8s/work/ca-config.json \
  -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
```

### kubeconfig 파일 생성 및 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/k8s/work/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

`kubeconfig` 파일을 배포합니다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_name in ${NODE_NAMES[@]}
do
  echo ">>> ${node_name}"
  scp kube-proxy.kubeconfig root@${node_name}:/etc/kubernetes/
done
```

### kube-proxy 설정 파일 생성

v1.10부터 일부 `kube-proxy` 옵션은 설정 파일로 지정할 수 있습니다. `--write-config-to` 옵션으로 설정 파일을 생성하거나 `KubeProxyConfiguration`을 참조하세요.

`kube-proxy` 설정 파일 템플릿을 생성합니다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > kube-proxy-config.yaml.template <<EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: ##NODE_IP##
clientConnection:
  burst: 200
  kubeconfig: "/etc/kubernetes/kube-proxy.kubeconfig"
  qps: 100
clusterCIDR: ${CLUSTER_CIDR}
healthzBindAddress: ##NODE_IP##:10256
hostnameOverride: ##NODE_NAME##
metricsBindAddress: ##NODE_IP##:10249
mode: "ipvs"
EOF
```

* **bindAddress:** 바인딩 주소
* **clientConnection.kubeconfig:** apiserver 연결용 `kubeconfig` 파일
* **clusterCIDR:** `kube-proxy`는 `--cluster-cidr` 값을 기준으로 클러스터 내부/외부 트래픽을 구분합니다. `--cluster-cidr` 또는 `--masquerade-all` 옵션을 지정해야 Service IP로의 요청에 SNAT를 수행합니다.
* **hostnameOverride:** `kubelet`과 동일한 값을 사용해야 합니다. 그렇지 않으면 `kube-proxy`가 노드를 찾지 못해 `ipvs` 규칙을 생성하지 않습니다.
* **mode:** `ipvs` 모드를 사용합니다.

각 노드별 `kube-proxy` 설정 파일을 생성하고 배포합니다:

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for (( i=0; i < 2; i++ ))
do
  echo ">>> ${NODE_NAMES[i]}"
  sed -e "s/##NODE_NAME##/${NODE_NAMES[i]}/" -e "s/##NODE_IP##/${NODE_IPS[i]}/" kube-proxy-config.yaml.template > kube-proxy-config-${NODE_NAMES[i]}.yaml
  scp kube-proxy-config-${NODE_NAMES[i]}.yaml root@${NODE_NAMES[i]}:/etc/kubernetes/kube-proxy-config.yaml
done
```

### kube-proxy systemd 유닛 파일 생성 및 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
cat > kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=${K8S_DIR}/kube-proxy
ExecStart=/opt/k8s/bin/kube-proxy \\
  --config=/etc/kubernetes/kube-proxy-config.yaml \\
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

```

`kube-proxy` systemd 유닛 파일을 배포합니다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_name in ${NODE_NAMES[@]}
do
  echo ">>> ${node_name}"
  scp kube-proxy.service root@${node_name}:/etc/systemd/system/
done
```

### kube-proxy 서비스 시작

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_name in ${NODE_NAMES[@]}
do
  echo ">>> ${node_name}"
  ssh root@${node_name} "mkdir -p ${K8S_DIR}/kube-proxy"
  ssh root@${node_name} "systemctl daemon-reload && systemctl enable kube-proxy && systemctl restart kube-proxy"
done
```

### 시작 결과 확인

```bash
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "systemctl status kube-proxy|grep Active"
done
```

`active (running)` 상태인지 확인합니다.

```bash
journalctl -u kube-proxy
```

### 리스닝 포트 확인

```bash
$ sudo netstat -lnpt|grep kube-prox
tcp        0      0 172.31.39.151:10249     0.0.0.0:*             LISTEN      2637/kube-proxy
tcp        0      0 172.31.39.151:10256     0.0.0.0:*             LISTEN      2637/kube-proxy
```

* **10249:** http prometheus 메트릭 포트
* **10256:** http healthz 포트

### ipvs 라우팅 규칙 확인

```bash
sudo apt install -y ipvsadm
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "/usr/sbin/ipvsadm -ln"
done
```

예상 출력:

```
>>> 172.31.39.151
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 rr
  -> 172.31.32.72:6443            Masq    1      2          0
  -> 172.31.39.151:6443           Masq    1      2          0
TCP  10.254.113.80:443 rr
  -> 172.31.39.151:4244           Masq    1      0          0
>>> 172.31.32.72
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 rr
  -> 172.31.32.72:6443            Masq    1      2          0
  -> 172.31.39.151:6443           Masq    1      2          0
TCP  10.254.113.80:443 rr
  -> 172.31.32.72:4244            Masq    1      0          0
```

위 결과에서 확인할 수 있듯이, https를 통해 Kubernetes 서비스 `kubernetes`에 접근하는 모든 요청은 `kube-apiserver` 노드의 6443 포트로 전달됩니다.