# 3.7 카프카 미러메이커2
카프카 미러메이커2는 서로 다른 두 개의 카프카 클러스터 간에 토픽을 복제하는 애플리케이션이다.
프로듀서와 컨슈머를 사용해서 직접 미러링하는 애플리케이션을 만들면 되지만 굳이 미러메이커2를 사용하는 이유는 토픽의 모든 것을 복제할 필요성이 있기 때문이다.

### 미러메이커2를 활용한 단방향 토픽 복제
미러메이커2를 사용하기 위해서는 config 디렉토리에 포함된 connect-mirror-maker.properties 파일을 수정해야 한다.
카프카 클러스터A(a-kafka:9092)에서 클러스터B(b-kafka:9092)로 토픽을 복제하는 설정 파일은 다음과 같다.
```properties
clusters = A, B

A.bootstrap.servers = a-kafka:9092
B.bootstrap.servers = b-kafka:9092

A->B.enabled = true
A->B.topics = topic1, topic2

B->A.enabled = false
B->A.topics = .*

replication.factor = 1

checkpoints.topic.replication.factor = 1
heartbeats.topic.replication.factor = 1
off-set-syncs.topic.replication.factor = 1
offset.storage.replication.factor = 1
status.storage.replication.factor = 1
config.storage.replication.factor = 1
```

## 미러메이커2를 활용한 지리적 복제

### 엑티브-스탠바이 클러스터 운영
서비스 애플리케이션들과 통신하는 카프카 클러스터 외에 재해 복구를 위해 임시 카프카 클러스터를 하나 더 구성하는 경우 액티브-스탠바이 클러스터로 운영할 수 있다.
### 액티브-액티브 클러스터 운영
글로벌 서비스를 운영할 경우 서비스 애플리케이션의 통신 지연을 최소화하기 위해 2개 이상의 클러스터를 두고 서로 데이터를 미러링하면서 사용할 수 있다.
### 허브 앤 스포크
각 팀에서 소규모 카프카 클러스터를 사용하고 있을 때 각 팀의 카프카 클러스터의 데이터를 한 개의 카프카 클러스터에 모아 데이터 레이크를 사용할 수 있다.