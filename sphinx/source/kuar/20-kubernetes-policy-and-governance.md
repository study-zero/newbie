# 20장. 쿠버네티스 클러스터에 대한 정책과 거버넌스

정책 : 쿠버네티스 리소스를 구성할 수 있는 방법에 대한 조건 및 제약의 집합
거버넌스 : 쿠버네티스 리소스에 대한 조직적 정책을 확인하고 강제화하는 과정

쿠버네티스에는 NetworkPolicy, PodSecurityPolicy 와 같은 다양한 타입의 정책이 존재

하지만 다음과 같이 쿠버네티스 리소스가 생성되기 전에 이러한 정책을 시행하려면, 내장 정책 리소스로는 한계가 존재

- 모든 컨테이너를 특정 컨테이너 레지스트리에서만 가져와야 한다.
- 모든 파드에는 부서 이름과 연락처 정보가 표시돼 있어야 한다.
- 모든 인그레스 호스트이름은 클러스터에서 고유해야 한다.

쿠버네티스의 `코어 확장성 컴포넌트`를 활용해 정책과 거버넌스를 달성할 수 있다.

> 코어 확장성 컴포넌트
> 쿠버네티스 클러스터의 기능을 확장 및 커스터마이징할 수 있도록 돕는 컴포넌트
> ex. CRD(Custom Resource Definition), Controller

## 승인 흐름

