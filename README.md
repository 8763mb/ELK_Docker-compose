<<<<<<< HEAD
# ELK_Docker-compose
ELK_Docker compose
=======
# ELK Stack 9.2 + Kafka 部署指南

本指南說明如何在本機以 Docker Compose 建置 **Elasticsearch 9.2.0、Logstash 9.2.0、Kibana 9.2.0**，並加入 **Apache Kafka 4.1.0** 作為日誌中介，同時啟用自簽 TLS 憑證與基本驗證。內容涵蓋備份、憑證產製、組態說明，以及啟動與測試流程。

---

## 架構概覽
- Elasticsearch 暴露 HTTPS (`https://localhost:9200`)，使用自簽 CA 與 `elastic` 帳號。
- Logstash 支援 `tcp`、`beats` 與 Kafka (`elk-input` topic) 多來源，並以 TLS 連線寫入 Elasticsearch。
- Kibana 與 Logstash 共用同一份 CA 以信任 Elasticsearch。
- Kafka 使用官方 `apache/kafka:4.1.0`（KRaft 模式），可由 Filebeat 或其他 Producer 發佈訊息，再由 Logstash 消費。

---

## 1. 備份與目錄
- 舊版組態已備份於 `backup/docker-compose.yml.bak`。
- 主要目錄結構：

```
/home/elk-stack/
├── backup/
│   └── docker-compose.yml.bak
├── certs/
│   ├── ca/
│   │   ├── ca.crt
│   │   └── ca.key
│   └── elasticsearch/
│       ├── elasticsearch.crt
│       ├── elasticsearch.csr
│       ├── elasticsearch.key
│       └── elasticsearch.cnf
├── docker-compose.yml
├── .env
├── logstash/
│   ├── config/
│   │   └── logstash.yml
│   └── pipeline/
│       └── logstash.conf
└── filebeat-8.11.0-linux-x86_64/ （如需改使用 Kafka output 可在此調整）
```

---

## 2. 產製 TLS 憑證
以下指令已在本專案中執行，若需重新建立可參考：

```bash
cd /home/elk-stack
mkdir -p certs/ca certs/elasticsearch

# 建立 Root CA
openssl genrsa -out certs/ca/ca.key 4096
openssl req -x509 -new -nodes -key certs/ca/ca.key \
  -sha256 -days 3650 \
  -subj "/C=TW/ST=Taiwan/L=Taipei/O=ELK Stack/OU=Ops/CN=elk-stack-root" \
  -out certs/ca/ca.crt

# 產生 Elasticsearch 憑證 (含 localhost 與 container 名稱)
cat <<'EOF' > certs/elasticsearch/elasticsearch.cnf
[ req ]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[ req_distinguished_name ]
C = TW
ST = Taiwan
L = Taipei
O = ELK Stack
OU = Ops
CN = elasticsearch

[ v3_req ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = elasticsearch
DNS.2 = localhost
IP.1 = 127.0.0.1
EOF

openssl genrsa -out certs/elasticsearch/elasticsearch.key 4096
openssl req -new -key certs/elasticsearch/elasticsearch.key \
  -out certs/elasticsearch/elasticsearch.csr \
  -config certs/elasticsearch/elasticsearch.cnf
openssl x509 -req -in certs/elasticsearch/elasticsearch.csr \
  -CA certs/ca/ca.crt -CAkey certs/ca/ca.key -CAcreateserial \
  -out certs/elasticsearch/elasticsearch.crt -days 825 -sha256 \
  -extensions v3_req -extfile certs/elasticsearch/elasticsearch.cnf

# 將 Root CA 複製到 Elasticsearch 掛載目錄
cp certs/ca/ca.crt certs/elasticsearch/ca.crt

# 調整權限，讓容器內的 elasticsearch (UID 1000) 可讀取
chown 1000:0 certs/elasticsearch/{elasticsearch.key,elasticsearch.crt,ca.crt}
chmod 640 certs/elasticsearch/{elasticsearch.key,elasticsearch.crt,ca.crt}
```

> 這份 CA (`certs/ca/ca.crt`) 已掛載到 Elasticsearch、Logstash 與 Kibana。若其他客戶端要連線，也需信任此 CA。

---

## 3. 組態重點

