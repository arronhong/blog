---
title: Introduction to Kafka
date: 2021-05-25 23:08:50
tags: kafka
categories: kafka
---

# Introduction to Kafka

:::info
閱讀Kafka the definitive guide的筆記
* Ch1 Meet Kafka
* Ch2 Installing Kafka
* Ch3 Kafka Producer: Writing Message to Kafka
* CH4 Kafka Consumer: Reading Data from Kafka
* Ch5 Kafka Internals
* Ch6 Relialbe Data Delivery
:::

## What is Kafka

分散式的event streaming系統。從多個端點獲取資料並將資料持久化存在disk，其他端點依照使用情境real time或是offline讀取並處理資料。Kafka可以在多種情境下使用，像是real time的金融交易系統、網站使用者行為追蹤與分析、sensor等IoT裝置資料運用以及microservices與event driven架構

### Terminology
![](https://i.imgur.com/cfgsyS6.png)

#### Message
相當於資料庫系統裡的一筆row或record。另外message可以有叫做key的metadata，key可以用來當作寫進哪個partition的參考

#### Topic
messages透過topic來分類，相當於資料庫系統裡的table或collection

#### Partition
一個topic由一到多個patitions組成，分散儲存在多個kafka brokers上。message以append的方式寫進partition，一個topic內的messages不保證時序，但一個partition內的messages保證。partition的設計與kafka提供的可靠性與可擴展性有很大的關係

#### Broker
存放data的server，讓producer(consumer)寫入(讀取)data。多個brokers組成cluster

#### Producer
將message送進broker的client

#### Consumer
從broker讀出message的client

#### Zookeeper
kafka透過zookeeper管理cluster內的配置，像是cluster內brokers、consumers與topics等的配置

## Producer
### How producer send message
#### Send api
1. 呼叫producer send api，並帶入topic、value以及optional key參數
2. 把value跟key serialize成bytes
3. 根據key決定partition或自定義partitioning的方法
4. 將message壓縮並append到buffer
5. 把message callback加到callback list
6. send api return，後續將message送到broker的工作由internal sender thread處理

![](https://i.imgur.com/KMvbnLN.png)


#### Sender thread
1. sender thread poll batch queues，每個batch queue拿一個batch(batch由多筆messages組成)
2. 根據batch要送到哪個broker分組
3. 將batch送到broker
4. 如果有用pipelining，sender不會等broker回應就繼續送下一個batch
5. 取得response後依序呼叫callback

![](https://i.imgur.com/tlEygcu.png)

#### When batch is ready
* batch size到達設定大小
* 超過收集的時間(linger time)
* 另一個要送到相同broker的batch已經ready
* call flush或close

### Producer configuration
#### acks
acks控制broker必須在message同步到多少replicas才能回response

| acks | throughput | latency | durability   |
| ---- | ---------- | ------- | ------------ |
| 0    | high       | low     | no guarantee |
| 1    | medium     | medium  | leader       |
| all  | low        | high    | ISR          |

* ack 0，基本上網路沒問題，producer送出訊息後不會管broker有沒有回response，基本上不保證data durability，有很高的throughput
* ack 1，等到partition leader將message寫到local
* ack all，等到所有in sync replica都將message寫進去

#### buffer.memory
producer buffer size，設置合理的buffer siez以避免當producer將messages送到buffer的速度比送到broker的速度還快，導致run out of memory。當buffer滿的時候，producer等待max.block.ms時間，否則拋出例外

#### compression.type
messages壓縮的方式，將message壓縮是提升performance的重要設定之一，壓縮可以降低network utilization及storage(kafka會將壓縮後的messages直接存在disk，之後也會直接整包送給consumer)。不同的壓縮方式有不同的壓縮比以及cpu運算時間，根據cpu等級與network bandwidth決定壓縮方式

#### request.timeout.ms
控制producer最多等待broker回應produce request的時間

#### metadata.fetch.timeout.ms
控制producer最多等待broker回應metadata request的時間

#### max.request.size
控制一個request的最大size

#### batch.size
batch可佔用memory最大size。batch size較大會佔用較多的buffer，反之較小會導致較頻繁的發送request

#### linger.ms
控制sender thread必須要等多久時間才能將batch送出。較高的linger time會有較好的throughput，反之有較高的latency

#### retries
當producer收到可重試的錯誤時，最多自動重試的次數

* retriable error : connection error、no leader
* non-retriable error : message too large、serialization error、authorized error

與retry.backoff.ms一起調整適當的值避免producer太快放棄或是浪費太多資源在重試

#### max.in.flight.requests.per.connection
在同一個connection下，可以pipelining多少個request。如果設定retries並且有pipeline，可能會導致messages的順序改變(e.q. 兩個batches pipeline送到broker，第一個batch fail並且重試，第二個成功後第一個batch重試成功)

### Sending strategy
send api是async讓sender thread把message送到broker，根據use case，應用程式通常會有以下方式使用send

#### Synchronous send
呼叫完send之後，application需等到broker回應成功或失敗，並根據回應處理後續邏輯

#### Fire and forget
不在乎message成功或失敗，送完之後直接準備下一個message。要注意的是，如果有設定retries，遇到retriable error，sender thread還是會自動重試，因此message失敗表示遇到non-retriable error或retry exhausted

#### Asynchronous send
持續地送訊息，當broker回應時才去呼叫先前註冊的callback，並在callback處理成功或失敗後續動作(log、metric)

### Partitioner
預設是將message的key hash並根據該topic的partition數量來決定partition。只要partition數量不變，就可以保證相同的key會是相同的partition，而當partition數量改變，相同key的新message有可能會是不同的partition。如果沒有給key，預設會用round robin的方式選擇partition

當某個key的message數量遠大於其他key，可能會導致特定broker的空間與loading過重，這時可以自訂partitioning的策略(有些driver不支援自訂partitioning)，像是把數量很大的kay寫到有較大儲存空間的partition，其他key平均在其他partitions

## Consumer
### Consumer group
一組consumers共同合作處理一些topics，也就是一組consumers分擔多個partitions的loading。透過增減member的數量來拓展處理messages的能力，因此topic擁有數量較多的partitions，在scale上較有彈性

一個partition只會被同一group內的一個member consume，因此member的數量超過partition的數量時，就會有consumer會閒置

![](https://i.imgur.com/AMPMDQV.png)

### Consumer Protocol
#### Startup
consumer起來時先問broker使用的api version，接著透過metadata request詢問cluster information(e.q. broker address、partition數量以及partition leader等)。在broker端會有一個group coordinator(一個broker，不同的group會有不同的broker)，coordinator負責處理consumer heartbeat、指派group leader以及consumer加入離開等事務。consumer接著會尋找coordinator並送JoinGroup reuqest給coordinator要求加入group，第一個加入group的consumer成為group leader，group leader會從coordinator得知group中所有members的資訊(資訊存在zookeeper)，group leader負責分配partitions並將名單送給coordinator，接著members詢問coordinator得知自己的assignment

![](https://i.imgur.com/eFQn39y.png)

#### Consumption
consumer開始拉取message時要先知道要從哪裡開始，可以透過fetch offset request得知該partition上次處理到哪個offset，fetch offset request不是必要的，只要consumer已經知道offset，那只要在fetch request時可以聲明即可，隨著fetch request不停地拉取messages，consumer還必須適時的向broker更新offset以及向coordinator發送heartbeat

![](https://i.imgur.com/kH20fYm.png)


#### Shutdown
consumer發送leave group request以gracefully shutdown

![](https://i.imgur.com/pFagoix.png)

### Poll loop
consumer poll api封裝大部分的動作，包含partition rebalance、送heatbeat及data fetching，這樣的設計讓application只要處理資料就好。consumer必須要在一定的時間內送出heartbeat，否則會被認為not alive，因此處理資料的時間必需短於session timeout的時間

poll api回傳messages，每筆message包含key、value、partition、offset和topic。poll可以設定參數，控制block多久時間來等待consumer buffer裡有資料，時間長短端看application想要多快拿回控制

### Rebalance
某些情況發生時，consumer group內的partitions需要重新分配，好讓每個partition都有被處理以及盡可能地平均分配
* consumer加入group
* consumer離開group
* topic有新增partition
* broker failure，該broker leader partition轉讓

consumer會定時送heartbeat給coordinator，當coordinator一段時間沒收到heartbeat時便認定consumer已經退出group，因此觸發rebalance。在執行rebalance的期間，整個group不會consume message。在coordinator尚未發現consumer已經退出(e.q. consumer crash)的這段時間，會使partition的messages暫停被consume，直到heartbeat session timeout。consumer明確地告知group coordinator退出，可使group coordinator立即觸發rebalance以便降低無法處理partition的gap

#### Rebalance listeners
在rebalance的前後，可以註冊callback來做一些處理，像是在rebalance前，commit你已經處理好的message等

* onPartitionsRevoked
rebalance開始前，consumers都停止comsume後。這裡可以讓application commit先前還沒commit的offset

* onPartitionsAssigned
partition重新分配後，consumer開始consume前

### Consumer configuration
#### fetch.min.bytes
控制broker最小回傳messages的size，如果可取得的messages size不夠，broker會block直到size夠才回傳。可以提高fetch.min.bytes來降低RTT，並且在consumer數量很多的時候可以降低brokers的loading

#### fetch.max.wait.ms
當messages size不足fetch.min.bytes的要求時，broker最多block多久就要回應request

#### max.partition.fetch.bytes
每個partition每次回傳給consumer的最大size，避免run out of consumer memory。需注意設置太大時，可能使data processing的時間過久，而無法在session timeout之前送heartbeat

這個設置是per partition的，因此要跟consumer總共處理partitions數量一起計算是否超過consumer memory。假設有20個partitions、5個consumers且max.partition.fetch.bytes是1MB，則每個consumer就需要大於4MB的可用memory。但實際上會需要更多的memory，因為group裡的consumer可能會離開

#### heartbeat.interval.ms
多久送heartbeats，通常與session.timeout.ms一起調整

#### session.timeout.ms
heartbeats有效時間，較低的timeout時間可以快速地偵測不預期退出的consumer，但也較容易誤判造成rebalance。反之較高的timeout會使恢復partition consume的gap較大，但也較不易誤判造成rebalance

#### auto.offset.reset
當consumer提供的offset不合法(offset已經被刪除或者offset不存在)時，要從哪裡(latest or earliest)開始consume

#### enable.auto.commit
當設為true時，會根據auto.commit.interval.ms時間自動commit offset

### Commits and offsets
當發生rebalance或group restart時，consumer可能會被指派不同的partition，透過追蹤partition offset，讓consumer可以接續partition上次的消費進度

consumer透過produce特殊的__consumer_offsets topic(0.9 and above)，將offset記錄在broker上的方式來追蹤每個group對partitions的處理進度

發生rebalance時，最近commit的offset比正在處理的offset還小時，會造成message重複處理。發生rebalance時，最近commit的offset比正在處理的offset還大時，會造成data lose

![](https://i.imgur.com/cWrIOGA.png)

#### Commit strategy
##### Automatic commit
application不用處理commit，commit會在poll api裡自動處理。當呼叫poll時，會去檢查是否超過上次commit加上auto.commit.interval.ms的時間，是的話就commit上一次poll的最大offset

#### Synchronous commit 
application自己掌控commit時機並等到commit response才做後續處理。commitSync api會等到broker回應response。commit失敗的時候會自動retry。頻繁的commit會降低throughput，反之會增加duplicate message的機會

#### Asynchronous commit 
application自己掌控commit時機，但不需要立刻知道commit結果。commitAsync api不等broker response，而是用callback的方式等到broker回應後才處理commit結果。commitAsync失敗後不會自動retry，因為自動retry可能會造成問題(e.q.假設一開始commit offset 2000時遇到短暫的connection問題，而隨後的offset 3000 commit成功了，這時offset 2000要retry並且成功就可能會把offset commit較小的位置)。如果想要在callback裡處理retry，就要注意commit順序的問題

## Kafka Internal
### Zookeeper
kafka用zookeeper來管理cluster，zookeeper會存放broker狀態、controller、topic及partition等。當broker啟動後會透過在zookeeper上建立ephemeral node的方式將自己的id註冊，當broker與zookeeper斷線時ephemeral node就會被移除，因此有新的broker加入或是移除時，透過監測這些ephemeral node的變化得知broker狀態

zookeeper存放cluster的結構

```
/
|--brokers
|    |--topics/[topic]/partitions/[partitionId]/state
|    |--ids/[brokerId]
|
|--controller_epoch
|--controller
|--consumers
|    |--[groupId]
|         |--ids/[consumerId]
|         |--owners/[topic]/[partitionId]
|
|--config
|    |--topics/[topic_name]
|    |--clients/[topic_name]
|
|--isr_change_notification
|--admin
     |--delete_topics/[topic_to_be_deleted] 
     |--preferred_replica_election
```

### Controller
cluster中的一個broker會被選為controller，controller要負責指派partition leaders，以及監控其他brokers是否failure。當controller得知有broker離開時，要將該broker的partition leadership指派給其他broker(通常是replica list的下一個replica)，然後通知所有brokers，誰是新的leader

第一個在zookeeper建立controller node的broker會成為controller，其他嘗試建立controller node的brokers會收到node exist的例外，接著其他brokers透過watch controller node的得知controller的變化。當controller離開cluster，controller node會被刪掉，其他brokers知到後嘗試去建立controller node成為controller，沒成為controller的brokers，會重新watch controller node

### Replication
partition replication是kafka提供availability以及durability的方式。每個partition只有一個leader叫做leader replica，所有producers與consumers都對leader發送request。其他的replicas叫做follower replicas，followers負責從leader replicate messages，當leader離開cluster，其中之一個follower會被選為leader。followers會對leader發送fetch request，request裡包含想要的offset，所以leader會知道所有followers同步到哪，如果follower超過replica.lag.time.max.ms時間沒有fetch，或是在該時間無法同步到最新的offset，就會被認為out of sync，out of sync的followers無法被選為leader，反之能在時間內同步到最新offset的replica稱為ISR(in sync replica)

#### Preferred leader
preferred leader是一開始topic被建立起來時，該topic所有partitions的leader配置，preferred leader會是下一個被選為leader的首選，因為最一開始的leader配置會是將partition leader最平均分在所有brokers的配置，因此期望preferred leader就是leader的狀況來達到負載均衡。當auto.leader.rebalance.enable=true時，preferred leader不是leader並且preferred leader處在in sync狀態時，會重新觸發leader election讓preferred leader成為leader

#### Partition allocation
建立topic時，kafka會在brokers間分配partitions，並且有以下目標
* 讓partitions在brokers間平均分散
* 讓partition以及每個replica在不同的broker上
* 如果有rack awareness，會盡量將partition與每個replica分散在不同的rack

分配partition到brokers的時候，不是用broker disk可存放空間來考量，而是partitions的數量。選擇好要將partition存在哪個broker之後，便要決定把partition寫到哪個目錄(log.dirs)，kafka會去計算每個目錄裡partitions的數量，並把新的partition存在數量最少的目錄，而不是考慮disk usage

### Request processing
produce與fetch request都必須送到partition leader，否則收到not leader error。kafka client會透過metadata request得知topic、partition、leader等資訊。所有的brokers都會把metadata存在cache，因此client可以問任何一個broker，client問完後也會存在cache

#### Produce request
當leader收到produce request時，會驗證
* user對這個topic有沒有權限
* acks參數是否正確
* 如果acks是all，是否有足夠的ISR可以同步。(當ISR的數量低於min.insync.replicas，broker會拒絕produce request)

如果沒有問題，leader就會寫進local disk(寫進filesystem cache，因此不保證何時寫進disk，kafka是以replica的方式做durability)，寫進disk後，leader會根據acks的設定回應，如果acks=all，等到所有followers都同步之後才回應

#### Fetch request
fetch request要求topics的partitions的offsets之後的messages。leader會檢查offset是否合法並存在，如果沒問題會用zero copy的方式從filesystem cache直接將messages送到network。client還可以控制broker回傳的資料量的上限(max.partition.fetch.bytes)與最小的資料量(fetch.min.bytes)，以及如果資料量未達最小限制broker可以收集多久(fetch.max.wait.ms)

![](https://i.imgur.com/fUbeKD9.png)

consumer只會取得已經同步到所有ISR的messages(high watermark)。leader會知道replicas同步狀況。這也表示replicas同步messages的速度會影響到consumers取的messages的速度(設定合理的replica.lag.time.max.ms)，因此為了提高reliability增加replica factor也會影響到throughput

![](https://i.imgur.com/eTDdfvh.png)

#### Other requests
除了metadata、produce、fetch requests之外，還有其他種類request，像是當partition有新leader時，controller發送LeaderAndIsr request通知所有brokers。request種類會持續增加或改進，在舊版本的protocol中，consumers透過zookeeper追蹤offsets，後來則是改成用特殊的topic來儲存追蹤offsets(OffsetCommitRequest、OffsetFetchRequest、ListOffsetsRequest)

client使用的request版本不能比broker支援的版本還新。較新的broker版本都可以向前相容舊版本的request

### File management
kafka最小儲存單位是partition，partition不能跨brokers，甚至不能跨disks(multiple disks if RAID is configured)。partition會被分成segments，每個segment大小根據size(log.segment.bytes)或是時間(log.segment.ms)決定，當kafka寫檔案時發現到達限制時，就會將segment關閉並開始新的segment。正在寫入的segment叫做active segment，active segment不會被刪除

#### Log retention
設定log.retention.ms控制log保存多久，實際上會去看segment的last modified time(mtime)，正常來說mtime會是segment關閉的時間。另一種用log.retention.bytes限制partition保存最大size。如果兩個同時設定的話，就看哪個先達成條件。另外log.segment.bytes的大小會影響log retention的時間，log.segment.bytes越大會使segment越晚關閉，越晚關閉表示時間計算越晚開始

#### File format
每個file(segment)裡包含messages與其offsets，格式就跟producer送上來的一樣，也跟consumer讀出的時候一樣，透過這樣的規範讓kafka可以使用zero copy也避免對訊息解壓縮後再壓縮來提高效率。每個message包含key、 value、offset、message size、checksum、version、壓縮方式以及timestamp。如果producer將所有messages壓縮一筆message，這筆message會被放在value的欄位，之後broker會將這筆message直接送給consumer，consumer收到後解壓縮就會拿到所有messages

![](https://i.imgur.com/poFSDZV.png)

#### Index
consumer會要求任何合法的offset，而這個offset可能存在任何segments中，因此kafka建立index segment將offset對應到log segment中的位置。segment file是由裡面最小的offset命名。index segment每筆record是由4 bytes的相對base offset以及4 bytes位置組成。當要找特定offset時，透過檔案名稱可以快速知道要去哪個index segment找，然後透過index segment內排序好的內容用binary sort快速找到該offset在log segment中的position

![](https://i.imgur.com/I0BkKOn.png)
![](https://i.imgur.com/uJhCBk5.png)

### Log compaction
log compaction提供另一種retention的方式，不將segment刪掉，而是把segment裡的logs壓起來，替每個key保留最新的value

![](https://i.imgur.com/6TrUs3S.png)

如何compact
1. segment分成clean與dirty兩個部分，clean是指先前已經compact過的部分，dirty則是尚未compact過的
2. kafka在開始時會啟動cleaner thread 
3. cleaner會選擇dirty messages比例最高的partition開始處理
4. cleaner iterate dirty messages的部分建立OffsetMap，這個map將key hash成16 bytes的值對應到8 bytes的offset，key相同就保留最大的offset
5. cleaner建立新的segment並從頭開始掃，如果key符合最新的offset或是key不在map裡就將message寫到新的segment
6. 最後將新的segment替換舊的

## Reliable Data Delivery
在建立可靠的系統時，必須先暸解kafka提供哪些guarantees，以及在哪些錯誤情況下會有怎樣的行為。kafka保證
* partition內messages的順序
* consumer只會讀到committed的message(committed message表示message同步到ISR。producer透過ack=all得知是否committed)

replication是kafka提供reliability的核心，replication factor控制replica的數量，控制可靠性與可用性的等級，相對的增加disk成本以及降低throughput

### Unclean leader election
unclean leader election表示允許讓out of sync replica選為leader，允許unclean leader可能導致data lose或data inconsistent，舉例來說，當ISR followers都crash或是同步太慢而只剩下leader時，因為ISR只有leader，此時寫入的messages都會被認定為committed，因此consumer就能讀到，然後leader crash且其中一個follower被選為leader並開始接受produce request，舊的leader回來發現offset與自己的不一致，會把自己不一致的部分刪掉，最後造成data lose與data inconsistent。可以設定(unclean.leader.election.enable)不允許unclean leader，但當發生ISR最後的leader離開時，partition變成offline直到原本的leader回來，如此保證資料一致並且沒有data lose，partition offline導致unavailability

### Minimum in-sync replicas
committed message表示message已經寫進多少ISR，即使ISR只有leader，這樣就容易造成上述問題。設定min.insync.replica控制必須有多少ISR才能寫入，因此只要ISR小於這個設定，producer就無法寫入訊息(NotEnoughReplicasException)，剩下的ISR變成read only

### Producer
acks=1且不允許unclean leader，當leader收到寫進local並回ack後就crash，follower還是in-sync(要認定out of sync需要時間)所以被選成leader，這筆message就會遺失，但不會inconsistent，因為consumer不會讀到

acks=all且不允許unclean leader，當發生retriable error時，producer沒適當的處理retry(像是沒處理或者還沒retry成功crash後沒保存下來等)也可能發生data lose

設定min.insync.replica結合，producer可以確保message寫進多少replicas

retry可能導致duplicate message，例如messages成功寫入ISR，但因為網路斷線producer沒收到ack所以retry(at least once)。常見的解法是用idempotent write或是用unique identifier

### Consumer
consumer在reliability方面就是處理commit offset的問題，如同先前提到的，在commit offset與process message間的rebalance或crash都可能造成duplicate data或是data lose，因此要做到exactly once必須用其他方法，像是將操作設計成idempotent

commit的頻率必須在效能與message重複的數量之間做取捨，commit的overhead與produce ack=all相似

有時在poll之後，messages沒辦法立刻處理完畢需要retry(像是DB短暫的無法連線)，此時或許可以先commit這次poll的last offset，把無法處理的messages存在buffer後繼續嘗試處理這些messages，但記得還是要呼叫poll來送heartbeat，因此使用pause確保poll不會拉出額外的messages，buffer清空後再使用resume繼續處理loop；另一個可能的方法是，將這些messages produce到另一個topic並且繼續下個loop，這時可以用另一組consumer group處理這些topic，或是同一組group同時處理兩個topic
    
### Validating system reliability
開發系統的時候，我們會有需求像是能花多少錢、用什麼等級的硬體、預期服務多少使用者、能接受的回應時間及能接受的容錯等等，而我們也需要知道外部系統提供什麼樣的保證以及什麼樣的限制，根據這些情境來配置外部系統的configuration，接下來會需要驗證配置是否符合需求。kafka definitive guide建議三個階段來驗證系統1. 驗證配置 2. 驗證應用端 3. 在production上監控

#### Validating configuration
將application的logic抽離，單純驗證client與broker的configuration。將kafka提供的VerifiableProducer設定成預想的producer configuration及produce rate，同樣地設定VerifiableConsumer，如此透過VerifiableProducer與VerifiableConsume的print得知結果是否符合需求。另外還建議跑一些狀況(kafka提供的[tests](https://github.com/apache/kafka/tree/trunk/tests))，像是Leader election、Controller election、Rolling restart、Unclean leader election等，當這些情況發生時，系統回復正常的時間是否能接受？是否掉messages？可以接受產生多少duplicate messages

#### Validating Applications
加入application logic跑整合測試，一樣試著跑像上述提到的failure conditions

#### Monitoring Reliability in Production
系統上線後，會需要監控client是否正常。監控producer兩個指標1. error rate以及retry rate，另外還需要監看log message是否有狀況(e.q. log是否很長報retry用完)；監控consumer lag指標，consumer lag表示consumer目前的offset離最新message有多遠，如果consmuer持續落後或是越來越遠就需要調整([linkedin提供的consumer lag check tool](https://github.com/linkedin/Burrow))

為了監測message從produce到consume是否符合需求上的即時，會需要紀錄produce messages數量、consume messages數量以及message從produce到被consume花了多少時間(version 0.10.0 message format就有timestamp，沒有的話建議produce時加上，並且建議加上一些metadata像是哪裡produce等方邊追蹤除錯)

### Choosing the number of partitions for a topic

partition是kafka平行處理訊息的單位，partition數量影響系統throughdput，我們透過可以期望的throughput來粗淺地計算partition需要的數量

1. topic期望的throughput (TT)
2. 一個producer可以承受的throughput (TP)
3. 一個consumer可以承受的throughput (TC)
4. 需要多少producer (NP = TT/TP) 
5. 需要多少consumer (NC = TT/TC)
6. partitions = max(NP, NC)

假設期望topic可以有1GB/sec的read，而一個consumer的能力是50MB/sec，因此最少需要20個consumers，就會是20個partitions；而假設希望topic可以有1GB/sec的write，而一個producer的能力是100MB/sec，因此最少需要10個producers及10個partitions。因此當有20個partitions時就能符合期望的throughput。

partition的數量雖然可以後來再增加，但partition數量改變後，相同的key就可能不會再被寫入原先的partition，因此這個key的order就無法保證。一個常見的實現是，一開始根據未來期望的throughput來建立partitions，以現在的throughput來建立broker，等到throughput上升擴展broker後，將部分partitions轉移到新的broker。

較多的partition，也表示kafka會開啟較多的file descriptor，確保作業系統fd limit的設定。

## Reference

* [Kafka the definitive guide](https://www.confluent.io/resources/kafka-the-definitive-guide/)
* [Tuning Kafka for low latency guaranteed messaging -- Jiangjie (Becket) Qin (LinkedIn), 6/15/16](https://www.youtube.com/watch?v=oQe7PpDDdzA)
* [How Kafka’s Storage Internals Work](https://thehoard.blog/how-kafkas-storage-internals-work-3a29b02e026)
* [Cloudera - Apache Kafka Guide](https://docs.cloudera.com/documentation/enterprise/latest/topics/kafka.html)
* [How to choose the number of topics/partitions in a Kafka cluster?](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster/)