![image](https://github.com/user-attachments/assets/77f8b012-c24c-44a4-8336-22888ca69e23)
<쿠버네티스 API 서버를 통한 API 요청 흐름>

`MutatingWebhookConfiguration` 리소스 생성을 통해 API 서버가 **승인 변경** 평가를 보낼 수 있는 웹훅의 엔드포인트를 구성할 수 있음

마찬가지로 `ValidatingWebhookConfiguration` 리소스 생성을 통해 API 서버가 승인 검증 평가를 보낼 수 있는 웹훅의 엔드포인트를 설정할 수 있음


> etcd
> - 쿠버네티스 클러스터 내 모든 리소스의 상태 및 메타데이터를 저장하는 key-value 저장소


## 게이트키퍼를 통한 정책과 거버넌스

쿠버네티스 자체적으로는 정책과 거버넌스 활성화를 위한 컨트롤러를 제공하지 않음

[게이트키퍼](https://open-policy-agent.github.io/gatekeeper/website/docs/) : 정의된 정책을 기반으로 리소스를 평가하고 쿠버네티스 리소스에 대해 생성이나 수정을 허용할지 여부를 결정하는 쿠버네티스 네이티브 정책 컨트롤러 (오픈소스 솔루션)

게이트키퍼는 CRD 를 사용해 구성과 관련된 새로운 쿠버네티스 자원 집합을 정의
</br>
-> kubectl 과 같은 친숙한 도구로 동작 가능

게이트키퍼는 자원에 대한 변경과 감사를 수행

### 개방형 정책 에이전트란?

[개방형 정책 에이전트(OPA)](https://www.openpolicyagent.org/) : 쿠버네티스 클러스터에서 정책을 정의하는 도구로, Rego 라는 언어를 사용하여 정책을 작성

게이트키퍼는 이 OPA 를 쿠버네티스 클러스터에 통합할 수 있는 컨트롤러

### 게이트키퍼 설치

**helm 패키지 매니저를 통해 게이트키퍼 설치**
``` shell
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper/gatekeeper --name-template=gatekeeper --namespace gatekeeper-system --create-namespace
```

**게이트키퍼 컴포넌트는 gatekeeper-system 네임스페이스 내에서 파드 형태로 실행되며, 승인 컨트롤러를 구성함을 확인**
``` shell
kubectl get pods -n gatekeeper-system

NAME                                             READY   STATUS    RESTARTS
gatekeeper-audit-6bbc9cc955-g4ppg                1/1     Running   0
gatekeeper-controller-manager-7d7f66bf68-8ts8c   1/1     Running   0
gatekeeper-controller-manager-7d7f66bf68-ltm4j   1/1     Running   0
gatekeeper-controller-manager-7d7f66bf68-x2fdw   1/1     Running   0
```

**웹훅 구성 방식 검토 ( `kubectl get validatingwebhookconfiguration -o yaml` )**

``` yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
...
webhooks:
- clientConfig:
    service: # (1)
      name: gatekeeper-webhook-service
      namespace: gatekeeper-system
      path: /v1/admit
  failurePolicy: Ignore # (3)
  namespaceSelector:
    matchExpressions:
    - key: admission.gatekeeper.sh/ignore # (2)
      operator: DoesNotExist
  rules:
  - apiGroups:
    - '*'
    apiVersions:
    - '*'
    operations:
    - CREATE
    - UPDATE
    resources:
    - '*'
```

(1) rules 에 해당하는 모든 리소스가 gatekeeper-system 네임스페이스에서 `gatekeeper-webhook-service` 라는 서비스로 실행되는 웹훅으로 전송된다.
<img width="657" alt="image" src="https://github.com/user-attachments/assets/dd967bb0-dc91-424f-bc82-c553e9a5951b" />
</br>

(2) `admission.gatekeeper.sh/ignore` 라벨이 지정된 네임스페이스의 리소스는 정책을 적용하지 않는다.
</br>

(3) 웹훅 (gatekeeper-webhook-service 서비스) 호출 실패 시, 쿠버네티스가 해당 요청을 무시하고 계속 처리하도록 한다.


### 정책 구성 - 컨테이너 이미지는 특정 레지스트리에서만 가져올 수 있게

**클러스터 관리자가 정책을 생성하는 방법**

1️⃣ 제약 조건 템플릿(게이트키퍼에서 정책을 만들기 위한 템플릿) 생성

``` yaml
# allowedrepos-constraint-template.yaml

apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
  annotations:
    description: Requires container images to begin with a repo string from a 
      specified list.
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos # ... (1)
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema: # ... (2)
          properties:
            repos:
              type: array
              items:
                type: string
  targets: 
    - target: admission.k8s.gatekeeper.sh # ... (3)
      rego: | # ... (4)
        package k8sallowedrepos

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not strings.any_prefix_match(container.image, input.parameters.repos)
          msg := sprintf("container <%v> has an invalid image repo <%v>, allowed repos are %v", [container.name, container.image, input.parameters.repos])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          not strings.any_prefix_match(container.image, input.parameters.repos)
          msg := sprintf("initContainer <%v> has an invalid image repo <%v>, allowed repos are %v", [container.name, container.image, input.parameters.repos])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.ephemeralContainers[_]
          not strings.any_prefix_match(container.image, input.parameters.repos)
          msg := sprintf("ephemeralContainer <%v> has an invalid image repo <%v>, allowed repos are %v", [container.name, container.image, input.parameters.repos])
        }
```
(1) K8sAllowedRepos 라는 CRD (여기서는 정책)를 정의한다.
</br>

(2) K8sAllowedRepos 정책을 생성할 때, 매개변수로 컨테이너 리포지토리 목록을 전달해야 한다.
</br>

(3) `admission.k8s.gatekeeper.sh` 라는 승인 컨트롤러가 정책을 실행한다.
</br>

(4) 정책의 실제 로직 / 쿠버네티스 리소스의 `spec.containers`, `spec.initContainers`, `spec.ephemeralContainers` 에 있는 컨테이너 이미지가 repos 리스트에 있는 레포지토리로 시작하는지 확인

2️⃣ 정책을 적용하고자 제약 조건 리소스를 생성

``` yaml
# allowedrepos-constraint.yaml

apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: repo-is-kuar-demo
spec:
  enforcementAction: deny # ... (1)
  match: # ... (2)
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - "default"
  parameters:
    repos:
      - "gcr.io/kuar-demo/"
```
(1) 규정을 준수하지 않는 리소스의 경우 거부된다.
- `dryrun` : 정책을 실행하되, 정책 위반 리소스 목록 확인 가능
- `warn` : 경고 메시지만 보낸다.
</br>

(2) 이 제약 조건의 범위를 기본 네임스페이스의 모든 파드로 제한한다.

**규정을 준수하는 리소스**

``` yaml
# compliant-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: kuard
spec:
  containers:
    - image: gcr.io/kuar-demo/kuard-amd64:blue
      name: kuard
      ports:
        - containerPort: 8080
          name: http
          protocol: TCP

# pod/kuard created
```

**규정 미준수 리소스**

``` yaml
# noncompliant-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: nginx-noncompliant
spec:
  containers:
    - name: nginx
      image: nginx

# Error from server (Forbidden): error when creating "STDIN": admission webhook "validation.gatekeeper.sh" denied the request
```
**감사**

`enforcementAction: dryrun`
``` yaml
# allowedrepos-constraint-dryrun.yaml

apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: repo-is-kuar-demo
spec:
  enforcementAction: dryrun
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - "default"
  parameters:
    repos:
      - "gcr.io/kuar-demo/"
```
**규정 미 준수 pod 생성 후, 주어진 제약 조건에 대해 규정을 준수하지 않은 리소스 목록 확인**

`kubectl get constraint repo-is-kuar-demo -o yaml`

``` yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
...
status:
  auditTimestamp: "2021-07-14T20:05:38Z"
  ...
  totalViolations: 1
  violations:
  - enforcementAction: dryrun
    kind: Pod
    message: container <nginx> has an invalid ..
    name: nginx-noncompliant
    namespace: default
```
### 변형

리소스가 규정을 준수하는지 확인하는 방법 말고, 규정을 준수하도록 리소스를 수정하는 방법은 없을까?

기본적으로 게이트키퍼는 검증 승인 웹훅으로 배포되지만, 변형 승인 웹훅으로 동작하도록 구성 가능

**변형 할당 리소스 생성**

``` yaml
# imagepullpolicyalways-mutation.yaml

apiVersion: mutations.gatekeeper.sh/v1alpha1
kind: Assign
metadata:
  name: demo-image-pull-policy
spec:
  applyTo:
  - groups: [""]
    kinds: ["Pod"]
    versions: ["v1"]
  match:
    scope: Namespaced
    kinds:
    - apiGroups: ["*"]
      kinds: ["Pod"]
    excludedNamespaces: ["system"]
  location: "spec.containers[name:*].imagePullPolicy"
  parameters:
    assign:
      value: Always
```
- 모든 파드의 imagePullPolicy 를 Always 로 설정
- system 네임스페이스는 제외

### 데이터 복제

생성하고자 하는 리소스의 특정 필드의 값을 다른 리소스의 필드 값과 비교하려는 니즈

ex. 인그레스의 호스트네임이 클러스터 전체에서 고유한지 확인

-> 게이트키퍼는 특정 리소스를 OPA 에 캐시해 리소스 간에 비교가 가능하도록 구성할 수 있음

**게이트키퍼 설정**

``` yaml
# config-sync.yaml

apiVersion: config.gatekeeper.sh/v1alpha1
kind: Config
metadata:
  name: config
  namespace: "gatekeeper-system"
spec:
  sync:
    syncOnly:
      - group: "networking.k8s.io"
        version: "v1"
        kind: "Ingress"
```

- 인그레스 리소스를 캐시
- ⚠️ 정책에 대한 평가를 수행하는 데 필요한 리소스만을 캐시해야 한다.


**제약 조건 템플릿 생성**

``` yaml
# uniqueingresshost-constraint-template.yaml

apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8suniqueingresshost
  annotations:
    description: Requires all Ingress hosts to be unique.
spec:
  crd:
    spec:
      names:
        kind: K8sUniqueIngressHost
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8suniqueingresshost

        identical(obj, review) {
          obj.metadata.namespace == review.object.metadata.namespace
          obj.metadata.name == review.object.metadata.name
        }

        violation[{"msg": msg}] {
          input.review.kind.kind == "Ingress"
          regex.match("^(extensions|networking.k8s.io)$", input.review.kind.group)
          host := input.review.object.spec.rules[_].host
          other := data.inventory.namespace[_][otherapiversion]["Ingress"][name]
          regex.match("^(extensions|networking.k8s.io)/.+$", otherapiversion)
          other.spec.rules[_].host == host
          not identical(other, input.review)
          msg := sprintf("ingress host conflicts with an existing ingress <%v>", [host])
        }
```
- `Rego` 섹션 - `data.inventory` 는 캐시 리소스를 의미하는 것으로, 새로운 인그레스의 host 값이 이미 존재하는(캐시에 존재하는) 다른 인그레스 리소스와 충돌하는지 검증하는 로직


### 메트릭

게이트키퍼는 지속적인 리소스 컴플라이언스 모니터링을 가능하게 하고자 프로메테우스 포맷으로 메트릭을 내보낸다.

### 정책 라이브러리

게이트키퍼 프로젝트의 핵심 신조 중 하나는 재사용 가능한 정책 라이브러리를 만드는 것

정책 라이브러리를 통해 정책을 공유하여 보일러플레이트 정책 작성 시간을 줄일 수 있음