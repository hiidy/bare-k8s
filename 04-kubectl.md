# kubectl 설치 및 구성

kubectl은 Kubernetes command-line 관리 도구다.

> **참고:**
> 
> 1. 별도 명시가 없으면 모든 작업은 k8s-01 노드에서 실행
> 2. 아래 과정은 한 번만 배포하면 된다. 생성된 kubeconfig 파일은 범용적이라 kubectl을 실행할 모든 머신의 `~/.kube/config`에 복사해서 사용 가능

---

## kubectl binary 다운로드 및 배포

```bash
cd /opt/k8s/work
wget https://dl.k8s.io/release/v1.33.3/bin/linux/amd64/kubectl
chmod +x kubectl
mv kubectl /opt/k8s/bin/
```

kubectl을 사용할 모든 노드에 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  scp /opt/k8s/bin/kubectl root@${node_ip}:/opt/k8s/bin/
  ssh root@${node_ip} "chmod +x /opt/k8s/bin/*"
done
```

---

## admin 인증서 및 private key 생성

kubectl은 https 프로토콜로 kube-apiserver와 통신한다. kube-apiserver는 kubectl 요청에 포함된 인증서를 검증하고 인가한다.

kubectl은 클러스터 관리에 사용되므로 최고 권한을 가진 admin 인증서를 생성한다.

```bash
cd /opt/k8s/work
cat > admin-csr.json <<EOF
{
  "CN": "admin",
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
      "O": "system:masters",
      "OU": "bare-k8s"
    }
  ]
}
EOF
```

- **O: system:masters**: kube-apiserver가 이 인증서를 사용하는 client 요청을 받으면 `system:masters` group 인증 identifier를 추가
- 기본 제공되는 ClusterRoleBinding `cluster-admin`이 Group `system:masters`를 Role `cluster-admin`에 바인딩하고 있어서 클러스터 운영에 필요한 최고 권한이 부여됨
- 이 인증서는 kubectl의 client 인증서로만 사용되므로 hosts 필드는 비워둔다

인증서 및 private key 생성

```bash
cd /opt/k8s/work
cfssl gencert -ca=/opt/k8s/work/ca.pem \
  -ca-key=/opt/k8s/work/ca-key.pem \
  -config=/opt/k8s/work/ca-config.json \
  -profile=kubernetes admin-csr.json | cfssljson -bare admin
ls admin*
```

- `[WARNING] This certificate lacks a "hosts" field.` 경고는 무시해도 된다.

---

## kubeconfig 파일 생성

kubectl은 kubeconfig 파일로 apiserver에 접근한다. 이 파일에는 kube-apiserver 주소와 인증 정보(CA 인증서 및 client 인증서)가 포함된다.

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh

# cluster parameter 설정
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/k8s/work/ca.pem \
  --embed-certs=true \
  --server=https://${NODE_IPS[0]}:6443 \
  --kubeconfig=kubectl.kubeconfig

# client 인증 parameter 설정
kubectl config set-credentials admin \
  --client-certificate=/opt/k8s/work/admin.pem \
  --client-key=/opt/k8s/work/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=kubectl.kubeconfig

# context parameter 설정
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin \
  --kubeconfig=kubectl.kubeconfig

# default context 설정
kubectl config use-context kubernetes --kubeconfig=kubectl.kubeconfig
```

- **--certificate-authority**: kube-apiserver 인증서를 검증하는 root 인증서
- **--client-certificate, --client-key**: 새로 생성한 admin 인증서와 private key. kube-apiserver와 https 통신 시 사용
- **--embed-certs=true**: ca.pem과 admin.pem 인증서 내용을 kubectl.kubeconfig 파일에 embed (그렇지 않으면 인증서 파일 경로가 기록되고, kubeconfig를 다른 머신에 복사할 때 인증서 파일도 따로 복사해야 해서 불편함)
- **--server**: kube-apiserver 주소. 여기서는 첫 번째 노드의 서비스를 지정

---

## kubeconfig 파일 배포

kubectl을 사용할 모든 노드에 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "mkdir -p ~/.kube"
  scp kubectl.kubeconfig root@${node_ip}:~/.kube/config
done
```