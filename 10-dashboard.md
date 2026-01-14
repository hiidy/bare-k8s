# 10 | Addons 배포 (1): Kubernetes Dashboard 배포

Kubernetes 클러스터를 운영하는 과정에서 `kube-apiserver`가 제공하는 REST API 인터페이스를 통해 작업할 수 있습니다. 또한 `kubectl` 명령줄 도구를 사용할 수도 있습니다. 하지만 더 편리하고 간단한 또 다른 방법은 프론트엔드(웹 UI)를 통해 조작하는 것입니다.

Kubernetes는 클러스터에 시각적으로 액세스할 수 있는 공식 dashboard 컴포넌트를 제공합니다. 이번 절에서는 Kubernetes dashboard 컴포넌트를 배포해 보겠습니다.

## Helm을 사용한 Dashboard 설치

Dashboard를 설치하는 가장 편리한 방법은 Helm을 사용하는 것입니다. 다음 명령을 실행하여 Helm으로 dashboard를 설치합니다

```bash
# 1. kubernetes-dashboard Helm 저장소 추가
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

# 2. Dashboard 설치
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard

# 3. Dashboard 컨트롤 플레인으로 트래픽 포워딩
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 9443:443 --address 0.0.0.0
```

모든 Pod가 `Running` 상태가 될 때까지 기다립니다

```bash
$ kubectl -n kubernetes-dashboard get pods
NAME                                                   READY   STATUS    RESTARTS   AGE
kubernetes-dashboard-api-94758c594-wh9zw               1/1     Running   0          114s
kubernetes-dashboard-auth-5fdd4c5fcb-ptzrz             1/1     Running   0          114s
kubernetes-dashboard-kong-76f95967c6-rvzfh             1/1     Running   0          114s
kubernetes-dashboard-metrics-scraper-74d8cb664-72pn6   1/1     Running   0          114s
kubernetes-dashboard-web-59b8766cc-t5mnz               1/1     Running   0          114s
```

모든 Pod가 `Running` 상태가 되면, 브라우저를 열고 `https://public-ip:9443/`에 접속합니다. 화면은 다음과 같습니다



로그인 화면이 나타나면 Bearer token을 통해 로그인해야 합니다. 따라서 먼저 Bearer token을 생성해야 합니다.

## Dashboard 로그인용 Token 및 Kubeconfig 설정 파일 생성

Dashboard는 기본적으로 token 인증만 지원하며 (client 인증서 인증은 미지원), Kubeconfig 파일을 사용할 경우 해당 파일에 token을 기록해야 합니다.

공식 문서를 참고하여 생성할 수 있습니다: [Creating sample user](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md). 또는 아래 튜토리얼을 따라 생성할 수도 있습니다.

### 로그인용 Bearer token 생성

```bash
kubectl -n kubernetes-dashboard create sa dashboard-admin # 1. ServiceAccount 생성
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin # ClusterRoleBinding 생성
DASHBOARD_LOGIN_TOKEN=$(kubectl -n kubernetes-dashboard create token dashboard-admin)
echo ${DASHBOARD_LOGIN_TOKEN}
```

출력된 token을 사용하여 Dashboard에 로그인합니다. 다음 로그인 시 편의를 위해 token을 별도로 저장해 두는 것이 좋습니다. 로그인 후 화면은 다음과 같습니다



### Token을 사용하는 KubeConfig 파일 생성

생성 명령은 다음과 같습니다

```bash
source /opt/k8s/bin/environment.sh
# 클러스터 매개변수 설정
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/cert/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=dashboard.kubeconfig

# 클라이언트 인증 매개변수 설정, 위에서 생성한 Token 사용
kubectl config set-credentials dashboard_user \
  --token=${DASHBOARD_LOGIN_TOKEN} \
  --kubeconfig=dashboard.kubeconfig

# 컨텍스트 매개변수 설정
kubectl config set-context default \
  --cluster=kubernetes \
  --user=dashboard_user \
  --kubeconfig=dashboard.kubeconfig

# 기본 컨텍스트 설정
kubectl config use-context default --kubeconfig=dashboard.kubeconfig

```

생성된 `dashboard.kubeconfig`를 사용하여 Dashboard에 로그인합니다.

## 참고 문헌

1. [https://github.com/kubernetes/dashboard/wiki/Access-control](https://github.com/kubernetes/dashboard/wiki/Access-control)
2. [https://github.com/kubernetes/dashboard/issues/2558](https://github.com/kubernetes/dashboard/issues/2558)
3. [https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
4. [https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.7.X-and-above](https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.7.X-and-above)
5. [https://github.com/kubernetes/dashboard/issues/2540](https://github.com/kubernetes/dashboard/issues/2540)