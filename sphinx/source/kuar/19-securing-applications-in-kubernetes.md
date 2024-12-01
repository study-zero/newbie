# 19장. 쿠버네티스에서 애플리케이션 보안

쿠버네티스는 다양한 보안 중심 API(Secury-focused API)를 제공한다.

심층 방어와 최소 권한의 원칙의 개념을 이해하는 것이 파드의 보안을 강화할 때 중요하다.

<blockquote>
<dl>
<dt>심층 방어(DID, Defense In Depth)</dt>
<dd>k8s를 포함하는 컴퓨팅 시스템에서 여러 계층에 걸쳐 보안 제어를 사용하는 개념</dd>
<dt>최소 권한의 원칙(principle of least privilege)</dt>
<dd>워크로드가 동작하는 데 필요한 리소스에만 접근할 수 있게 하는 것</dd>
</dl>
</blockquote>

<br/>

## SecurityContext 이해

**SecurityContext**는 파드와 컨테이너에 적용될 수 있는 보안 중심 필드(security-focused field)의 집합이다.

SeuciryContext에서 다루는 보안 제거의 예는 다음과 같다.

- 사용자 권한 및 접근 제어
- 읽기 전용 루트 파일 시스템
- 권한 상승 허용
- Seccomp, AppArmor, SELinux 프로필과 라벨 할당
- 특권(privileged) 또는 비권한(unprevileged)으로 실행

