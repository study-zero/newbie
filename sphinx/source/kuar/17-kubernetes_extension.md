# 17장.  쿠버네티스 확장

**쿠버네티스 클러스터에 쿠버네티스가 자체적으로 제공하는 기능이 아닌 새로운 기능을 추가하는 것**

처음부터 쿠버네티스는 쿠버네티스 API를 확장 가능하게 설계됨

클러스터 운영자는 플러그인을 추가하여 클러스터 커스터마이징 가능

- Helm / Istio / Argo CD
- 클러스터 관리자 권한이 필요

## 확장 지점

클러스터를 직접 확장하기 전, 기존 쿠버네티스 API 범위 내에서 가능한 상황을 고려할 것

### API 서버 확장 (새로운 리소스 타입 추가)

새로운 리소스 생성
``` yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: loadtests.beta.kuar.com
spec:
  group: beta.kuar.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                service:
                  type: string
                scheme:
                  type: string
                requestsPerSecond:
                  type: integer
                paths:
                  type: array
                  items:
                    type: string
  scope: Namespaced
  names:
    plural: loadtests
    singular: loadtest
    kind: LoadTest
    shortNames:
    - lt
```
- `name`: <리소스 복수형>. <api 그룹>
- `served`: 쿠버네티스 API 서버가 해당 버전의 리소스를 서비스할 것인지 여부 (v1 API 버전을 통해 LoadTest 리소스에 접근 가능/불가능)
- `storage`: 쿠버네티스 API 서버의 백업 저장소에 데이터를 저장하는 데 사용되는 버전인지 여부 (v1, v2 버전에 대해 리소스를 생성했다면, 둘 중 하나만 true 여야 함)
- `scope`: 클러스터 전역에서 사용될지 or 특정 네임스페이스 내에서만 사용될지

새로운 LoadTest 객체 생성

``` yaml
apiVersion: beta.kuar.com/v1
kind: LoadTest
metadata:
  name: my-loadtest
spec:
  service: my-service
  scheme: https
  requestsPerSecond: 1000
  paths:
  - /index.html
  - /login.html
  - /shares/my-shares/
```

`kubectl get loadtests`
- CRD 생성 전 `error: the server doesn't have a resource type "loadtests"`
- CRD 생성 후 `No resources found in default namespace.`
- LoadTest 객체 생성 후 `my-loadtest   64s`

**LoadTest 사용자 정의 컨트롤러**
- LoadTest 객체가 정의될 때 특정 조치를 취하는 컨트롤러
- API 서버와의 상호작용을 통해 LoadTest 생성, 삭제 및 변경 사항을 모니터링
- 폴링 기반 접근 방식 vs 모니터링 API
    - 지연 시간과 오버헤드가 없는 **모니터링 API**를 활용!
    - 버그 없이 활용하기 위해 client-go 라이브러리의 Informer 패턴처럼 안정적인 매커니즘을 사용!


### API 요청에 대한 승인 컨트롤러 추가

```{figure} /images/kuar/ch17/access-control-overview.svg
:alt: image
:width: 720px
:class: white-bg
```

[https://kubernetes.io/docs/concepts/security/controlling-access/](https://kubernetes.io/docs/concepts/security/controlling-access/)

- 승인 컨트롤러 구분 [https://kubernetes.io/docs/concepts/security/controlling-access/](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
    - Mutating Admission Controller: 기본 값 설정
    - Validating Admission Controller: 유효성 검증


**사용자 정의 승인 컨트롤러를 통한 유효성 검사**

API 객체 생성 요청이 유효한 스키마인지 검사

동적 승인 제어 시스템(웹훅)을 통해 승인 컨트롤러를 클러스터에 추가할 수 있음

웹훅은 HTTP 애플리케이션으로, AWS Lambda와 같은 FaaS 형태로 실행될 수 있음

``` yaml
apiVersion: admissionregistration.io/v1beta1
kind: ValidatingWebhookConfiguration
metadata:
  name: kuar-validator
webhooks:
- name: validator.kuar.com
  rules:
  - apiGroups:
    - "beta.kuar.com"
    apiVersions:
    - v1
    operations:
    - CREATE
    resources:
    - loadtests
  clientConfig:
    url: <웹훅을 위한 URL>
    caBundle: <base64로 인코딩된 CA 인증서>
```
웹훅은 HTTPS를 통해서만 접근 가능

즉, 웹훅을 제공하려면 인증서를 생성해야 함

1. 웹훅 URL에 대한 개인키, 인증서 서명 요청 생성
2. API 서버로 인증서 서명 요청을 보내 인증서를 받음
3. 해당 인증서를 사용하여 SSL 기반 승인 컨트롤러를 생성

기본값을 제공하기 위해서는 kind 필드를 `ValidatingWebhookConfiguration` 대신 `MutatingWebhookConfiguration`을 사용하면 됨


## 사용자 정의 리소스를 위한 패턴

- 저스트 데이터
    - 애플리케이션의 단순 정보를 저장하고 검색하는 용도로 API 서버를 서용
    - 사용자 정의 리소스를 순수 데이터만으로 구성
    - 확장성과 사용의 편의성 측면에서 컨피그맵 보다 유용
    - ex. 애플리케이션의 카나리 배포를 위한 컨피규레이션 (하나를 만들어두면 다른 애플리케이션에서 활용 가능 ?)
- 컴파일러
    - 추상화 패턴을 의미하는 것으로, LoadTest 확장을 예로 들 수 있음
    - 오퍼레이터 패턴과 달리 온라인 상태 유지 관리 기능은 존재하지 않음
- 오퍼레이터
    - 사용자 정의 리소스를 사용하여 애플리케이션의 배포 및 관리를 자동화하는 패턴으로, 쿠버네티스 원칙인 control loop를 따름
    - 생성된 리소스의 온라인 프로액티비 관리 기능을 제공
        - 데이터베이스의 스냅샷 백업
        - 새 버전의 소프트웨어 사용 가능 시 업그레이드 알림

[kubebuilder 프로젝트](https://github.com/kubernetes-sigs/kubebuilder?tab=readme-ov-file)  라이브러리를 활용하여 쿠버네티스 API 확장을 쉽고 안정적으로 구축할 수 있음

## 요약

쿠버네티스 API의 확장성은 **에코시스템을 강화**하고, **클러스터 커스터마이징**을 용이하게 함