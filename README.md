# BÀI TẬP SESSION 17 (MD5) - BÀI TẬP 5 (HW5)
## Triển khai Filebeat thay vì Logstash TCP: Tối ưu hóa tài nguyên & Cách ly Ứng dụng (Sidecar Pattern)

---

## 1. Mục tiêu kiến thức & Tình huống nghiệp vụ

- **Mục tiêu kiến thức**:
  - Nắm vững mô hình kiến trúc **Log Shipper** (sử dụng **Filebeat** viết bằng Go) thay thế cho việc ứng dụng bắn log trực tiếp qua mạng Socket TCP.
  - Hiểu nguyên lý tách biệt hoàn toàn ứng dụng nghiệp vụ khỏi hạ tầng logging (**Decoupling Architecture**): Ứng dụng chỉ ghi log ra file cục bộ trên đĩa, Filebeat đóng vai trò Sidecar âm thầm đọc file và vận chuyển sang Elasticsearch.
  - Biết cách cấu hình `filebeat.yml` (bóc tách JSON log, cấu hình template và index động) và kết nối với cụm Docker Compose.

- **Tình huống nghiệp vụ**:
  - Sau bài học từ sự cố sập luồng chính do tắc nghẽn TCP ở HW4, Dev team StoreX quyết định thay đổi hoàn toàn kiến trúc:
    1. **Spring Boot**: Không phụ thuộc vào bất kỳ kết nối mạng socket nào khi ghi log. Ứng dụng chỉ ghi log ra file trên đĩa (`storex.log`) bằng cơ chế `RollingFileAppender` truyền thống của Logback.
    2. **Filebeat**: Triển khai agent siêu nhẹ Filebeat cùng mount chung thư mục log (volume). Filebeat tự động đọc dòng log mới (tailing/harvesting) và đẩy trực tiếp vào Elasticsearch một cách an toàn.

---

## 2. Phân tích so sánh Kiến trúc: Logstash TCP vs. Filebeat Log Shipper

```
                       [ MÔ HÌNH SO SÁNH 2 KIẾN TRÚC ]

   1. KIẾN TRÚC CŨ: LOGSTASH TCP (Rủi ro cao)
   ┌─────────────┐
   │ Spring Boot │──(Ghi trực tiếp qua mạng TCP Socket:5000)──► [ LOGSTASH ] ──► [ ELASTICSEARCH ]
   └─────────────┘  * Mạng chậm / Logstash nghẽn => TREO ỨNG DỤNG!
  
   2. KIẾN TRÚC MỚI: FILEBEAT LOG SHIPPER (An toàn tuyệt đối)
   ┌─────────────┐
   │ Spring Boot │──(Ghi cục bộ xuống đĩa buffered I/O)──┐
   └─────────────┘                                        │
                                                          ▼
                                                  ┌──────────────┐
                                                  │  storex.log  │ (Thư mục dùng chung Volume)
                                                  └──────────────┘
                                                          ▲
                                                          │ (Tailing file bất đồng bộ)
                                                  ┌───────┴──────┐
                                                  │   FILEBEAT   │ (Agent Go cực nhẹ ~30MB)
                                                  └───────┬──────┘
                                                          │ (Push HTTP có cơ chế Backpressure)
                                                          ▼
                                                 ┌─────────────────┐
                                                 │  ELASTICSEARCH  │
                                                 └─────────────────┘
```

### Bảng so sánh kỹ thuật:

| Tiêu chí | Logstash TCP (Kiến trúc cũ) | Filebeat Log Shipper (Kiến trúc chuẩn) |
| :--- | :--- | :--- |
| **Ngôn ngữ phát triển** | Java / JRuby (Chạy trên máy ảo JVM) | Go (Biên dịch trực tiếp ra mã máy native binary) |
| **Tài nguyên tiêu thụ** | Nặng: Chiếm từ **500MB đến 1GB+ RAM**, tốn nhiều CPU | Siêu nhẹ: Chỉ tiêu thụ khoảng **20MB - 50MB RAM**, CPU gần như bằng 0 |
| **Tác động đến Ứng dụng** | **Rất nguy hiểm**: Có thể làm block Worker Thread khi mạng chập chờn hoặc buffer socket đầy | **An toàn tuyệt đối**: Ứng dụng chỉ ghi file đĩa cục bộ, hoàn toàn không biết tới sự tồn tại của EFK |
| **Cơ chế chống mất dữ liệu** | Log trong RAM socket dễ mất khi server crash hoặc restart | **Bền vững (Registry Offset)**: Filebeat ghi nhớ vị trí con trỏ đọc trên đĩa. Nếu Elasticsearch downtime, Filebeat dừng lại và tiếp tục gửi tiếp khi ES online |
| **Khả năng điều tiết (Backpressure)** | Kém hơn, dễ gây nghẽn tầng socket ứng dụng | Rất mạnh: Tự động điều chỉnh tốc độ đọc file theo đúng khả năng tiếp nhận của Elasticsearch |

