# BÀI TẬP SESSION 17 (MD5) - BÀI TẬP 4 (HW4)
## Phân tích và Khắc phục lỗi Lost Log & Nghẽn Hệ thống khi Logstash Quá tải (AsyncAppender Pattern)

---

## 1. Mục tiêu kiến thức & Tình huống nghiệp vụ

- **Mục tiêu kiến thức**:
  - Hiểu sâu sắc cơ chế hoạt động của cơ chế hàng đợi (**Queue**) và vùng đệm (**Buffer**) trong kiến trúc logging tập trung.
  - Phân tích được hiệu ứng sập đổ dây chuyền (**Cascading Failure / Thread Starvation**) khi sử dụng Appender truyền dữ liệu qua mạng dạng đồng bộ.
  - Nắm vững cách cấu hình **`AsyncAppender`** trong Logback / Spring Boot và hiểu bản chất từng thông số kỹ thuật then chốt (`queueSize`, `neverBlock`, `discardingThreshold`).
  - Nắm được mô hình kiến trúc mở rộng chuẩn Production sử dụng Message Broker (Kafka/RabbitMQ) làm lớp đệm bền vững.

- **Tình huống nghiệp vụ**:
  - Vào sự kiện Flash Sale, hệ thống bán hàng **StoreX** ghi nhận lượng truy cập khổng lồ, phát sinh lưu lượng log lên tới **10.000 dòng log/giây**.
  - Cụm Logstash tiếp nhận không kịp xử lý dẫn đến nghẽn mạng và chậm phản hồi.
  - Hệ quả nghiêm trọng: Ứng dụng Backend Spring Boot bị lỗi **Timeout** liên tục, các luồng xử lý đơn hàng bị đóng băng, khách hàng không thể thanh toán, gây gián đoạn và thiệt hại doanh thu trực tiếp cho doanh nghiệp.

---

## 2. Phân tích nguyên nhân gốc rễ (Root Cause Analysis)

### 2.1. Vì sao Appender TCP đồng bộ lại làm chậm và sập toàn bộ ứng dụng chính?

Cấu hình ban đầu của ứng dụng StoreX:
```xml
<appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>logstash:5000</destination>
    <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
</appender>
```

```
           [ CƠ CHẾ NGHẼN CỦA ĐỒNG BỘ: HIỆU ỨNG SẬP DÂY CHUYỀN (THREAD STARVATION) ]

   Khách hàng gửi Request
            │
            ▼
    ┌────────────────────────────────────────────────────────┐
    │  TOMCAT THREAD POOL (Giới hạn mặc định: 200 threads)    │
    │                                                        │
    │  [Worker Thread 1] ──► Xử lý Đơn hàng                  │
    │                           │                            │
    │                           ▼                            │
    │                     log.info(...)                      │
    │                           │                            │
    │                           ▼  (Ghi mạng Socket TCP)     │
    │                   ┌───────────────┐                    │
    │                   │ TCP Socket    │                    │
    │                   │ OS Send Buffer│ ──► [ LOGSTASH ]   │
    │                   └───────────────┘       (BỊ QUÁ TẢI) │
    │                           │               (Xử lý chậm) │
    │                           ▼                            │
    │             Buffer đầy -> socket.write() BỊ BLOCK      │
    │             Thread 1 bị TREO vô thời hạn!             │
    │                                                        │
    │  [Worker Thread 2] ──► log.info(...) ──► BỊ BLOCK      │
    │  [Worker Thread 3] ──► log.info(...) ──► BỊ BLOCK      │
    │  ...                                                   │
    │  [Worker Thread 200] ─► log.info(...) ──► BỊ BLOCK     │
    └────────────────────────────────────────────────────────┘
                                │
                                ▼
         CẠN KIỆT TOÀN BỘ WORKER THREAD TRONG THREAD POOL!
         Hệ thống sập hoàn toàn (504 Gateway Timeout / Drop kết nối)
         Chỉ vì một dòng lệnh ghi Log!
```