[19-1-kuard-pod-securitycontext.yaml](https://github.com/kubernetes-up-and-running/examples/blob/master/19-1-kuard-pod-securitycontext.yaml)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
    - image: gcr.io/kuar-demo/kuard-amd64:blue
      name: kuard
      securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          privileged: false
      ports:
        - containerPort: 8080
          name: http
          protocol: TCP
```

<blockquote>
<dl>
<dt>runAsRoot</dt>
<dd>컨테이너가 루트 사용자로 실행되는지 여부를 지정<br/>
설정이 true고 루트 사용자로 실행 중인 경우 컨테이너 구동이 실패</dd>

<dt>runAsUser/runAsGroup</dt>
<dd>컨테이너가 실행될 때 사용할 사용자 및 그룹 ID를 지정</dd>

<dt>fsGroup</dt>
<dd>파드가 마운트하는 볼륨의 파일 시스템 그룹 ID를 지정<br/>
fsGroupChangePolicy를 사용해 정확한 동작 구성 가능</dd>

<dt>allowPrivilegeEscalation</dt>
<dd>컨테이너 프로세스가 상위 프로세스보다 더 많은 권한을 얻을 수 있는지 여부<br/>
privieged: true인 경우 해당 설정도 trueㄹ로 설정됨</dd>

<dt>privileged</dt>
<dd>컨테이너를 호스트와 동일한 권한으로 상승시킨 상태로 실행</dd>

<dt>readOnlyRootFilesystem</dt>
<dd>컨테이너의 루트 파일 시스템을 읽기 전용으로 마운트</dd>
</dl>
</blockquote>

```console
$ k exec -it kuard -- ash
// 참고:
// double dash (--)는 kubectl 명령의 인자와 컨테이너 내에서 실행할 명령을 구분하는 역할을 함
// https://kubernetes.io/docs/tasks/debug/debug-application/get-shell-running-container/

$ id
uid=1000 gid=3000 groups=2000,3000
// uid(User IDentifier): 각 사용자를 고유하게 식별하는 숫자
// gid(Group IDentifier): 각 그룹을 고유하게 식별하는 숫자
// groups: 여러 사용자를 포함하는 실제 그룹 집합(gid는 groups를 식별하는데 사용하는 숫자)

$ ps
PID   USER     TIME  COMMAND
    1 1000      0:00 {kuard} /mnt/rosetta /kuard
   29 1000      0:00 {ash} /mnt/rosetta /bin/ash
assertion failed [!result.is_error]: Failed to create temporary file (ThreadContextFcntl.cpp:84 create_tempfile)
 Trace/breakpoint trap (core dumped)

$ touch file
touch: file: Read-only file system

```

SecurityContext에서 다루는 핵심 운영체제 제어 집합의 목록

<blockquote>
<dl>
<dt>Capabilities</dt>
<dd>워크로드가 동작하는 데 필요할 수 있는 권한 그룹의 추가나 제거를 허용<br/>
예를 들어 최소 권한의 원칙을 위해 파드에 루트 권한 부여 대신 NET_ADMIN를 허용하여 네트워킹 구성 권한을 부여할 수 있다.</dd>
<dt>AppArmor</dt>
<dd>프로세스가 접근할 수 있는 파일을 제어<br/>
container.apparmor.security.beta.kubernetes.io/<container_name>:<profile_ref> 어노테이션을 통해 적용<br/>
<profile_ref>의 종류는 runtime/default, localhost/<path to profile>, unconfined(기본값)</dd>
<dt>Seccomp(Secure Computing)</dt>
<dd>syscall에 대한 필터를 생성</dd>
<dt>SELinux</dt>
<dd>파일 및 프로세스에 대한 접근 제어<br/>
기본적으로 각 컨테이너에 임의의 SELinux 콘텍스트가 할당됨</dd>
</dl>
</blockquote>

<br/>

보안 제어 적용을 확인하기 위해 [amicontained](https://github.com/genuinetools/amicontained)를 사용할 것이다.

[19-2-amicontained-pod.yaml](https://github.com/kubernetes-up-and-running/examples/blob/master/19-2-amicontained-pod.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: amicontained
spec:
  containers:
    - image: jess/amicontained:v0.4.9
      name: amicontained
      command: [ "/bin/sh", "-c", "--" ]
      args: [ "amicontained" ]
```

```console
$ k logs amicontained
Container Runtime: not-found
Has Namespaces:
        pid: true
        user: true
User Namespace Mappings:
        Container -> 0  Host -> 501     Range -> 1
        Container -> 1  Host -> 100000  Range -> 1000000
AppArmor Profile: unconfined_u:unconfined_r:spc_t:s0
Capabilities:
        BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_bind_service net_raw sys_chroot mknod audit_write setfcap
Seccomp: disabled
```

[19-3-amicontained-pod-securitycontext.yaml](https://github.com/kubernetes-up-and-running/examples/blob/master/19-3-amicontained-pod-securitycontext.yaml)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: amicontained
  annotations:
    container.apparmor.security.beta.kubernetes.io/amicontained: "runtime/default"
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - image: jess/amicontained:v0.4.9
      name: amicontained
      command: [ "/bin/sh", "-c", "--" ]
      args: [ "amicontained" ]
      securityContext:
        capabilities:
            add: ["SYS_TIME"]
            drop: ["NET_BIND_SERVICE"]
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        privileged: false
```

```console
$ k logs amicontained
Error from server (BadRequest): container "amicontained" in pod "amicontained" is not available

$ k get po
NAME           READY   STATUS     RESTARTS   AGE
amicontained   0/1     AppArmor   0          55s
kuard          1/1     Running    0          31m

$ k describe po amicontained
...
Events:
  Type     Reason     Age   From                 Message
  ----     ------     ----  ----                 -------
  Normal   Scheduled  80s   default-scheduler    Successfully assigned default/amicontained to clu-worker
  Warning  AppArmor   80s   kubelet, clu-worker  Cannot enforce AppArmor: AppArmor is not enabled on the host

// AppArmor 설정 제거 후 재배포
// sys_teim이 추가됨
$ k logs amicontained
Container Runtime: not-found
Has Namespaces:
        pid: true
        user: true
User Namespace Mappings:
        Container -> 0  Host -> 501     Range -> 1
        Container -> 1  Host -> 100000  Range -> 1000000
AppArmor Profile: unconfined_u:unconfined_r:spc_t:s0
Capabilities:
        BOUNDING -> chown dac_override fowner fsetid kill setgid setuid setpcap net_raw sys_chroot sys_time mknod audit_write setfcap
Seccomp: filtering
```

<br/>

### SecurityContext 문제
SecurityContext의 모든 필드를 직접 구성해 보안 제어의 베이스라인을 적용하는 것은 어렵다.

AppArmor, seccomp, SELinux 프로파일과 콘텍스트의 생성 및 관리는 쉽지 않고 구성 중 오류 발생이 쉽다.

에러가 발생하면 어플리케이션의 기능을 손상시킨다.

이러한 문제를 해결하기 위해 Seccomp 프로파일을 쉽게 생성하고 관리할 수 있는 [Security Profiles Operator](https://github.com/kubernetes-sigs/security-profiles-operator)라는 것이 있다.

<br/>

## 파드 보안
파드 보안(Pod Security)은 유효성 검사를 수행한다.

유효성 검사는 SecurityContext가 적용되지 않은 리소스 생성을 허용하지 않는다.

<br/>

### 파드 보안이란?
파드 보안을 사용하면 파드에 대해 서로 다른 보안 프로파일을 선언할 수 있다.

이는 파드 보안 표준(pod security standard)로 알려져 있으며 **네임스페이스 수준**에서 적용된다.

파드 보안 표준은 SecurityContext와 같이 보안에 민감한 필드와 관련 값의 모음이다.

파드 보안 표준은 Mode와 Level로 구성된다.

Mode는 정책 위반 시 행동을 결정하고 Level은 권한을 결정한다.

세 가지 파드 보안 표준(Level)은 다음과 같다.

<blockquote>
<dl>
<dt>baselines</dt>
<dd>쉬운 온보딩이 가능하며 가장 일반적인 권한 상승을 제공</dd>
<dt>restricted</dt>
<dd>보안의 모범 사례가 될 수 있도록 고강도로 제한<br/>
워크로드가 중단될 수 있음</dd>
<dt>privileged</dt>
<dd>개방적이며 제한이 없음</dd>
</dl>
</blockquote>

각 파드의 보안 표준은 파드 명세서의 필드 목록과 허용되는 값을 정의한다.

- `spec.securityContext`
- `spec.containers[*].securityContext`
- `spec.containers[*].ports`
- `spec.volumes[*].hostPath`
- ...

전체 필드는 [공식 문서](https://kubernetes.io/docs/concepts/security/pod-security-standards/)에서 확인할 수 있다.

각 표준은 주어진 모드를 사용해 네임스페이스에 적용된다.

세 가지 모드(Mode)는 다음과 같다.

<blockquote>
<dl>
<dt>Enforce</dt>
<dd>정책을 위반하는 모든 파드를 거부</dd>
<dt>Warn</dt>
<dd>정책을 위반하는 파드를 허용하며 경고 메시지를 표시</dd>
<dt>Audit</dt>
<dd>정책을 위반하는 파드를 허용하며 감사 로그와 감사 메시지를 생성</dd>
</dl>
</blockquote>

### 파드 보안 표준 적용

파드 보안 표준은 라벨을 사용해 네임스페이스에 적용된다.
- 필수: `pod-security.kubernetes.io/<MODE>: <LEVEL>`
- 선택: `pod-security.kubernetes.io/<MODE>-version: <VERSION>`

여러 모드를 사용해 서로 다른 표준에 대한 행동을 다양하게 결정할 수 있다.

[19-4-baseline-ns.yaml](https://github.com/kubernetes-up-and-running/examples/blob/master/19-4-baseline-ns.yaml)
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: baseline-ns
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.22
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.22
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.22
```

처음 정책을 배포하는 것은 어려운 작업이나<br/>
dry-run 명령을 통해 파드 보안 표준을 위반하는 기존 워크로드를 쉽게 확인할 수 있다.

```console
// --dry-run=server 표현은 1.18 버전부터 deprecated됨
// --dry-run=true로 대체되었으며 동일한 표현
// https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/576-dry-run/README.md#removing-a-deprecated-flag
$ k label --dry-run=true --overwrite ns \
--all pod-security.kubernets.io/enforce=baseline
```

baseline-ns를 만들고 아래 파드를 생성한다.

<a id="19-6-kuard-pod.yaml" href="https://github.com/kubernetes-up-and-running/examples/blob/master/19-6-kuard-pod.yaml">19-6-kuard-pod.yaml</a>
```console
$ k apply -f - --namespace baseline-ns << EOF
apiVersion: v1
kind: Pod
metadata:
  name: kuard
  labels:
    app: kuard
spec:
  containers:
    - image: gcr.io/kuar-demo/kuard-amd64:blue
      name: kuard
      ports:
        - containerPort: 8080
          name: http
          protocol: TCP
      securityContext: # 추가해봄
          allowPrivilegeEscalation: true
          readOnlyRootFilesystem: false
          privileged: false # true일 경우 아래 메시지
EOF
// output
Error from server (Forbidden): error when creating "STDIN": pods "kuard" is forbidden: violates PodSecurity "baseline:v1.22": privileged (container "kuard" must not set securityContext.privileged=true)
```

실제로 해보니 warn, audit 로그가 남지 않아서 최신 쿠버네티스는 동작이 달라진 게 아닐지 추정

## 서비스 계정 관리
**서비스 계정(service account)**는 파드 내부에서 실행되는 워크로드에 ID를 제공하는 쿠버네티스 리소스다.

RBAC를 서비스 계정에 적용해 쿠버네티스 API를 통해 ID가 접근할 수 있는 리소스를 제어할 수 있다(14장).

애플리케이션에서 쿠버네티스 API가 필요하지 않은 경우 최소 권한 원칙에 따라 접근을 비활성화 해야한다.

쿠버네티스는 각 네임스페이스에 기본 서비스 계정을 생성한다.

이 계정이 자동으로 모든 파드에 대한 서비스 계정으로 설정된다.

이 서비스 계정에는 각 파드에 자동으로 마운트되고 쿠버네티스 API에 접근하는 데 사용되는 토큰이 포함돼 있다.

이 동작을 비활성화 하려면 서비스 계정 컨피규레이션에 automountServiceAcountToken: false를 추가해야한다.

[19-7-service-account.yaml](https://github.com/kubernetes-up-and-running/examples/blob/master/19-7-service-account.yaml)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
automountServiceAccountToken: false
```

<br/>

## 역할 기반 접근 제어
RBAC(14장)를 워크로드의 보안 상태(security posture)를 보완하는 데 적용할 수 있다. 

<br/>

## 런타임 클래스
쿠버네티스는 Container Runtime Interface(CRI)를 통해 노드 운영체제의 컨테이너 런타임과 상호작용한다.

컨테이너 런타임은 구현 방식에 따라 더 강력한 보안을 보장하는 것을 포함해 다양한 수준의 격리를 제공할 수 있다.

Kata container, Firecracker, gViosr와 같은 프로젝트의 경우 중첩 가상화에서 정교한 시스템콜 필터링에 이르기까지 다양한 격리 메커니즘을 기반으로 한다.

쿠버네티스는 사용자가 워크로드 유형에 따라 컨테이너 런타임을 선택할 수 있는 유연함을 제공한다.

RuntimeClass API는 컨테이너 런타임을 선택할 수 있도록 도입됐다.

```{note}
새로운 런타임 클래스는 클러스터 관리자에 의해 구성되어야 한다.
워크로드가 올바른 노드에 스케쥴링되려면 nodeSelector나 [toleration](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)이 필요할 수 있다.
```

파드 명세서에서 runtimeClassName을 지정해 런타임 클래스를 사용할 수 있다.


```{figure} /images/kuar/ch19/figure9-1.png
:alt: image
:width: 360px

Figure 19-1. RuntimeClass Flow Diagram
```

[19-8-kuard-pod-runtimeclass.yaml](https://github.com/kubernetes-up-and-running/examples/blob/master/19-8-kuard-pod-runtimeclass.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard
  labels:
    app: kuard
spec:
  runtimeClassName: firecracker
  containers:
    - image: gcr.io/kuar-demo/kuard-amd64:blue
      name: kuard
      ports:
        - containerPort: 8080
          name: http
          protocol: TCP
# Error from server (Forbidden): error when creating "STDIN": pods "kuard" is forbidden: pod rejected: RuntimeClass "firecracker" not found
# 
# $ kubectl get runtimeclass
# No resources found in default namespace.
```

런타임 클래스를 사용하면 워크로드가 민감한 정보를 처리하거나 신뢰할 수 없는 코드를 실행하는 경우 전박적인 보안 사항을 보완할 수 있다.


<br/>

## 네트워크 정책
ingress 및 egress 네트워크 정책을 모두 생성할 수 있는 네트워크 정책 API가 있다.

네트워크 정책은 파드를 선택할 수 있는 라벨을 사용해 구성된다.

인그레스와 같은 네트워크 정책은 실제로 연결된 쿠버네티스 컨트롤러와 함께 제공되지 않으며<br/>
네트워크 정책 리소스 생성 시 동작하는 컨트롤러를 설치하지 않은 경우 적용되지 않는다.

네트워크 정책 리소스는 Calico, Cilium, Weave Net과 같은 네트워크 플러그인으로 구성된다.

**NetworkPolicy** 리소스는 podSelector(필수), policyTypes, ingress, egress 섹션으로 구성된다.

podSelector 필드가 비어있는 경우 네임스페이스의 모든 파드에 매치된다.

podSelector 필드에는 matchLabels 섹션도 포함될 수 있다.

파드가 NetworkPolicy 리소스와 매치되는 경우 모든 ingress와 egress 커뮤니케이션에 대해 명시적으로 정의해야한다.

그렇지 않은 경우 **모든 통신은 차단**될 것이다.

파드가 여러 네트워크 정책 리소스와 매치되는 경우 정책이 추가된다.

기본적으로 모든 트래픽을 차단하려고 하는 경우 예제 19-9와 같이 차단(deny) 룰을 만들 수 있다.

[19-9-networkpolicy-default-deny.yaml](https://github.com/kubernetes-up-and-running/examples/blob/master/19-9-networkpolicy-default-deny.yaml)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

네트워크 정책 집합의 예를 살펴보고 워크로드를 보호하는 방법을 소개하겠다.

```console
$ k create ns kuard-networkpolicy
```

<a href="#19-6-kuard-pod.yaml">19-6-kuard-pod.yaml</a>의 파드를 생성

```console
$ k expose pod kuard --port=8080 --target-port=8080 \
--namespace kuard-networkpolicy

$ k run test-source --rm -ti --image busybox /bin/sh \
--namespace kuard-networkpolicy

# wget -q kuard:8080 -O -
<!doctype html>
...

$ k apply -f - --namespace kuard-networkpolicy << EOF
heredoc> apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF

# wget -q --timeout=1 kuard:8080 -O -
```

default-deny-ingress 적용 시 기본 적용된 차단 네트워크 정책으로 인해 테스트 출발지 파드에서 더 이상 kuard 파드에 접근할 수 없게 됐다.

테스트 출발지에서 kuard 파드로 접근을 허용하는 네트워크 정책을 생성해보자.

[19-12-networkpolicy-kuard-allow-test-source.yaml](https://github.com/kubernetes-up-and-running/examples/blob/master/19-12-networkpolicy-kuard-allow-test-source.yaml)
```console
$ k apply -f - --namespace kuard-networkpolicy << EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: access-kuard
spec:
  podSelector:
    matchLabels:
      app: kuard
  ingress:
    - from:
      - podSelector:
          matchLabels:
            run: test-source
EOF

$ k run test-source --rm -ti --image busybox /bin/sh \
--namespace kuard-networkpolicy

# wget -q kuard:8080 -O -
<!doctype html>
...
```

<br/>

## 서비스 메시
서비스 메시(15장)는 워크로드의 보안 상태를 높이는 데도 사용할 수 있다.

서비스 메시는 서비스를 기반으로 하는 protocol-aware 정책 컨피규레이션을 허용하는 접근 정책을 제공한다.

서비스 메시는 일반적으로 모든 서비스 간 통신에서 상호 TLS를 구현해서 통신이 암호화될 뿐만 아니라 서비스 ID도 확인한다.

<br/>

## 보안 벤치마크 도구
쿠버네티스 클러스터에 대해 보안 벤치마크를 실행해 컨피규레이션이 baseline을 충족하는지 확인할 수 있는 몇 가지 도구가 있다.

대표적인 도구가 [kube-bench](https://github.com/aquasecurity/kube-bench)다.

kube-bench를 [쿠버네티스용 CIS 벤치마크](https://www.cisecurity.org/benchmark/kubernetes)를 실행할 수 있다.


```console
$ k apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/refs/heads/main/job.yaml
job.batch/kube-bench created

$ k logs job.batch/kube-bench
[INFO] 4 Worker Node Security Configuration
[INFO] 4.1 Worker Node Configuration Files
[FAIL] 4.1.1 Ensure that the kubelet service file permissions are set to 600 or more restrictive (Automated)
...
```

<br/>

## 이미지 보안
파드 내의 코드와 애플리케이션을 안전하게 유지하는 것이 중요하다.

컨테이너 이미지 보안의 기본 사항에는 컨테이너 이미지 레지스티리가 알려진 코드 취약점에 대해 정적 스캔을 수행하는지 확인하는 것이 포함돼 있다.

또한 이미지가 실행된 이후에 취약점을 식별하고 침입과 같은 잠재적으로 악의적인 활동을 찾는 런타임 스캔을 수행하는 도구가 필요하다.

보안 스캔 외에도 컨테이너 이미지의 내용을 최소화해 불필요한 의존성을 제거하는 도구도 있다.

보안 취약점이 발견되면 이미지를 빠르게 수정하고 다시 배포할 수 있도록, 지속적으로 배포하는 시스템에 이미지 보안을 투자하는 것이 중요하다.

<br/>

## 요약

다양한 보안 중심 API와 리소스를 다뤘다.

SecurityContext를 이용한 심층 방어와 allowPrivilegeEscalation를 이용한 최소 권한의 원칙을 연습해보고 쿠버네티스의 기본 보안을 개선해볼 수 있었다.
