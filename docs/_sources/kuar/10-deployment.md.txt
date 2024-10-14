# 10장. 디플로이먼트
```{figure} /images/kuar/ch10/deployment.png
:alt: image
:width: 400px
:reference: https://medium.com/the-programmer/working-with-deployments-roll-back-and-rolling-updates-bb4299488e34
```

**디플로이먼트** 객체는 **새 버전의 릴리스**를 관리하고자 사용한다.<br/>
이는 파드, 레플리카셋, 서비스가 애플리케이션의 단일 인스턴스를 구축하는데 사용되는 것이랑 다른 점이다.

디플로이먼트는 버전에 관계 없이 배포된 애플리케이션을 나타내는 개념이다.<br/>
디플로이먼트를 통해 다른 버전으로 쉽게 이동하는 **롤아웃**을 수행할 수 있다.<br/>

롤아웃 과정에서 개별 파드는 상태 검사를 거치고<br/>
사용자가 정의한 시간 만큼 기다리며 신중하게 진행된다.<br/>
너무 많은 실패가 발생하면 배포를 중지한다.

디플로이먼트를 사용하면 다운타임이나 에러 없이 새로운 소프트웨어 버전을 쉽고 안정적으로 롤아웃할 수 있다.<br/>
이는 쿠버네티스 클러스에서 실행되는 디플로이먼트 컨트롤러에 의해 제어된다.

<br/>


## 첫 번째 디플로이먼트
```console
$ k apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kuard
spec:
  selector:
    matchLabels:
      run: kuard
  replicas: 1
  template:
    metadata:
      labels:
        run: kuard # .spec.selector.matchLabels와 동일해야 함
    spec:
      containers:
      - name: kuard
        image: gcr.io/kuar-demo/kuard-amd64:blue
EOF
```

```{note}
`.spec.selector.matchLabels`와 `.spec.template.metdata.labels`가 일치하지 않으면 아래와 같은 오류가 발생한다.

    The Deployment "kuard" is invalid: spec.template.metadata.labels: Invalid value: map[string]string{"run":"apple"}: `selector` does not match template `labels`
```

```console
# 파일로 생성시
$ kubectl create -f kuard-deployment.yaml

# 디플로이먼트 객체의 샐렉터 확인
$ k get deploy kuard -o jsonpath --template {.spec.selector.matchLabels}
{"run":"kuard"}

# 동일 라벨의 replicaSets 확인
$ k get rs --selector=run=kuard
NAME               DESIRED   CURRENT   READY   AGE
kuard-79fd97bf74   1         1         1       0m11s
```

디플로이먼트는 레플리카셋을 제어한다.

```console
# 디플로이먼트의 크기 재조정
$ k scale deploy kuard --replicas=2
deployment.apps/kuard scaled

# replicaSets 상태 확인
$ k get rs --selector=run=kuard
NAME               DESIRED   CURRENT   READY   AGE
kuard-79fd97bf74   2         2         2       3m26s
```

디플로이먼트로 제어되는 레플리카셋을 수동으로 조정할 경우<br/>
replicaset-controller에 의해 디플로이먼트에서 지정한 replicas의 상태로 복구된다.

이는 쿠버네티스가 온라인 자가 치유 시스템이며<br/>
디플로이먼트 객체가 최상위 레벨로 레플리카셋을 관리하고 있기 때문이다.

```console
# 파드 상태 확인
$ k get po --selector=run=kuard
NAME                     READY   STATUS    RESTARTS   AGE
kuard-79fd97bf74-2rqc2   1/1     Running   0          74s
kuard-79fd97bf74-gvl7g   1/1     Running   0          10m

# 디플로이먼트의 크기 재조정
$ k scale rs kuard-79fd97bf74 --replicas=1
deployment.apps/kuard scaled

# replicaSets 상태 확인
$ k get rs --selector=run=kuard
NAME               DESIRED   CURRENT   READY   AGE
kuard-79fd97bf74   2         2         2       3m26s

# 파드 상태 확인
$ k get po --selector=run=kuard
NAME                     READY   STATUS    RESTARTS   AGE
kuard-79fd97bf74-d5dxc   1/1     Running   0          2s  # 삭제된 자리에 새 파드가 생김
kuard-79fd97bf74-gvl7g   1/1     Running   0          11m # 유지
```

<br/>

## 디플로이먼트 생성

