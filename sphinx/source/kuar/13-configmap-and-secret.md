# 13장. 컨피그맵과 시크릿
컨테이너 이미지는 가능한한 재사용할 수 있게 만드는 것이 좋다.<br/>
<strong>컨피그맵\(ConfigMap\)</strong>과 <strong>시크릿\(Secret\)</strong>을 사용하면 이에 도움을 준다.

컨피그맵은 워크로드에 대한 컨피규레이션 정보를 제공하는 데 사용된다.<br/>
시크릿은 컨피그맵과 유사하지만 TLS 인증서, 자격증명 같은 민감한 정보를 담는 데 사용된다.

```{note}
[워크로드](https://kubernetes.io/ko/docs/concepts/workloads/)란 쿠버네티스에서 구동되는 애플리케이션이다.<br/>
워크로드를는 일련의 파드 집합 내에서 실행된다.<br/>
빌트인(built-in) 워크로드 리소스의 종류로는 Deployment, ReplicaSet 등이 있다.
```

<br/>

## 컨피그맵
컨피그맵은 작은 파일 시스템을 정의하는 k8s 객체와 비슷하다.<br/>
또는 환경 변수나 커맨드라인을 정의하는 변수의 집합이다.

<br/>

### 컨피그맵 생성
- my-config.txt
```ini
parameter1 = value1
parameter2 = value2
```

- kubectl create
```sh
kubectl create configmap my-config \
  --from-file=my-config.txt \
  --from-literal=extra-param=extra-value \
  --from-literal=another-param=another-value
```

- kubectl get
```console
$ k get cm my-config -o yaml
apiVersion: v1
data:
  another-param: another-value
  extra-param: extra-value
  my-config.txt: |-
    parameter1 = value1
    parameter2 = value2
kind: ConfigMap
metadata:
  creationTimestamp: "2024-10-20T08:40:51Z"
  name: my-config
  namespace: default
  resourceVersion: "532629"
  uid: d97a3300-7d1a-4566-bde2-14c92e7e648e
```

<br/>

### 컨피그맵 사용
<dl>
<dt>파일 시스템</dt>
<dd>컨피그맵을 파드에 마운트할 수 있다.</dd>
<dt>환경 변수</dt>
<dd>환경 변수의 값을 동적으로 설정할 수 있다.</dd>
<dt>커맨드라인 인수</dt>
<dd>커맨드라인을 동적으로 생성할 수 있다.</dd>
</dl>

- kuard-config.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard-config
spec:
  containers:
    - name: test-container
      image: gcr.io/kuar-demo/kuard-amd64:blue
      imagePullPolicy: Always
      command:
        - "/kuard"
        - "$(EXTRA_PARAM)" # 커맨드라인 사용
      env:
        - name: ANOTHER_PARAM # 환경 변수 사용
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: another-param
        - name: EXTRA_PARAM
          valueFrom:
            configMapKeyRef:
              name: my-config
              key: extra-param
      volumeMounts:
        - name: config-volume
          mountPath: /config
      volumes:
        - name: config-volume
          configMap: # 파일 시스템 사용
            name: my-config
```

- kubectl apply
```sh
kubectl apply -f kuard-config.yaml
```

<br/>

## 시크릿
비빈번호, 보안 토큰, 개인키 같은 데이터를 저장하는 **시크릿** 객체가 있다.<br/>
시크릿은 클러스터 etcd 스토리지에 평문으로 저장된다.

<br/>

### 시크릿 생성
```console
$ curl -o kuard.crt https://storage.googleapis.com/kuar-demo/kuard.crt
$ curl -o kuard.key https://storage.googleapis.com/kuar-demo/kuard.key
$ kubectl create secret generic kuard-tls \
  --from-file=kuard.crt \
  --from-file=kuard.key
secret/kuard-tls created
$ k describe secret kuard-tls
Name:         kuard-tls
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
kuard.crt:  1050 bytes
kuard.key:  1679 bytes

```
<br/>

### 시크릿 사용
쿠버네티스 API를 호출하거나 시크릿 볼륨(Secret volume)을 사용한다.<br/>
시크릿 볼륨은 kubelet에 의해 관리되며 파드 생성 시점에 생성된다.<br/>
시크릿은 tmpfs 볼륨(RAM 디스크)에 저장되며 노드의 디스크에는 기록되지 않는다.

- kuard-secret.yml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard-tls
spec:
  containers:
    - name: kuard-tls
      image: gcr.io/kuar-demo/kuard-amd64:blue
      imagePullPolicy: Always
      volumeMounts:
        - name: tls-certs
      mountPath: "/tls"
      readOnly: true
      volumes:
        - name: tls-certs
          secret:
            secretName: kuard-tls
```
<br/>

### 사설 컨테이너 레지스트리
쿠버네티스가 사설 레지스트리에 저장된 이미지 사용을 할 때 자격증명 저장 용도로 시크릿을 사용할 수 있다.

```console
$ kubectl create secret docker-registry my-image-pull-secret \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email-address>
```

-  kuard-secret-ips.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
name: kuard-tls
spec:
  containers:
    - name: kuard-tls
      image: gcr.io/kuar-demo/kuard-amd64:blue
      imagePullPolicy: Always
      volumeMounts:
        - name: tls-certs
      mountPath: "/tls"
      readOnly: true
      imagePullSecrets:
        - name: my-image-pull-secret
      volumes:
        - name: tls-certs
          secret:
            secretName: kuard-tls
```
<br/>

## 명명 규칙
컨피그맵과 시크릿의 키는 regex 제한이 있다.

```regex
^[.[?[a-zAZ0-9[([.[?[a-zA-Z0-9[+[-_a-zA-Z0-9[?)*$
```
<br/>

## 컨피그맵과 시크릿 관리

<br/>

### 조회

<br/>

### 생성

<br/>

### 업데이트

<br/>

## 요약

<br/>
