# k8s 클러스터 노드 구성하기


### Master, Worker 노드 공통 구성
1. swap off하기
    
    ```jsx
    #swapoff -a
    #sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
    ```
2. CRI 설치 전 네트워크 구성하기 (런타임을 사용 가능한 상태로 만들기 위한 사전 설정)

```jsx
// https://kubernetes.io/ko/docs/setup/production-environment/container-runtimes/
echo '======== [6] 컨테이너 런타임 설치 ========'
echo '======== [6-1] 컨테이너 런타임 설치 전 사전작업 ========'
echo '======== [6-1] iptable 세팅 ========'
cat <<EOF |tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF |tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

3. CRI 설치하기

```jsx
// https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository
// Install using the apt repository
// Before you install Docker Engine for the first time on a new host machine, you need to set up the Docker apt repository. Afterward, you can install and update Docker from the repository.

// Set up Docker's apt repository.
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

// Install the Docker packages.
// Specific version - 호환버전은 공식문서 참고!
// https://containerd.io/releases/
// To install a specific version of Docker Engine, start by listing the available versions in the repository:
apt list --all-versions docker-ce
docker-ce/noble 5:29.1.2-1~ubuntu.24.04~noble <arch>
docker-ce/noble 5:29.1.1-1~ubuntu.24.04~noble <arch>
...

// kubernetes 버전 1.34를 기준으로 버전 고정하기
sudo apt install -y \
  containerd.io=2.1.5-1~ubuntu.24.04~noble \
  docker-ce=5:27.3.1-1~ubuntu.24.04~noble \
  docker-ce-cli=5:27.3.1-1~ubuntu.24.04~noble \
  docker-buildx-plugin=0.16.2-1~ubuntu.24.04~noble \
  docker-compose-plugin=2.29.7-1~ubuntu.24.04~noble

sudo apt-mark hold \
  containerd.io \
  docker-ce \
  docker-ce-cli \
  docker-buildx-plugin \
  docker-compose-plugin

// Latest
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

```

4. **cgroup 드라이버 설정하기**
    1. default가 cgroupfs이지만, OS에서 init을 systemd로 사용한다면 호환성을위해 systemd로 변경해주기
    
    ```jsx
    #ps -p 1 -o comm=
    // systemd인 경우 호환성을 위해 변경해주기
    // Kubernetes 공식문서에서 kubelet 과 컨테이너 런타임(containerd, CRI-O)의 cgroup driver를 systemd 로 일치시킬 것을 권장
    ```
    

systemd cgroup 드라이버 환경 설정하기 

```jsx
//etc/containerd/config.toml 의 systemd cgroup 드라이버를 runc 에서 사용하려면, 다음과 같이 설정한다.
sudo rm /etc/containerd/config.toml
sudo containerd config default | sudo tee /etc/containerd/config.toml

// /etc/containerd/config.toml에서 수정하기
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
    
# systemctl restart containerd 
    
// 스크립트화
// containerd config default > /etc/containerd/config.toml
sudo rm /etc/containerd/config.toml
sudo containerd config default | sudo tee /etc/containerd/config.toml
sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
```

5. kubeadm 설치하기

```jsx
// 사전작업
sudo apt-get update
# apt-transport-https는 더미 패키지일 수 있다. 그렇다면 해당 패키지를 건너뛸 수 있다
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
//쿠버네티스 패키지 리포지터리용 공개 샤이닝 키
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

// 쿠버네티스 apt 리포지터리를 추가
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

6. 쿠버네티스 편의기능

```jsx
// kubectl 자동완성 기능
// bash-completion 설치
sudo apt update
sudo apt install -y bash-completion

// kubectl 자동완성 설정
echo "source <(kubectl completion bash)" >> ~/.bashrc

// kubectl 단축명령(alias) 추가
echo 'alias k=kubectl' >> ~/.bashrc

// 단축명령에도 자동완성 적용
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc

// 적용
source ~/.bashrc
```


### Master 구성 작업
1. 클러스터 init 구성
```jsx
// Phase별 args는 [공식문서 참고](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)
kubeadm init --pod-network-cidr=172.31.32.0/19 --apiserver-advertise-address 172.31.0.7
```

2. kubectl 설정

```jsx
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

// join 정보 저장
kubeadm join 172.31.0.7:6443 --token puxczb.h6nl5k7ddse2s45e \
	--discovery-token-ca-cert-hash sha256:4886b22b58b8d965f55dcbe1121075946a73818b23277973f10acb87bbf4b75c
```

3. 파드 네트워크 설치 (calico)

```jsx
// 지원 버전과 필요 네트워크 포트 확인
//https://archive-os-3-28.netlify.app/calico/3.28/getting-started/kubernetes/requirements
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.5/manifests/tigera-operator.yaml
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.28.5/manifests/custom-resources.yaml
// custom-resources.yaml에서 cidr을 수정

```