쿠버네티스 컨피규레이션은 선언형 관리 방식을 사용해야 한다(yaml or json).

```console
$ k get deploy kuard -o yaml > kuard-deployment.yaml
$ k replace -f kuard-deployment.yaml --save-config
```

> `kubectl replace`의 [--save-config 옵션](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_replace/)은 현재 객체의 설정을 어노테이션에 담을 때 사용한다.<br/>
> 추후에 `kubectl apply`를 사용할 경우 스마트한 병합에 활용된다.


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kuard
  namespace: default
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      run: kuard
  strategy: # 새로운 소프트웨어의 롤아웃이 진행되는 방법
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate # 또는 Recreate 2가지 지원
  template:
    metadata:
      labels:
        run: kuard
    spec:
      containers:
      - image: gcr.io/kuar-demo/kuard-amd64:blue
        imagePullPolicy: IfNotPresent
        name: kuard
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

<br/>

## 디플로이먼트 관리

`kubectl describe` 명령으로 디플로이먼트에 관한 자세한 정보를 얻을 수 있다.

```console
$ k describe deploy kuard
Name:                   kuard
Namespace:              default
CreationTimestamp:      Mon, 14 Oct 2024 15:53:42 +0900
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               run=kuard
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  run=kuard
  Containers:
   kuard:
    Image:         gcr.io/kuar-demo/kuard-amd64:blue
    Port:          <none>
    Host Port:     <none>
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none> # 이전 버전의 rs
NewReplicaSet:   kuard-79fd97bf74 (2/2 replicas created) # 현재 버전의 rs
Events:
  Type    Reason             Age                From                   Message
  ----    ------             ----               ----                   -------
  Normal  ScalingReplicaSet  25m                deployment-controller  Scaled up replica set kuard-79fd97bf74 to 1
  Normal  ScalingReplicaSet  14m (x5 over 22m)  deployment-controller  Scaled up replica set kuard-79fd97bf74 to 2 from 1
```

`OldReplicaSets`와 `NewReplicaSet`은 디플로이먼트에서 현재 관리하고 있는 레프리카셋 개체의 상태를 표현한다.<br/>
디플로이먼트가 롤아웃 중이면 두 상태 모두 값을 갖고, 완료되면 `OldReplicaSets`가 `none`으로 설정된다.

배포를 위한 `kubectl rollout` 명령도 있다.
<br/>


## 디플로이먼트 업데이트

디플로이먼트를 활요한 가장 일반적인 두 가지 작업은 확장과 애플리케이션 업데이트다.

<br/>


### 디플로이먼트 확장

선언형 설정 변경을 통해 디플로이먼트를 확장하려면 `.spec.repliacas`를 변경한다.

```yaml
spec:
  replicas: 3
```

변경을 저장하고 커밋 후에 아래 명령을 사용한다.

```console
$ k apply -f kuard-deployment.yaml
```

<br/>


### 컨테이너 이미지 업데이트

소포트웨어의 새 버전을 롤아웃하려면 선언형 설정에서 컨텡이너 이미지를 업데이트 한다.<br/>
업데이트에 대한 정보를 기록하려면 탬플릿에 애노테이션을 추가한다.<br/>
애노테이션 추가 자체가 설정 변경으로 인식되어 롤아웃을 유발하므로 간단한 확장 시에는 수정하지 않는다.

```yaml
...
spec.template.spec.containers:
  - image: gcr.io/kuar-demo/kuard-amd64:green
    imagePullPolicy: Always
...
spec.template.metadta.annotations:
  kubernetes.io/change-cuase: "Update to green kuard"
```

디플로이먼트를 아래처럼 업데이트한다.

```console
$ k apply -f kuard-deployment.yaml
```

롤아웃 상태를 아래처럼 모니터링할 수 있다.

```console
$ k rollout status deploy kuard
deployment "kuard" successfully rolled out
```

디플로이먼트에 의해 관리되는 이전 및 새로운 레플리카 셋을 사용중인 이미지와 함께 확인할 수 있다.<br/>
롤백을 대비해 이전 및 새로운 레플리카셋 모두 보관된다.

