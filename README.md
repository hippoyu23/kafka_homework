# kafka_homework

## 整體架構基本理解(我怎麼理解Kafka)

### Kafka是什麼
我把Kafka想成一個「很可靠的訊息中繼站/公告欄」:
- Producer(送訊息的人)把訊息丟進 Kafka
- Kafka會把訊息保存起來(不是丟完就不見)
- Consumer(取訊息的人)再從 Kafka 把訊息讀走
這樣Producer跟Consumer就不用直接互相連線，也不會因為其中一邊暫時掛掉就整個卡住。

### Kafka裡面有哪些角色
**1) Broker(Kafka伺服器)**
- 一台Kafka服務就是一台broker
- 例如我在單機版本跑1台broker，在HA版本跑3台broker
- 多台broker的好處是可以做副本(replication)，提高可用性

**2) Topic(訊息分類)**
- Topic可以理解成「不同類別的公告欄」
- Producer送訊息時要指定topic，Consumer也從特定topic讀訊息

**3) Partition（切分）**
- 一個topic可以切成多個partition
- partition的用意是讓資料可以平行處理、提升吞吐量
- 同一個partition內的訊息會保持順序

**4) Replication(副本)與Leader/Follower**
- 每個partition可以有多個副本(replica)，分散在不同broker上
- 其中一個副本會被選為leader，讀寫都主要走leader
- 其他副本是follower，負責跟leader同步資料
- 如果某台broker掛掉，Kafka可以把leader換到其他副本(failover)

**5) ISR(In-Sync Replicas)**
- ISR是「目前同步狀態正常」的副本集合
- 當某台broker掛掉或同步落後太多，它會被移出ISR
- 我在HA驗證時停掉一台broker後，describe看到ISR變少，代表Kafka有偵測到故障並調整同步集合

### 我這份作業的部署架構
我把部署分成兩個環境，避免互相覆蓋:

**(1) 單機環境(single-broker)**
- Docker Compose跑1台Kafka broker
- 用`kafka_check.sh`做端到端驗證(produce/consume)
- 目的:先確認Kafka核心功能正常(能送、能存、能讀)

**(2) HA環境(ha-3brokers)**
- Docker Compose跑3台Kafka broker(同一個cluster)
- 建立`replication-factor=3`的topic
- 故障驗證:停掉其中一台broker後仍可produce/consume，並觀察ISR變化
- 目的:證明我理解並實作至少一項HA機制(多broker+replication)

### 目標
這份作業我把重點放在:
1. Kafka能成功啟動並持續運作
2. 使用Docker+Docker Compose部署
3. 照README的步驟可以完整重現環境與驗證結果

---

## 環境需求(我使用的版本)
- Windows+WSL(Ubuntu 24.04)
- Docker Desktop(需開啟WSL integration)
- Docker/Docker Compose:
  ```bash
  docker --version
  docker compose version
  ```
- kafka_check.sh腳本需要Kafka CLI(我裝在WSL):
  ```bash
  kafka-topics.sh --version
  kafka-console-producer.sh --version || true
  kafka-console-consumer.sh --version || true
  ```

---

## 我如何確保「Kafka能持續運作」
啟動 Kafka:
```bash
docker compose up -d
docker compose ps
```

我用```docker compose ps```確認container狀態是Up

並用```docker logs $CONTAINER_NAME(這邊要換成正確的container name)```確認沒有一直重啟或fatal error

或是```docker logs -f broker```

---

## 如何重現
我把所有需要的檔案都放在repo:
- ```docker-compose.yml```:描述Kafka的部署方式
- ```kafka_check.sh```:一鍵驗證Kafka是否正常(produce/consume端到端)
- ```README.md```:提供從零開始的操作步驟

重現步驟:

**1) 進到專案資料夾**
```bash
cd single-broker
```
**2) 啟動kafka**
```bash
docker compose up -d
docker compose ps
```
**3) 健康檢查**
```bash
chmod +x kafka_check.sh
./kafka_check.sh localhost:9092 healthcheck-topic
```
假設成功會看到:```✅ SUCCESS: Kafka is working (produce/consume verified)```

**收工與重啟**

關閉:
```bash
docker compose down
```
重啟:
```bash
docker compose up -d
docker compose ps
```

---

# HA 實作(High Availability)

## 我選擇實作的HA措施是什麼?
我選擇實作的HA措施是:

**(1)多broker(3台)**

**(2)topic replication(replication-factor=3)**

並用「停掉其中一台 broker」來做故障驗證。

### 為什麼選這個?
因為我認為這是Kafka最典型、也最容易驗證的HA機制:
- 多broker可以避免單點故障(不會一台掛掉就整個不能用)
- replication-factor=3代表同一份資料會有3份副本分散在不同broker
- 某台broker掛掉時，Kafka可以用剩下的副本繼續提供服務
- 我也可以透過`Isr`(同步副本集合)是否變化來證明HA有運作

---

## HA 架構說明
### (1) 建立3 broker Kafka cluster(KRaft mode)
我在`ha-3brokers/`用Docker Compose部署三台broker(kafka1/kafka2/kafka3)  
其中每台broker同時扮演broker與controller(KRaft)，並透過容器內部網路用service name互相溝通

