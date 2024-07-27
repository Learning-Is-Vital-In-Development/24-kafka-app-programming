# 3.6 카프카 커넥트
카프카 커넥트는 카프카 오픈소스에 포함된 툴 중 하나로 데이터 파이프라인 생성 시 반복 작업을 줄이고 효율적인 전송을 이루기 위한 애플리케이션이다.
커넥터는 프로듀서 역할을 하는 `소스커넥터`와 컨슈머 역할을 하는 `싱크 커넥터` 2가지로 구성되어 있다.
사용자가 커넥터를 사용하여 파이프라인을 생성할 때 컨버터와 트랜스폼기능을 옵션으로 추가할 수 있다.
- 컨버터
  - 데이터 처리를 하기 전에 스키마를 변경하도록 도와준다.
- 트랜스폼
  - 데이터 처리 시 각 메시지 단위로 데이터를 간단하게 변환하기 위한 용도로 사용된다.

## 커넥트를 실행하는 방법
커넥트를 실행하는 방법은 2가지가 있다. 하나는 단일 모드 커넥트이고 두번째는 분산 모드 커넥트 이다.
### 단일 모드 커넥트
단일 모드 커넥트는 하나의 커넥터만 실행하는 방법이다.
![image](https://github.com/user-attachments/assets/472dd440-5648-445f-83b7-95614568ab46)

단일 모드 커넥트 설정 파일은 connect-standalone.properties 파일이다.
```properties
# 카프카와 연동할 카프카 클러스터 호스트
# 2개 이상 연동할 경우 콤마(,)로 구분
bootstrap.servers=kafka:9092
 
# 데이터를 카프카에 저장하고 카프카에서 가져올 때 데이터를 변환하는데 사용
# JsonConverter, StringConverter, ByteArrayConverter 기본 제공
# 사용하지 않으려면 false로 설정
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=true
value.converter.schemas.enable=true
 
# 로컬 파일에 오프셋 저장
# 소스/싱크 커넥터가 데이터 처리 시점을 저장하기 위해 사용
# 예시) 파일 소스 커넥터: 특정 파일을 읽어 토픽에 저장할 때, 몇 번째 줄까지 읽었는지 저장
# 저장할 경로 및 처리 완료한 테스크의 오프셋 커밋 주기
# 데이터 처리하는데 중요한 정보임으로 접근에 유의
offset.storage.file.filename=/tmp/connect.offsets
offset.flush.interval.ms=10000
 
# 커넥터의 디렉터리 주소
# 여러개의 경우 콤마(,)로 구분
# 컨버터(converter)와 트랜스폼(transform)도 프러그인으로 추가 가능
plugin.path=/usr/local/share/java,/usr/local/share/kafka/plugins
```

단일 모드 커넥트는 커넥트 설정파일과 함께 커넥터 설정파일도 정의하여 실행해야 한다.
커넥터 설정파일은 config 디렉토리에 connect-file-source.properties 파일로 저장되어 있다.
```properties
# 커넥터 이름, 커넥트에서 유일해야 함
name=local-file-source
# 커넥터 클래스 명, 카프카 기본 제공 클래스인 FileStreamSource로 지정
connector.class=FileStreamSource
# 테스크 수, 다수의 파일을 읽어 토픽에 저장하고 싶으면 테스크 수를 늘려 병렬 처리하면 됨
tasks.max=1
# 읽을 파일 위치
file=test.txt
# 데이터를 저장할 토픽 이름
topic=connect-test
```

단일 커넥트를 실행하는 명령어는 다음과 같다.
```shell
$ ./bin/connect-standalone.sh ./config/connect-standalone.properties  ./config/connect-file-source.properties
````

### 분산 모드 커넥트
분산 모드 커넥트는 여러 대의 커넥터를 실행하는 방법이다. 분산 모드 커넥트는 단일 모드 커넥트와 다르게 2개 이상의 프로세스가 1개의 그룹으로 묶여서 운영된다.
![image](https://github.com/user-attachments/assets/2a016420-2c2d-4041-a4ea-638098a0d6e5)
 
분산 모드 커넥트 설정 파일은 connect-distributed.properties 파일이다.
```properties
bootstrap.servers=kafka:9092
 
# 다수의 커넥트 프로세스들을 묶을 그룹 이름
# 같은 그룹으로 지정된 커넥트들에서 커넥터가 실행되면 커넥트들에 분산되어 실행됨
# 커넥트 중 한 대에서 이슈가 발생해도 다른 커넥트에서 커넥터가 안전하게 실행됨
group.id=connect-cluster
 
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=true
value.converter.schemas.enable=true
 
# 분산 모드 커넥트는 오프셋 정보를 내부 토픽(internal topic)에 저장
# 내부 토픽: https://docs.confluent.io/platform/current/streams/developer-guide/manage-topics.html#internal-topics
offset.storage.topic=connect-offsets
offset.storage.replication.factor=1
config.storage.topic=connect-configs
config.storage.replication.factor=1
status.storage.topic=connect-status
status.storage.replication.factor=1
 
offset.flush.interval.ms=10000
 
plugin.path=/usr/local/share/java,/usr/local/share/kafka/plugins
```

분산 모드 커넥트는 분산 커넥트 설정 파일만 있으면 된다.

분산 모드 커넥트를 실행하는 명령어는 다음과 같다.
```shell
$ ./bin/connect-distributed.sh ./config/connect-distributed.properties
```

분산 모드 커넥트가 실행되고 난 이후에는 RESTAPI를 통해 커넥터를 추가, 삭제, 실행, 중지할 수 있다.

```shell
// 사용가능한 플러그인 리스트 조회
$ curl -X GET localhost:8083/connector-plugins
[{"class":"io.debezium.connector.db2.Db2Connector","type":"source","version":"1.8.1.Final"},{"class":"io.debezium.connector.mongodb.MongoDbConnector","type":"source","version":"1.8.1.Final"},...{"class":"org.apache.kafka.connect.mirror.MirrorSourceConnector","type":"source","version":"1"}]

// 파일 소스 커넥터 생성
$ curl -X POST -H "Content-Type: application/json" --data '{"name": "local-file-source", "config": {"connector.class":"FileStreamSourceConnector", "tasks.max":"1", "file":"/tmp/test.txt", "topic":"connect-test" }}' http://localhost:8083/connectors
 
{"name":"local-file-source","config":{"connector.class":"FileStreamSourceConnector","tasks.max":"1","file":"/tmp/test.txt","topic":"connect-test","name":"local-file-source"},"tasks":[],"type":"source"}

// 파일 소스 커넥터 실행 확인
$ curl -X GET localhost:8083/connectors/local-file-source/status
{"name":"local-file-source","connector":{"state":"RUNNING","worker_id":"172.22.0.4:8083"},"tasks":[{"id":0,"state":"RUNNING","worker_id":"172.22.0.4:8083"}],"type":"source"}

// 실행 중인 커넥터 (active connector) 리스트 확인
$ curl -X GET localhost:8083/connectors
["local-file-source","course-outbox-connector","user-outbox-connector","local-file-sink"]

//실행 중인 커넥터 (active connector) 리스트 삭제
$ curl -X DELETE localhost:8083/connectors/local-file-source
$ curl -X GET localhost:8083/connectors
["course-outbox-connector","user-outbox-connector","local-file-sink"]
```

## 3.6.1 소스 커넥터
소스 커넥터는 소스 애플리케이션 또는 소스 파일로부터 데이터를 가져와 토픽으로 넣는 역할을 한다.
소스 커넥터를 만들 때는 `connect-api` 라이브러리를 추가해야 한다.
소스 커넥터를 만들 때 필요한 클래스는 2개다. 첫번째는 `SourceConnector` 인터페이스를 구현한 클래스, 두번째는 `SourceRecord` 클래스를 반환하는 `SourceTask` 인터페이스를 구현한 클래스다.

## 3.6.2 싱크 커넥터
싱크 커넥터는 토픽의 데이터를 타깃 애플리케이션 또는 타깃 파일로 저장하는 역할을 한다.
싱크 커넥터를 만들 때는 `connect-api` 라이브러리를 추가해야 한다.
싱크 커넥터를 만들 때 필요한 클래스는 2개다. 첫번째는 `SinkConnector` 인터페이스를 구현한 클래스, 두번째는 `SinkRecord` 클래스를 받는 `SinkTask` 인터페이스를 구현한 클래스다.