### 3.1 `docker-compose.yml`
```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:9.2.0
    container_name: elasticsearch
    environment:
      - node.name=elasticsearch
      - cluster.name=elk-cluster
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD:-changeme}
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=true
      - xpack.security.http.ssl.verification_mode=certificate
      - xpack.security.http.ssl.certificate_authorities=/usr/share/elasticsearch/config/certs/ca.crt
      - xpack.security.http.ssl.certificate=/usr/share/elasticsearch/config/certs/elasticsearch.crt
      - xpack.security.http.ssl.key=/usr/share/elasticsearch/config/certs/elasticsearch.key
      - xpack.security.transport.ssl.enabled=true
      - xpack.security.transport.ssl.verification_mode=certificate
      - xpack.security.transport.ssl.certificate_authorities=/usr/share/elasticsearch/config/certs/ca.crt
      - xpack.security.transport.ssl.certificate=/usr/share/elasticsearch/config/certs/elasticsearch.crt
      - xpack.security.transport.ssl.key=/usr/share/elasticsearch/config/certs/elasticsearch.key
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
      - ./certs/elasticsearch:/usr/share/elasticsearch/config/certs:ro
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:9.2.0
    container_name: logstash
    environment:
      - LS_JAVA_OPTS=-Xmx512m -Xms512m
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD:-changeme}
    volumes:
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
      - ./certs/ca:/usr/share/logstash/config/certs:ro
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    depends_on:
      - elasticsearch
      - kafka
    networks:
      - elk

  kibana:
    image: docker.elastic.co/kibana/kibana:9.2.0
    container_name: kibana
    environment:
      - SERVER_HOST=0.0.0.0
      - SERVER_PUBLICBASEURL=${KIBANA_PUBLIC_URL:-http://localhost:5601}
      - ELASTICSEARCH_HOSTS=https://elasticsearch:9200
      - ELASTICSEARCH_SERVICEACCOUNT_TOKEN=${KIBANA_SERVICE_ACCOUNT_TOKEN}
      - ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=/usr/share/kibana/config/certs/ca.crt
    volumes:
      - ./certs/ca:/usr/share/kibana/config/certs:ro
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    networks:
      - elk

  kafka:
    image: apache/kafka:4.1.0
    container_name: kafka
    hostname: kafka
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      KAFKA_NUM_PARTITIONS: 3
      KAFKA_LOG_DIRS: /var/lib/kafka/data
    volumes:
      - kafka-data:/var/lib/kafka/data
    ports:
      - "9092:9092"
    networks:
      - elk

volumes:
  elasticsearch-data:
    driver: local
  kafka-data:
    driver: local

networks:
  elk:
    driver: bridge
```

> 啟動前請設定 `ELASTIC_PASSWORD` 環境變數（可放在 `.env` 或 shell 內 export）。

### 3.2 `logstash/config/logstash.yml`
```yaml
xpack.monitoring.elasticsearch.hosts: [ "https://elasticsearch:9200" ]
xpack.monitoring.elasticsearch.username: "elastic"
xpack.monitoring.elasticsearch.password: "${ELASTIC_PASSWORD}"
xpack.monitoring.elasticsearch.ssl.certificate_authority: "/usr/share/logstash/config/certs/ca.crt"
```

### 3.3 `logstash/pipeline/logstash.conf`
```ruby
input {
  tcp {
    port => 5000
    type => "nginx_access"
  }
  beats {
    port => 5044
  }
  kafka {
    bootstrap_servers => "kafka:9092"
    topics => ["elk-input"]
    group_id => "logstash"
    auto_offset_reset => "latest"
  }
}

filter {
  if [type] == "nginx_access" {
    grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
  }

  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
    }
    date { match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ] }
  }
}

output {
  elasticsearch {
    hosts => ["https://elasticsearch:9200"]
    index => "logstash-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "${ELASTIC_PASSWORD}"
    cacert => "/usr/share/logstash/config/certs/ca.crt"
    ssl => true
  }
  stdout { codec => rubydebug }
}
```

---

## 4. 啟動服務
1. 準備環境變數  
   - 編輯 `.env`（已隨專案建立），設定強密碼並先留空 Kibana token：
     ```dotenv
     ELASTIC_PASSWORD=<請改成強密碼>
     KIBANA_SERVICE_ACCOUNT_TOKEN=
     KIBANA_PUBLIC_URL=http://<伺服器對外位址>:5601
     ```
     > `KIBANA_PUBLIC_URL` 會用在 Kibana 對外產生的絕對連結，請改成外部電腦實際要使用的 URL，否則瀏覽器可能被導向到 `http://localhost:5601`。
   - 匯入環境變數供後續指令使用：
     ```bash
     set -a; source .env; set +a
     ```
2. 預先拉取映像並啟動核心服務（Kibana 暫不啟動）：
   ```bash
   docker compose pull
   docker compose up -d elasticsearch kafka logstash
   ```
3. 確認 Elasticsearch 正常：
   ```bash
   docker compose logs -f elasticsearch
   curl --tlsv1.2 --cacert certs/ca/ca.crt -u elastic:$ELASTIC_PASSWORD https://localhost:9200
   ```
