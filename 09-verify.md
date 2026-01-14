# 09 | Kubernetes 클러스터 기능 검증

이전 배포 단계를 거쳐 Kubernetes가 정상적으로 작동하는 데 필요한 컴포넌트들을 성공적으로 설치했습니다. 따라서 이번 시간에는 Kubernetes 클러스터를 검증하고, 정상적으로 작동하는지 확인하여 클러스터가 성공적으로 배포되었는지 확인해보겠습니다.

**주의:** 특별히 명시되지 않은 경우, 본 문서의 모든 작업은 `k8s-01` 노드에서 수행되며, 이후 원격으로 파일을 배포하고 명령을 실행합니다.

## 노드 상태 확인

모두 `Ready` 상태이면 정상입니다.

```bash
$ kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
k8s-01   Ready    <none>   3h49m   v1.33.3
k8s-02   Ready    <none>   3h49m   v1.33.3
```

## 테스트 파일 생성

테스트용 `nginx-ds` DaemonSet과 `nginx-ds` Service를 생성합니다

```yaml
cd /opt/k8s/work
cat > nginx-ds.yml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-ds
  labels:
    app: nginx-ds
spec:
  type: NodePort
  selector:
    app: nginx-ds
  ports:
  - name: http
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      app: nginx-ds
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:
      - name: my-nginx
        image: nginx:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
EOF
```

DaemonSet 및 Service 리소스 생성

```bash
$ kubectl create -f nginx-ds.yml
```

## 각 노드의 Pod IP 연결성 확인

```bash
$ kubectl get pods -o wide -l app=nginx-ds
NAME             READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
nginx-ds-8nf5p   1/1     Running   0          36s   10.0.1.20    k8s-02   <none>           <none>
nginx-ds-9g827   1/1     Running   0          36s   10.0.0.163   k8s-01   <none>           <none>
```
모든 Node에서 위 2개의 Pod IP로 각각 ping을 보내 연결되는지 확인합니다

```bash
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh ${node_ip} "ping -c 1 10.0.1.20"
    ssh ${node_ip} "ping -c 1 10.0.0.163"
  done
```

## Service IP 및 포트 접근성 확인

```bash
$ kubectl get svc -l app=nginx-ds
NAME       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-ds   NodePort   10.254.10.91   <none>        80:31427/TCP   2m43s

```

확인 결과:
● Service Cluster IP: `10.254.10.91`
● 서비스 포트: `80`
● NodePort 포트: `31427`

모든 Node에서 Service IP로 curl 실행

```bash
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh ${node_ip} "curl -s 10.254.10.91"
  done

```

Nginx 환영 페이지 내용이 출력되어야 합니다:

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## 서비스의 NodePort 접근성 확인

모든 Node에서 실행

```bash
source /opt/k8s/bin/environment.sh
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh ${node_ip} "curl -s ${node_ip}:31427"
  done
```

Nginx 환영 페이지 내용이 출력되어야 합니다.