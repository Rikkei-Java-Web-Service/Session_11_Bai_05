# Bài 5: Chiến lược kiểm thử cho hệ thống Booking Service

## Phần 1 - Phân tích Logic

### Thách thức kiểm thử khi hệ thống có nhiều service giao tiếp với nhau

Khi hệ thống Booking System mở rộng với nhiều service (UserService, BookingService, NotificationService) giao tiếp qua API REST, các thách thức kiểm thử chính bao gồm:

#### 1. Tính phụ thuộc chéo giữa các service
- **BookingService** phụ thuộc vào **UserService** để xác thực người dùng và **NotificationService** để gửi thông báo.
- Nếu UserService lỗi, BookingService cũng bị ảnh hưởng dù logic booking hoàn toàn đúng.
- Khó xác định lỗi thuộc về service nào khi nhiều service cùng liên quan.

#### 2. Dữ liệu không nhất quán
- Mỗi service có database riêng. Khi một booking được tạo, cần đồng bộ dữ liệu giữa UserService (lịch sử đặt lịch) và BookingService (trạng thái booking).
- Network failures có thể gây ra trạng thái không nhất quán (ví dụ: booking tạo thành công nhưng notification gửi thất bại).

#### 3. Khó mô phỏng môi trường thực
- Trong Unit Test, ta mock các dependency. Nhưng mock không phản ánh chính xác hành vi thực của API REST (latency, timeout, error responses).
- Các lỗi chỉ xuất hiện khi các service thực sự giao tiếp với nhau (integration bugs).

#### 4. Tại sao chỉ Unit Test là CHƯA ĐỦ?

| Loại lỗi | Unit Test phát hiện? | Cần loại test nào? |
|---|---|---|
| Bug logic trong một method | ✅ Có | Unit Test |
| API contract sai (request/response format) | ❌ Không | API Test |
| Database query sai | ❌ Không | Integration Test |
| Luồng nghiệp vụ end-to-end bị lỗi | ❌ Không | E2E Test |
| Timeout/retry giữa các service | ❌ Không | Integration Test |
| UI + Backend không đồng bộ | ❌ Không | E2E Test |

#### Vai trò của từng loại test:

- **Unit Test**: Kiểm thử logic nghiệp vụ trong từng method/class riêng lẻ. Chạy nhanh, dễ debug. Mock tất cả dependency.
- **Integration Test**: Kiểm thử tương tác giữa các thành phần (ví dụ: Service ↔ Database, Service ↔ Service). Phát hiện lỗi cấu hình, query, mapping.
- **API Test**: Kiểm thử REST API endpoints. Đảm bảo request/response contract đúng, status code chính xác, validation hoạt động.
- **End-to-End Test**: Kiểm thử toàn bộ luồng nghiệp vụ từ đầu đến cuối. Phát hiện lỗi tổng hợp mà các test đơn lẻ không thể tìm thấy.

---

## Phần 2 - Thiết kế chiến lược kiểm thử

### 1. Mô hình Test Pyramid

```
         /‾‾‾‾‾‾‾‾‾‾‾‾‾\
        /   E2E Tests    \          ~5-10% tổng số test
       /    (ít nhất)     \
      /___________________\
     /                     \
    /  Integration Tests    \       ~20-30% tổng số test
   /   (vừa phải)           \
  /_________________________\
 /                           \
/      Unit Tests              \    ~60-70% tổng số test
/   (nhiều nhất, nhanh nhất)   \
/________________________________\
```

**Tỷ lệ đề xuất cho Booking System:**

| Loại Test | Tỷ lệ | Mục tiêu | Thời gian chạy |
|---|---|---|---|
| **Unit Test** | 60-70% | Kiểm thử logic nghiệp vụ từng method | < 1 phút |
| **Integration Test** | 20-30% | Kiểm thử tương tác service-database, service-service | 2-5 phút |
| **E2E Test** | 5-10% | Kiểm thử luồng nghiệp vụ quan trọng nhất | 5-15 phút |

