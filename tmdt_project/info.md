Bản phân tích toàn diện về dự án của bạn.

### 1. Project Overview

*   **Core Purpose**: Nền tảng này là một thị trường lao động (marketplace) được thiết kế để kết nối hai nhóm người dùng chính: **Nhà tuyển dụng (Employer)** có nhu cầu tuyển dụng lao động phổ thông cho các công việc thời vụ hoặc dự án ngắn hạn, và **Người lao động (Freelancer)** tìm kiếm các công việc này. Hệ thống quản lý toàn bộ quy trình từ đăng tin, ứng tuyển, giao kết hợp đồng, thực hiện công việc theo các mốc (milestones), cho đến thanh toán.
*   **Completion State**: Dự án đã hoàn thành các luồng nghiệp vụ cốt lõi. Các module chính như Quản lý người dùng, Đăng tin tuyển dụng, Nộp hồ sơ (Proposal), Gửi đề nghị (Offer), và Quản lý hợp đồng (Contract) đã được triển khai và hoạt động. Các module phụ trợ như Chat, Thanh toán qua cổng thanh toán, và Giải quyết tranh chấp vẫn đang trong giai-đoạn-lên-kế-hoạch hoặc chưa hoàn thiện.

### 2. Tech Stack & Architecture

*   **Primary Technologies**:
    *   **Backend**: Java 17, Spring Boot 3.
    *   **Frontend**: React, TypeScript, Vite.
    *   **Database**: MongoDB được sử dụng làm cơ sở dữ liệu chính cho các microservices.
    *   **Build & Dependency Management**: Maven cho backend, npm cho frontend.
    *   **Infrastructure**: Docker và Docker Compose để quản lý các dịch vụ hạ tầng cục bộ (MongoDB, Redis, Kafka).

*   **Microservices Architecture**:
    *   Hệ thống được cấu trúc theo mô hình microservices, với mỗi domain nghiệp vụ được tách thành một service riêng biệt (ví dụ: `user-service`, `job-service`, `contract-service`).
    *   Mỗi service là một ứng dụng Spring Boot độc lập, có cơ sở dữ liệu riêng (mặc dù hiện tại có thể đang dùng chung một instance MongoDB nhưng khác database/collection).

*   **Cross-Service Communication**:
    *   **Current State**: Giao tiếp giữa các service chủ yếu được thực hiện thông qua các lời gọi REST API trực tiếp. Hiện tại, **chưa có API Gateway** tập trung được triển khai; các service gọi thẳng đến endpoint của nhau.
    *   **Asynchronous Communication**: Mặc dù Kafka đã được thiết lập trong `docker-compose.yml`, việc sử dụng nó cho các quy trình bất đồng bộ (ví dụ: tạo thông báo, cập nhật trạng thái) chưa được thể hiện rõ trong các module đã hoàn thiện.
    *   **Distributed Transactions**: Hiện tại **chưa áp dụng Saga pattern** hoặc các cơ chế two-phase commit. Các giao dịch được xử lý trong phạm vi của từng service. Ví dụ, khi một "Offer" được chấp nhận, `contract-service` sẽ được gọi để tạo "Contract", nhưng đây là một chuỗi các lời gọi API tuần tự, chưa có cơ chế rollback tự động phức tạp giữa các service.

### 3. Directory Structure

Đây là cấu trúc thư mục cấp cao của workspace, tập trung vào backend:

```
g:\tmdt_project/
├── backend/
│   ├── pom.xml             # Parent POM cho tất cả module backend
│   ├── infra/
│   │   └── docker-compose.yml # Định nghĩa local infra (DBs, message broker)
│   ├── libs/
│   │   └── common/         # Chứa code dùng chung (exception handlers, base DTOs)
│   ├── services/
│   │   ├── user-service/     # Quản lý User, Profile, Auth
│   │   ├── job-service/      # Quản lý Job Postings
│   │   ├── contract-service/ # Quản lý Contract, Milestone, Offer, Proposal
│   │   ├── chat-service/     # (Chưa hoàn thiện)
│   │   ├── payment-service/  # (Chưa hoàn thiện)
│   │   └── ...
│   └── doc/                  # Tài liệu phân tích, thiết kế, và tóm tắt
└── frontend/
    └── app/                  # Source code của ứng dụng React
```

*   `infra`: Chứa các tệp cấu hình để chạy môi trường phát triển cục bộ. `docker-compose.yml` là trung tâm, định nghĩa các container cho MongoDB, Redis, và Kafka.
*   `libs/common`: Một module Maven chứa các lớp tiện ích, cấu hình bảo mật cơ bản, và các trình xử lý ngoại lệ (exception handlers) được chia sẻ giữa các microservices để tránh lặp code.
*   `services`: Thư mục quan trọng nhất, chứa mã nguồn cho từng microservice. Mỗi thư mục con là một dự án Spring Boot độc lập, tuân thủ kiến trúc layer (controller, service, repository, domain).

### 4. Core Modules Deep-Dive

