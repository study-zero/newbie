# 11장. 데몬셋


**데몬셋**은 모든 노드에 에이전트나 데몬을 실행하기 위한 쿠버네티스 객체

**데몬셋**은 파드의 복사본이 쿠버네티스 클러스터의 노드 집합에서 실행되게 함

**데몬셋**은 레플리카셋과 유사하게, 클러스터의 원하는 상태와 관찰된 상태를 동일하게 만듬

노드당 하나의 파드를 실행하기 원한다면, 레플리카셋이 아닌 **데몬셋**을 사용해야 함

*ex) 로그 수집기, 모니터링 에이전트...*


## 데몬셋 스케줄러

**데몬셋**은 노드 셀렉터를 사용하지 않는 한, 모든 노드에 파드의 복제본을 생성함

**노드 셀렉터**를 사용하면, 일치하는 라벨 집합을 갖는 노드로 범위를 제한함

새로운 노드가 클러스터에 추가될 경우, **데몬셋 컨트롤러**는 파드가 없음을 확인하고 새로운 노드에 파드를 추가함

```{figure} /images/kuar/ch11/k8s-cluster.png
:alt: image
:width: 720px
```
- **데몬셋 컨트롤러**는 Controller Manager 중 하나
  - 컨트롤러 매니저는 Control Plane 컴포넌트 중 하나
  - _ex) Deployment Controller, Replication Controller, DaemonSet Controller_
- 일반적으로 쿠버네티스 스케줄러에 의해 파드가 실행되는 노드가 선택되지만, 데몬셋 파드는 데몬셋 컨트롤러에 의해 생성되고 각 노드에 할당됨


## 데몬셋 생성

``` yaml
# fluentd.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:v0.14.10
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lob/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```
- `fluentd`: 로그 수집 및 처리를 위해 사용되는 오픈소스 SW
- 레플리카셋과 달리, 데몬셋은 파드 템플릿 명세를 반드시 갖고 있어야 함

**`$ kubectl describe daemonset fluentd`**
```{figure} /images/kuar/ch11/describe-daemonset.png
:alt: image
:width: 540px
```
- 클러스터에 새 노드가 추가되면, fluentd 파드가 해당 노드에 자동으로 배포됨!


## 데몬셋을 특정 노드로 제한

노드에 라벨 추가

`kubectl label nodes minikube ssd=true`


노드 셀렉터 사용
```
nodeSelector:
     ssd: "trues"
```

**`$ kubectl describe daemonset fluentd`**
```{figure} /images/kuar/ch11/describe-daemonset-with-selector.png
:alt: image
:width: 540px
```



## 데몬셋 업데이트

디플로이먼트와 유사하게, **롤링업데이트 전략** 사용

**spec.minReadySeconds**
- 롤링업데이트가 파드를 업그레이드하기 전, 파드가 준비 상태로 있어야 하는 시간
- *30~60초*


**spec.updateStrategy.rollingUpdate.maxUnavailable**
- 롤링업데이트로 동시에 업데이트할 수 있는 파드의 수
- *1개*

