쿠버네티스 실습을 해보려고 컨트롤러 노드 1개와 워커 노드 2개를 준비했다.

마스터 노드에서 `kubeadm init`으로 초기화 후 토큰을 생성한 뒤에 나오는 `kubeadm join`을 워커 노드에 그대로 복붙하면 되는데, 이게 워커 노드가 두 개라서 할만하지, 100개 이상 되버리면 어떻게 하지? 라는 생각에 다른 방법이 있나 찾아봤다.

당연히 개발 형님들이 워커 노드에 자동으로 쿠버네티스를 설치해주는 도구를 만들어놨다.

- 워커 노드에는 `kubeadm`, `kube-proxy`, `kubelet`, `containerd`가 설치되어 있어야 한다.

각 요소들의 역할은 다음과 같다.

### containerd
컨테이너 런타임으로, 도커에서 컨테이너를 실행할 때 사용하는 것과 같다.

쿠버네티스는 원래 `CRI(Container Runtime Interface)`로 도커를 사용하다가, 최근 버전에는 따로 `containerd`를 사용하도록 변경되었다.

- 실제로 nginx, redis와 같은 컨테이너가 있는 파드를 실행하는 역할
- `kubelet`이 런타임에게 "이 파드를 실행해" 라고 요청

### kubelet
- 워커 노드의 핵심 에이전트
- 컨트롤 플레인에서 파드를 스케줄하면, 해당 노드에서 kubelet이 실행한다.
- 파드 상태 보고도 kubelet이 담당

### kube-proxy

- 서비스 IP/Port 를 클러스터 네트워크로 연결
- 파드와 외부 서비스와의 통신을 대리해주는 역할

---

쿠버네티스 자동 설치 도구들로 여러 개가 있는데 나는 그 중 `kubespray`를 이용했다. 

kubespray는 `Ansible` 을 이용해 k8s 클러스터를 설치한다.

물론입니다. 지금까지 **Kubespray를 사용해 쿠버네티스 클러스터를 구축한 전체 과정**을 블로그 포스팅용으로 깔끔하게 정리해드릴게요. 초심자 기준으로 단계별 설명과 함께 각 명령어를 포함했습니다.

---

# 설치 과정

## 📌 노드 구성

* 클러스터 구성

  * `k8s-controller` (마스터): 10.0.10.172
  * `k8s-worker-1`: 10.0.10.140
  * `k8s-worker-2`: 10.0.10.89
* 모든 노드에 `ubuntu` 계정, 비밀번호 기반 SSH 로그인 가능

---

## 1️⃣ 사전 준비 (컨트롤러 노드에서 실행)

### 필요한 패키지 설치

```bash
sudo apt update
sudo apt install -y python3-pip python3-dev libffi-dev libssl-dev git
pip3 install --upgrade pip
```

### Kubespray 클론 및 요구 사항 설치

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
sudo pip3 install -r requirements.txt
```

---

## 2️⃣ 인벤토리 설정

### 인벤토리 템플릿 복사

```bash
cp -rfp inventory/sample inventory/mycluster
```

### 호스트 구성 파일 생성 (`inventory/mycluster/hosts.yaml`)

```yaml
all:
  hosts:
    k8s-controller:
      ansible_host: 10.0.10.172
      ip: 10.0.10.172
      access_ip: 10.0.10.172
    k8s-worker-1:
      ansible_host: 10.0.10.140
      ip: 10.0.10.140
      access_ip: 10.0.10.140
    k8s-worker-2:
      ansible_host: 10.0.10.89
      ip: 10.0.10.89
      access_ip: 10.0.10.89

  children:
    kube_control_plane:
      hosts:
        k8s-controller:
    kube_node:
      hosts:
        k8s-worker-1:
        k8s-worker-2:
    etcd:
      hosts:
        k8s-controller:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```

---

## 3️⃣ SSH 인증 키 배포

컨트롤러 노드에서 워커 노드로 SSH 키 복사:

```bash
ssh-keygen -t rsa  # (없으면 생성)
ssh-copy-id ubuntu@10.0.10.140
ssh-copy-id ubuntu@10.0.10.89
```

**ubespray는 containerd나 crio 같은 컨테이너 런타임과 통신하기 위해 crictl 명령어를 사용**

VERSION="v1.28.0"
curl -LO https://github.com/kubernetes-sigs/cri-tools/releases/download/${VERSION}/crictl-${VERSION}-linux-amd64.tar.gz
sudo tar -C /usr/local/bin -xzvf crictl-${VERSION}-linux-amd64.tar.gz
rm crictl-${VERSION}-linux-amd64.tar.gz

---

## 4️⃣ 클러스터 설치 시작

```bash
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root cluster.yml
```

---

## 5️⃣ 설치 후 점검

### 클러스터 노드 확인

```bash
kubectl get nodes
```

### 시스템 파드 상태 확인

```bash
kubectl get pods -A
```

---

## 6️⃣ 테스트: Nginx 배포

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get svc nginx
```

```bash
curl http://<워커노드 IP>:<할당된 NodePort>
```

---

## 7️⃣ etcd 문제 해결 (선택적)

> Kubespray 설치 후 `etcd` 서비스가 수동 구성된 경우

### etcd 환경 설정 (`/etc/etcd.env`)

```bash
ETCD_NAME=k8s-controller
ETCD_DATA_DIR=/var/lib/etcd
ETCD_INITIAL_CLUSTER=k8s-controller=https://10.0.10.172:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_LISTEN_PEER_URLS=https://10.0.10.172:2380
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://10.0.10.172:2380
ETCD_ADVERTISE_CLIENT_URLS=https://10.0.10.172:2379
ETCD_LISTEN_CLIENT_URLS=https://10.0.10.172:2379,https://127.0.0.1:2379
ETCD_CLIENT_CERT_AUTH=true
ETCD_TRUSTED_CA_FILE=/etc/ssl/etcd/ssl/ca.pem
ETCD_CERT_FILE=/etc/ssl/etcd/ssl/member-k8s-controller.pem
ETCD_KEY_FILE=/etc/ssl/etcd/ssl/member-k8s-controller-key.pem
ETCD_PEER_CLIENT_CERT_AUTH=true
ETCD_PEER_TRUSTED_CA_FILE=/etc/ssl/etcd/ssl/ca.pem
ETCD_PEER_CERT_FILE=/etc/ssl/etcd/ssl/member-k8s-controller.pem
ETCD_PEER_KEY_FILE=/etc/ssl/etcd/ssl/member-k8s-controller-key.pem
```

### systemd 서비스 파일 (`/etc/systemd/system/etcd.service`)

```ini
[Unit]
Description=etcd
After=network.target

[Service]
Type=notify
User=root
EnvironmentFile=/etc/etcd.env
ExecStart=/usr/local/bin/etcd
NotifyAccess=all
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
```

### 적용 및 실행

```bash
sudo systemctl daemon-reload
sudo systemctl restart etcd
sudo systemctl status etcd
```

> 에러 메시지: `bind: address already in use` 발생 시 → 기존 etcd 프로세스 종료

```bash
sudo lsof -i :2380
sudo pkill -f etcd
sudo systemctl restart etcd
```

---