### (2) 建立RF=3的topic
我建立topic `ha-topic`，設定:
- partitions=1
- replication-factor=3

因此`describe`應該會看到:
- Replicas會有3個broker
- Isr一開始會是3個broker(代表都同步)

---

## HA驗證方式
我用兩個方向驗證:

### A) 功能驗證:故障後仍可produce/consume
- 正常狀態先produce/consume
- 停掉其中一台broker(例如kafka2)
- 再produce/consume一次
- 若仍能成功送出並讀到訊息，代表叢集在部分節點故障下仍可用

### B) 狀態驗證:觀察ISR變化
- 故障前:`Isr`應包含3台(例如`3,1,2`)
- 故障後:被停掉的 broker 應從`Isr`消失(例如變成 `3,1`)
- 這代表Kafka有偵測到故障並調整同步副本集合，但服務仍維持可用

---

## 重現步驟(HA)

**1)避免port衝突**
如果先前有跑單機版本，請先關掉(是為了避免9092被占用):
```bash
cd single-broker
docker compose down
```
**2)啟動HA(3 brokers)**
```bash
cd ha-3brokers
docker compose up -d
docker compose ps
```
預期會看到:
```
[+] up 4/4
 ✔ Network ha-3brokers_default Created                                                                                           0.0s
 ✔ Container kafka1            Created                                                                                           0.2s
 ✔ Container kafka3            Created                                                                                           0.2s
 ✔ Container kafka2            Created                                                                                           0.2s
NAME      IMAGE                COMMAND                  SERVICE   CREATED        STATUS                  PORTS
kafka1    apache/kafka:3.7.1   "/__cacert_entrypoin…"   kafka1    1 second ago   Up Less than a second   0.0.0.0:9092->9092/tcp, [::]:9092->9092/tcp
kafka2    apache/kafka:3.7.1   "/__cacert_entrypoin…"   kafka2    1 second ago   Up Less than a second   0.0.0.0:9094->9092/tcp, [::]:9094->9092/tcp
kafka3    apache/kafka:3.7.1   "/__cacert_entrypoin…"   kafka3    1 second ago   Up Less than a second   0.0.0.0:9096->9092/tcp, [::]:9096->9092/tcp
```
**3)建立RF=3的topic**
```bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic ha-topic --partitions 1 --replication-factor 3
```
會顯示topic已創建
```
Created topic ha-topic.
```
查看describe:
```
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic ha-topic
```
預期會看到:
```
Topic: ha-topic TopicId: cYlTOVQ2S_GbPnABm_jWrQ PartitionCount: 1 ReplicationFactor: 3 Configs:   Topic: ha-topic Partition: 0 Leader: 1 Replicas: 1,2,3 Isr: 1,2,3
```
**4)正常狀態端到端驗證(produce一筆然後consume一筆)**
```bash
MSG1="ha-$(date +%s)-$RANDOM"
echo "$MSG1" | kafka-console-producer.sh --bootstrap-server localhost:9092 --topic ha-topic
```

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic ha-topic --from-beginning --max-messages 1
```
預期會看到:
```
Processed a total of 1 messages
```
**5)故障模擬**
```bash
docker stop kafka2
```
**6)故障後仍可用(produce一筆然後consume兩筆)**
再多送一筆資料
```bash
MSG2="ha-$(date +%s)-$RANDOM"
echo "$MSG2" | kafka-console-producer.sh --bootstrap-server localhost:9092 --topic ha-topic
```
再讀回來(兩筆，避免consumer等不到更多訊息而造成超時)
```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic ha-topic --from-beginning --max-messages 2
```
預期會看到:
```
ha-1770916549-5380
ha-1770916590-5944
Processed a total of 2 messages
```
**7)觀察ISR變化(HA的證據)**
```bash
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic ha-topic
```
預期會看到(可以發現2不見了):
```
Topic: ha-topic TopicId: cYlTOVQ2S_GbPnABm_jWrQ PartitionCount: 1 ReplicationFactor: 3 Configs:   Topic: ha-topic Partition: 0 Leader: 1 Replicas: 1,2,3 Isr: 1,3
```
**8)把kafka2開回來**
```bash
docker start kafka2
```

---

## 遇到的困難與解法

### (1)Port 9092被占用(無法啟動）
現象:出現`Bind for 0.0.0.0:9092 failed: port is already allocated`

原因:另一套 Kafka（單機或 HA）還在跑，占用 9092

解法:先用正確的資料夾把服務關掉
```bash
cd single-broker && docker compose down
```
或(真的卡住時)
```bash
docker ps --format "table {{.Names}}\t{{.Ports}}" | grep 9092 || true
```
### (2)container名稱衝突(broker已存在)
現象:出現`The container name "/broker" is already in use`

原因:之前的broker container還存在(可能狀態是stopped)，但新的一套compose想用同名 container

解法:
```bash
docker rm -f broker
```
### (3)HA初版設定有timeout(broker互連/advertised listeners)
現象:produce/consume會timeout或retry

原因:Docker容器內的localhost只指向自己，broker之間互連要用service name(kafka1/kafka2/kafka3)

解法:把listeners拆成INTERNAL/EXTERNAL，broker間走INTERNAL，主機CLI走EXTERNAL