```console
$ k get rs -o wide
NAME               DESIRED   CURRENT   READY   AGE    CONTAINERS   IMAGES                               SELECTOR
kuard-666c976497   3         3         3       108s   kuard        gcr.io/kuar-demo/kuard-amd64:green   pod-template-hash=666c976497,run=kuard
kuard-79fd97bf74   0         0         0       38m    kuard        gcr.io/kuar-demo/kuard-amd64:blue    pod-template-hash=79fd97bf74,run=kuard
```

일시적으로 롤아웃을 중단하거나 재개할 경우 아래처럼 사용할 수 있다.

```console
# 중단
$ k rollout pause deploy kuard
deployment.apps/kuard paused

# 재개
$ k rollout resume deploy kuard
deployment.apps/kuard resumed
```
<br/>


### 롤아웃 이력
디플로이먼트는 롤아웃 이력(rollout history)를 유지 관리한다.<br/>
이력을 통해 이전 상태를 확인하고 특정 버전으로 롤백할 수 있다.<br/>

```console
$ k rollout history deploy kuard
deployment.apps/kuard 
REVISION  CHANGE-CAUSE
1         <none>
2         Update to green kuard
```

변경 이력(revision history)은 가장 오래된 것부터 최신의 것 순서로 제공된다.<br/>
새로운 롤아웃을 수행할 때마다 고유한 변경 번호(revision)가 부여된다.

특정 버전의 자세한 내용은 `--revision` 옵션을 통해 확인할 수 있다.

```console
$ k rollout history deploy kuard --revision=2
deployment.apps/kuard with revision #2
Pod Template:
  Labels:       pod-template-hash=666c976497
        run=kuard
  Annotations:  kubernetes.io/change-cause: Update to green kuard
  Containers:
   kuard:
    Image:      gcr.io/kuar-demo/kuard-amd64:green
    Port:       <none>
    Host Port:  <none>
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
  Node-Selectors:       <none>
  Tolerations:  <none>
```

blue 버전으로 한 번 더 업데이트를 수행하면 아래와 같다.

```console
$ k rollout history deploy kuard
deployment.apps/kuard 
REVISION  CHANGE-CAUSE
1         <none>
2         Update to green kuard
3         Update to blue kuard
```

방금 전 수행한 업데이트를 롤백하면 아래처럼 마지막 롤아웃을 취소할 수 있다.<br/>
undo 명령은 부분적으로 완료되거나 와전히 완료된 롤아웃을 모두 취소할 수 있다.<br/>
undo 명령은 이전 버전으로 새로운 롤아웃을 실행시키는 방식으로 진행된다.

undo 명령은 체크인된 매니페스트와 실제 클러스터에서 실행 중인 항목과 불일치를 유발할 수 있기 때문에<br/>
선언형 설정 yaml을 되돌리고 `kubectl apply`를 적용하는 것이 실제 운영 환경과의 일치를 더 쉽게 유지할 수 있다.


```console
# 마지막 롤아웃 취소
$ k rollout undo deploy kuard
deployment.apps/kuard rolled back

# 롤아웃 이력 확인
$ k rollout history deploy kuard
deployment.apps/kuard 
REVISION  CHANGE-CAUSE
1         <none>
3         Update to blue kuard
4         Update to green kuard

# replicaSets 상태 확인
$ k get rs -o wide
NAME               DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES                               SELECTOR
kuard-666c976497   3         3         3       12m     kuard        gcr.io/kuar-demo/kuard-amd64:green   pod-template-hash=666c976497,run=kuard
kuard-79fd97bf74   0         0         0       49m     kuard        gcr.io/kuar-demo/kuard-amd64:blue    pod-template-hash=79fd97bf74,run=kuard
kuard-b64c97699    0         0         0       3m11s   kuard        gcr.io/kuar-demo/kuard-amd64:blue    pod-template-hash=b64c97699,run=kuard
```

2번 리비전이 사라지고 롤백에 재활용되어 4번 리비전 번호가 새로 부여됐다.

`--to-revision` 플래그를 사용해 특정 리비전으로 롤백할 수도 있다.


```console
# 3번 리비전으로 롤백
$ k rollout undo deploy kuard --to-revision=3
deployment.apps/kuard rolled back

# 롤아웃 이력 확인
$ k rollout history deploy kuard
deployment.apps/kuard 
REVISION  CHANGE-CAUSE
1         <none>
4         Update to green kuard
5         Update to blue kuard
```

3번 리비전이 사라지고 롤백에 재활용되어 5번 리비전 번호가 새로 부여됐다.<br/>
`--to-revision=0` 플래그를 사용하면 이전 배포 버전으로 롤백이 가능하다.

