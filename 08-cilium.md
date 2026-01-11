# 08 | 네트워크 컴포넌트 Cilium + CoreDNS 배포


## Cilium 배포

Cilium을 설치하기 전에 먼저 `kube-proxy`가 설치되어 있어야 합니다.

Cilium 공식 사이트에서는 상세한 설치 문서를 제공합니다: [Cilium Quick Installation](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/). 공식 문서를 따라 설치해도 되고, 이 튜토리얼을 계속 따라 해도 됩니다.

### Cilium 설치 명령 (Cilium CLI 설치)

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm -f cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

### Cilium 네트워크 플러그인 설치

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo add cilium https://helm.cilium.io/
helm repo update
helm install cilium cilium/cilium --namespace kube-system --set image.tag=1.16.2
```

위 명령어를 실행하면 클러스터에 Cilium 네트워크 플러그인이 설치됩니다. 다음 명령어로 설치 진행 상황을 확인할 수 있습니다.

```bash
$ kubectl -n kube-system get pods|grep cilium
cilium-envoy-2g75d                1/1     Running   0              171m
cilium-envoy-jbwmt                1/1     Running   0              171m
cilium-glf5n                      1/1     Running   0              171m
cilium-kgjrc                      1/1     Running   0              171m
cilium-operator-6d897dfd6-hwg9c   1/1     Running   7 (160m ago)   4d9h
cilium-operator-6d897dfd6-rwlsq   1/1     Running   7 (161m ago)   4d9h
```

Pod들이 모두 `Running` 상태라면 Cilium이 올바르게 설치된 것입니다. `helm install cilium cilium/cilium` 명령을 실행하면 Kubernetes 클러스터에 다음 컴포넌트들이 설치되는 것을 볼 수 있습니다.

* **cilium-operator (Deployment):** Cilium의 고급 기능을 조정하고 관리하는 역할을 합니다.
* **cilium (DaemonSet):** Cilium CNI 네트워크 플러그인입니다.
* **cilium-envoy (DaemonSet):** Envoy 프록시 Pod들로, Cilium과 통합되어 향상된 네트워크 기능을 제공합니다.

또한 다음 명령어를 통해서도 Cilium이 올바르게 설치되었는지 확인할 수 있습니다.

```bash
$ cilium status --wait
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium                   Desired: 2, Ready: 2/2, Available: 2/2
DaemonSet              cilium-envoy             Desired: 2, Ready: 2/2, Available: 2/2
Deployment             cilium-operator          Desired: 2, Ready: 2/2, Available: 2/2
Containers:            cilium                   Running: 2
                       cilium-envoy             Running: 2
                       cilium-operator          Running: 2
                       clustermesh-apiserver
                       hubble-relay
Cluster Pods:          0/0 managed by Cilium
Helm chart version:    1.18.5
Image versions         cilium             quay.io/cilium/cilium:1.17.6@sha256:2c92fb05962a346eaf0ce11b912ba434dc10bd54b9989e970416681f4a069628: 2
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.34.12-1765374555-6a93b0bbba8d6dc75b651cbafeedb062b2997716@sha256:3108521821c6922695ff1f6ef24b09026c94b195283f8bfbfc0fa49356a156e1: 2
                       cilium-operator    quay.io/cilium/operator-generic:v1.18.5@sha256:36c3f6f14c8ced7f45b40b0a927639894b44269dd653f9528e7a0dc363a4eb99: 2
```

## 클러스터 네트워크 검증

다음 명령어를 실행하여 2개의 테스트용 Pod를 생성합니다.

```bash
cd /opt/k8s/work
cat > dnsutils-for-cilium-test-ds.yaml <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: dnsutils-for-cilium-test
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      app: dnsutils-for-cilium-test
  template:
    metadata:
      labels:
        app: dnsutils-for-cilium-test
    spec:
      containers:
      - name: my-dnsutils
        image: gcr.io/kubernetes-e2e-test-images/dnsutils:1.3
        imagePullPolicy: IfNotPresent
        command:
          - tail
          - "-f"
          - "/dev/null"
        ports:
        - containerPort: 80
EOF
kubectl create -f dnsutils-for-cilium-test-ds.yaml
```

다음 명령어로 Pod 생성 상태를 확인합니다.

```bash
$ kubectl get pods -o wide -l app=dnsutils-for-cilium-test
NAME                             READY   STATUS    RESTARTS   AGE   IP          NODE     NOMINATED NODE   READINESS GATES
dnsutils-for-cilium-test-97ssk   1/1     Running   0          58s   10.0.1.78   k8s-02   <none>           <none>
dnsutils-for-cilium-test-m9gnf   1/1     Running   0          58s   10.0.0.71   k8s-01   <none>           <none>