### 2.2. Chuỗi sự kiện gây sập hệ thống (Step-by-step Failure Analysis):
1. **Lưu lượng tăng đột biến**: Vào đợt Sale, hàng ngàn khách hàng thao tác đồng thời tạo ra 10.000 log/s.
2. **Logstash bị quá tải**: Do tài nguyên CPU, I/O hoặc Elasticsearch ghi chậm, Logstash không kịp đọc dữ liệu từ Socket TCP.
3. **Tràn bộ đệm TCP (TCP Window Full)**: Bộ đệm socket của hệ điều hành phía máy chủ ứng dụng bị đầy (TCP Zero Window), không thể gửi thêm byte nào qua mạng.
4. **Luồng chính bị khóa (Thread Blocking)**: Khi gọi `log.info(...)`, luồng xử lý chính (`http-nio-8080-exec-x`) phải đợi phương thức `socket.write()` hoàn tất. Do socket bị nghẽn, luồng này bị treo và chuyển sang trạng thái `BLOCKED` / `WAITING`.
5. **Cạn kiệt Thread Pool (Thread Pool Exhaustion)**: Web Server (Tomcat) có số lượng thread giới hạn (thường là 200 threads). Từng thread xử lý xong phần logic nghiệp vụ nhưng bị giữ lại ở bước ghi log, dẫn đến toàn bộ 200 threads đều bị treo.
6. **Sập hệ thống toàn diện**: Các request tiếp theo từ khách hàng không có thread nào phục vụ, dẫn đến lỗi `Connection Refused` hoặc `HTTP 504 Gateway Timeout`.

---

## 3. Giải pháp kỹ thuật: Cơ chế Bất đồng bộ với `AsyncAppender`

### 3.1. Sơ đồ kiến trúc AsyncAppender

```
             [ KIẾN TRÚC BẤT ĐỒNG BỘ BẢO VỆ LUỒNG CHÍNH (ASYNCAPPENDER) ]

   Khách hàng gửi Request
            │
            ▼
    ┌──────────────────────────────────────────────────────────────┐
    │  TOMCAT WORKER THREAD                                        │
    │                                                              │
    │  1. Xử lý nghiệp vụ thanh toán (Business Logic)              │
    │  2. log.info(...)                                            │
    │  3. Đẩy log vào BlockingQueue trong RAM (Chỉ mất vài micro-giây) │
    │  4. Ngay lập tức giải phóng thread để phục vụ khách khác!    │
    └──────────────────────────────┬───────────────────────────────┘
                                   │
                                   ▼ (Non-blocking enqueue)
                    ┌──────────────────────────────┐
                    │     IN-MEMORY RING BUFFER    │
                    │        (BlockingQueue)       │
                    │     Dung lượng: 16.384 log   │
                    └──────────────┬───────────────┘
                                   │
                                   ▼ (Background Daemon Thread)
                    ┌──────────────────────────────┐
                    │     ASYNC WORKER THREAD      │
                    │   (Chạy nền ngầm độc lập)   │
                    │  - Lấy log từ Queue ra       │
                    │  - Gửi qua TCP tới Logstash  │
                    └──────────────┬───────────────┘
                                   │
                                   ▼ (TCP Socket: 5000)
                            [ LOGSTASH : 5000 ]
```

### 3.2. Phân tích chi tiết các thuộc tính cốt lõi của `AsyncAppender`

