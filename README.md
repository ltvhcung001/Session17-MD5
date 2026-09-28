# BÀI TẬP SESSION 17 (MD5) - BÀI TẬP 1 (HW1)
## Khắc phục lỗi kết nối giữa Kibana và Elasticsearch trong cụm ELK/EFK Stack

---

## 1. Mục tiêu kiến thức & Tình huống nghiệp vụ

- **Mục tiêu kiến thức**:
  - Nắm vững kiến trúc và cách thiết lập các thành phần trong bộ công cụ **Elasticsearch**, **Fluentd/Logstash**, và **Kibana** (EFK / ELK Stack).
  - Hiểu rõ vai trò của từng cổng mạng (network ports) và giao thức truyền thông trong hệ sinh thái Elastic.
- **Tình huống nghiệp vụ**:
  - Đội ngũ phát triển (Dev team) quyết định ứng dụng ELK/EFK để quản lý và phân tích log tập trung.
  - Một bạn Dev đã soạn thảo file `docker-compose.yml`, nhưng khi triển khai hệ thống, service Kibana liên tục ghi log cảnh báo lỗi `Elasticsearch is unreachable` và giao diện web hiển thị lỗi `Kibana server is not ready yet`.

---

## 2. Phân tích nguyên nhân gốc rễ (Root Cause Analysis)

### 2.1. Bản chất sự khác biệt giữa Port 9200 và Port 9300 của Elasticsearch

Elasticsearch mặc định lắng nghe trên 2 cổng mạng phục vụ hai mục đích hoàn toàn khác nhau:

| Tiêu chí | Cổng 9200 (HTTP REST API) | Cổng 9300 (Transport Protocol) |
| :--- | :--- | :--- |
| **Giao thức** | HTTP / HTTPS (JSON payload) | Giao thức nhị phân Java nội bộ (Binary TCP) |
| **Mục đích sử dụng** | Cổng giao tiếp chuẩn RESTful API | Giao tiếp nội bộ giữa các node trong cụm (Node-to-Node, Clustering) |
| **Đối tượng kết nối** | **Kibana**, Logstash, Fluentd, Trình duyệt web, REST Client (Postman, Curl), Ứng dụng Backend | Các Elasticsearch Node khác trong Cluster, Java Transport Client (legacy) |

```
                                  [ KIẾN TRÚC KẾT NỐI ELK ]

   Trình duyệt (Browser)
            │
            ▼ (HTTP - Port 5601)
     ┌──────────────┐
     │    KIBANA    │
     └──────┬───────┘
            │
            │  [X] SAI: http://elasticsearch:9300 (Gửi HTTP vào cổng Binary TCP -> Lỗi)
            │  [v] ĐÚNG: http://elasticsearch:9200 (Gửi HTTP REST API vào cổng HTTP)
            ▼
     ┌──────────────┐
     │ELASTICSEARCH │◄── (Cổng 9300: Dành riêng node-to-node cluster giao tiếp nhị phân)
     └──────────────┘
```

### 2.2. Lý do phát sinh lỗi trong cấu hình ban đầu
- Trong cấu hình ban đầu, biến môi trường của Kibana được khai báo:
  ```yaml
  ELASTICSEARCH_HOSTS=http://elasticsearch:9300
  ```
- Kibana sử dụng HTTP client để gửi các truy vấn REST API đến cổng `9300`. Vì cổng `9300` của Elasticsearch chỉ hiểu giao thức nhị phân Transport thay vì HTTP, yêu cầu của Kibana bị từ chối hoặc không phản hồi đúng định dạng HTTP.
- Hậu quả: Kibana không thể khởi tạo kết nối (handshake) với Elasticsearch, liên tục cảnh báo `Elasticsearch is unreachable`, chuyển sang trạng thái chờ và hiển thị lỗi `Kibana server is not ready yet` khi người dùng truy cập.

### 2.3. Các điểm cần hoàn thiện bổ sung
1. **Sửa biến môi trường kết nối**: Đổi `ELASTICSEARCH_HOSTS` thành `http://elasticsearch:9200`.
2. **Ánh xạ cổng máy host (Port Mapping)**:
   - Service `kibana`: Mở cổng `5601:5601` để truy cập dashboard từ máy host (`http://localhost:5601`).
   - Service `elasticsearch`: Mở cổng `9200:9200` để kiểm tra trạng thái cluster và phục vụ truy vấn HTTP từ máy host.
3. **Quản lý bộ nhớ Heap & Tương thích Kernel (ES_JAVA_OPTS & JDK_JAVA_OPTIONS)**:
   - Thiết lập `ES_JAVA_OPTS=-Xms512m -Xmx512m` giúp Elasticsearch hoạt động ổn định, tránh chiếm dụng quá nhiều RAM trên môi trường phát triển cục bộ.
   - Thêm `JDK_JAVA_OPTIONS=-XX:-UseContainerSupport` để xử lý triệt để lỗi NPE cgroup v2 của JVM trên các nhân Linux/Docker hiện đại.
4. **Phụ thuộc khởi động (depends_on)**: Đảm bảo Kibana khởi tạo sau khi Elasticsearch được tạo.

---

## 3. Cấu hình Docker Compose hoàn chỉnh (`docker-compose.yml`)

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

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.0
    container_name: kibana
    environment:
      # Sửa cổng kết nối từ 9300 (transport) về đúng cổng 9200 (HTTP REST API)
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

## 4. Hướng dẫn khởi chạy & Kiểm tra kết quả

### 4.1. Khởi chạy hệ sinh thái
Mở terminal tại thư mục chứa file `docker-compose.yml` và chạy:
```bash
docker compose up -d
```

### 4.2. Kiểm tra trạng thái của các container
```bash
docker compose ps
```

### 4.3. Kiểm tra Elasticsearch REST API
Truy cập trực tiếp từ máy host:
```bash
curl http://localhost:9200
```
Phản hồi thành công sẽ trả về JSON chứa thông tin cụm, phiên bản Elasticsearch (`7.17.0`) và khẩu hiệu `"You Know, for Search"`.

### 4.4. Kiểm tra giao diện người dùng Kibana
- Truy cập trình duyệt tại địa chỉ: `http://localhost:5601`
- Kết quả mong muốn: Giao diện Kibana tải thành công, không gặp cảnh báo `Elasticsearch is unreachable` hay lỗi `server is not ready yet`.

---

## 5. Kết luận

- Việc phân định rõ giữa **Cổng REST API (9200)** và **Cổng Transport (9300)** là nguyên lý cơ bản và quan trọng khi thiết lập cụm Elasticsearch cũng như tích hợp các dịch vụ vệ tinh (Kibana, Logstash, Fluentd, Beat agents).
- Sau khi cấu hình đúng `ELASTICSEARCH_HOSTS=http://elasticsearch:9200` và ánh xạ cổng `5601:5601`, Kibana kết nối liền mạch tới Elasticsearch, hệ thống sẵn sàng cho việc thu thập và trực quan hóa log.
