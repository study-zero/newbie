# 8장. 인그레스를 통한 HTTP 로드밸런싱

## 인그레스
- 쿠버네티스 자체의 HTTP 기반 로드밸런싱 시스템
- 즉, 들어오는 연결을 해석해 올바른 업스트림 서버로 보내는 **트래픽 전달**의 역할을 수행


## 인그레스 컨트롤러

``` {note}

클러스터 내의 인그레스가 작동하려면, 인그레스 컨트롤러가 실행되고 있어야 한다. 

즉, 적어도 하나의 인그레스 컨트롤러를 선택하고 이를 클러스터 내에 설치해야 한다.

(https://kubernetes.io/ko/docs/concepts/services-networking/ingress-controllers/)

```
</br>

인그레스 컨트롤러는 인그레스 프록시와 인그레스 오퍼레이터로 구성됨
- **인그레스 프록시**
    - `LoadBalancer` 타입의 서비스를 사용해 클러스터 외부에 노출됨
    - 들어오는 요청을 업스트림 서버로 보냄
- **인그레스 오퍼레이터**
    - 쿠버네티스 API에서 인그레스 객체를 읽고 모니터링
    - 인그레스 리소스에 명시된 것처럼 트래픽을 라우팅하도록 인그레스 프록시를 재구성


## 인그레스 구현
- 다른 쿠버네티스 객체처럼 인그레스 객체를 만들고 수정할 수 있지만, 이는 껍데기일뿐 구현은 사용자에 달려있음
- 1) 일반적으로 사용할 수 있는 HTTP 로드밸런서는 하나가 아니기 때문
- 2) 공통 확장 기능이 추가되기 전에 인그레스 객체가 쿠버네티스에 추가됐기 때문


## 인그레스 실습을 준비
**컨투어 설치**

컨투어는 수 많은 인그레스 컨트롤러 중 하나
</br>

**업스트림 서비스 생성**
```{figure} /images/kuar/ch08/service-creation.png
:alt: image
:width: 540px
```

``` {note}
Mac + Minikube를 위한 설정
- Contour ❌ → NGINX 인그레스 ✅
- 로컬 hosts 파일 변경 (`/etc/hosts`)
```

## 인그레스 사용
### 가장 간단한 사용법
= 모든 트래픽을 업스트림 서비스에 전달!

**simple-ingress.yaml 작성**
``` yaml
# simple-ingress.yaml v1
apiVersion: networking.k8s.io/v1
kind: ingress
metadata:
  name: simple-ingress
spec:
  service:
    name: alpaca
    port:
      number: 8080
```

**인그레스 생성**
``` zsh
# 인그레스를 생성한다
kubectl apply -f simple-ingress.yaml
```

**트러블 슈팅**

``` {Error}
no kind "ingress" is registered for version "networking.k8s.io/v1" in scheme "pkg/api/legacyscheme/scheme.go:30"
```

``` yaml
# simple-ingress.yaml v2
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
spec:
  service:
    name: alpaca
    port:
      number: 8080
```

``` {Error}
strict decoding error: unknown field "spec.service"
```

``` yaml
# simple-ingress.yaml v3
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-ingress
spec:
  defaultBackend:
    service:
      name: alpaca
      port:
        number: 8080
```
**결과**
alpaca.example.com / bandicoot.example.com → alpaca

### 호스트 이름 사용
= HTTP 호스트 헤더를 기반으로 트래픽을 전달

**host-ingress.yaml 작성 (트러블 슈팅 이후 버전)**
``` yaml
# host-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
spec:
  defaultBackend:
    service:
      name: be-default
      port:
        number: 8080
  rules:
    - host: alpaca.example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: alpaca
              port:
                number: 8080
```

**결과**
alpaca.example.com → alpaca
bandicoot.example.com → be-default


### 경로 사용
= HTTP 요청의 경로를 기반으로 트래픽을 전달

**path-ingress.yaml 작성 (트러블 슈팅 이후 버전)**
``` yaml
# path-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
spec:
  rules:
    - host: bandicoot.example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: bandicoot
              port:
                number: 8080
        - path: /a/
          pathType: Prefix
          backend:
            service:
              name: alpaca
              port:
                number: 8080
```

**결과**
bandicoot.example.com/a → alpaca
bandicoot.example.com/ → bandicoot

## 인그레스 심화편
### 다중 인그레스 컨트롤러 실행
인그레스 리소스가 인그레스 컨트롤러에 대한 특정 구현을 요청할 수 있도록 `IngressClass` 리소스가 존재

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: hello-world.example
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 8080
```
- `spec.ingressClassName` 필드를 사용하여 특정 **인그레스 리소스를 명시**

### 다중 인그레스 객체
하나의 인그레스 컨트롤러에 여러 개의 인그레스 객체를 지정할 수 있음

그러나 중복/충돌하는 컨피규레이션을 지정하면 동작이 정의되지 않을 수 있음

### 인그레스와 네임스페이스
여러 보안 유의 사항으로 때문에, 인그레스 객체는 동일한 네임스페이스에 있는 업스트림 서비스만 참조 가능

> 하나의 인그레스 컨트롤러에 여러 개의 인그레스 객체를 지정할 수 있음

이 경우, 한 네임스페이스의 인그레스 객체가 다른 네임스페이스에 문제를 유발할 수 있음

### 경로 재작성
HTTP 요청에 대한 경로를 수정하는 데 사용할 수 있음
- /sub-path → /app-path

경로를 지정할 때 정규식도 지원하는 구현체가 여럿 존재함

### TLS 제공
시크릿(secret)을 정의하면 인증서가 업로드되고, 이러면 인그레스 객체에서 인증서를 참조할 수 있음

시크릿을 만드는 방법
```
kubectl create secret tls <시크릿 이름> --cert <인증서 pem 파일> -key <개인키 pem 파일>
```

### 인그레스의 대체 구현
- 클라우드 서비스 제공자(CSP)가 제공하는 인그레스 구현 (ex. AWS Load Balancer Controller)
- NGINX 인그레스 컨트롤러
- Envoy 기반 인그레스 컨트롤러
- Traefik
- …

### 인그레스의 미래
``` {Tip}
"Gateway API가 미래다"
 
Gateway API
- 네트워킹 전담 쿠버네티스(SIG)에서 개발 중인 프로젝트
- HTTP 밸런싱에 중점을 두고 있지만, Layer 4 밸런싱을 제어하기 위한 리소스도 포함됨
```

```{figure} /images/kuar/ch08/gateway-api.png
:alt: image
:width: 540px
```
(https://gateway-api.sigs.k8s.io/)

