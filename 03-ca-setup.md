# CA 인증서 생성

Kubernetes의 각 컴포넌트는 보안을 위해 x509 인증서로 통신을 암호화하고 인증한다.

- **CA (Certificate Authority)**: 이후 생성하는 인증서들을 서명하는 데 사용하는 self-signed root 인증서
- CA 인증서는 클러스터의 모든 노드에서 공유하며, 한 번만 생성하면 된다
- 인증서 생성에는 CloudFlare의 PKI toolset [cfssl](https://github.com/cloudflare/cfssl)을 사용

> 별도 명시가 없으면 모든 작업은 k8s-01 노드에서 실행한다.

---

## cfssl 설치

```bash
cd /opt/k8s/work
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssl_1.6.5_linux_amd64 -O cfssl
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssljson_1.6.5_linux_amd64 -O cfssljson
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.5/cfssl-certinfo_1.6.5_linux_amd64 -O cfssl-certinfo
chmod +x cfssl cfssljson cfssl-certinfo
mv cfssl cfssljson cfssl-certinfo /opt/k8s/bin/
```

---

## CA config 파일 생성

CA config 파일은 root 인증서의 사용 시나리오(profile)와 세부 parameter(usage, 만료 시간, server auth, client auth, encryption 등)를 설정한다

```bash
cd /opt/k8s/work
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ],
        "expiry": "876000h"
      }
    }
  }
}
EOF
```

- **signing**: 이 인증서로 다른 인증서를 서명할 수 있음 (생성된 ca.pem 인증서에 `CA=TRUE` 설정됨)
- **server auth**: client가 이 인증서로 server가 제공한 인증서를 검증할 수 있음
- **client auth**: server가 이 인증서로 client가 제공한 인증서를 검증할 수 있음
- **"expiry": "876000h"**: 인증서 유효기간 100년

---

## CSR (Certificate Signing Request) 파일 생성

```bash
cd /opt/k8s/work
cat > ca-csr.json <<EOF
{
  "CN": "kubernetes-ca",
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
  ],
  "ca": {
    "expiry": "876000h"
  }
}
EOF
```

- **CN (Common Name)**: kube-apiserver가 인증서에서 이 필드를 추출해 요청 username으로 사용. 브라우저는 이 필드로 웹사이트의 유효성을 검증
- **O (Organization)**: kube-apiserver가 인증서에서 이 필드를 추출해 요청 user가 속한 group으로 사용
- kube-apiserver는 추출한 User와 Group을 RBAC 인가의 user identifier로 사용

> **주의:**
> 
> 1. 각 인증서 CSR 파일의 CN, C, ST, L, O, OU 조합은 서로 달라야 한다. 같으면 `PEER'S CERTIFICATE HAS AN INVALID SIGNATURE` 에러가 발생할 수 있다.
> 2. 이후 생성하는 인증서의 CSR 파일에서는 CN을 다르게 설정하고 (C, ST, L, O, OU는 동일하게 유지) 구분한다.

---

## CA 인증서 및 private key 생성

```bash
cd /opt/k8s/work
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
ls ca*
```

---

## 인증서 파일 배포

```bash
cd /opt/k8s/work
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
do
  echo ">>> ${node_ip}"
  ssh root@${node_ip} "mkdir -p /etc/kubernetes/cert"
  scp ca*.pem ca-config.json root@${node_ip}:/etc/kubernetes/cert
done
```

---

## Reference

1. [Various CA Certificate Types](https://github.com/kubernetes-sigs/apiserver-builder-alpha/blob/master/docs/concepts/auth.md)