리비전은 기본적으로 최대 10까지 존재하며 최대값은 `.sepc.revisionHistoryLimit`으로 정의된다.

<br/>


## 디플로이먼트 전략

롤아웃 전략으로 재생성(Recreate)와 롤링업데이트(RollingUpdate) 2가지가 지원된다.

<br/>


### 재생성 전략

```{figure} /images/kuar/ch10/recreate.png
:alt: image
:width: 300px
:reference: https://medium.com/codex/kubernetes-deployment-rolling-updates-and-rollbacks-explained-e3efa6557368
```

새로운 이미지를 사용하도록 레플리카셋을 업데이트하고<br/>
디플로이먼트와 관련된 모든 파드를 종료한다.<br/>
더 이상 복제본이 존재하지 않으면 새로운 이미지로 모든 파드를 재생성 한다.

빠르고 간단하지만 서비스 다운타임(downtime)이 발생한다.<br/>
약간의 다운타임이 허용되는 테스트 목적을 사용해야 한다.


<br/>


### 롤링업데이트 전략

```{figure} /images/kuar/ch10/rolling-update.png
:alt: image
:width: 440px
:reference: https://medium.com/codex/kubernetes-deployment-rolling-updates-and-rollbacks-explained-e3efa6557368
```

한 번에 전체 파드가 아닌 일부 파드를 업데이트해 모든 파드가 새 버전의 소프트웨어를 실행할 때 까지<br/>
점진적으로 업데이트를 수행한다.

사용자와 직접 상호 작용하는 서비스에 선호되며<br/>
재생성 전략보다 업데이트 속도가 느리지만 다운타임이 없고 안정적이다.<br/>
롤아웃 중에도 사용자 트래픽을 수신하며 새로운 버전의 서비스로 변경할 수 있다.

<br/>

#### 서비스의 다중 버전 관리
중요한 점은 업데이트 과정에서 일정 기간 동안 새로운 버전과<br/>
이전 버전 서비스 요청을 모두 수신하고 트래픽을 처리한다는 것이다.

각 버전의 소프트웨워와 클라이언트는 약간의 **버전 차이가 발생하더라도 서로 통신이 가능**해야 한다.

예를 들어 롤아웃 중에 버전 1 클라이언트는 버전 2 서버로 API 요청을 보낼 수 있다.<br/>
이 경우 두 버전 간의 호환성이 보장되어야 한다.

이는 자바스크립트 클라이언트에만 해당되는 것이 아니라 컴파일된 클라이언트 라이브러리에서도 동일하다.<br/>
라이브러리는 무거운 백엔드 클라이언트를 사용하지 않고 API contract를 통해 분리된 아키텍처를 구현함으로써 해결될 수 있다.

```{figure} /images/kuar/ch10/architecture.png
:alt: image
:width: 440px
```


<br/>

#### 롤링업데이트 설정

롤링업데이트 동작을 조정하는 데에 `maxUnavailable`과 `maxSurge` 매개변수가 사용된다.

<dl>
<dt>maxUnavailable</dt>
<dd>롤링업데이트 과정에서 사용 불가능한 파드의 최대 갯수(절댓값 또는 백분율)를 설정한다.</dd>
<dt>maxSurge</dt>
<dd>롤아웃을 수행하고자 생성할 수 있는 추가 리소스의 갯수(절댓값 또는 백분율)를 설정한다.</dd>
</dl>

`maxUnavailable` 매개변수는 롤링업데이트 진행 속도를 조정하는 데 도움을 준다.

예를 들어 `maxUnavailable`를 50%로 설정한 경우 아래와 같은 단계를 거친다.

1. 기존 레플리카셋을 원래 크기의 50%로 축소한다.
2. 새로운 레플리카셋을 최대 2개의 복제본으로 확장한다.
3. 기존 레플리카셋을 0으로 축소한다.
2. 새로운 레플리카셋을 최대 4개의 복제본으로 확장한다.

- 파드 수
    | 단계 | 기존 레플리카셋 | 새로운 레플리카 셋 |
    | -- | -- | -- |
    | 0 | 4 | 0 |
    | 1 | 2 | 0 |
    | 2 | 2 | 2 |
    | 3 | 0 | 2 |
    | 4 | 0 | 4 |

