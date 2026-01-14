# vanilla-k8s

kubeadm 없이 바이너리 직접 설치 방식으로 Kubernetes 클러스터를 구성하는 프로젝트입니다. Debian 12을 기준으로 2노드(혼합 etcd/master/worker) 구성을 다루며, 필요한 인증서, kubeconfig, systemd 유닛, 네트워크 설정을 단계별로 제공합니다.

## 목표

- 바이너리 직접 배포 방식으로 Kubernetes 클러스터를 구축
- x509 인증서 기반 보안 통신
- control plane 고가용성(2노드) 구성
- kube-proxy IPVS 모드
- Cilium 기반 CNI, CoreDNS 등 기본 애드온 구성

## 구성 요약

- OS: Debian 12 권장
- 노드: 2대 (etcd/master/worker 혼합)
- etcd: v3.6.2 (TLS 통신)
- control plane:
  - kube-apiserver: L4 nginx 프록시 기반 HA
  - kube-controller-manager: leader election
  - kube-scheduler: leader election
- kubelet: TLS bootstrap 및 자동 인증서 갱신
- kube-proxy: IPVS 모드
- 애드온: CoreDNS, metrics-server, Dashboard, EFK, Harbor

## 주요 컴포넌트 버전

| 컴포넌트 | 버전 |
| --- | --- |
| kubernetes | v1.33.3 |
| etcd | 3.6.2 |
| containerd | 2.1.3 |
| runc | v1.3.0 |
| cilium | 1.17.6 |
| coredns | 1.9.4 |
| dashboard | 1.5.0 |
| prometheus-operator | v0.76.2 |
| prometheus | v2.54.1 |
| grafana | 11.2.0 |
| nginx | 1.29.0 |
| nerdctl | v2.1.3 |
| elasticsearch | 8.5.1 |
| crictl | 1.33.0 |

## 네트워크/리소스 기본값

- Service CIDR: `10.254.0.0/16`
- Pod CIDR: `172.30.0.0/16`
- NodePort 범위: `30000-32767`
- Control plane endpoint: `https://127.0.0.1:8443` (kube-nginx)

## 설치 흐름

아래 문서 순서대로 진행합니다.

1. [01-component-versions-and-configuration.md](01-component-versions-and-configuration.md)
   - 버전, 요구 사항, 핵심 구성 전략
2. [02-system-init.md](02-system-init.md)
   - OS 초기화, 네트워크/커널 파라미터, 패키지, 환경 변수 설정
3. [03-ca-setup.md](03-ca-setup.md)
   - CA 및 기본 인증서 구성
4. [04-kubectl.md](04-kubectl.md)
   - kubectl 설치 및 admin kubeconfig 구성
5. [05-etcd.md](05-etcd.md)
   - etcd 클러스터 구성 및 검증
6. [06-master.md](06-master.md)
   - control plane 구성 (apiserver/controller-manager/scheduler)
7. [07-worker.md](07-worker.md)
   - 워커 노드 구성 (kubelet/kube-proxy/containerd) 및 kube-apiserver HA 프록시
8. [08-cilium.md](08-cilium.md)
   - Cilium + CoreDNS 배포 및 네트워크 검증

## 환경 설정 포인트

- `/opt/k8s/bin/environment.sh`에 클러스터 IP, 노드명, etcd endpoint 등을 정의
- 노드 추가 시 `NODE_IPS`, `NODE_NAMES`, `ETCD_ENDPOINTS`, `ETCD_NODES`, CSR hosts 항목을 함께 업데이트
- 모든 노드는 외부 네트워크 접근 가능해야 하며, 내부망 통신을 권장

## 주의 사항

- swap, firewall, SELinux 비활성화 필요
- 인증서 CN/O 조합 중복 시 인증 오류 발생 가능
- kube-apiserver는 insecure port(8080)를 사용하지 않음

## 참고

각 문서는 실 배포 명령을 포함하므로, 환경에 맞게 IP/호스트명을 변경한 뒤 실행하세요.