```

Pod들이 모두 `Running` 상태가 되면, 한 Pod 내부에서 다른 Pod의 IP 주소로 ping 통신이 가능한지 테스트합니다.

```bash
$ kubectl exec -it dnsutils-for-cilium-test-97ssk -- ping 10.0.0.71
PING 10.0.0.71 (10.0.0.71): 56 data bytes
64 bytes from 10.0.0.71: seq=0 ttl=63 time=0.718 ms
64 bytes from 10.0.0.71: seq=1 ttl=63 time=0.415 ms
64 bytes from 10.0.0.71: seq=2 ttl=63 time=0.744 ms
...

```

## CoreDNS 배포

Kubernetes 클러스터 내에서 서비스 디스커버리(Service Discovery)를 구현하려면, 서비스 도메인 이름을 Pod IP로 해석하여 IP 주소를 통해 통신할 수 있도록 해주는 DNS 컴포넌트가 필수적입니다. 현재 CoreDNS는 Kubernetes 환경에서 가장 널리 사용되는 DNS 솔루션입니다.

이 섹션에서는 CoreDNS를 단계별로 배포하여 클러스터 내 서비스의 DNS 해석이 원활하게 이루어지도록 하는 방법을 소개하겠습니다.

### CoreDNS 다운로드 및 구성

```bash
cd /opt/k8s/work
git clone https://github.com/coredns/deployment.git
mv deployment coredns-deployment
```

### CoreDNS 배포

```bash
cd /opt/k8s/work/coredns-deployment/kubernetes
source /opt/k8s/bin/environment.sh
./deploy.sh -i ${CLUSTER_DNS_SVC_IP} -d ${CLUSTER_DNS_DOMAIN} | kubectl apply -f -
```

### CoreDNS 기능 확인

다음 3가지 방식을 통해 CoreDNS 기능이 정상인지 확인할 수 있습니다.

1. CoreDNS 컴포넌트가 정상적으로 실행 중인지 확인
2. Pod에서 Kubernetes 서비스에 접근
3. DNS 구성을 확인하고 DNS 해석(Resolution) 테스트

구체적인 작업은 다음과 같습니다.

#### 1. CoreDNS 컴포넌트 정상 실행 확인

```bash
$ kubectl get pods -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS   AGE
coredns-5bd9dd46c6-8kvb6   1/1     Running   0          39s
```

coredns Pod가 `Running` 상태이므로 coredns가 성공적으로 배포되었습니다.

#### 2. Pod에서 Kubernetes 서비스 접근

새로운 Deployment를 생성합니다.

```bash
cd /opt/k8s/work
cat > my-nginx.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
EOF
kubectl create -f my-nginx.yaml

```

해당 Deployment를 expose 하여 my-nginx 서비스를 생성합니다.

```bash
$ kubectl expose deploy my-nginx
service "my-nginx" exposed

$ kubectl get services my-nginx -o wide
NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE   SELECTOR
my-nginx   ClusterIP   10.254.27.60   <none>        80/TCP    30s   run=my-nginx
```

다른 Pod를 생성하여 `/etc/resolv.conf`에 kubelet이 구성한 `--cluster-dns`와 `--cluster-domain`이 포함되어 있는지 확인하고, `my-nginx` 서비스를 위에서 표시된 Cluster IP `10.254.67.218`로 해석할 수 있는지 확인합니다.

#### 3. DNS 구성 확인 및 DNS 해석 테스트

테스트용 `dnsutils` Pod를 생성합니다.

```bash
cd /opt/k8s/work
cat > dnsutils-ds.yml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: dnsutils-ds
  labels:
    app: dnsutils-ds
spec:
  type: NodePort
  selector:
    app: dnsutils-ds
  ports:
  - name: http
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: dnsutils-ds
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      app: dnsutils-ds
  template:
    metadata:
      labels:
        app: dnsutils-ds
    spec:
      containers:
      - name: my-dnsutils
        image: gcr.io/kubernetes-e2e-test-images/dnsutils:1.3
        imagePullPolicy: IfNotPresent
        command:
          - tail
          - "-f"
          - "/dev/null"
        ports:
        - containerPort: 80
