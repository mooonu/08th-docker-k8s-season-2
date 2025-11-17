
## k8s 아키텍처의 이해

### 물리적 계층
실제 하드웨어 자원으로 구성된다
- Master Node: 클러스터 전체를 관리하고 제어하는 노드
- Worker Node: 실제 컨테이너 워크로드가 실행되는 노드

### 논리적 계층
추상화된 리소스들로 구성된다
- Namespace: 리소스를 논리적으로 분리하는 단위
- Service: Pod들을 네트워크로 노출시키는 추상화 계층
- Ingress: 외부에서 클러스터 내부 서비스로 접근할 수 있게 하는 규칙
- Pods: 컨테이너를 실행하는 가장 작은 배포 단위

## k8s 핵심 컴포넌트 (물리적 계층 관점)

### Master Node

**kube-apiserver**
- k8s API를 노출하는 쿠버네티스 컨트롤 플레인 컴포넌트
	- `kubectl` 명령어는 이 API 서버와 통신
	- 오직 kube-apiserver만이 etcd에 직접 접근 가능

**etcd**
- 클러스터의 모든 데이터를 저장하는 분산 key-value 저장소
- etcd가 손상되면 클러스터 전체 정보가 손실된다

**kube-scheduler**
- 노드가 배정되지 않은 새로 생성된 파드를 감지하고, 실행할 워커 노드를 선택하는 컨트롤 플레인 컴포넌트

**kube-controller-manager**
- 컨트롤러 프로세스를 실행하는 컨트롤 플레인 컴포넌트 (여러 컨트롤러의 집합체)
- 각 컨트롤러는 분리된 프로세스이지만, 복잡성을 낮추기 위해 모두 단일 바이너리로 컴파일되고, 단일 프로세스 내에서 실행된다.

> 컨트롤 플레인 컴포넌트란?
> 클러스터 전체를 관리하고 제어하는 마스터 노드의 
> 핵심 컴포넌트 집합
> - kube-apiserver
> - etcd
> - kube-scheduler
> - kube-controller-manager

### Worker Node

**kubelet**
- 클러스터의 각 노드에서 실행되는 에이전트
- kube-apiserver와 통신하며 Pod에서 컨테이너가 확실하게 동작하도록 관리
	- Pod와 컨테이너의 상태를 모니터링 (Health Check)

**kube-proxy**
- 클러스터 네트워크 규칙을 관리
- 클라이언트 요청을 적절한 Pod로 전달하는 네트워크 프록시
- iptables를 활용해 트래픽 라우팅 규칙 설정
- Service의 Cluster IP로 들어온 요청을 실제 Pod IP로 전달

## Pod 실행 과정

`kubectl apply -f deployment.yaml`

### 흐름
kubectl → kube-apiserver (HTTPS) → 인증/인가 → Validation → etcd에 저장

### 과정
- kube-controller-manager 안의 Deployment Controller가 etcd 감시 (api-server 통해서) 
	- 이후 ReplicaSet 생성, 이를 api-server를 통해 etcd에 저장
- kube-controller-manager 안의 ReplicaSet Controller에서도 etcd를 감시하다가 필요한 pod 객체를 api-server를 통해 etcd에 저장 (해당 객체는 pending 상태이며 어떤 노드에 갈지 정하지 않은 상태임)
- kube-scheduler에서 이 스케쥴링되지 않은 etcd를 감시하다가 pending 상태를 본 후 파드가 생성될 노드를 지정하게 된다. 마찬가지로 api-server를 통해 etcd에 저장
- kubelet은 계속 etcd를 감시하다가 자기 노드에 할당된 pod를 발견하면 pod를 실행하게 되는 구조조

## Service

Service는 **추상적인 개념**이자 **설정 정보**
- Service = iptables/IPVS 규칙 + 가상 IP

### 타입
- ClusterIP: 기본 동일 클러스터에서만 사용
- NodePort: 서비스 하나에 모든 노드의 지정된 포트를 할당
- LoadBalancer: public cloud에서 많이 사용

### 동작 과정
1. Service를 생성하면 API Server가 etcd에 해당 정보들을 저장한다.
2. kube-controller-manager의 Endpoint Controller가 즉각 Service 생성을 감지하고, 클러스터에 동일한 라벨의 pod들을 검색 (예: `app: backend`)
3. Pod의 해당 ip들을 수집한 후 endpoint 리소스를 생성하면서 etcd에 endpoint 객체를 저장한다.

## kube-proxy의 동작
Service와 Pod를 실제로 연결하는 핵심 컴포넌트:

1. Service Added 이벤트 감지
    - kube-apiserver를 통해 새로운 Service 생성 감지
    - Cluster IP로 들어오는 요청을 처리할 iptables 규칙 생성
2. Endpoint Added 이벤트 감지
    - 새로운 Pod가 Service에 추가됨을 감지
    - Cluster IP로 들어온 트래픽을 실제 Pod IP로 전달하는 규칙 추가

> kube-apiserver는 
> Linux 커널과 유사한 역할로, 모든 이벤트의 중심에 있음

## ingress

- 가장 많이 사용되는 것은 Nginx Ingress Controller
	- 트래픽 흐름
		- 사용자 - LB - Ingress Controller(Nginx) - Service(Cluster IP) - Pod(애플리케이션 컨테이너)

### 참고 자료
- [k8s ingress nginx github](https://github.com/kubernetes/ingress-nginx/blob/main/deploy/static/provider/cloud/deploy.yaml)

### 배포 전략
- Deployment: 일반적인 애플리케이션 배포 방식 (일부 노드에만 존재)
- DaemonSet: 모든 노드에 하나씩 배포 (전체 노드에 존재)