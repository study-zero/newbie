# 5장. 파드
<br/>

## 파드의 필요성

서비스 레벨에 차이가 있으면서 한 머신에 있어야 할 공생 관계를 가진 경우를 컨트롤 하기 위해서

- local filesystem을 공유하는 git 동기화 컨테이너와 웹 서빙 컨테이너
- MSA에서 각 어플리케이션과 해당 어플리케이션의 정보 및 네트워크를 컨트롤하는 사이드카

<br/>

## 쿠버네티스에서의 파드

- 쿠버네티스 컨테이너의 배포 단위로 같은 파드에 속한 컨테이너 들은 모두 같은 머신 혹은 node에 속한다.
- 동일 파드
    - 동일 ip
    - 포트
    - 네트워크 네임스페이스
    - 호스트 명(UTS 네임스페이스)
        - UTS 네임스페이스 - 리눅스 커널의 네임스페이스 중 하나로 시스템의 호스트명과 도메인 이름을 격리하는 역할을 함, kubernetes에서 pod안의 모든 컨테이너들은 동일한 uts네임스페이스를 공유하고 그 결과 pod내의 모든 컨테이너들은 동일한 호스트명을 가짐. 이를 통해 하나의 pod내에서 실행 중인 여러 컨테이너들은 같은 호스트를 사용할 수 있게 되고 이것이 네트워크 상호작용을 간단하게 만드는 효과를 만든다고 한다.
    - 프로세스 간 통신 채널로 동일 파드에 속하는 컨테이너는 서로 소통 가능

<br/>

## 파드에 대한 생각

- 파드 안의 존재들은 모두 같은 스케일링 전략을 가져야 한다.
- 각기 다른 머신에 존재하여도 문제없이 실행될 수 있다면 같은 pod에 놓는 것은 좋지 못한 선택일 수 있다.
    - ex) database와 웹서버

<br/>

## 파드 매니페스트

- 정의: 쿠버네티스 api 객체를 단순히 텍스트 파일 형태로 표현한 것
- declarative configuration
    - 상태를 지정하고 해당 상태를 유지하도록 조치를 취하는 서비스로 제출하는 것
- 처리과정
    1. 매니페스트 파일을 API 서버에 제출
    2. 매니페스트 검증 및 수락 + 처리
    3. 영구 저장소(etcd)에 저장
        1. etcd: Kubernetes의 분산 키-값 저장소로, 클러스터의 상태 정보를 모두 저장합니다. 이에는 파드 매니페스트를 비롯한 클러스터 내의 모든 오브젝트의 상태(예: 파드, 서비스, 컨피그맵, 시크릿, 노드 정보 등)가 포함됩니다.(gpt)
    4. 스케줄러가 노드에 파드를 배치
    5. kubelet에 의해 파드 실행
    6. 매니페스트 파일에 기반하여 실제 객체를 지속적으로 관리

<br/>

## 파드 생성

- kubectl run
    
    ```bash
    user@AL02502768 kind % kubectl run kuard --generator=run-pod/v1 \
    --image=gcr.io/kuar-demo/kuard-amd64:blue
    pod/kuard created
    ```
    
    - —generator 옵션
        - 종류
            - `run-pod/v1`: Pod를 생성하기 위한 생성기.
            - `deployment/apps.v1`: Deployment를 생성하기 위한 생성기.
        - 구 버전에서 쓰이는 방식이고 현재는 다음과 같은 방식으로 대체되었다고 한다.
            
            ```bash
            kubectl run <pod-name> --image=<image-name> --restart=Never
            ```
            
            ![alt text](image.png)
            
- 생성된 pods 확인
    
    ```bash
    user@AL02502768 kind % kubectl get pods  
    NAME    READY   STATUS    RESTARTS   AGE
    kuard   1/1     Running   0          15m
    ```
    
- 특정 파드 삭제
    
    ```bash
    user@AL02502768 kind % kubectl delete pods/kuard
    pod "kuard" deleted
    ```
    

<br/>

## 파드 매니페스트 생성

- yaml이나 json으로 생성 가능하지만 주로 yaml 사용(comment)
- key-value 스타일
- metadata vs spec section
    - metadata section: 파드와 그 라벨에 대한 설명
    - spec: 컨테이너 목록 설명
- 예시코드:
    
    ```yaml
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
    ```
    

<br/>

## 파드 실행

