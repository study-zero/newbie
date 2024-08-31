# 4장. 공통 kubectl 명령
**kubectl** 커맨드라인 유틸리티는 쿠버네티스 API와 상호작용하는 강력한 도구다.

<br/>

## 네임스페이스
```{figure} /images/kuar/ch04/namespace.png
:alt: image
:width: 520px
```

**네임스페이스**는 쿠버네티스 객체들의 집합을 담고 있는 폴더와 같다.

kubectl은 기본적으로 default 네임스페이스와 상호작용한다.

다른 네임스페이스를 사용하고 싶다면 `--namespace` 플래그를 추가할 수 있다.

모든 네임스페이스와 상호작용 하고 싶다면 `--all-namespaces` 또는 `-A` 플래그를 추가해야 한다.

```sh
# 특정 네임스페이스 사용
kubectl get po --namespace=mystuff
# 모든 네임스페이스 사용
kubectl get po --all-namespaces
# 또는 kubectl -A
```

<br/>

## 컨텍스트
```{figure} /images/kuar/ch04/context.png
:alt: image
:width: 400px
```

**컨텍스트**는 세 가지 요소인 *클러스터*, *사용자*, *네임스페이스*로 이루어진다.

예를 들어 `dev-frontend` 컨텍스트는 "`development` 클러스터의 `frontend` 네임스페이스에 접근하는데 `developer` 사용자 자격증명을 사용하라고 알려준다." [#](https://kubernetes.io/ko/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0-%EC%82%AC%EC%9A%A9%EC%9E%90-%EC%BB%A8%ED%85%8D%EC%8A%A4%ED%8A%B8-%EC%A0%95%EC%9D%98)

컨텍스트는 주로 `$HOME/.kube/config` 경로에 존재하는 kubectl 컨피규레이션 파일<sup>kubeconfig</sup>에 기록돼있다.

특정 네임스페이스를 기본값으로 하는 컨텍스트는 아래와 같이 생성하고 사용할 수 있다.

```sh
# 컨텍스트 생성
kubectl config set-context my-context --namespace=mystuff
# 컨텍스트 사용
kubectl config use-context my-context
```

<br/>

## 쿠버네티스 API 객체 조회
```{figure} /images/kuar/ch04/kubernetes-api.png
:alt: image
:width: 540px
```

쿠버네티스의 모든 객체는 RESTful 리소스로 표현된다.

각 쿠버네티스 객체는 고유의 HTTP 경로에 존재한다.

아래와 같이 pods 목록을 받아볼 수 있다.

이는 `kubectl get pods`와 동일하다.

```console
// 쿠버네티스 API 프록시 설정(localhost:port)
$ kubectl proxy 8001
// 쿠버네티스 API 호출
$ curl -is 127.0.0.1:8001/api/v1/namespaces/default/pods
HTTP/1.1 200 OK
...
{
  "kind": "PodList",
  "apiVersion": "v1",
  ...
}
```

`kubectl get <리소스 이름>`을 실행할 경우 현재 네임스페이스에 존재하는 모든 리소스 목록을 조회할 수 있다.

특정 리소스를 조회하고자 할 경우에는 `kubectl get <리소스 이름> <객체 이름>`을 사용할 수 있다.

`--watch`를 사용하면 특정 리소스를 지속적으로 모니터링할 수 있다.

`kubectl`은 API 결과를 사람이 읽기 쉽게 가공하여 일부 세부 사항을 생략한다.

객체의 전체 정보를 얻고자 할 경우에는 `-o json` 또는 `-o yaml` 플래그를 사용해 해당 형식으로 조회할 수 있다.

헤더를 생략하고자 할 경우에는 `--no-headers` 플래그를 사용할 수 있다.

객체의 특정 필드를 추출할 때는 JSONPath 쿼리 언어를 사용할 수 있다.

```console
$ kubectl get pods podId -o jsonpath --template="{.status.podIP}{'\n'}"
$ kubectl get pods -o jsonpath --template="{.items[*].metadata.name}{'\n'}"
$ kubectl get pods -o jsonpath --template='{"name\timage\tpodIP\n"}{range .items[*]}{@.metadata.name}{"\t"}{@.spec.containers[*].image}{"\t"}{@.status.podIP}{"\n"}{end}'
name    image   podIP
pod0    image0:latest      10.244.1.1
pod1    image1:latest      10.244.1.2
pod2    image2:latest      10.244.1.3
```

쉼표로 구분된 타입 목록으로 여러 객체를 동시에 확인할 수 있다.

```sh
kubectl get pods,services
```

특정 객체의 자세한 정보를 얻고 싶을 경우 `describe` 명령을 사용할 수 있다.

```sh
kubectl describe <리소스 이름> <객체 이름>
```

쿠버네티스 객체 타입에 대해 지원되는 필드 목록을 보려면 `explain` 명령을 사용할 수 있다.

```sh
kubectl explain pods
```

```{note}
모든 쿠버네티스 객체 종류는 `kubectl api-resources`로 조회할 수 있다.
```

<br/>

## 쿠버네티스 객체 생성, 수정 삭제
```{figure} /images/kuar/ch04/apply.png
:alt: image
:width: 540px
```

쿠버네티스 API의 객체들은 JSON이나 YAML 파일로 표현된다.

YAML이나 JSON 파일들로 쿠버네티스 객체를 생성, 수정, 삭제할 수 있다.

`kubectl apply -f <파일명>`을 통해 객체 생성 및 수정이 가능하다.

수정 명령은 파일 내용과 다른 설정을 가진 객체들만 수정한다.

생성 중인 객체가 이미 클러스터에 존재할 경우 변경사항 없이 해당 명령은 종료된다.

```sh
# 객체 생성, 수정
kubectl apply -f obj.yaml
```

실제 변경없이 `apply` 명령의 적용 사항을 확인하려면 `--dry-run=client`(로컬에서 확인) 또는 `--dry-run=server`(서버에서 검증) 플래그를 사용할 수 있다.

`apply` 명령은 객체의 애노테이션에 이전 컨피규레이션 이력을 기록한다.

`apply` 명령에 `edit-last-applied`, `set-last-applied`, `view-last-applied` 명령을 함께 사용해 객체의 어노테이션 기록된 `last-applied-configuration`을 편집, 설정, 조회할 수 있다.

```sh
kubectl apply edit-last-applied deploy/myDeploy
kubectl apply set-last-applied deploy/myDeploy -f <파일명>
kubectl apply view-last-applied deploy/myDeploy
```

쿠버네티스 객체의 직접 편집은 `edit` 명령으로 가능하다.

`edit` 명령으로 편집된 파일을 저장하면 쿠버네티스 클러스터에 자동으로 업로드한다.

```sh
# 객체 직접 편집
kubectl edit <리소스 이름> <객체 이름>
```

객체 삭제는 `delete` 명령으로 가능하며 사용자의 확인을 받지 않는다.

```sh
# 객체 삭제
kubectl delete -f obj.yaml
# 객체 삭제
kubectl delete <리소스 이름> <객체 이름>
```

<br/>

## 객체 라벨링과 애노테이션
```{figure} /images/kuar/ch04/labels.png
:alt: image
:width: 400px
```

`label`과 `annotation`은 메타데이터라는 점이 동일하나 다음과 같은 차이점이 있다.

`label`은 `selector`로 활용되는 것처럼 쿠버네티스 운영에 활용되는 데이터이고,

`annotation`은 운영에 활용되지 않지만 사용자에게 필요한 정보나 3rd party가 사용할 수 있는 데이터이다.

라벨과 애노테이션은 덮어쓰기를 허용하지 않아 `--overwrite` 플래그를 사용하거나 삭제 후 재지정하여야한다.

자세한 내용은 6장에서 다룬다.

```sh
# 레이블 지정
kubectl label pods myPod color=red
# 레이블 삭제
kubectl label pods myPod color-
# 어노테이션 지정
kubectl annotate pods myPod color=blue
# 어노테이션 삭제
kubectl annotate pods myPod color-
```

<br/>

## 디버깅 명령
```{figure} /images/kuar/ch04/logging-architecture.png
:alt: image
:width: 400px
```

현재 동작 중인 컨테이너의 로그는 아래와 같이 확인할 수 있다.

```sh
# 파드 로그 조회
kubectl logs <파드 이름>
# 파드 로그 조회(컨테이너 지정)
kubectl logs <파드 이름> -c <컨테이너 이름>
# 파드 로그 조회(실시간)
kubectl logs <파드 이름> -f
```

```{note}

쿠버네티스에서 파드의 로그는 `stdout`의 redirect를 통해 kubelet 내에 저장된다. [#](https://stackoverflow.com/a/58718841/16783410)

쿠버네티스 1.2 버전 이상부터 [아래 명령어로 노드에 ssh 접속을 할 수 있다](https://kubernetes.io/docs/tasks/debug/debug-cluster/kubectl-node-debug/).

`k debug node/my-node -it --image=ubuntu`

호스트 파일은 `/host`에 마운트된다.

컨테이너 로그는 아래 위치에서 확인할 수 있다.

`cd /host/var/log/pods/...`

```

현재 실행 중인 컨테이너에 명령을 실행하고자 할 경우 `exec` 명령을 사용한다.

컨테이너에서 실행 중인 프로세스의 콘솔에 접근하려면 `attach` 명령을 사용한다.

```sh
kubectl exec -it <파드 이름> -- bash
kubectl attach -it <파드 이름>
```

```{note}
`attach` 명령 실행 시 아래와 같은 오류가 발생할 수 있다.

    error: Unable to use a TTY - container test did not allocate one

이 경우 아래와 같이 `.spec.template.spec.containers[].tty`와 `.spec.template.spec.containers[].stdin`를 `true`로 설정해야한다.

    spec: 
      containers: 
      - name: web 
        image: web:latest 
        tty: true 
        stdin: true

```

`cp` 명령을 사용해 컨테이너에 파일을 붙여넣거나 가져올 수 있다.

```sh
kubectl cp <파드 이름>:</path/to/remote/file> </path/to/local/file>
```

네트워크를 통해 파드에 접근하고자 할 경우 `port-forward` 명령을 사용해 로컬 머신에서 파드로 네트워크 트래픽을 전달 할 수 있다.

이는 퍼블릭 네트워크에 노출시키 않고 컨테이너로 안전하게 네트워크 트래픽을 터널링할 수 있다.

```sh
# 포트는 <로컬 머신 포트>:<원격 컨테이너 포드> 순서
kubectl port-forward <파드 이름> 8080:80
# 서비스 객체로 연결. 서비스 내의 단일 파드로만 요청이 전달됨을 유의.
kubectl port-forward svc/<서비스 이름> 8080:80
```

쿠버네티스 이벤트를 확인하려면 `events` 객체로 조회할 수 있다.

```sh
# 이벤트 조회
kubectl get events
# 실시간 이벤트 조회
kubectl get events --watch
# 모든 네임스페이스 이벤트 조회
kubectl get events -A
```

클러스터의 리소스 사용 현황 조회는 아래와 같이 할 수 있다.

```console
// 노드의 리소스 사용량 조회
$ kubectl top nodes
NAME                       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
my-cluster-control-plane   135m         6%     933Mi           11%       
my-cluster-worker          38m          1%     420Mi           5%        
my-cluster-worker2         43m          2%     411Mi           5%        
my-cluster-worker3         39m          1%     364Mi           4% 

// 파드의 리소스 사용량 조회
$ kubectl top pods
NAME      CPU(cores)   MEMORY(bytes)   
pod0      10m          51Mi            
pod1      9m           51Mi            
pod2      9m           51Mi            
pod3      9m           60Mi            
pod4      9m           67Mi            
pod5      11m          62Mi 
```

```{note}
`error: Metrics API not available` 오류가 발생할 경우 metrics server 설치가 필요하다.

- [kubernetes-sigs/metrics-server](https://github.com/kubernetes-sigs/metrics-server?tab=readme-ov-file#installation)

`kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`

설치 후에도 명령이 실행되지 않는다면 아래와 같이 `metrics-server` 객체의 args에 2개 항목을 추가해준다.
`kubectl edit deploy metrics-server -n kube-system`

      spec:
        containers:
        - args:
          - --kubelet-insecure-tls
          - --kubelet-preferred-address-types=InternalIP

```

<br/>

## 클러스터 관리

```{figure} /images/kuar/ch04/cordon-drain.png
:alt: image
:width: 400px
```

클러스터 관리를 위해 특정 노드를 차단(cordon)하고 비울(drain) 수 있다.

노드를 차단하면 해당 노드에 팟이 스케줄링되지 않는다.

노드에 대한 비우기를 설정하면 해당 노드에서 실행 중인 모든 파드가 제거된다.

수리나 업그레이드를 위해 물리 머신을 제거하려면 `cordon` 및 `drain` 수행 후 안전하게 제거할 수 있다.

머신이 수리되면 `uncordon` 명령으로 파드 스케줄링을 다시 활성화할 수 있다.

시스템 재시작과 같이 긴 시간이 소요되지 않을 경우 일반적으로 차단하거나 비울 필요는 없다.

```console
// 파드 스케줄링 중지
$ kubectl cordon my-node3
node/myNode cordoned

// 노드 상태 확인
$ kubectl get nodes
NAME               STATUS                     ROLES           AGE   VERSION
my-control-plane   Ready                      control-plane   47h   v1.31.0
my-node1           Ready                      <none>          47h   v1.31.0
my-node2           Ready                      <none>          47h   v1.31.0
my-node3           Ready,SchedulingDisabled   <none>          47h   v1.31.0

// 클러스터 내 파드 모두 제거
$ kubectl drain myNode

// 파드 스케줄링 재활성화
$ kubectl uncordon myNode

// 노드 상태 확인
$ kubectl get nodes
NAME               STATUS                     ROLES           AGE   VERSION
my-control-plane   Ready                      control-plane   47h   v1.31.0
my-node1           Ready                      <none>          47h   v1.31.0
my-node2           Ready                      <none>          47h   v1.31.0
my-node3           Ready                      <none>          47h   v1.31.0
```

<br/>

## 명령 자동 완성
`kubectl`은 셸과의 통합(integration)을 지원한다.

먼저 아래와 같은 패키지를 설치해야한다.

```sh
# macos
brew install bash-completion
# centos/redhat
yum install bash-completion
# debian/ubuntu
apt-get install bash-completion
```

위 패키지가 설치되었다면 아래 명령어로 자동 완성 기능을 추가할 수 있다.

`.bashrc`나 `.zshrc` 등에 추가하여 셸 실행 시 자동으로 추가할 수 있다.

```sh
# bash
source <(kubectl completion bash)
# zsh
source <(kubectl completion zsh)
```

<br/>

## 클러스터 조회의 대안
`kubectl`외에도 쿠버네티스 클러스터와 상호작용하기 위한 도구들이 있다.

- Visual Studio Code Extensions [#](https://marketplace.visualstudio.com/items?itemName=ms-kubernetes-tools.vscode-kubernetes-tools)
- IntelliJ Plugins [#](https://plugins.jetbrains.com/plugin/10485-kubernetes)
- Eclipse Tools [#](https://marketplace.eclipse.org/content/ekube)

관리형 쿠버네티스 사용 시 대부분 웹 기반 UX가 제공된다.

[Rancher dashboard](https://github.com/rancher/dashboard)와 [Headlamp project](https://github.com/headlamp-k8s/headlamp) 등 여러 오픈소스 GUI 대시보드가 있다.

<br/>

## 요약
`kubectl`은 쿠버네티스 클러스터에서 애플리케이션을 관리하는 강력한 도구다.

4장에서는 `kubectl`의 일반적인 사용 방법을 소개했다.

`help` 명령어를 통해 더 많은 정보를 확인할 수 있다.

```sh
kubectl help
kubectl help <명령 이름>
```