### 2. Công cụ kiểm thử phù hợp

| Công cụ | Mục đích | Áp dụng cho |
|---|---|---|
| **JUnit 5** | Framework test chính | Tất cả loại test |
| **Mockito** | Mock dependency trong Unit Test | Unit Test |
| **AssertJ** | Assertion thông minh, dễ đọc | Tất cả loại test |
| **Spring Boot Test** | Bootstrap Spring context cho integration test | Integration Test |
| **@DataJpaTest** | Test JPA repository với H2 in-memory DB | Repository layer |
| **@WebMvcTest** | Test Controller layer (không load full context) | API/Controller Test |
| **REST Assured** | Test REST API endpoint với cú pháp BDD | API Test, E2E Test |
| **Testcontainers** | Chạy database thực (MySQL, PostgreSQL) trong Docker | Integration Test |

**Ví dụ áp dụng cho Booking System:**

```
BookingService
├── Unit Test (JUnit 5 + Mockito + AssertJ)
│   ├── BookingServiceTest          → Mock UserService, NotificationService
│   ├── BookingValidatorTest        → Test validation logic
│   └── PricingCalculatorTest       → Test tính giá
│
├── Integration Test (Spring Boot Test)
│   ├── BookingRepositoryTest       → @DataJpaTest, test query với H2
│   ├── BookingControllerTest       → @WebMvcTest, test REST endpoints
│   └── BookingServiceIntTest       → Full context, test service + DB
│
└── E2E Test (REST Assured)
    ├── BookingFlowTest             → Tạo booking → xác nhận → thông báo
    └── CancelBookingFlowTest       → Hủy booking → hoàn tiền → thông báo
```

### 3. Sử dụng JaCoCo để đo Coverage

#### Cấu hình JaCoCo trong build.gradle:

```groovy
plugins {
    id 'jacoco'
}

jacoco {
    toolVersion = "0.8.11"
}

jacocoTestReport {
    reports {
        xml.required = true
        html.required = true
    }
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                counter = 'LINE'
                minimum = 0.80  // Tối thiểu 80% Line Coverage
            }
        }
        rule {
            limit {
                counter = 'BRANCH'
                minimum = 0.70  // Tối thiểu 70% Branch Coverage
            }
        }
    }
}

test {
    finalizedBy jacocoTestReport
}

check {
    dependsOn jacocoTestCoverageVerification
}
```

#### Tại sao cần thiết lập ngưỡng coverage tối thiểu?

1. **Đảm bảo chất lượng tối thiểu**: Ngăn chặn code mới được merge mà không có test đầy đủ.
2. **Phát hiện sớm regression**: Nếu coverage giảm, có nghĩa là code mới chưa được test hoặc test cũ bị xóa.
3. **Tạo văn hóa viết test**: Khi ngưỡng được enforce trong CI/CD, developer buộc phải viết test.
4. **Giảm lỗi trên staging/production**: Coverage cao (đặc biệt Branch Coverage) giúp phát hiện bug sớm trước khi deploy.

### 4. Kiểm thử giao tiếp giữa các service

#### Chiến lược 1: Contract Testing (Kiểm thử hợp đồng)
- Đảm bảo API contract giữa BookingService và UserService không thay đổi bất ngờ.
- Sử dụng **Spring Cloud Contract** hoặc **Pact** để tạo contract test.
- Provider (UserService) publish contract → Consumer (BookingService) verify.

#### Chiến lược 2: Integration Test với MockServer/WireMock
- Sử dụng **WireMock** để mock HTTP responses từ UserService, NotificationService.
- BookingService gọi API thực tế đến WireMock → kiểm tra xử lý response đúng.

