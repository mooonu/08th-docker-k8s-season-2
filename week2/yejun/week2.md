## Ingress

쿠버네티스의 Ingress는 외부에서 쿠버네티스 클러스터 내부로 들어오는 네트워크 요청 즉, Ingress 트래픽을 어떻게 처리할지 정의할 수 있다. 기존 방식은 NodePort, Loadbalancer 타입의 서비스를 노출시켜 외부 사용자가 진입할 수 있도록 하였다.

하나의 클러스터에서 여러 가지 서비스를 운영한다면 외부 연결을 어떻게 할 수 있는지?

NodePort를 이용하면 서비스 개수만큼 포트를 오픈하고 사용자에게 어떤 포트인지 알려줘야 합니다

쿠버네티스에서 외부에서 접근 가능한 가장 기본적인 방식이 **NodePort**

<img width="749" height="458" alt="스크린샷 2025-12-02 오후 9 48 18" src="https://github.com/user-attachments/assets/fe575017-be46-4e79-a8fd-f01729a4b93a" />

하지만 **서비스가 여러 개일 경우 다음과 같은 문제가 발생**

### **서비스 개수만큼 서로 다른 포트를 열어야 한다**

- 각 서비스는 **30000~32767 범위의 포트(NodePort)** 중 하나를 배정받음
- 서비스가 많아질수록 외부에 노출해야 하는 포트도 계속 증가함
    
    → `service A: 30080`, `service B: 30081`, `service C: 30082` …
    

### **사용자에게 각 서비스의 포트를 알려줘야 한다**

- 사용자는 어떤 서비스인지 알고 해당 NodePort로 직접 접근해야 함
    
    ```
    http://<노드 IP>:30080  → 서비스 A
    http://<노드 IP>:30081  → 서비스 B
    ```
    

이 방식은 **확장성이 떨어지고**,

**도메인 기반 라우팅(URL path, host 기반 라우팅)이 불가능**하다는 한계가 있음.

<img width="1478" height="530" alt="image" src="https://github.com/user-attachments/assets/704a0a32-5cf6-4708-8ee4-ab4fc3c1fce2" />

외부로부터 들어오는 요청에 대한 로드 밸런싱, TLS/SSL 인증서 처리, 특정 HTTP 경로의 라우팅 등을 Ingress를 통해 정의할 수 있다. 

외부 요청을 어떻게 처리할 것인지를 정의하는 집합인 Ingress를 정의한 뒤, 이를 Ingress Controller라고 부르는 **인그레스 매니지드 로드밸런서**에 적용함으로써 추상화된 단계에서 서비스 처리 로직을 정의할 수 있다. 

### 잦은 Nat로 인한 성능이슈

Ingress Controller가 가지고 있는 라우팅 정보는 서비스를 향하는데, 서비스에서 Endpoints를 노출하는 설정과 Ingress Controller가 해당 endpoints를 읽어와 저장할 수 있다면 불필요한 Nat를 줄일 수 있고 이는 오버헤드 감소 효과를 가져올 수 있음.

<img width="888" height="792" alt="스크린샷 2025-12-02 오후 9 50 27" src="https://github.com/user-attachments/assets/74873897-8d96-444e-8ce0-ff24975d3356" />

1. Ingress → Service 를 파싱
2. Service → Endpoints(혹은 EndpointsSlice)를 조회
3. 각 Pod의 IP:Port 리스트를 NGINX/HAProxy의 **백엔드 서버 엔트리로 직접 구성**
    1. --feature-gates=EndpointSlice=true,EndpointSliceProxying=true

즉, **Service ClusterIP로 트래픽을 보내지 않는다.**

```jsx
 Client → Ingress Controller(Pod) → Pod(Endpoints 직접 연결)
 [확인방법]
 # kubectl get endpointslice -n default
 # kubectl exec -it <controller-pod> -n ingress-nginx -- cat /etc/nginx/nginx.conf
upstream default-my-app-80 {
    server 10.244.1.5:8080;
    server 10.244.2.7:8080;
    server 10.244.3.9:8080;
}

Ingress Controller Pod의 IP가 source이면 Nat 발생하지 않음
tcpdump -i eth0 host <client pod ip>

```

### 외부 클라이언트와의 연결

<img width="744" height="577" alt="스크린샷 2025-12-02 오후 9 51 25" src="https://github.com/user-attachments/assets/2533533f-6a18-4f07-b856-647b889af360" />

외부망에 Nodeport로 접근할 수 있는 Edge Component를 구성하여 연결할 수 있음

```jsx
[외부 사용자] → [Edge LB] → [NodePort] → [Ingress Controller]
```

이러한 구성은 **클라이언트의 Source IP를 유지할 수 없다.** 모든 요청은 엣지 장비의 IP로 보이게 되는 문제가 있음

Proxy Protocol을 통해 헤더에 Source IP를 유지할 수 있음

[구성 방법]

LB에서 send-proxy옵션을 통한 X-Real-Ip와 같은 필드를 정의한다.

Ingress-controller의 [ConfigMap](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#use-proxy-protocol)을 통해 Nginx의 use-proxy-protocol 옵션을 yes로 활용한다.

```jsx
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
data:
  use-proxy-protocol: "true"
```

### Ingress-Nginx의 Retirement, 앞으로의 표준 Gateway API와 Service Mesh