- apply 명령 사용
    
    ```bash
    user@AL02502768 kind % kubectl apply -f kuard-pod.yaml
    pod/kuard created
    ```
    

<br/>

## 파드 조회

- get 명령 사용
    
    ```bash
    user@AL02502768 kind % kubectl get pods               
    NAME    READY   STATUS    RESTARTS   AGE
    kuard   1/1     Running   0          20s
    ```
    

<br/>

## 파드 세부 사항

- describe 명령어 사용

```bash
user@AL02502768 kind % kubectl describe pods kuard
# 파드에 대한 기본적인 정보
Name:         kuard
Namespace:    default
Priority:     0
Node:         kind-control-plane/10.89.1.4
Start Time:   Mon, 02 Sep 2024 01:28:18 +0900
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                ""{"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"name":"kuard","namespace":"default"},"spec":{"containers":[{"image":"gcr.io/..."
Status:       Running
IP:           10.244.0.6
IPs:
  IP:  10.244.0.6

## 파드 내 컨테이너에 대한 설명
Containers:
  kuard:
    Container ID:   containerd://33deba34b9f3e27905b686f25d97bd5b46e4af1d184231f74e7bcd183187e2a7
    Image:          gcr.io/kuar-demo/kuard-amd64:blue
    Image ID:       gcr.io/kuar-demo/kuard-amd64@sha256:1ecc9fb2c871302fdb57a25e0c076311b7b352b0a9246d442940ca8fb4efe229
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 02 Sep 2024 01:28:18 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-5btk2 (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-5btk2:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute for 300s
                             node.kubernetes.io/unreachable:NoExecute for 300s
## 이벤트 정보
Events:
  Type    Reason     Age    From                         Message
  ----    ------     ----   ----                         -------
  Normal  Scheduled  8m58s  default-scheduler            Successfully assigned default/kuard to kind-control-plane
  Normal  Pulled     8m58s  kubelet, kind-control-plane  Container image "gcr.io/kuar-demo/kuard-amd64:blue" already present on machine
  Normal  Created    8m58s  kubelet, kind-control-plane  Created container kuard
  Normal  Started    8m58s  kubelet, kind-control-plane  Started container kuard
```

- man page
    
    ```bash
    user@AL02502768 kind % kubectl describe --help    
    Show details of a specific resource or group of resources
    
     Print a detailed description of the selected resources, including related resources such as events or controllers. You
    may select a single object by name, all objects of that type, provide a name prefix, or label selector. For example:
    
      $ kubectl describe TYPE NAME_PREFIX
      
     will first check for an exact match on TYPE and NAME_PREFIX. If no such resource exists, it will output details for
    every resource that has a name prefixed with NAME_PREFIX.
    
    Use "kubectl api-resources" for a complete list of supported resources.
    
    Examples:
      # Describe a node
      kubectl describe nodes kubernetes-node-emt8.c.myproject.internal
      
      # Describe a pod
      kubectl describe pods/nginx
      
      # Describe a pod identified by type and name in "pod.json"
      kubectl describe -f pod.json
      
      # Describe all pods
      kubectl describe pods
      
      # Describe pods by label name=myLabel
      kubectl describe po -l name=myLabel
      
      # Describe all pods managed by the 'frontend' replication controller (rc-created pods
      # get the name of the rc as a prefix in the pod the name).
      kubectl describe pods frontend
    
    Options:
      -A, --all-namespaces=false: If present, list the requested object(s) across all namespaces. Namespace in current
    context is ignored even if specified with --namespace.
      -f, --filename=[]: Filename, directory, or URL to files containing the resource to describe
      -k, --kustomize='': Process the kustomization directory. This flag can't be used together with -f or -R.
      -R, --recursive=false: Process the directory used in -f, --filename recursively. Useful when you want to manage
    related manifests organized within the same directory.
      -l, --selector='': Selector (label query) to filter on, supports '=', '==', and '!='.(e.g. -l key1=value1,key2=value2)
          --show-events=true: If true, display events related to the described object.
    
    Usage:
      kubectl describe (-f FILENAME | TYPE [NAME_PREFIX | -l label] | TYPE/NAME) [options]
    
    Use "kubectl options" for a list of global command-line options (applies to all commands).
    ```
    
    - 관련 설명 필요

<br/>

## 파드 삭제

```bash
user@AL02502768 kind % kubectl delete pods/kuard
pod "kuard" deleted
```
<br/>

## 파드에 접근

### 로그를 통해 더 많은 정보 얻기

