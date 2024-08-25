# 3장. 쿠버네티스 클러스터 배포
## 쿠버네티스 설치
<img src="/newbie/_static/images/kuar/ch03/nav_logo.svg" width="240px"/>

<!-- 
```{image} images/kuar/ch03/nav_logo.svg
:alt: Select Parameters
:width: 240px
:align: center
```
-->
<!-- https://stackoverflow.com/a/73306746 -->

이전 장에서 앱 컨테이너를 성공적으로 구축했다.

이번 장에서는 분산 시스템에 컨테이너를 배포하기 위한 클러스터 배포를 알아본다.

쿠버네티스 클러스터 관리는 복잡해서 퍼블릭 클라우드에서 제공하는 관리형 솔루션을 사용하면 좋다.

대신 비용을 지불해야하고 클라우드에 연결되어있어야 한다.

로컬 개발도 가능하며, 미니큐브나 docker-in-docker 기반 Kind를 사용할 수 있다.

미니큐브는 단일 노드만 실행 가능하며, Kind는 다중 노드 클러스터를 구축할 수 있다.

<br/>

### 퍼블릭 클라우드
#### 구글 클라우드 플랫폼(GCP)
<img src="/newbie/_static/images/kuar/ch03/google-cloud-logo.svg" width="240px" style="background: white; padding: 1rem;"/>