| Thuộc tính | Giá trị tối ưu | Ý nghĩa kỹ thuật & Tác động thực tế |
| :--- | :--- | :--- |
| **`queueSize`** | `16384` hoặc `65536` | Kích thước hàng đợi trong bộ nhớ RAM (mặc định là `256`). Với 10.000 log/s, giá trị 256 sẽ bị đầy trong chưa đầy 0.03 giây. Tăng lên 16384 hoặc 65536 tạo ra vùng đệm đủ lớn để ứng phó với các đợt lưu lượng tăng vọt (traffic spikes). |
| **`neverBlock`** | **`true`** | **THUỘC TÍNH QUAN TRỌNG NHẤT**: Mặc định là `false` (nghĩa là khi hàng đợi đầy, Worker Thread vẫn sẽ bị BLOCK). Khi cấu hình `neverBlock=true`, nếu hàng đợi bị đầy, Logback sẽ tự động vứt bỏ (DROP) log mới thay vì bắt luồng chính phải chờ. **Quy tắc vàng: Thà mất một vài dòng log chứ tuyệt đối không để sập hệ thống bán hàng.** |
| **`discardingThreshold`** | `20` | Ngưỡng dung lượng hàng đợi còn trống để bắt đầu loại bỏ log (tính theo %). Khi dung lượng trống < 20%, hệ thống tự động loại bỏ các log mức thấp (`TRACE`, `DEBUG`, `INFO`) để ưu tiên giữ lại các log nghiêm trọng (`WARN`, `ERROR`). |
| **`includeCallerData`** | `false` | Mặc định là `false`. Tuyệt đối không bật `true` trên môi trường tải cao vì việc trích xuất thông tin Class/Method/Line Number đòi hỏi chi phí StackTrace Inspection rất đắt đỏ về CPU. |
| **`maxFlushTime`** | `5000` | Thời gian tối đa (ms) cho phép background thread tiếp tục xả hết log còn tồn đọng trong hàng đợi khi ứng dụng tắt (Graceful Shutdown). |

---

## 4. File cấu hình hoàn chỉnh (`logback-spring.xml`)

File cấu hình chi tiết đặt tại `src/main/resources/logback-spring.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="30 seconds">

    <!-- Cấu hình tên ứng dụng -->
    <springProperty scope="context" name="appName" source="spring.application.name" defaultValue="storex-service"/>

    <!-- 1. Console Appender hiển thị log cục bộ -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- 2. TCP Appender gửi log sang Logstash -->
    <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
        <destination>logstash:5000</destination>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"app_name":"${appName}"}</customFields>
        </encoder>
        <connectionStrategy>
            <roundRobin>
                <connectionTimeout>5000</connectionTimeout>
            </roundRobin>
        </connectionStrategy>
    </appender>

    <!-- 3. AsyncAppender: Giải pháp xử lý bất đồng bộ chống nghẽn luồng chính -->
    <appender name="ASYNC_LOGSTASH" class="ch.qos.logback.classic.AsyncAppender">
        <!-- Tham chiếu tới Logstash TCP Appender -->
        <appender-ref ref="LOGSTASH" />

        <!-- 
            Dung lượng hàng đợi đệm (Queue Size):
            Mặc định là 256. Với lưu lượng cao (10.000 logs/s), cần tăng kích thước hàng đợi
            để có đủ không gian đệm khi Logstash gặp độ trễ (ví dụ: 16384 hoặc 65536).
        -->
        <queueSize>16384</queueSize>

        <!-- 
            Ngưỡng loại bỏ log (Discarding Threshold):
            Mặc định là 20. Khi hàng đợi chỉ còn trống dưới 20%, Logback sẽ tự động bỏ qua
            các log mức thấp (TRACE, DEBUG, INFO) để nhường hàng đợi cho log quan trọng (WARN, ERROR).
        -->
        <discardingThreshold>20</discardingThreshold>

        <!-- 
            neverBlock: THUỘC TÍNH CỐT LÕI ĐỂ KHÔNG BLOCK ỨNG DỤNG CHÍNH
            - true: Khi hàng đợi bị đầy (buffer full), drop log ngay lập tức và trả quyền điều khiển 
                    lại cho Worker Thread (Non-blocking). Ứng dụng chính tiếp tục chạy bình thường.
            - false (mặc định): Luồng Worker Thread bị TREO (BLOCK) chờ hàng đợi có chỗ trống,
                    khiến toàn bộ luồng xử lý đơn hàng bị đóng băng khi Logstash nghẽn.
        -->
        <neverBlock>true</neverBlock>

        <!-- Thời gian tối đa (ms) chờ đẩy hết log trong queue khi ứng dụng dừng (Graceful Shutdown) -->
        <maxFlushTime>5000</maxFlushTime>

        <!-- Tắt includeCallerData để tăng tốc độ ghi log, tránh tốn CPU cho việc truy vết stacktrace -->
        <includeCallerData>false</includeCallerData>
    </appender>

    <!-- Root Logger sử dụng Appender bất đồng bộ ASYNC_LOGSTASH -->
    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="ASYNC_LOGSTASH" />
    </root>

</configuration>
```