```java
@SpringBootTest
@AutoConfigureWireMock(port = 8089)
class BookingServiceIntegrationTest {

    @Test
    void shouldCreateBooking_whenUserExists() {
        // Stub UserService response
        stubFor(get(urlEqualTo("/api/users/user-001"))
            .willReturn(aResponse()
                .withStatus(200)
                .withBody("{\"id\":\"user-001\",\"name\":\"Nguyen Van A\"}")
            ));

        // Test BookingService gọi UserService
        Booking booking = bookingService.createBooking("user-001", "2024-01-15", "09:00");
        assertThat(booking.getStatus()).isEqualTo("CONFIRMED");
    }
}
```

#### Chiến lược 3: REST Assured cho API Test
```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class BookingApiTest {

    @LocalServerPort
    int port;

    @Test
    void shouldReturn201_whenCreateBookingSuccessfully() {
        given()
            .port(port)
            .contentType(ContentType.JSON)
            .body(new BookingRequest("user-001", "2024-01-15", "09:00"))
        .when()
            .post("/api/bookings")
        .then()
            .statusCode(201)
            .body("status", equalTo("CONFIRMED"))
            .body("userId", equalTo("user-001"));
    }
}
```

### 5. Tự động hóa test trong quá trình build

#### CI/CD Pipeline đề xuất:

```
┌─────────────┐     ┌──────────────┐     ┌───────────────┐     ┌──────────────┐
│  Git Push    │────▶│  Unit Tests  │────▶│  Integration  │────▶│  Coverage    │
│  / PR        │     │  (< 1 min)   │     │  Tests (5min) │     │  Report      │
└─────────────┘     └──────────────┘     └───────────────┘     └──────────────┘
                                                                       │
                                                                       ▼
                    ┌──────────────┐     ┌───────────────┐     ┌──────────────┐
                    │   Deploy     │◀────│  E2E Tests    │◀────│  Coverage    │
                    │   Staging    │     │  (10 min)     │     │  Check       │
                    └──────────────┘     └───────────────┘     └──────────────┘
```

#### Cấu hình Gradle để chạy test tự động:

```groovy
// Trong build.gradle
tasks.named('test') {
    useJUnitPlatform()

    // Chạy unit test song song để tăng tốc
    maxParallelForks = Runtime.runtime.availableProcessors().intdiv(2) ?: 1

    // Bắt buộc test phải pass trước khi build
    testLogging {
        events "passed", "skipped", "failed"
        exceptionFormat "full"
    }
}

// Task riêng cho integration test
tasks.register('integrationTest', Test) {
    useJUnitPlatform {
        includeTags 'integration'
    }
    shouldRunAfter tasks.named('test')
}

// Build phải qua cả unit test và integration test
tasks.named('check') {
    dependsOn 'integrationTest'
}
```

#### Lợi ích của tự động hóa:

1. **Phát hiện lỗi sớm**: Mỗi commit đều được test tự động.
2. **Giảm phụ thuộc kiểm thử thủ công**: Developer không cần chạy test manually.
3. **Tăng tốc phát triển**: Feedback nhanh, sửa lỗi sớm, giảm chi phí sửa bug.
4. **Đảm bảo tính nhất quán**: Tất cả developer đều phải tuân thủ cùng tiêu chuẩn test.
5. **Ngăn regression**: Test cũ vẫn chạy khi thêm tính năng mới.

---

## Tổng kết chiến lược

| Thành phần | Chiến lược | Công cụ |
|---|---|---|
| Logic nghiệp vụ | Unit Test + Mock | JUnit 5, Mockito, AssertJ |
| Database layer | Integration Test | @DataJpaTest, H2, Testcontainers |
| API endpoints | Controller Test + API Test | @WebMvcTest, REST Assured |
| Service-to-service | Contract Test + WireMock | Spring Cloud Contract, WireMock |
| Luồng nghiệp vụ | E2E Test | REST Assured, Selenium |
| Coverage | Đo và enforce ngưỡng | JaCoCo (Line ≥ 80%, Branch ≥ 70%) |
| CI/CD | Tự động chạy trong pipeline | Gradle + GitHub Actions / Jenkins |