---

## 3. Cấu hình chi tiết

### 3.1. File cấu hình Filebeat (`filebeat.yml`)

Cấu hình đọc file `storex.log` trong volume chia sẻ, tự động bóc tách các trường JSON lên root level và gửi trực tiếp sang Elasticsearch:

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/storex/storex.log
      - /var/log/storex/*.log
    # Tự động parse chuỗi JSON thành các trường có cấu trúc trên Elasticsearch
    json.keys_under_root: true
    json.overwrite_keys: true
    json.add_error_key: true

setup.template.settings:
  index.number_of_shards: 1

output.elasticsearch:
  hosts: ["http://elasticsearch:9200"]
  index: "storex-filebeat-%{+yyyy.MM.dd}"

setup.template.name: "storex-filebeat"
setup.template.pattern: "storex-filebeat-*"
setup.ilm.enabled: false
```

### 3.2. Cấu hình Docker Compose (`docker-compose.yml`)

Khai báo service `filebeat` chạy cùng mạng `elk-network` và mount chung thư mục log `./logs`:

```yaml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "JDK_JAVA_OPTIONS=-XX:-UseContainerSupport"
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - elk-network

  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.0
    container_name: logstash
    ports:
      - "5000:5000"
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro
    environment:
      - "LS_JAVA_OPTS=-Xms512m -Xmx512m"
      - "JDK_JAVA_OPTIONS=-XX:-UseContainerSupport"
    depends_on:
      - elasticsearch
    networks:
      - elk-network

  filebeat:
    image: docker.elastic.co/beats/filebeat:7.17.0
    container_name: filebeat
    user: root
    command: ["filebeat", "-e", "--strict.perms=false"]
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - ./logs:/var/log/storex:ro
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    networks:
      - elk-network

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    networks:
      - elk-network

networks:
  elk-network:
    driver: bridge
```

---

## 4. Hướng dẫn khởi chạy & Kiểm thử thực tế

### 4.1. Khởi chạy toàn bộ hệ thống
```bash
docker compose up -d
```

### 4.2. Ghi log vào file `logs/storex.log`
Mô phỏng ứng dụng Spring Boot ghi dữ liệu log có cấu trúc xuống file:
```bash
echo '{"app":"StoreX","level":"INFO","timestamp":"2026-09-28T10:40:00Z","logger":"com.storex.order.OrderService","orderId":"ORD-5544","amount":320.50,"message":"Order #5544 created successfully via Filebeat"}' >> ./logs/storex.log
```

### 4.3. Kiểm tra log của container Filebeat
```bash
docker logs filebeat --tail 25
```
Filebeat lập tức phát hiện file thay đổi, mở harvester và gửi batch dữ liệu sang Elasticsearch với trạng thái phản hồi `200 OK`.

### 4.4. Kiểm tra Index và Document trên Elasticsearch
1. **Kiểm tra danh sách index**:
   ```bash
   curl http://localhost:9200/_cat/indices?v
   ```
   *Kết quả*: Xuất hiện index `storex-filebeat-2026.09.28`.

2. **Truy vấn nội dung document**:
   ```bash
   curl http://localhost:9200/storex-filebeat-*/_search?pretty
   ```
   *Kết quả*: Các trường JSON như `app: StoreX`, `orderId: ORD-5544`, `amount: 320.50`, `message` được lưu trữ đầy đủ dưới dạng structured document.

### 4.5. Trực quan hóa trên giao diện Kibana
1. Truy cập `http://localhost:5601`.
2. Vào **Stack Management** > **Index Patterns** > Tạo pattern `storex-filebeat-*`.
3. Vào tab **Discover** để theo dõi và tìm kiếm log real-time mà ứng dụng StoreX không hề hay biết tới sự tồn tại của EFK.

---

## 5. Kết luận

- Việc chuyển đổi từ Logstash TCP sang **Filebeat Sidecar** là giải pháp chuẩn công nghiệp giúp:
  1. Bảo vệ ứng dụng chính khỏi mọi nguy cơ tắc nghẽn mạng hoặc sập dây chuyền.
  2. Tiết kiệm tài nguyên máy chủ nhờ sự gọn nhẹ của Filebeat.
  3. Đảm bảo tính toàn vẹn dữ liệu log nhờ cơ chế registry offset và backpressure.
