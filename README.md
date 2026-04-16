# Kiến trúc Hệ thống - DDD & Clean Architecture

Source code được thiết kế dựa trên các nguyên lý của **Domain-Driven Design (DDD)** và **Clean Architecture** . Mục tiêu của kiến trúc này là đảm bảo tính độc lập của Business Logic, dễ dàng mở rộng, bảo trì và dễ dàng viết Unit Test.

Dự án sử dụng chiến lược **Multi-Module Maven** kết hợp với Spring Boot để phân định ranh giới vật lý (physical boundaries) giữa các layer, qua đó dùng chính trình biên dịch (compiler) để ép buộc tuân thủ quy tắc phụ thuộc (Dependency Rule).

## Cấu trúc Layer

Kiến trúc được chia thành 5 Layer độc lập, phục vụ các mục đích cụ thể:

### 1. `domain` 
- **Trách nhiệm:** Lưu trữ toàn bộ "trái tim" của phần mềm. Đây là nơi chứa các Aggregates, Entities, Value Objects, cũng như khai báo các **Interfaces** của Service và Repository.
- **Quy tắc phụ thuộc:** Phải mang tính độc lập nhất. Module này không biết về cơ sở dữ liệu, giao diện người dùng hay cách dữ liệu đi qua mạng. (Lưu ý: Ở dự án này có phụ thuộc nhỏ vào `spring-boot-starter-data-jpa` để tận dụng các annotation cấu trúc theo hệ sinh thái Spring).
- *Lợi ích:* Đảm bảo Core Business Logic luôn tĩnh, không bị rò rỉ hay thay đổi khi nâng cấp, thay thế thư viện bề mặt.

### 2. `controller`
- **Trách nhiệm:** Xử lý nhận giao tiếp từ thế giới bên ngoài vào hệ thống thông qua REST API.
- **Thành phần:** Chứa các `@RestController` và các Data Transfer Objects (DTOs) dùng để hứng request và serialize JSON cho response.
- **Quy tắc phụ thuộc:** Phụ thuộc vào `domain` để gọi các use case thông qua Interfaces, hoàn toàn không được biết về lớp `service`.

### 3. `service`
- **Trách nhiệm:** Triển khai (implement) các logic và hạ tầng từ các Interfaces đã được định nghĩa ở `domain`. Module này sẽ chứa các hành vi tương tác trực tiếp với cơ sở dữ liệu (H2 Database), các thao tác hạ tầng và xử lý nghiệp vụ nặng.
- **Quy tắc phụ thuộc:** Phụ thuộc ngược lại vào `domain`. Đây là minh chứng đặc trưng cho nguyên lý **Dependency Inversion (SOLID)** cốt lõi của Clean Architecture.

### 4. `authserver` 
- **Trách nhiệm:** Tập trung xác thực (Authentication) và phân quyền (Authorization) của ứng dụng, sử dụng Spring Security. Xử lý JWT Tokens.
- **Lợi ích:** Tách biệt vấn đề bảo mật một cách độc lập mà không xáo trộn lõi nghiệp vụ.

### 5. `application`
- **Trách nhiệm:** Đây là module cao nhất dùng để khởi chạy dự án, chứa class có `@SpringBootApplication`. Nó đóng vai trò là một "Composition Root".
- **Quy tắc phụ thuộc:** Thấy và "gom" tĩnh tất cả các module còn lại (`controller`, `service`, `authserver`, v.v.). Nhờ có Application module, IoC Container của Spring mới có thể inject các Object như Repository Impl từ `service` vào các interface đang chờ sẵn tại `domain` hay `controller`.

---

## Đánh giá

1. **Bảo vệ ranh giới cực độ (Strict Boundary Enforcement):** Bằng cách tách thành Maven Modules thay vì chỉ xếp vào các packages trong 1 source nguyên khối, ta tránh được việc các kĩ sư phần mềm (đặc biệt là Junior) vô tình bypass layer (ví dụ: truy vấn DB thẳng từ biến Controller bằng Repository). Lỗi "Dependency" sẽ được ném thẳng ở thời điểm biên dịch (Compile-time) thay vì chờ code review phát hiện.
2. **Khả năng thay thế và tiến hoá độc lập (High Pluggability):** Giả sử định hướng tương lai chuyển từ REST API hiện tại sang gRPC hoặc GraphQL, chỉ cần tạo ra một module `grpc-delivery` trỏ thẳng vào `domain` thay vì đập bỏ sửa đổi logic. Việc thay thế từ hệ quản trị H2/MySQL sang MongoDB cũng tương tự, viết `service-mongo` trỏ vào `domain`.
3. **Mở rộng quy mô phát triển (Scalability for Enterprise Teams):** Có thể giao module `domain` thiết kế các Contract (Interface) trước, trong khi Team Frontend làm việc song song qua Mock. Team Infra cũng sẽ độc lập phát triển tại layer `service`. Tính kết dính lỏng (loose coupling) đạt mức xuất sắc.

---

## Hướng dẫn Tương tác API (Development)

Trong giai đoạn này, có các API đang hiện hữu để trải nghiệm:

- **H2 Database Console:** `http://localhost:8080/h2-console`

**Customer Endpoints:**
- Liệt kê toàn bộ khách hàng: `GET http://localhost:8080/customers`
- Thông tin chi tiết khách hàng: `GET http://localhost:8080/customers/<UUID>`
- Tạo khách hàng mới: `POST http://localhost:8080/customers?name=<name>&job=<job>`
- Xóa tất cả khách hàng: `DELETE http://localhost:8080/customers`
