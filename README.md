# kafka_homework

## 整體架構基本理解(我怎麼理解Kafka)

### Kafka是什麼?
我把Kafka想成一個「很可靠的訊息中繼站/公告欄」:
- Producer(送訊息的人)把訊息丟進 Kafka
- Kafka會把訊息保存起來(不是丟完就不見)
- Consumer(取訊息的人)再從 Kafka 把訊息讀走
這樣Producer跟Consumer就不用直接互相連線，也不會因為其中一邊暫時掛掉就整個卡住。

---

### Kafka裡面有哪些主要角色
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

---

### 我這份作業的部署架構
我把部署分成兩個環境，避免互相覆蓋:

**(A) 單機環境(single-broker)**
- Docker Compose跑1台Kafka broker
- 用`kafka_check.sh`做端到端驗證(produce/consume)
- 目的:先確認Kafka核心功能正常(能送、能存、能讀)

**(B) HA環境(ha-3brokers)**
- Docker Compose跑3台Kafka broker(同一個cluster)
- 建立`replication-factor=3`的topic
- 故障驗證:停掉其中一台broker後仍可produce/consume，並觀察ISR變化
- 目的:證明我理解並實作至少一項HA機制(多broker+replication)

---

## 思考過程、取捨與可重現性(我怎麼做、為什麼這樣做)

### 目標
這份作業我把重點放在:
1. Kafka能成功啟動並持續運作
2. 使用Docker+Docker Compose部署
3. 照README的步驟可以完整重現環境與驗證結果

---

### 我如何確保「Kafka能持續運作」
啟動 Kafka:
```bash
docker compose up -d
docker compose ps
```

我用```docker compose ps```確認container狀態是Up

並用```docker logs $CONTAINER_NAME(這邊要換成正確的container name)```確認沒有一直重啟或fatal error

### 如何重現
我把所有需要的檔案都放在repo:
- ```docker-compose.yml```:描述Kafka的部署方式
- ```kafka_check.sh```:一鍵驗證Kafka是否正常(produce/consume端到端)
- ```README.md```:提供從零開始的操作步驟

重現步驟:

**1) 進到專案資料夾**
```bash
cd ~/single-broker
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