- logs 명령어 사용
    
    ```bash
    user@AL02502768 kind % kubectl logs kuard
    2024/09/01 16:45:52 Starting kuard version: v0.10.0-blue
    2024/09/01 16:45:52 **********************************************************************
    2024/09/01 16:45:52 * WARNING: This server may expose sensitive
    2024/09/01 16:45:52 * and secret information. Be careful.
    2024/09/01 16:45:52 **********************************************************************
    2024/09/01 16:45:52 Config: 
    {
      "address": ":8080",
      "debug": false,
      "debug-sitedata-dir": "./sitedata",
      "keygen": {
        "enable": false,
        "exit-code": 0,
        "exit-on-complete": false,
        "memq-queue": "",
        "memq-server": "",
        "num-to-gen": 0,
        "time-to-run": 0
      },
      "liveness": {
        "fail-next": 0
      },
      "readiness": {
        "fail-next": 0
      },
      "tls-address": ":8443",
      "tls-dir": "/tls"
    }
    2024/09/01 16:45:52 Could not find certificates to serve TLS
    2024/09/01 16:45:52 Serving on HTTP on :8080
    ```
    

### exec을 사용해 컨테이너에서 명령 실행

```bash
user@AL02502768 kind % kubectl exec -it kuard -- ash 
~ $ exit
```

- 참고사항: bash는 없음
    
    ```bash
    user@AL02502768 kind % kubectl exec -it kuard -- bash
    error: Internal error occurred: error executing command in container: failed to exec in container: failed to start exec "56fbf70a8f69023e46ca86000a3415fa23e0d3302d0e5598aa361169953148ba": OCI runtime exec failed: exec failed: unable to start container process: exec: "bash": executable file not found in $PATH: unknown
    ```
    

### 컨테이너 내외부로 파일 복사

- 안티패턴이지만 장애를 빠르게 해결하는 방안으로는 사용 가능
- 단 사용 후 이미지에 반영하여야 함

<br/>

## 상태 검사

- 프로세스 상태 검사 → 파드를 항상 살아 있는 상태로 유지할 수 있음
    - 검사 실패 시 프로세스 재시작
- 그러나 주요 프로세스를 항상 실행하게 하는 단순한 방식이어서 데드락 같은 상황을 처리할 수 없음
- 각 어플리케이션 별로 특화된 상태 검사가 필요
    - 프로브라고 불리며 매니페스트에 정리

### 활성 프로브

- 활성 프로브를 추가한 매니페스트
    
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: kuard
    spec:
      containers:
        - image: gcr.io/kuar-demo/kuard-amd64:blue
          name: kuard
          livenessProbe:
            httpGet:
              path: /healthy
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 1
            failureThreshold: 3
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
    ```
    
- 강제 실패 시:
    
    ```bash
    87s         Normal    Created     pod/kuard   Created container kuard
    87s         Normal    Started     pod/kuard   Started container kuard
    87s         Warning   Unhealthy   pod/kuard   Liveness probe failed: HTTP probe failed with statuscode: 500
    87s         Normal    Killing     pod/kuard   Container kuard failed liveness probe, will be restarted
    ```
    

### 준비 프로브

- 사용자 요청을 받을 수 있는 상태인지에 대한 체크
- 실패시 성공할 때까지 로드밸런서에서 제외, 주기적으로 시도하여 성공시 다시 트래픽 받음

### 시작 프로브

- 컨테이너가 시작될 때 한 번 실행됨. 이 프로브가 성공할 때까지 쿠버네티스는 컨테이너가 정상적으로 시작되고 있다고 가정하며, 성공하면 준비 및 라이브니스 프로브가 활성화됨
- 만약 시작 프로브가 실패하면, 컨테이너는 재시작되거나 실패 상태로 간주될 수 있음.

### 상태 검사의 기타 기입

tcp와 exec 프로브 존재

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard
spec:
  containers:
    - image: gcr.io/kuar-demo/kuard-amd64:blue
      name: kuard
      readinessProbe:
        tcpSocket:
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 1
      livenessProbe:
        httpGet:
          path: /healthy
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 1
        failureThreshold: 3
      ports:
        - containerPort: 8080
          name: http
          protocol: TCP

```
<br/>

## 리소스 관리
### 리소스 관리가 필요한 이유
- 컴퓨트 노드의 전체 사용률을 높여서 비용 효율성을 높이기 위해
- 이를 위해서는 각 컨테이너가 필요한 리소스를 쿠버네티스에 알려주어야 할 필요성이 있음

