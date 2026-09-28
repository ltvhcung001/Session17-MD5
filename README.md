# BÀI TẬP SESSION 17 (MD5) - BÀI TẬP 2 (HW2)
## Cấu hình Logstash nhận luồng log JSON từ Spring Boot qua giao thức TCP

---

## 1. Mục tiêu kiến thức & Tình huống nghiệp vụ

- **Mục tiêu kiến thức**:
  - Biết cách cấu hình file `logstash.conf` với đầy đủ 3 khối: `input`, `filter` (tùy chọn), và `output`.
  - Hiểu cơ chế hoạt động của các Codec trong Logstash, đặc biệt là `codec => json_lines` khi tiếp nhận luồng dữ liệu truyền qua TCP Socket từ ứng dụng (như Spring Boot với thư viện `logstash-logback-encoder`).
  - Đảm bảo dữ liệu log gửi sang Elasticsearch là **dữ liệu có cấu trúc (Structured Data)** thay vì một chuỗi text thô (Raw String).

- **Tình huống nghiệp vụ**:
  - Hệ thống Backend **StoreX** (xây dựng bằng Spring Boot) cấu hình Logback (`logstash-logback-encoder`) để bắn log trực tiếp qua TCP Socket tới Logstash ở cổng `5000` dưới định dạng JSON.
  - Tuy nhiên, file `logstash.conf` ban đầu cấu hình input `tcp` mà không chỉ định codec (hoặc dùng codec `plain` mặc định), dẫn đến việc Logstash không thể parse được các trường JSON hoặc lưu toàn bộ log thành một chuỗi chuỗi string thô trong trường `message`, không phục vụ được nhu cầu tìm kiếm, lọc theo trường (`field filter`) và vẽ biểu đồ phân tích trên Kibana.

---

## 2. Phân tích nguyên nhân gốc rễ & Giải pháp

### 2.1. Phân tích nguyên nhân lỗi (Root Cause Analysis)

```
                            [ LUỒNG LOG TỪ STOREX QUA LOGSTASH VÀO ELASTICSEARCH ]

   Spring Boot (StoreX)
   Logback TCP Appender
            │
            ▼ (JSON Stream qua TCP:5000)
    ┌───────────────┐
    │   LOGSTASH    │
    │  input { tcp }│
    └───────┬───────┘
            │
            ├─ [X] codec => plain (mặc định)
            │      -> Lưu nguyên cục: { message: "{\"app\":\"StoreX\",\"level\":\"INFO\"...}" }
            │      -> Không thể filter, query theo từng field riêng biệt!
            │
            └─ [v] codec => json_lines
                   -> Tự động tách từng dòng và parse JSON thành fields:
                      { "app": "StoreX", "level": "INFO", "orderId": 1001, ... }
                      
            ▼ (Gửi sang Elasticsearch)
    ┌───────────────┐
    │ ELASTICSEARCH │ (Lập chỉ mục các trường có kiểu dữ liệu rõ ràng: string, number, date)
    └───────────────┘
```

1. **Hành vi mặc định của Logstash TCP Input**:
   - Khi cấu hình `input { tcp { port => 5000 } }` mà không khai báo codec, Logstash sử dụng codec mặc định là `line` hoặc `plain`.
   - Mỗi chuỗi byte gửi qua socket chỉ được đọc vào dưới dạng text thô và gán vào trường `message`. Toàn bộ cấu trúc key-value của log JSON bị mất, biến thành string thuần túy.

2. **Tại sao phải là `json_lines` thay vì `json`?**
   - Ứng dụng Backend gửi log qua kết nối TCP dạng liên tục (Streaming Socket / Keep-Alive connection), trong đó mỗi bản ghi log kết thúc bằng ký tự xuống dòng `\n` (NDJSON - Newline Delimited JSON).
   - Codec `json` thông thường chờ nhận hết luồng dữ liệu (hoặc đóng socket) mới parse, dẫn đến treo hoặc parse lỗi khi kết nối TCP duy trì lâu dài.
   - Codec `json_lines` được thiết kế chuyên biệt cho TCP Stream: tự động tách từng dòng theo `\n` và parse ngay lập tức từng bản ghi JSON thành các document có cấu trúc.

---

## 3. Cấu hình chi tiết

### 3.1. File cấu hình Logstash (`logstash.conf`)

Đã sửa đổi input bằng việc thêm `codec => json_lines`:

```ruby
input {
  tcp {
    port => 5000
    # Sửa lỗi: Bổ sung codec json_lines để Logstash tự động phân tách theo từng dòng và parse JSON
    codec => json_lines
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "storex-logs-%{+YYYY.MM.dd}"
  }
  stdout {
    codec => rubydebug
  }
}
```

### 3.2. Cấu hình Docker Compose (`docker-compose.yml`)

Bổ sung service `logstash` đồng bộ với cụm Elasticsearch và Kibana:

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

### 4.1. Khởi động toàn bộ cụm dịch vụ
```bash
docker compose up -d
```

### 4.2. Gửi log JSON mẫu qua cổng TCP 5000
Có thể sử dụng PowerShell để gửi gói tin log có cấu trúc vào cổng 5000:

```powershell
$client = New-Object System.Net.Sockets.TcpClient("localhost", 5000)
$stream = $client.GetStream()
$writer = New-Object System.IO.StreamWriter($stream)
$writer.WriteLine('{"app":"StoreX","level":"INFO","logger":"com.storex.service.OrderService","orderId":"ORD-9988","amount":150.75,"message":"Payment processed successfully"}')
$writer.Flush()
$client.Close()
```

### 4.3. Kiểm tra kết quả trong Elasticsearch
Truy vấn các bản ghi log trong index `storex-logs-*`:
```bash
curl http://localhost:9200/storex-logs-*/_search?pretty
```

Kết quả phản hồi xác nhận Logstash đã parse thành công từng trường JSON riêng biệt:
- Trường `app`: `"StoreX"`
- Trường `level`: `"INFO"`
- Trường `orderId`: `"ORD-9988"`
- Trường `amount`: `150.75` (dạng number)
- Trường `message`: `"Payment processed successfully"`

Thay vì lưu toàn bộ chuỗi JSON vào một field `message` thô, dữ liệu đã được bóc tách có cấu trúc hoàn chỉnh.

---

## 5. Kết luận

- Việc sử dụng `codec => json_lines` trên `tcp` input là giải pháp tối ưu và bắt buộc khi nhận log dạng streaming JSON từ các ứng dụng Java / Spring Boot.
- Logstash hoàn thành nhiệm vụ phân giải (parsing) trực tiếp tại tầng input, giúp Elasticsearch lập chỉ mục chính xác từng trường dữ liệu, tối ưu hóa hiệu năng truy vấn và phân tích trên Kibana Dashboard.
