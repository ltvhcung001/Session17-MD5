# BÀI TẬP SESSION 17 (MD5) - BÀI TẬP 3 (HW3)
## Phân loại và Định tuyến Log theo Index động trên Elasticsearch bằng Logstash

---

## 1. Mục tiêu kiến thức & Tình huống nghiệp vụ

- **Mục tiêu kiến thức**:
  - Biết cách sử dụng **biến động (dynamic field references / string interpolation)** trong khối `output` của Logstash để tự động tạo và định tuyến log vào các Index khác nhau cho từng Microservice.
  - Nắm vững các quy tắc đặt tên Index trong Elasticsearch (bắt buộc viết thường, không chứa ký tự đặc biệt) và cách xử lý tiền xử lý (pre-processing) dữ liệu với filter `mutate`.
- **Tình huống nghiệp vụ**:
  - Hệ thống Microservices gồm nhiều dịch vụ như `user-service`, `order-service`, `payment-service`... cùng gửi log về Logstash.
  - Cấu hình cũ để cố định `index => "logstash-%{+YYYY.MM.dd}"`, dẫn đến toàn bộ log của tất cả các service bị tống chung vào một index duy nhất.
  - **Hệ quả tiêu cực**:
    - Khó phân quyền bảo mật (RBAC): Đội ngũ phụ trách `order-service` không nên xem các thông tin nhạy cảm của `user-service`.
    - Không thể tối ưu hóa chính sách vòng đời chỉ mục (Index Lifecycle Management - ILM): Mỗi dịch vụ có nhu cầu lưu trữ (retention policy) khác nhau (ví dụ: `order-service` cần lưu 1 năm, `user-service` debug chỉ cần lưu 14 ngày).
    - Giảm hiệu năng truy vấn khi dung lượng dữ liệu phình to.
  - Quản trị viên yêu cầu tách riêng từng Index theo tên ứng dụng dựa trên trường `app_name` do ứng dụng Spring Boot (`logback-spring`) gửi kèm trong chuỗi JSON.

---

## 2. Phân tích giải pháp kỹ thuật

### 2.1. Cơ chế nội suy chuỗi biến động (Dynamic Field References) trong Logstash

```
                          [ ĐỊNH TUYẾN INDEX ĐỘNG THEO APP_NAME ]

   user-service  ──[TCP:5000 {"app_name":"user-service",...}]──► ┌──────────┐
                                                                 │          │ ──► Index: user-service-2026.09.28
   order-service ──[TCP:5000 {"app_name":"order-service",...}]──► │ LOGSTASH │ ──► Index: order-service-2026.09.28
                                                                 │          │ ──► Index: unknown-service-2026.09.28 (fallback)
   other-service ──[TCP:5000 (không có app_name)]───────────────► └──────────┘
```

- Trong Logstash, cú pháp `%{[tên_trường]}` hoặc `%{tên_trường}` cho phép truy xuất giá trị của trường dữ liệu trong sự kiện log.
- Cú pháp `%{+YYYY.MM.dd}` định dạng thời gian dựa trên trường `@timestamp` của sự kiện.
- Bằng cách kết hợp:
  ```ruby
  index => "%{[app_name]}-%{+YYYY.MM.dd}"
  ```
  Logstash sẽ tự động trích xuất giá trị trường `app_name` của từng bản ghi để sinh tên Index tương ứng trên Elasticsearch theo ngày.

### 2.2. Xử lý chuẩn hóa với Filter Mutate
Elasticsearch quy định rất nghiêm ngặt về định dạng tên Index:
1. **Chỉ được dùng chữ thường (lowercase)**: Nếu ứng dụng gửi `app_name: "Order-Service"`, Elasticsearch sẽ từ chối tạo index và quăng lỗi `InvalidIndexNameException`.
2. **Xử lý trường hợp thiếu `app_name`**: Nếu bản ghi log không có `app_name`, cần gán giá trị dự phòng mặc định (fallback) như `unknown-service` để tránh sinh ra tên index lỗi `%{app_name}-...`.

Khối filter hoàn chỉnh:
```ruby
filter {
  if [app_name] {
    mutate {
      lowercase => [ "app_name" ]
    }
  } else {
    mutate {
      add_field => { "app_name" => "unknown-service" }
    }
  }
}
```

---

## 3. Cấu hình hoàn chỉnh

### 3.1. File cấu hình Logstash (`logstash.conf`)

```ruby
input {
  tcp {
    port => 5000
    codec => json_lines
  }
}

filter {
  # Chuẩn hóa app_name về chữ thường để tuân thủ quy định đặt tên Index của Elasticsearch
  if [app_name] {
    mutate {
      lowercase => [ "app_name" ]
    }
  } else {
    mutate {
      add_field => { "app_name" => "unknown-service" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    # Định tuyến động tên index theo từng ứng dụng/service: %{[app_name]}-%{+YYYY.MM.dd}
    index => "%{[app_name]}-%{+YYYY.MM.dd}"
  }
  stdout {
    codec => rubydebug
  }
}
```

### 3.2. Cấu hình Docker Compose (`docker-compose.yml`)

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

### 4.1. Khởi chạy toàn bộ hệ thống
```bash
docker compose up -d
```

### 4.2. Gửi đồng thời luồng log từ hai dịch vụ `user-service` và `order-service`
Sử dụng script gửi qua cổng TCP 5000:

```powershell
$client = New-Object System.Net.Sockets.TcpClient("localhost", 5000)
$stream = $client.GetStream()
$writer = New-Object System.IO.StreamWriter($stream)

# Gửi log của user-service
$userLog = '{"app_name":"user-service","level":"INFO","message":"User login successful","userId":"user_001"}'
$writer.WriteLine($userLog)

# Gửi log của order-service
$orderLog = '{"app_name":"order-service","level":"INFO","message":"Order created successfully","orderId":"order_999"}'
$writer.WriteLine($orderLog)

$writer.Flush()
$client.Close()
```

### 4.3. Kiểm tra danh sách Index trên Elasticsearch
Kiểm tra qua API `GET _cat/indices?v`:
```bash
curl http://localhost:9200/_cat/indices?v
```

Kết quả phản hồi trên Elasticsearch hiển thị 2 index độc lập:
```text
health status index                           docs.count
yellow open   order-service-2026.09.28                 1
yellow open   user-service-2026.09.28                  1
```

### 4.4. Kiểm tra nội dung document trong từng Index riêng biệt
- **Truy vấn log `user-service`**:
  ```bash
  curl http://localhost:9200/user-service-*/_search?pretty
  ```
  *Kết quả trả về*: Bản ghi có `app_name: "user-service"`, `userId: "user_001"`, `message: "User login successful"`.

- **Truy vấn log `order-service`**:
  ```bash
  curl http://localhost:9200/order-service-*/_search?pretty
  ```
  *Kết quả trả về*: Bản ghi có `app_name: "order-service"`, `orderId: "order_999"`, `message: "Order created successfully"`.

---

## 5. Kết luận

- Việc áp dụng biến động `%{[app_name]}-%{+YYYY.MM.dd}` trong Logstash Output giải quyết triệt để bài toán phân loại và cách ly dữ liệu log theo từng service trong kiến trúc Microservices.
- Hệ thống đảm bảo tính mở rộng cao: khi có thêm bất kỳ dịch vụ mới nào (ví dụ `payment-service`, `inventory-service`), Logstash sẽ tự động sinh Index mới tương ứng mà không cần phải cấu hình lại hay khởi động lại pipeline.