### 기호 표기 방법
- literals:
  - 메모리 자원을 설정할 때 사용하는 표기 방식으로 기본 단위는 byte
  - MB/GB/PB와 MiB/GiB/PiB의 차이 비교
    - 1 MB = 1000KB
    - 1 MIM = 1024KIB
- millicores:
  - CPU 리소스를 설정할 때 사용하는 단위
    - 1 core == 1000m

### 리소스 요청
- 개념: 해당 어플리케이션이 보장받아야 하는 최소 리소스 량
- 예시코드
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: kuard
  spec:
    containers:
      - image: gcr.io/kuar-demo/kuard-amd64:blue
        name: kuard
        resources:
          requests:
            cpu: "500m"
            memory: "128Mi"
        ports:
          - containerPort: 8080
            name: http 
            protocol: TCP
  ```
- 리눅스 커널 cpu-shares:
  - [관련링크](https://www.redhat.com/sysadmin/cgroups-part-two)

### 리소스 제한
- 개념: 해당 어플리케이션이 받을 수 있는 최대 리소스 량
- 예시코드
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: kuard
  spec:
    containers:
      - image: gcr.io/kuar-demo/kuard-amd64:blue
        name: kuard
        resources:
          requests:
            cpu: "500m"
            memory: "128Mi"
          limits:
            cpu: "800m"
            memory: "256Mi"
        ports:
          - containerPort: 8080
            name: http 
            protocol: TCP
  ```
</br>

## 볼륨을 통한 데이터 보존
### 파드에서 볼륨 사용
- 해당 파드 매니페스트에서 컨테이너가 접근할 수 있는 모든 볼륨을 정의
- 예시
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: kuard
  spec:
    volumes:
      - name: "kuard-data"
        hostPath: #워커 노드 임의의 위치를 컨테이너에 마운트 할 수 있는 예약 볼륨
          path: "/var/lib/kuard"
  ```
### 컨테이너에서 볼륨 사용
- 특정 컨테이너에 마운트될 볼륨과 각 볼륨들이 마운트될 경로 정의
- 예시
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: kuard
  spec:
    volumes:
      - name: "kuard-data"
        hostPath: #워커 노드 임의의 위치를 컨테이너에 마운트 할 수 있는 예약 볼륨
          path: "/var/lib/kuard"
    containers:
      - image: gcr.io/kuar-demo/kuard-amd64:blue
        name: kuard
        volumeMounts:
          - mountPath: "/data"
            name: "kuard-data"
        ports:
          - containerPort: 8080
            name: http
            protocol: TCP
  ```

</br>

## 파드에서 볼륨을 사용하는 다양한 방법
### 통신/동기화
- 두 컨테이너 사이의 볼륨 공유
```yml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  volumes:
    - name: shared-data
      emptyDir: {}  # 이 예제에서는 emptyDir을 사용합니다. Pod의 수명 동안만 데이터를 저장합니다.
  containers:
    - name: container-1
      image: nginx
      volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html  # 첫 번째 컨테이너에서의 마운트 경로

    - name: container-2
      image: busybox
      command: [ "sh", "-c", "echo 'Hello from container-2' > /shared-data/message.txt && sleep 3600" ]
      volumeMounts:
        - name: shared-data
          mountPath: /shared-data
```
### 캐시
- emptyDir 볼륨 공유
```yml
apiVersion: v1
kind: Pod
metadata:
  name: empty-dir-example
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - mountPath: /tmp/emptydir
      name: temp-storage
  volumes:
  - name: temp-storage
    emptyDir: {}
```
- emptydir
  - pod가 생성될 때 자동으로 만들어지며, Pod가 삭제되면 함께 삭제됩
  - 일반적으로는 로컬 디스크에 데이터를 저장하지만 메모리를 기반으로 저장할 수도 있음
    ```yml
    volumes:
    - name: temp-storage
      emptyDir:
        medium: Memory
    ```
### 영구데이터
- 클라우드 같은 원격 네트워크 스토리지 볼륨 사용
### 호스트 파일 시스템 필요시
- hostPath라는 볼륨을 사용
```yml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-example
spec:
  containers:
  - name: hostpath-container
    image: nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: hostpath-volume
  volumes:
  - name: hostpath-volume
    hostPath:
      path: /data/nginx
      type: Directory
```