EOF
kubectl create -f dnsutils-ds.yml
```

테스트 Pod가 성공적으로 생성되었는지 확인합니다.

```bash
$ kubectl get pods -lapp=dnsutils-ds -o wide
kubectl exec -it dnsutils-ds-9zmbk -- cat /etc/resolv.conf
NAME                READY   STATUS    RESTARTS   AGE   IP          NODE     NOMINATED NODE   READINESS GATES
dnsutils-ds-4z9pj   1/1     Running   0          37s   10.0.1.42   k8s-02   <none>           <none>
dnsutils-ds-8x6hk   1/1     Running   0          37s   10.0.0.84   k8s-01   <none>           <none>
```

DNS 해석 구성이 올바른지 확인합니다.

```bash
$ kubectl exec -it dnsutils-ds-8x6hk -- cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.254.0.2
options ndots:5
```

DNS를 사용하여 Kubernetes 내부 서비스 도메인을 성공적으로 해석할 수 있는지 확인합니다.

```bash
$ kubectl exec -it dnsutils-ds-8x6hk -- nslookup kubernetes
Server:         10.254.0.2
Address:        10.254.0.2#53

Name:   kubernetes.default.svc.cluster.local
Address: 10.254.0.1

$ kubectl exec -it dnsutils-ds-8x6hk -- nslookup my-nginx
Server:         10.254.0.2
Address:        10.254.0.2#53

Name:   my-nginx.default.svc.cluster.local
Address: 10.254.27.60
```

DNS가 다음 2가지 유형의 Kubernetes 내부 서비스 도메인을 성공적으로 해석하는 것을 볼 수 있습니다.

* **Kubernetes 내장 서비스 도메인:** kubernetes -> 10.254.0.1
* **사용자가 생성한 외부 서비스 도메인:** my-nginx -> 10.254.27.60

DNS를 사용하여 공용 인터넷 도메인을 성공적으로 해석할 수 있는지 확인합니다.

```bash
$ kubectl exec -it dnsutils-ds-8x6hk -- nslookup www.google.com
Server:         10.254.0.2
Address:        10.254.0.2#53

Non-authoritative answer:
Name:   www.google.com
Address: 142.250.76.132
Name:   www.google.com
Address: 2404:6800:400a:808::2004
```

DNS가 `www.google.com`을 성공적으로 해석하고 있으며, 해석된 IP 주소는 `142.250.76.132`임을 확인할 수 있습니다.

## 더 포괄적인 Kubernetes 네트워크 테스트

위의 배포 과정을 통해 우리는 네트워크 플러그인과 DNS 컴포넌트를 성공적으로 배포했습니다. 이로써 Kubernetes의 네트워크 기능이 온전해졌으므로, Kubernetes 네트워크에 대해 더 포괄적인 테스트를 진행할 수 있습니다.

Cilium 공식 사이트에서는 Kubernetes 네트워크를 종합적으로 테스트할 수 있는 테스트 기능을 제공합니다. 다음 명령어를 실행하여 테스트할 수 있습니다.

```bash
$ cilium connectivity test
ℹ️  Monitor aggregation detected, will skip some flow validation steps
✨ [k8s-cluster] Creating namespace for connectivity check...

---------------------------------------------------------------------------------------------------------------------
📋 Test Report
---------------------------------------------------------------------------------------------------------------------
✅ 69/69 tests successful (0 warnings)

```

`69/69 tests successful (0 warnings)`는 모든 Kubernetes 클러스터 네트워크 연결성 테스트가 통과했음을 의미합니다.
****
**주의할 점:**

* Kubernetes 연결성 테스트는 하나 이상의 Pod에서 열린 파일(open files)이 너무 많아 배포에 실패할 수 있습니다. 만약 `too many open files`와 같은 오류가 발생하면, inotify 리소스 제한을 늘려야 할 수 있습니다.
* `cilium connectivity test`는 테스트 케이스가 매우 많으며, 네트워크 기능에 대해 더욱 포괄적이고 엄격한 테스트를 수행합니다. 그중 일부 케이스는 클러스터 구성이나 이미지 다운로드 실패 등의 이유로 실패할 수 있으며, 일부 실패한 테스트 케이스는 정상적인 클러스터 기능에 영향을 주지 않을 수도 있습니다. 따라서 테스트 결과는 참고용으로만 활용하세요.

## 참고 문헌

1. [https://community.infoblox.com/t5/Community-Blog/CoreDNS-for-Kubernetes-Service-Discovery/ba-p/8187](https://community.infoblox.com/t5/Community-Blog/CoreDNS-for-Kubernetes-Service-Discovery/ba-p/8187)
2. [https://coredns.io/2017/03/01/coredns-for-kubernetes-service-discovery-take-2/](https://coredns.io/2017/03/01/coredns-for-kubernetes-service-discovery-take-2/)
3. [https://www.cnblogs.com/boshen-hzb/p/7511432.html](https://www.cnblogs.com/boshen-hzb/p/7511432.html)
4. [https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns)