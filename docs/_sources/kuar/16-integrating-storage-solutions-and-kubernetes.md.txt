# 16장. 스토리지 솔루션과 쿠버네티스의 연계
상태 비저장 방식으로 마이크로서비스를 구축하면 신뢰성를 확보하고 관리가 용이하다.<br/>
하지만 서비스 운영 시 어느 시점에는 어딘가에 저장된 데이터가 필요하다.<br/>
컨테이너 오케스트레이션 솔루션 상의 애플리케이션은 불변하고 선언형인 상태로 관리될 필요가 있다.<br/>
기존 애플리케이션은 데이터 중력(data gravity)로 인해 컨테이너로 운영되더라도<br/>
격리된 상태로 구축되지 않고 기존 VM과 데이터를 공유할 수 있어서 분리가 필요한 복잡성이 요구된다.

15장에서는 쿠버네티스 환경에서 스토리지를 컨테이너형 마이크로서비스로<br/>연계하기 위한 다양한 접근 방법을 살펴본다.

## 외부 서비스 가져오기
데이터베이스가 실행되고 있는 기존 시스템과 통합하는 방법이다.<br/>
레거시 서버와 서비스를 쿠버네티스에 포함시킬 경우<br/>
쿠버네티스에서 제공하는 모든 내장 네이밍 및 서비스 탐색의 기본 기능 같은 이점을 활용할 수 있다.

```yaml
kind: Service
metadata:
  name: my-database
  # note 'test' namespace here
  namespace: test
```

위와 같은 서비스는 아래와 같은 포인터를 가진다.
- my-database.test.svc.cluster.internal

네임스페이스를 prod로 설정할 경우 아래와 같은 포인터를 가진다.
- my-database.prod.svc.cluster.internal

### 셀렉터가 없는 서비스

데이터베이스가 실행 중인 특정 서버를 가리키는 DNS를 서비스와 연결할 수 있다.

*예제 15-1. dns-service.yaml*
```yaml
kind: Service
apiVersion: v1
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: database.company.com
```

ExternalName 타입의 서비스를 생성하면 사용자가 지정한 외부 서비스를 가리키는 CNAME 레코드를 쿠버네티스 DNS 서비스에 등록한다.

클러스터 내에서 호스트 이름 external-database.svc.default.cluster의 DNS를 조회하면 DNS 프로토콜은 database.company.com의 이름으로 별칭을 부여한다.<br/>
그후 이 이름을 외부 데이터베이스 서버의 IP 주소로 해석한다.

IP 주소로 외부 서비스를 쿠버네티스로 가져오는 것도 가능하다.<br/>
이 경우 서비스와 엔드포인트 객체를 직접 등록해주어야한다.<br/>
또한 서버의 IP 주소의 최산화 책임을 가지게 되기에 고정 IP 사용 또는 자동으로 엔드포인트를 업데이트해주는 절차를 마련해야한다.

*예제 15-2. external-ip-service.yaml*
```yaml
kind: Service
apiVersion: v1
metadata:
  name: external-ip-database
```

*예제 15-3. external-ip-endpoints.yaml*
```yaml
kind: Endpoints
apiVersion: v1
metadata:
  name: external-ip-database
subsets:
  - addresses:
  - ip: 192.168.0.1
    ports:
      - port: 3306
 ```



### 외부 서비스의 제약 사항: 상태 검사
외부 서비스는 health check를 수행하지 않아서 별도로 신뢰성을 확인해야한다.

## 신뢰할 수 있는 싱글톤 실행
쿠버네티스 기본 객체를 활용하되 복제본을 생성하지 않으면 관리가 간단하다.

신뢰할 수 있는 분산 시스템 구축 원칙에 반하는 것으로 보일 수 있지만<br/>
기존 시스템이 단일 가상머신이나 단일 물리머신인 경우가 많고<br/>
소규모 애플리케이션의 경우 제한된 다운타임은 복잡성을 줄일 수 있는 합리적인 트레이드오프 관계로 볼 수 있다.

다음절에서 설명하는 것과 같이 레플리카셋을 사용하면 싱글톤이어도 장애를 어느 정도 대비할 수 있다.

### MySQL 싱글톤 실행
MySQL을 쿠버네티스 내에서 싱글톤으로 실행하기 위해 다음과 같은 세 가지 기본 객체를 생성한다.

- MySQL 애플리케이션과 독립적인 영구 볼륨
- MySQL 애플리케이션 파드
- MySQL 파드를 노출하기 위한 서비스