GKE(Google Kubernetes Engine)을 사용한다.
- [https://cloud.google.com/sdk/docs/install-sdk?hl=ko](https://cloud.google.com/sdk/docs/install-sdk?hl=ko)

```sh
$ gcloud auth login
# project number 쓰지 않도록 주의
$ gcloud config set project PROJECT_ID
$ gcloud config set compute/zone us-west1-a
$ gcloud components install gke-gcloud-auth-plugin
$ gcloud container clusters create kuar-cluster --num-nodes=3
$ gcloud container clusters list
$ gcloud container clusters get-credentials kuar-cluster --region=us-west1-a
```

```sh
$ k get namespaces
NAME                 STATUS   AGE
default              Active   3h53m
gke-managed-cim      Active   3h52m
gke-managed-system   Active   3h52m
gmp-public           Active   3h52m
gmp-system           Active   3h52m
kube-node-lease      Active   3h53m
kube-public          Active   3h53m
kube-system          Active   3h53m
```

- Troubleshooting
```sh
# .bashrc 등에 환경변수 설정하지 않을 시 executable file not found in $PATH
# couldn't get current server API group list 오류 발생
export PATH=".../google-cloud-sdk/bin:$PATH"
```

<!--
```
# export USE_GKE_GCLOUD_AUTH_PLUGIN=True
# gcloud components update
```

- 신뢰할 수 있는 IP 설정
```
gcloud container clusters update kuar-cluster \
    --enable-master-authorized-networks \
    --master-authorized-networks 8.8.8.8/32
```
-->

<br/>

#### 애저(AZ)
```sh
az group create --name=kuar --location=westus
az aks create --resoure-group=kuar --name=kuar-cluster
az aks get-credentials --resource-group=kuar --name=kuar-cluster
```

<br/>

#### 아마존 웹 서비스(AWS)
```
eksctl create cluster
# eksctl create cluster --help
```

### 로컬 환경
#### 미니큐브
<img src="/newbie/_static/images/kuar/ch03/minikube-logo-full.png" width="240px" style="background: white; padding: 0.5rem;"/>

간단한 단일 노드 클러스트를 구출할 수 있다.

도커 데스크톱이 이미 설치돼 있는 경우 단일 머신 쿠버 네티스가 함께 제공된다.

분산된 클러스터의 신뢰성은 제공하지 않는다.

특정 기능은 미니큐브에서 사용할 수 없거나 제한된 방식으로 동작한다.

미니큐브를 사용하려면 머신에 하이퍼바이저를 설치해야한다. ex. virtualbox

- [https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download)

```sh
# 시작
$ minikube start
# 중지
$ minikube stop
# 삭제
$ minikube delete
```

<br/>

#### Kind(도커 환경)
<img src="/newbie/_static/images/kuar/ch03/kind-logo.png" width="240px" style="background: white; padding: 0.5rem;"/>

여러 쿠버네티스 클러스터 노드를 시뮬레이션하고자 가상머신 대신 도커 컨테이너를 이용한다.

kind는 kubernetes in docker를 의미한다.

```sh
# 생성
$ kind create cluster --wait 5m
# 확인
$ kubectl cluster-info
# 삭제
$ kind delete cluster
```

- [https://kind.sigs.k8s.io/docs/user/quick-start/](https://kind.sigs.k8s.io/docs/user/quick-start/)

<br/>

## 쿠버네티스 클라이언트

kubectl은 공식적인 쿠버네티스 클라이언트다.

kubectl은 쿠버네티스 API와 상호작용하기 위한 커맨드라인 도구다.

kubectl을 사용해 파드, 레플리카셋(ReplicaSet) 및 서비스 같은 대부분의 쿠버네티스 객체를 관리할 수 있다.

<br/>

### 클러스터 상태 확인
```sh
$ kubectl version
Client Version: v1.31.0
Kustomize Version: v5.4.2
Server Version: v1.31.0
```

Server Version은 쿠버네티스 API의 버전이고,<br/>
Client Version은 kubectl의 버전이다.<br/>
[Kustomize](https://kustomize.io/)는 [쿠버네티스 리소스를 템플릿이나 DSL을 사용하지 않고 선언형으로 사용자화하는 도구](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/)이다.<br/>
Kustomize는 `kubectl -k ...`로 사용할 수 있는 built-in 솔루션이고<br/>
patch라는 방식을 통해 리소스의 특정 키/벨류를 새로운 값으로 바꿀 수 있다.

<br/>

```sh
$ kubectl get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE   ERROR
controller-manager   Healthy   ok        
scheduler            Healthy   ok        
etcd-0               Healthy   ok  
```

**controller-manager**는 클러스터의 동작을 제어하는 다양한 컨트롤러를 실행한다.<br/>
예를 들어 서비스의 모든 복제본에 대해 사용 가능 여부와 정상 동작 여부를 확인한다.

**scheduler**는 클러스터의 다른 노드에 다른 파드를 배치하는 역할을 담당한다.

**[etcd](https://etcd.io/)** 서버는 모든 API 객체가 저장된 클러스터의 저장소다.

- etcd 키를 조회해보자!

```sh
$ kubectl get po -n kube-system | grep etcd
$ kubectl exec -it etcd-my-cluster-control-plane -n kube-system -- /bin/sh
# https://stackoverflow.com/a/53519231
$ ETCDCTL_API=3 \
  etcdctl \
  --endpoints 127.0.0.1:2379 \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key \
  get / \
  --prefix \
  --keys-only

...
/registry/pods/kube-system/coredns-6f6b679f8f-4p56d
/registry/pods/kube-system/coredns-6f6b679f8f-8qsj6
/registry/pods/kube-system/etcd-my-cluster-control-plane
...

$ ETCDCTL_API=3 \
  etcdctl \
  --endpoints 127.0.0.1:2379 \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key \
  get /registry/deployments/kube-system/coredns \
  --prefix
k8s
apps/v1
Deployment
...
```

<br/>

### 쿠버네티스 노드 조회
```sh
$ kubectl get nodes
NAME                       STATUS   ROLES           AGE   VERSION
Kube0                      Ready    control-plane   19h   v1.31.0
Kube1                      Ready    <none>          19h   v1.31.0
Kube2                      Ready    <none>          19h   v1.31.0
Kube3                      Ready    <none>          19h   v1.31.0
```

쿠버네티스 노드는 control-plane 노드와 worker 노드로 구분된다.

control-plane 노드는 API 서버, 스케줄러 등과 같이 클러스터를 관리하는 작업이 실행된다.<br/>
control-plane 노드에는 스케줄링을 수행하지 않는데, 사용자 워크로드(workload; 작업부하)가 클러스터 전체 운영에 영향을 주지 않기 위함이다.

worker 노드는 사용자 컨테이너 등이 실행된다.

```sh
$ kubectl describe nodes Kube0
Name:               my-cluster-control-plane
Roles:              control-plane
Labels:             beta.kubernetes.io/arch=arm64
                    beta.kubernetes.io/os=linux
...
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Sun, 25 Aug 2024 22:04:33 +0900   Sun, 25 Aug 2024 02:13:27 +0900   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Sun, 25 Aug 2024 22:04:33 +0900   Sun, 25 Aug 2024 02:13:27 +0900   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Sun, 25 Aug 2024 22:04:33 +0900   Sun, 25 Aug 2024 02:13:27 +0900   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Sun, 25 Aug 2024 22:04:33 +0900   Sun, 25 Aug 2024 02:13:51 +0900   KubeletReady                 kubelet is posting ready status
...
Capacity:
  cpu:                2
  ephemeral-storage:  104266732Ki
  memory:             8104728Ki
  pods:               110
Allocatable:
  cpu:                2
  ephemeral-storage:  104266732Ki
  memory:             8104728Ki
  pods:               110
...
System Info:
  Kernel Version:             6.8.11-300.fc40.aarch64
  OS Image:                   Debian GNU/Linux 12 (bookworm)
  Operating System:           linux
  Architecture:               arm64
  Container Runtime Version:  containerd://1.7.18
  Kubelet Version:            v1.31.0
...
PodCIDR:                      10.244.0.0/24
PodCIDRs:                     10.244.0.0/24
...
Non-terminated Pods:          (10 in total)
  Namespace                   Name                                                CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                                ------------  ----------  ---------------  -------------  ---
  kube-system                 coredns-6f6b679f8f-4p56d                            100m (5%)     0 (0%)      70Mi (0%)        170Mi (2%)     20h
  kube-system                 coredns-6f6b679f8f-8qsj6                            100m (5%)     0 (0%)      70Mi (0%)        170Mi (2%)     20h
 
```

쿠버네티스는 머신에서 실행 중인 파드가 요청한 리소스의 요청(reuqest)과 제한(limit)을 추적할 수 있다.

파드에서 요청(reuqest)한 리소스는 노드에서 그 만큼의 사용을 보장해주지만<br/>
제한(limit)은 파드가 사용할 수 있는 리소스의 최대량이다.<sup>5장</sup>

파드의 제한(limit)은 요청(request)보다 적용 우선순위가 높을 수 있고,<br/>
이 경우 여분의 리소스가 최선 노력(best-effort) 기반으로 제공된다.

하지만 이 리소스가 노드에 존재하는 것은 보장되지 않는다.

<br/>

## 쿠버네티스 컴포넌트
쿠버네티스를 구성하는 많은 컴포넌트는 실제로 쿠버네티스 자체를 통해 배포된다.
해당 컴포넌트들은 `kube-system` 네임스페이스에서 실행된다.

### 쿠버네티스 프록시
쿠버네티스 프록시(proxy)는 쿠버네티스 클러스터 내의 부하 분산된 서비스(load-balanced services)로 네트워크 트래픽을 라우팅하는 역할을 한다.

프록시는 모든 노드에 위치한다.

쿠버네티스에는 데몬셋(DaemonSet)<sup>11장</sup>이라고 하는 API 객체가 있으며, 이 객체를 통해 프록시가 배포된다.

```sh
$ kubectl get daemonsets -n kube-system kube-proxy
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-proxy   4         4         4       4            4           kubernetes.io/os=linux   20h
```

<br/>

### 쿠버네티스 DNS

쿠버네티스는 DNS 서버를 실행해 클러스터에 정의된 서비스의 이름 지정과 검색 기능을 제공한다.

쿠버네티스 DNS는 복제된 서비스(replicated servies) 형태로 실행된다.

클러스터 크기에 따라 하나 이상의 DNS 서버가 실행된다.

쿠버네티스 DNS는 디플로이먼트(deployment) 형태로 실행되어 복제본(replicas)이 관리된다.

DNS 서버에 대한 로드밸런싱을 수행하는 쿠버네티스 서비스도 있다.

```sh
$ kubectl get deploy -n kube-system | grep dns
coredns   2/2     2            2           20h

$ kubectl get svc -n kube-system | grep dns
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   20h

$ kubectl exec -it ... -- /bin/sh
> $ cat /etc/resolv.conf 
> search default.svc.cluster.local svc.cluster.local cluster.local dns.podman
> nameserver 10.96.0.10
> options ndots:5
```

<br/>

### 쿠버네티스 UI
클라우드 서비스 사용 시 GUI는 보통 제공업체에서 제공된다.

제공업체에서 제공하지 않거나 클러스터 내의 GUI를 선호하는 경우 Kubernetes Dashboard를 설치할 수 있다.

- [https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)

```sh
# Add kubernetes-dashboard repository
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
# Deploy a Helm Release named "kubernetes-dashboard" using the kubernetes-dashboard chart
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard

# https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

kubectl -n kubernetes-dashboard create token admin-user
```

<br/>

## 요약

쿠버네티스 클러스터를 설치하고 생성된 클러스터를 탐색하기 위한 몇 가지 명령을 실행해봤다.

4장에서는 kubectl 도구를 활용하는 방법을 살펴본다.