---

## 5. Kiến trúc đệm mở rộng cho hệ thống quy mô lớn (Enterprise Production)

Mặc dù `AsyncAppender` với `neverBlock=true` giải quyết triệt để vấn đề nghẽn luồng chính, nhưng việc lưu hàng đợi trong RAM của ứng dụng vẫn có 2 hạn chế:
1. **Rủi ro mất mát dữ liệu (Data Loss)**: Nếu server bị sập đột ngột (Crash / Out of Memory / Mất điện), số log đang nằm trong RAM chưa kịp gửi sẽ mất vĩnh viễn.
2. **Giới hạn bộ nhớ RAM**: Nếu hàng đợi quá lớn, ứng dụng sẽ tốn nhiều heap memory và tăng thời gian dừng do Garbage Collection (GC Pause).

### Kiến trúc giải pháp với Message Broker (Kafka Buffer):

```
                   [ KIẾN TRÚC LOGGING MỞ RỘNG DOANH NGHIỆP ]

   ┌──────────────────┐
   │ Spring Boot Apps │
   │  (Microservices) │
   └────────┬─────────┘
            │  (Đẩy nhanh qua Kafka Producer)
            ▼
   ┌──────────────────────────────────────────────┐
   │             APACHE KAFKA CLUSTER             │
   │  - Lớp đệm bền vững ghi trực tiếp xuống đĩa │
   │  - Khả năng chịu tải hàng trăm nghìn log/s   │
   │  - Lưu trữ dữ liệu an toàn từ 3 đến 7 ngày   │
   └──────────────────────┬───────────────────────┘
                          │
                          ▼ (Pull theo tốc độ xử lý của Consumer)
                 ┌──────────────────┐
                 │ LOGSTASH CLUSTER │
                 └────────┬─────────┘
                          │
                          ▼
                 ┌──────────────────┐
                 │  ELASTICSEARCH   │
                 └──────────────────┘
```

- **Nguyên lý hoạt động**:
  - Ứng dụng Spring Boot đẩy log vào topic `app-logs` của **Kafka** hoặc **RabbitMQ**.
  - Kafka đóng vai trò như một **bể chứa đệm khổng lồ (Shock Absorber)**, có khả năng lưu trữ hàng triệu log xuống đĩa an toàn.
  - Logstash hoạt động với tư cách là Kafka Consumer: chủ động kéo log về theo tốc độ xử lý tối đa của nó. Dù Logstash có bị bảo trì hay chậm trong nhiều giờ, ứng dụng chính vẫn hoạt động trơn tru 100% và không có bất kỳ dòng log nào bị mất mát.

---

## 6. Kết luận

1. **Nguyên tắc bất di bất dịch trong Thiết kế Hạ tầng**: Tác vụ phụ (như Ghi Log / Metrics / Tracing) **tuyệt đối không bao giờ được phép làm ảnh hưởng hoặc kéo sập luồng xử lý nghiệp vụ chính** của ứng dụng.
2. Luôn bọc các Network Appender qua **`AsyncAppender`** với thuộc tính **`neverBlock=true`** để đảm bảo tính sẵn sàng cao (**High Availability**) cho toàn hệ thống.
3. Khi lưu lượng vượt ngưỡng hàng chục nghìn log/s, kiến trúc nên tích hợp thêm **Message Broker (Kafka)** để đảm bảo cả hai tiêu chí: **Không nghẽn ứng dụng** và **Không mất mát log**.