```{note}
재생성 전략은 `maxUnavailble`을 100%로 설정한 롤링업데이트 전략과 동일하게 수행된다.
```

`maxSurge` 설정이 없는 위와 같은 방식은 서비스 가용성을 줄이는 방법을 통해 동작하며<br/>
현재 최대 본제본의 수보다 크게 확장하는 것이 불가능하다.

`maxUnavailable`을 0으로 설정하고 `maxSurge` 매개변수를 사용하면<br/>
서비스 사용성을 100% 미만으로 덜어뜨리지 않고 롤아웃을 수행할 수 있다.

예를 들어 `maxUnavailable`를 0으로, `maxSurge`를 50%로 설정한 경우 아래와 같은 단계를 거친다.

1. 새로운 레플리카셋을 최대 원래 크기의 50%인 복제본으로 확장한다.
2. 기존 레플리카셋을 2로 축소한다.
3. 새로운 레플리카셋을 최대 4개의 복제본으로 확장한다.
4. 기존 레플리카셋을 0으로 축소한다.

- 파드 수
    | 단계 | 기존 레플리카셋 | 새로운 레플리카 셋 |
    | -- | -- | -- |
    | 0 | 4 | 0 |
    | 1 | 4 | 2 |
    | 2 | 2 | 2 |
    | 3 | 2 | 4 |
    | 4 | 0 | 4 |

```{note}
블루/그린 배포(blue/green deployment)는 `maxSurge`를 100%로 설정한 롤링업데이트 전략과 동일하게 수행된다(2단계).
```

<br/>

### 서비스 안정을 위한 느린 롤아웃

단계별 롤아웃은 새로운 소프트웨어 버전의 정상적이고 안정적인 서비스 제공이 목적이므로<br/>
디플로이먼트 컨트롤러는 다음 파드 업데이트 전에 현재 업데이트를 수행한 파드의 준비 상태를 확인한다.<<br/>
이는 파드의 readiness check(5장 참고)를 통해 이루어진다.

파드가 준비 상태가 되어도 메모리 누수나 전체 요청의 1%에서 발생하는 오류 등<br/>
특정 시간 이후 발생하는 오류가 있기 때문에 다음 파드의 업데이트로 넘어가기 전에<br/>
`.spec.minReadySeconds` 설정을 통해 일정 시간 대기할 수 있다.

`.spec.minReadySeconds`를 60으로 설정하면 디플로이먼트가 다음 파드의 업데이트로 이동하기 전에<br/>
**파드의 상태가 정상임을 확인한 후 60초 동안 대기**한다.

시스템이 대기하는 시간을 제한하는 타임아웃도 `.spec.progressDeadlineSeconds`를 통해 설정가능하다.<br/>
이는 특정 배포 진행 단계가 데드라인 시간 안에 진행되지 않으면 배포가 실패한 것으로 간주하고<br/>
남아있는 단계들이 모두 종료된다.<br/>
파드를 생성하거나 삭제될 때마다 타임아웃 시계는 0으로 초기화된다.

```{figure} /images/kuar/ch10/deployment-cycle.png
:alt: image
:width: 400px
```

<br/>


## 디플로이먼트 삭제


아래와 같은 명령형 명령을 통해 디플로이먼트를 삭제할 수 있다.

```console
# 디플로이먼트 및 레플리카셋, 파드 삭제
$ k delete deploy kuard

# 디플로이먼트 삭제 및 레플리카셋, 파드 유지
$ k delete deploy kuard --cascade=false

# 선언형 설정을 통해 디플로이먼트 삭제
$ k delete -f kuard-deployment.yaml
```
<br/>


## 디플로이먼트 모니터링

디플로이먼트 상태는 `kubectl describe deployment`를 통해 확인할 수 있으며<br/>
`.status.conditions` 배열에 나타난다.

배포가 실패할 때 `Type=Progressing, Status=False`로 표시된다.

<br/>


## 요약

쿠버네티스의 주목표는 신뢰할 수 있는 분산 시스템을 쉽게 구축하고 배포할 수 있게 하는 것이다.

이는 소프트웨어 서비스에 대한 새 버전의 롤아웃을 정기적으로 관리하는 것을 의미한다(배포는 계속될 것이므로).

디플로이먼트는 서비스에 대한 안정적인 롤아웃과 롤아웃 관리를 위해 중요한 요소다.


<br/>