4. 產生 Kibana 專用 Service Account Token（名稱可自訂，範例為 `kibana-token-1`）：
   ```bash
   docker compose exec elasticsearch \
     bin/elasticsearch-service-tokens create elastic/kibana kibana-token-1
   ```
   指令會輸出 `SERVICE_TOKEN ... = <token>`，複製 `<token>`。
5. 將 token 寫回 `.env` 並重新載入：
   ```bash
   sed -i "s|KIBANA_SERVICE_ACCOUNT_TOKEN=.*|KIBANA_SERVICE_ACCOUNT_TOKEN=<token>|" .env
   set -a; source .env; set +a
   ```
6. 啟動 Kibana（或重新啟動若先前失敗）：
   > 若 `.env` 有更新，請先 `set -a; source .env; set +a` 再重啟服務。
   ```bash
   docker compose up -d kibana
   docker compose up -d logstash   # 確保 Logstash 再次啟動就緒
   docker compose logs -f kibana
   ```
7. 確認全部服務皆運行：
   ```bash
   docker compose ps
   ```

---

## 5. 建立 Kafka Topic
啟動後建立預設 topic（可依需求調整 partitions）：

```bash
docker compose exec kafka kafka-topics.sh \
  --bootstrap-server kafka:9092 \
  --create --topic elk-input \
  --partitions 3 --replication-factor 1
```

可用 `kafka-console-consumer.sh` 驗證訊息：
```bash
docker compose exec kafka kafka-console-consumer.sh \
  --bootstrap-server kafka:9092 --topic elk-input --from-beginning
```

---

## 6. Filebeat 與其他 Producer
- 若使用 Filebeat，建議改成 Kafka output：

  ```yaml
  output.kafka:
    hosts: ["kafka:9092"]
    topic: "elk-input"
    compression: gzip
    ssl.certificate_authorities: ["/path/to/ca.crt"]  # 若 Kafka 啟用 TLS 再加入
  ```

- 若仍需直接送至 Logstash，保留 `output.logstash`。Logstash input 會同時處理 `beats` 與 Kafka。

---

## 7. 常見問題與排除
- **遠端瀏覽器無法開啟 Kibana（但 `telnet <host> 5601` 成功）**
  - 確認 `.env` 的 `KIBANA_PUBLIC_URL` 已改成實際對外網址（非 `localhost`）。
  - 重新載入環境變數並重啟 Kibana：
    ```bash
    set -a; source .env; set +a
    docker compose up -d kibana
    ```
  - 於瀏覽器使用與 `KIBANA_PUBLIC_URL` 相同的網址重新登入。
- **`docker compose ps` 看不到 Logstash**
  - 重新啟動 Logstash 並查看日誌：
    ```bash
    docker compose up -d logstash
    docker compose logs -f logstash
    ```
- **TLS 連線測試失敗**
  - 確認憑證權限（`chown 1000:0`、`chmod 640`）並再次測試：
    ```bash
    curl --tlsv1.2 --cacert certs/ca/ca.crt -u elastic:$ELASTIC_PASSWORD https://localhost:9200
    ```

---

## 8. 驗證與使用
- Kibana 入口：`http://localhost:5601`，以 `elastic / $ELASTIC_PASSWORD` 登入。
- 首次登入需在 Kibana 中信任自簽 CA，可將 `certs/ca/ca.crt` 匯入瀏覽器或 OS 信任庫。
- 在 Kibana 建立索引樣板 `logstash-*` 後即可於 Discover 檢視資料。
- 監控與診斷：
  ```bash
  docker compose exec elasticsearch curl --cacert config/certs/ca.crt -u elastic:$ELASTIC_PASSWORD https://localhost:9200/_cat/health?v
  docker compose exec logstash curl http://localhost:9600/_node/stats/pipeline?pretty
  ```

---

## 9. 常見操作
- **重新產生憑證**：刪除 `certs/*.crt`、`*.key` 等檔案後，按照第 2 節重新執行命令，並重啟容器。
- **還原舊設定**：將 `backup/docker-compose.yml.bak` 複製回專案根目錄即可。
- **升級版本**：修改 `docker-compose.yml` 的映像標籤後重新 `docker compose pull && docker compose up -d`。

---

完成以上步驟後，即可透過 Kafka 將多來源日誌送入 Logstash，再安全地寫入 Elasticsearch 9.2，並在 Kibana 上檢視。若需要進一步整合 Beats、APM 或 Alerting，可在既有 TLS/驗證架構下繼續擴充。***
>>>>>>> master