*예제 15-4. nfs-volume.yaml*
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: database
  labels:
  volume: my-volume
spec:
  accessModes:
    - ReadWriteMany
  capacity:
  storage: 1Gi
  nfs: # nfs
    server: 192.168.0.1
    path: "/exports"
```

아래와 같이 영구 볼륨을 생성 가능하다.
```sh
kubectl apply -f nfs-volume.yaml
```

```{note}
[Network File System(NFS)](https://en.wikipedia.org/wiki/Network_File_System)란 1984년 Sun Microsystems에서 개발한 분산 파일 시스템 프로토콜로 클라이언트 컴퓨터의 사용자가 로컬 저장소에 액세스하는 것과 매우 유사하게 컴퓨터 네트워크를 통해 파일에 액세스할 수 있도록 한다.
```

*예제 15-5. nfs-volume-claim.yaml*
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: database
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
    v olume: my-volume
```

위와 같이 PVC를 통해 영구 볼륨 사용 요청을 할 수 있다.<br/>
굳이 PV가 아닌 PVC를 통해 영구 볼륨을 연결하는 이유는 파드에 대한 정의를 스토리지의 정의에서 분리하기 위함이다(일종의 프록시 느낌).

*예제 15-6. mysql-replicaset.yaml*
```yaml
apiVersion: extensions/v1
kind: ReplicaSet
metadata:
  name: mysql
   # labels so that we can bind a Service to this Pod
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: database
        image: mysql
        resources:
          requests:
            cpu: 1
            memory: 2Gi
        env:
          # 보다 나은 보안은 11장 참고
          - name: MYSQL_ROOT_PASSWORD
        value: some-password-here
        livenessProbe:
        tcpSocket:
        port: 3306
        ports:
          - containerPort: 3306
        volumeMounts:
          - name: database
            # /var/lib/mysql가 MySQL이 데이터를 저장하는 장소
            mountPath: "/var/lib/mysql"
        volumes:
          - name: database
            persistentVolumeClaim:
            claimName: database
```

레플리카셋을 사용해서 파드가 다운될 경우 정상 노드에 스케쥴링을 보장할 수 있다.<br/>
단일 머신 장애 등을 방지하기 위함이다.

*예제 15-7. mysql-service.yaml*
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
ports:
  - port: 3306
    protocol: TCP
    selector:
    app: mysql
```

### 동적 볼륨 프로비저닝
StorageClass 객체를 통해 동적 볼륨 프로비저닝을 사용할 수 있다.

클러스터는 여러 개의 다른 스토리지 클래스가 설치될 수 있다.<br/>
NFS 서버용과 iSCSI 블록 저장소용 등 스토리지 클래스가 함께 존재할 수 있다.

*예제 15-8. storageclass.yaml*
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: default
  annotations:
    storageclass.beta.kubernetes.io/is-default-class: "true"
  labels:
    kubernetes.io/cluster-service: "true"
provisioner: kubernetes.io/azure-disk
```

*예제 15-9. dynamic-volume-claim.yaml*
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: default # default 스토리지 클래스 연결
spec:
  accessModes:
    - ReadWriteOnce
  resources:
  requests:
  storage: 10Gi
```

## 스테이트풀셋을 통한 쿠버네티스 네이티브 스토리지
상태 비저장 애플리케이션이 비식별성을 가지는 것과 달리<br/>
상태 저장 애플리케이션은 식별성을 필요로 한다.

쿠버네티스 1.5에서 스테이트풀셋이 소개됐다.

### 스테이트풀셋 속성
스테이트풀셋은 레플리카셋과 유사하게 복제된 파드의 그룹으로 구성된다.<br/>
하지만 아래와 같은 차이점이 있다.

- 각 복제본은 고유한 인덱스와 함께 영구 호스트 이름을 갖는다.
    (예. database-0, database-1 등)
- 각 복제본은 인덱스 번호가 가장 낮은 순서부터 높은 순서로 생성되고,
    이전 인덱스 번호의 파드가 안정적이고 이용 가능한 상태가 될 때까지 복제본은 생성되지 않는다.
    이러한 특성은 확장(scale up) 시에도 동일하게 적용된다.
- 스테이트풀셋이 삭제될 때 각각의 관리되는 복제본 파드는 가장 높은 번호에서 낮은 순으로 삭제된다.
    이는 축소(scale down)하는 데도 적용된다.

### 스테이트풀셋을 통한 몽고DB 수동 복제
*예제 15-10. mongo-simple.yaml*
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo"
  replicas: 3
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
        - name: mongodb
          image: mongo:3.4.1
          command:
            - mongod
            - --replSet
            - rs0 # 복제본 세트 이름 지정
          ports:
            - containerPort: 27017
          name: peer
```

*예제 15-11. mongo-service.yaml*
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  ports:
    - port: 27017
  name: peer
  clusterIP: None
  selector:
    app: mongo
```

아래와 같이 DNS 항목을 확인할 수 있다.
```console
$ kubectl run -it --rm --image busybox busybox ping mongo-1.mongo
```

아래 처럼 첫번째 파드에 접속해 replicaset을 수동 설정할 수 있다.
```console
$ kubectl exec -it mongo-0 mongo
> rs.initiate( {
  _id: "rs0",
  members:[ { _id: 0, host: "mongo-0.mongo:27017" } ]
  });
  OK
> rs.add("mongo-1.mongo:27017");
> rs.add("mongo-2.mongo:27017");
```

### 몽고DB 클러스터 생성 자동화
컨피그맵 객체에 스크립트를 저장하여 몽고DB 클러스터 생성 자동화에 서용할 수 있다.

```yaml
...
- name: init-mongo
  image: mongo:3.4.1
  command:
    - bash
    - /config/init.sh
  volumeMounts:
    - name: config
      mountPath: /config
  volumes:
  - name: config
    configMap:
      name: "mongo-init"
```

*Example 15-12. mongo-configmap.yaml*
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-init
data:
  init.sh: |
    #!/bin/bash
    # Need to wait for the readiness health check to pass so that the
    # mongo names resolve. This is kind of wonky.
    until ping -c 1 ${HOSTNAME}.mongo; do
    echo "waiting for DNS (${HOSTNAME}.mongo)..."
    sleep 2
    done
    until /usr/bin/mongo --eval 'printjson(db.serverStatus())'; do
    echo "connecting to local mongo..."
    sleep 2
    done
    echo "connected to local."
    HOST=mongo-0.mongo:27017
    until /usr/bin/mongo --host=${HOST} --eval 'printjson(db.serverStatus())'; do
    echo "connecting to remote mongo..."
    sleep 2
    done
    echo "connected to remote."
    if [[ "${HOSTNAME}" != 'mongo-0' ]]; then
    until /usr/bin/mongo --host=${HOST} --eval="printjson(rs.status())" \
    | grep -v "no replset config has been received"; do
    echo "waiting for replication set initialization"
    sleep 2
    done
    echo "adding self to mongo-0"
    /usr/bin/mongo --host=${HOST} \
    --eval="printjson(rs.add('${HOSTNAME}.mongo'))"
    fi
    if [[ "${HOSTNAME}" == 'mongo-0' ]]; then
    echo "initializing replica set"
    /usr/bin/mongo --eval="printjson(rs.initiate(\
    {'_id': 'rs0', 'members': [{'_id': 0, \
    'host': 'mongo-0.mongo:27017'}]}))"
    fi
    echo "initialized"
    while true; do
    sleep 3600
    done
```

*Example 15-13. mongo.yaml*
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo"
  replicas: 3
  template:
metadata:
  labels:
    app: mongo
  spec:
    containers:
    - name: mongodb
      image: mongo:3.4.1
      command:
        - mongod
        - --replSet
        - rs0
      ports:
        - containerPort: 27017
      name: web
      # This container initializes the mongodb server, then sleeps.
    - name: init-mongo
      image: mongo:3.4.1
      command:
        - bash
        - /config/init.sh
      volumeMounts:
        - name: config
          mountPath: /config
      volumes:
        - name: config
          configMap:
          name: "mongo-init"
```

### 영구 볼륨과 스테이트풀셋

```yaml
...
volumeMounts:
  - name: database
    mountPath: /data/db
```
```yaml
volumeClaimTemplates:
  - metadata:
    name: database
    annotations:
    volume.alpha.kubernetes.io/storage-class: anything
    spec:
    accessModes: [ "ReadWriteOnce" ]
    resources:
    requests:
    storage: 100Gi
```
### 마지막 단계: 준비 프로브

```yaml
...
livenessProbe:
  exec:
    command:
      - /usr/bin/mongo
      - --eval
      - db.serverStatus()
  initialDelaySeconds: 10
  timeoutSeconds: 10
```