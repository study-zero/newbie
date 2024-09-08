# 6장. 라벨과 애노테이션
라벨은 파드와 레플리카셋 같은 쿠버네티스 객체에 연결할 수 있는 키/값 쌍으로, 쿠버네티스 객체에 식별 정보를 첨부하는데 매우 유용하다. 라벨은 객체를 그룹화하기 위한 기초를 제공한다.
애노테이션은 도구와 라이브러리에서 활용할 수 있게 비식별 정보를 유지해 설계된 키/값 쌍이다. 라벨과 다르게 쿼리, 필터링 또는 파드를 서로 구분하기 위한 수단이 아니다.

## 라벨
라벨은 객체의 메타데이터를 식별하는 기능을 제공한다.
라벨의 키는 두 부분으로 나눌 수 있는데, **선택적 접두사**와 **이름**이며 슬래시(/)로 구분된다.
접두사가 지정된 경우, 반드시 DNS 하위 도메인이어야 한다.
```{figure} /images/kuar/ch06/label.png
:alt: image
:width: 540px
```

### 라벨 적용

**Before**
``` bash
# 디플로이먼트를 생성
kubectl run alpaca-prod \
--image=gcr.io/kuar-demo/kuard-amd64:blue \
--replicas=2 \
--labels="ver=1,app=alpaca,env=pod"
```
> error: unknown flag: --replicas
> 
> The `--replicas` option was deprecated in Kubernetes 1.18 and removed in a Kubernetes 1.21.

**After**
``` bash
# 디플로이먼트를 생성
kubectl create deployment alpaca-prod \
--image=gcr.io/kuar-demo/kuard-amd64:blue \
--replicas=2

# 라벨 적용
kubectl label deployments alpaca-prod ver=1 app=alpaca env=prod --overwrite=true
```


**<라벨 기반의 벤다이어그램>**
```{figure} /images/kuar/ch06/diagram.png
:alt: image
:width: 540px
```
- app, env, ver 별 그룹핑 가능

### 라벨 수정
``` bash
# 라벨 수정
kubectl label deployments alpaca-prod ver=1 "canary=false"
```

> error: 'canary' already has a value (true), and --overwrite is false
> --overwrite=true를 추가해야 함


``` bash
# 라벨 제거
kubectl label deployments alpaca-prod ver=1 "canary-"
```

### 라벨 셀렉터
**라벨 셀렉터**는 라벨의 집합을 기반으로 쿠버네티스 객체를 필터링하는데 사용된다.

**현재 클러스터에서 실행 중인 모든 파드의 상태 확인**
``` bash
kubectl get pods --show-labels
```

```{figure} /images/kuar/ch06/show-labels.png
:alt: image
:width: 540px
```

- `pod-template-hash`: 디플로이먼트에 적용되는 라벨로, 특정 파드가 어떤 템플릿 버전에서 생성됐는지 추적할 수 있음

**ver 라벨이 2로 설정된 파드만을 나열**
``` bash
kubectl get pods --selector="ver=2"

or

kubectl get pods -l "ver=2"
```

**app 라벨이 bandicoot 이면서 ver 라벨이 2로 설정된 파드만을 나열**
``` bash
kubectl get pods --selector="app=bandicoot,ver=2"
```


**app 라벨이 alpaca 또는 bandicoot로 설정된 파드만을 나열**
``` bash
kubectl get pods --selector="app in (alpaca,bandicoot)"
```

**셀렉터 연산자**
```{figure} /images/kuar/ch06/label-selector.png
:alt: image
:width: 540px
```


### API 객체의 라벨 셀렉터
쿠버네티스에서 특정 객체가 다른 객체 집합을 참조하고자 **라벨 셀렉터**를 사용한다.
이때, **구문 분석된 구조(parsed structure)** 를 사용한다.

``` YAML
# app=alpaca, ver in (1, 2)
selector:  
  matchLabels:
    app: alpaca  
  matchExpressions:  
    - {key: ver, operator: In, values: [1, 2]}
```

``` YAML
# app=alpaca, ver=1
selector:
  app: alpaca
  ver: 1
```


**라벨은 쿠버네티스에서 애플리케이션을 서로 연결하고 관리해주는 강력한 도구다.**
- 파드의 여러 복제본을 생성하고 유지 관리하는 레플리카셋의 경우, **셀렉터**를 통해 관리 중인 파드를 찾는다.
- 서비스 로드밸런서의 경우, **셀렉터 쿼리**를 통해 트래픽을 전달해야 하는 파드를 찾는다.

## 애노테이션
**애노테이션**은 **메타데이터**의 유일한 목적인, 쿠버네티스 객체에 **추가적인 정보를 저장**할 수 있는 장소를 제공한다.

**라벨 vs 애노테이션**
확실하지 않은 경우, **애노테이션**을 통해 객체에 정보를 추가하고
셀렉터에서 사용하려는 경우, **라벨**을 사용한다.

**애노테이션은 어떤 용도로 사용되는가**
- 객체의 최신 업데이트에 대한 이유 추적
- 라벨에 적합하지 않은 빌드, 릴리즈 또는 이미지 관련 정보 첨부
- UI의 시각적 품질이나 유용성을 향상시키고자 추가 데이터 제공
- ...

**애노테이션의 포맷**
- 라벨 키와 동일한 포맷을 사용
- 추가적인 정보 제공에 주로 사용되므로, 키의 '네임스페이스' 부분이 보다 중요
- 키의 예시) deployment.kubernetes.io/change-cause

**애노테이션은 모든 쿠버네티스 객체의 공통 metadata 섹션에 정의된다.**
``` YAML
...
metadata:
  annotations:
    example.com/icon-url: "https://example.com/icon.png"
...
```