*   **Authentication Module (`user-service`)**:
    *   **Flow**: Người dùng đăng ký và đăng nhập thông qua các endpoint trong `user-service`. Sau khi xác thực thành công, service sẽ cấp phát một cặp JSON Web Tokens (JWT): một **Access Token** (thời gian sống ngắn, ví dụ 1 giờ) và một **Refresh Token** (thời gian sống dài, ví dụ 7 ngày). Access Token được sử dụng để xác thực các yêu cầu API tiếp theo. Refresh Token được dùng để lấy Access Token mới mà không cần đăng nhập lại.
    *   **Technologies**: Spring Security được sử dụng để quản lý quá trình xác thực và phân quyền. JWT được tạo và xác minh trong service này.
    *   **Role Management**: Hệ thống định nghĩa các vai trò (roles) như `EMPLOYER`, `FREELANCER`, `ADMIN`. Spring Security sử dụng các vai trò này để bảo vệ các endpoint, đảm bảo chỉ người dùng có quyền phù hợp mới có thể truy cập các tài nguyên nhất định.

*   **Job Posting Management (`job-service`)**:
    *   **Entities & Relationships**:
        *   `Job`: Thực thể trung tâm, chứa các thông tin như `title`, `description`, `budget`, `status` (`OPEN`, `IN_PROGRESS`, `COMPLETED`), và `employerId`.
        *   `Proposal`: Đại diện cho một hồ sơ ứng tuyển của `Freelancer` cho một `Job`. Nó có mối quan hệ nhiều-một với `Job` (một `Job` có thể có nhiều `Proposal`).
        *   `Offer`: Khi `Employer` chọn một `Proposal`, họ sẽ tạo ra một `Offer`. `Offer` có mối quan hệ một-một với `Proposal` được chọn.
    *   **API Flow**:
        1.  `POST /jobs`: `Employer` tạo một tin tuyển dụng mới.
        2.  `GET /jobs`: Bất kỳ ai (cả đã xác thực và chưa) cũng có thể xem danh sách các công việc đang `OPEN`.
        3.  `GET /jobs/{jobId}`: Xem chi tiết một công việc.
        4.  `POST /jobs/{jobId}/proposals`: `Freelancer` nộp hồ sơ ứng tuyển.
        5.  `GET /jobs/{jobId}/proposals`: `Employer` xem danh sách các hồ sơ đã ứng tuyển vào công việc của mình.
        6.  `POST /proposals/{proposalId}/offers`: `Employer` gửi một đề nghị làm việc cho `Freelancer` đã được chọn.

*   **Other Modules**:
    *   **Contract Module (`contract-service`)**: Xử lý logic sau khi một `Offer` được `Freelancer` chấp nhận. Nó tự động tạo ra một `Contract` và các `Milestones` ban đầu. Module này quản lý toàn bộ vòng đời của hợp đồng, từ khi bắt đầu, thực hiện, cho đến khi hoàn thành hoặc bị hủy.

### 5. Database Schema (High-Level)

Dựa trên các lớp Entity trong mã nguồn, đây là cấu trúc dữ liệu chính (ánh xạ từ MongoDB documents):

*   `users`: Lưu thông tin đăng nhập (`email`, `password`, `role`, `status`).
*   `profiles`: Lưu thông tin chi tiết của người dùng (`full_name`, `avatar_url`, `skills`, `company_name`). (Quan hệ 1-1 với `users`).
*   `jobs`: Lưu thông tin các tin đăng tuyển dụng (`title`, `description`, `budget`, `status`, `employerId`).
*   `proposals`: Lưu các hồ sơ ứng tuyển (`jobId`, `freelancerId`, `coverLetter`, `bidAmount`, `status`). (UNIQUE constraint trên `jobId` và `freelancerId`).
*   `offers`: Lưu các đề nghị công việc (`jobId`, `proposalId`, `employerId`, `freelancerId`, `status`).
*   `contracts`: Lưu các hợp đồng đã được ký kết (`offerId`, `jobId`, `employerId`, `freelancerId`, `totalValue`, `status`). (Quan hệ 1-1 với `offers`).
*   `milestones`: Các mốc thanh toán/công việc trong một hợp đồng (`contractId`, `amount`, `dueDate`, `status`).
*   `deliverables`: Sản phẩm bàn giao cho một milestone (`milestoneId`, `file_url`, `description`).

### 6. Coding Conventions & Standards

*   **Layered Architecture**: Mã nguồn trong mỗi service được phân tách rõ ràng thành các lớp:
    *   `@RestController`: Chỉ chịu trách nhiệm xử lý HTTP request/response và validation DTO.
    *   `@Service`: Chứa toàn bộ business logic, không phụ thuộc vào HTTP.
    *   `@Repository`: Lớp truy cập dữ liệu, sử dụng Spring Data MongoDB.
*   **Dependency Injection**: Sử dụng **Constructor Injection** một cách nhất quán để tiêm các dependency, giúp mã nguồn dễ đọc và dễ dàng cho việc viết unit test.
*   **DTO Pattern**: Sử dụng các đối tượng truyền dữ liệu (Data Transfer Objects) để tách biệt giữa API layer và domain model. Điều này giúp API linh hoạt và tránh lộ chi tiết của database schema.
*   **Centralized Exception Handling**: Sử dụng `@ControllerAdvice` để xử lý các ngoại lệ một cách tập trung, trả về các thông báo lỗi nhất quán dưới dạng JSON.
*   **Configuration Management**: Cấu hình được quản lý thông qua các tệp `application.properties` hoặc `application.yml`, với các giá trị nhạy cảm (secrets) được khuyến khích đặt trong biến môi trường hoặc tệp `.env` (đã được `.gitignore`).