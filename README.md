# kafka_nifi

## Kafka Consumer로 Metric Data DB insert

- 시나리오 : telegraf로 vcenter에서 발생되는 로그를 주기적으로 가져와서 Kafka Producer로 전달

  (telegraf + Kafka Producer  => NiFi Kafka Consumer + Maridb insert) 

  1. NiFi에서 Kafka Consumer process를 통해 로그 데이터 전달 받음
  2. flowfile를 1차 가공하여 각 상태(cpu, memory, net, power등등)중 원하는 상태를 Attribute로 생성
  3. attribute를 저장할 수 있는 형태로 가공 하여 DB insert .

![전체프로세스](https://github.com/watso77/kafka_nifi/blob/master/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202020-08-12%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%2010.51.30.png)


### Kafka 설치

#### Download 

- apache quick start 참고 (https://kafka.apache.org/quickstart)

- ``` ba
  > tar -xzf kafka_2.12-2.5.0.tgz
  > cd kafka_2.12-2.5.0
  ```



### 설정 

- zookeeper 설정 : **config/zookeeper.properties**

  ``` bash
  dataDir=/home/centos/apps/zookeeper/data          <<zookeeper dir
  
  # the port at which the clients will connect
  clientPort=2181
  # disable the per-ip limit on the number of connections since this is a non-production config
  maxClientCnxns=0
  
  server.1=server01:15000:16000
  server.2=server02:15000:16000
  server.3=server03:15000:16000
  ```

- kafka 설정 : **config/server.properties**

  ``` bash
  1) Server Basics
  ############################# Server Basics #############################
  
  # The id of the broker. This must be set to a unique integer for each broker.
  broker.id=1                << 서버마다 다른 아이디를 줘야 함
  
  2) Zookeeper
  ############################# Zookeeper #############################
  
  # Zookeeper connection string (see zookeeper docs for details).
  # This is a comma separated host:port pairs, each corresponding to a zk
  # server. e.g. "127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002".
  # You can also append an optional chroot string to the urls to specify the
  # root directory for all kafka znodes.
  zookeeper.connect=server01:2181,server02:2181,server03:2181
  
  
  # Timeout in ms for connecting to zookeeper
  zookeeper.connection.timeout.ms=6000
  ```



### 서버 실행

- zookeeper  실행 : 

  ``` bash
  bin/zookeeper-server-start.sh config/zookeeper.properties
  ```

  

- kafka 실행 :

  ```
  bin/kafka-server-start.sh config/server.properties
  ```



### 예제 실행

#### 토픽 생성

``` bash
# telegraf topic 생성
$ bin/kafka-topics.sh --create --bootstrap-server MariaDB_primary:9092 --replication-factor 1 --partitions 1 --topic telegraf
```

#### Producer 실행

``` bash
bin/kafka-console-producer.sh --bootstrap-server MariaDB_primary:9092 --topic telegraf
```

#### Consumer 실행

``` bash
bin/kafka-console-consumer.sh --bootstrap-server MariaDB_primary:9092 --topic telegraf --from-beginning
```

### 확인

- 토픽확인 

  ``` bash
  $  bin/kafka-topics.sh --list --zookeeper MariaDB_primary
  ```

- Kafka process 확인

  ``` bash
  $ jps 
  ```

- 토픽삭제

  ``` bash
  bin/kafka-topics.sh --delete --zookeeper MariaDB_primary:9092 --topic test_vcenter_metrics 
  or 
  bin/kafka-topics.sh --zookeeper MariaDB_primary:2181 --alter --topic test_vcenter_metrics --config retention.ms=1000
  
  --force : 
  ```


### Telegraf로 vsphere metric 정보 수집

-  설치

  - 요구사항 : go language 설치

``` bash
# CentOS
sudo yum -y install telegraf

# 서버 실행
./telegraf --config telegraf.conf

```

- 설정 

  1. output을 Kafka 로 설정
     - broker 3대 vcenter-metrics topic에 전달. 
     - data_format은 json으로 한다.

  ``` bash
  # Configuration for the Kafka server to send metrics to
   [[outputs.kafka]]
     ## URLs of kafka brokers
     brokers = ["MariaDB_primary:9092", "MariaDB_standby01:9092",  "MariaDB_standby02:9092" ]
     ## Kafka topic for producer messages
     topic = "vcenter-metrics"
     
     data_format = "json"
  ```

  2. input 설정

  ``` bash
  [[inputs.vsphere]]
     ## List of vCenter URLs to be monitored. These three lines must be uncommented
     ## and edited for the plugin to work.
     vcenters = [ "https://10.50.251.56/sdk","https://10.50.251.57/sdk" ]
     username = "administrator@vsphere.local"
     password = "xxxxxxx"
     
     insecure_skip_verify = true  <== 보안검사 우회를 위해 true로 설정.
  ```

  

### NiFi 설치

### Process 설계

- Kafka Consumer

  - 편집속성

    Kafka Brokers : MariaDB_primary:9092,MariaDB_standby01:9092,MariaDB_standby02:9092

    Topic Name : vcenter-metrics

    Group ID : test_nifi

    ![스크린샷 2020-08-12 오전 11.06.04](https://github.com/watso77/kafka_nifi/blob/master/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202020-08-12%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%2011.06.04.png)

    

- Split Record로 flowfile 나누기

  ![스크린샷 2020-08-12 오전 11.08.41](https://github.com/watso77/kafka_nifi/blob/master/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202020-08-12%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%2011.08.41.png)

- Attribute 생성 (EvaluateJsonPath)

  - metric으로 받을 모든 데이터를 Attribute로 저장 한다.

    ![스크린샷 2020-08-12 오전 11.09.57](https://github.com/watso77/kafka_nifi/blob/master/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202020-08-12%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%2011.09.57.png)

- AttributeToJson

  - 위에서 저장한 attribute를 Json형태로 변환 한다.

    ![스크린샷 2020-08-12 오전 11.12.28](https://github.com/watso77/kafka_nifi/blob/master/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202020-08-12%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%2011.12.28.png)

- RouteOnAttribte로 저장 데이터 분리

  ![스크린샷 2020-08-12 오전 11.11.25](https://github.com/watso77/kafka_nifi/blob/master/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202020-08-12%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%2011.11.25.png)

- DB Insert (PutDatabaseRecord)

  - RecordReader를 JsonTreeReader로 설정
  - DBCPConnectionPool로 DB Connection정보 설정
  - Statement Type : INSERT
  - Table Name  : Attribute 속성 중 name속성을 테이블이름으로 사용한다.

![스크린샷 2020-08-12 오전 11.13.18](https://github.com/watso77/kafka_nifi/blob/master/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202020-08-12%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%2011.13.18.png)

- DBCPConnectionPool 설정

  다른 정보는 DB Connection Pool 설정이나, Database Driver Location(s)에 사용하고자 하는 driver를 잡아줘야 한다.

  ![스크린샷 2020-08-12 오전 11.16.54](https://github.com/watso77/kafka_nifi/blob/master/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202020-08-12%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%2011.16.54.png)



### 참고사항

- table schema ddl

``` sql
CREATE TABLE `vsphere_vm_cpu` (
  `name` varchar(100) DEFAULT NULL,
  `cluster_name` varchar(100) DEFAULT NULL,
  `cpu` varchar(100) DEFAULT NULL,
  `dcname` varchar(100) DEFAULT NULL,
  `esxhostname` varchar(100) DEFAULT NULL,
  `host` varchar(100) DEFAULT NULL,
  `usage_average` varchar(100) DEFAULT NULL,
  `idle_summation` varchar(100) DEFAULT NULL,
  `io_time` varchar(100) DEFAULT NULL,
  `metric_name` varchar(100) DEFAULT NULL,
  `demand_average` varchar(100) DEFAULT NULL,
  `guest` varchar(100) DEFAULT NULL,
  `guesthostname` varchar(100) DEFAULT NULL,
  `latency_average` varchar(100) DEFAULT NULL,
  `moid` varchar(100) DEFAULT NULL,
  `source` varchar(100) DEFAULT NULL,
  `timestamp` varchar(100) DEFAULT NULL,
  `uuid` varchar(100) DEFAULT NULL,
  `vcenter` varchar(100) DEFAULT NULL,
  `vmname` varchar(100) DEFAULT NULL,
  `readiness_average` varchar(100) DEFAULT NULL,
  `ready_summation` varchar(100) DEFAULT NULL,
  `run_summation` varchar(100) DEFAULT NULL,
  `usagemhz_average` varchar(100) DEFAULT NULL,
  `used_summation` varchar(100) DEFAULT NULL,
  `wait_summation` varchar(100) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `vsphere_vm_mem` (
  `name` varchar(100) DEFAULT NULL,
  `clustername` varchar(100) DEFAULT NULL,
  `dcname` varchar(100) DEFAULT NULL,
  `esxhostname` varchar(100) DEFAULT NULL,
  `guest` varchar(100) DEFAULT NULL,
  `guesthostname` varchar(100) DEFAULT NULL,
  `hosts` varchar(100) DEFAULT NULL,
  `moid` varchar(100) DEFAULT NULL,
  `source` varchar(100) DEFAULT NULL,
  `timestamp` varchar(100) DEFAULT NULL,
  `usage_average` varchar(100) DEFAULT NULL,
  `uuid` varchar(100) DEFAULT NULL,
  `vcenter` varchar(100) DEFAULT NULL,
  `vmname` varchar(100) DEFAULT NULL,
  `swapin_average` varchar(100) DEFAULT NULL,
  `swapinrate_average` varchar(100) DEFAULT NULL,
  `swapout_average` varchar(100) DEFAULT NULL,
  `swapoutrate_average` varchar(100) DEFAULT NULL,
  `vmmemctl_average` varchar(100) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `vsphere_vm_net` (
  `name` varchar(100) DEFAULT NULL,
  `bytesrx_average` varchar(100) DEFAULT NULL,
  `bytestx_average` varchar(100) DEFAULT NULL,
  `clustername` varchar(100) DEFAULT NULL,
  `dcname` varchar(100) DEFAULT NULL,
  `esxhostname` varchar(100) DEFAULT NULL,
  `guest` varchar(100) DEFAULT NULL,
  `guesthostname` varchar(100) DEFAULT NULL,
  `hosts` varchar(100) DEFAULT NULL,
  `moid` varchar(100) DEFAULT NULL,
  `source` varchar(100) DEFAULT NULL,
  `timestamp` varchar(100) DEFAULT NULL,
  `usage_average` varchar(100) DEFAULT NULL,
  `uuid` varchar(100) DEFAULT NULL,
  `vcenter` varchar(100) DEFAULT NULL,
  `vmname` varchar(100) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `vsphere_vm_power` (
  `name` varchar(100) DEFAULT NULL,
  `power_average` varchar(100) DEFAULT NULL,
  `clustername` varchar(100) DEFAULT NULL,
  `dcname` varchar(100) DEFAULT NULL,
  `esxhostname` varchar(100) DEFAULT NULL,
  `guest` varchar(100) DEFAULT NULL,
  `guesthostname` varchar(100) DEFAULT NULL,
  `hosts` varchar(100) DEFAULT NULL,
  `moid` varchar(100) DEFAULT NULL,
  `source` varchar(100) DEFAULT NULL,
  `timestamp` varchar(100) DEFAULT NULL,
  `usage_average` varchar(100) DEFAULT NULL,
  `uuid` varchar(100) DEFAULT NULL,
  `vcenter` varchar(100) DEFAULT NULL,
  `vmname` varchar(100) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

```

