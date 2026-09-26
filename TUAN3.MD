# TIẾN ĐỘ TUẦN 3: THIẾT KẾ CƠ SỞ DỮ LIỆU QUAN HỆ

> **Đề tài:** Xây dựng ứng dụng web quản lý và chuẩn hóa hồ sơ tác giả, công bố khoa học trên Scopus phục vụ Trường Đại học Công nghệ thông tin và Truyền thông (ICTU).
> **Tuần thực hiện:** Tuần 3 — Nhiệm vụ 1: Thiết kế CSDL quan hệ (ERD + Data Dictionary + SQL DDL).
> **Căn cứ:** Đặc tả FR/NFR Tuần 2 (`BAO_CAO_TIEN_DO_TUAN_2.md`); Kết quả khảo sát 660 bản ghi Tuần 1 (`scopus-ictu-data-survey.md`); Chuẩn PostgreSQL 15.
> **Quy ước đặt tên:** `snake_case` cho toàn bộ tên bảng và cột; khóa chính (PK) là cột `id` (BIGSERIAL); khóa ngoại (FK) có hậu tố `_id`; tất cả bảng đặt trong schema `scopus_ictu`.
> **Người soạn:** Trương Quang Huy — SV ngành CNTT, ICTU.

---

## 1. TỔNG QUAN THIẾT KẾ CƠ SỞ DỮ LIỆU

Hệ thống Scopus ICTU cần lưu trữ toàn bộ dữ liệu từ khi nhập tệp CSV thô (22 trường theo khảo sát), qua các giai đoạn xử lý pipeline (parse, chuẩn hoá, Match Score), đến khi Admin phê duyệt và kết xuất báo cáo. Bên cạnh dữ liệu nghiệp vụ, hệ thống còn phải đảm bảo **khả năng kiểm toán bất biến** (NFR-02) và **lưu vết truy cập trái phép** (NFR-04). Từ đó, CSDL được thiết kế gồm **10 bảng** chia thành ba nhóm chức năng.

### 1.1. Phân nhóm bảng

| Nhóm | Bảng | Mục đích |
|---|---|---|
| **Quản trị & Bảo mật** | `users`, `upload_batches` | Quản lý tài khoản, phân quyền RBAC (NFR-04); theo dõi lô nhập dữ liệu |
| **Nghiệp vụ chính** | `raw_scopus_data`, `lecturers`, `author_extracted`, `approval_queue`, `standardized_data` | Pipeline 4 giai đoạn: nhập → bóc tách → phê duyệt → chuẩn hoá |
| **Kiểm toán & Truy vết** | `audit_log`, `export_audit_log`, `security_audit_log` | Lưu vết bất biến theo NFR-02 và NFR-04 |

### 1.2. Nguyên tắc thiết kế chính

- **Idempotency (NFR-01):** Bảng `raw_scopus_data` có cột `eid` UNIQUE NOT NULL — đảm bảo re-import cùng một tệp CSV không sinh bản ghi mới; 660/660 bản ghi thực tế đều có EID đầy đủ theo khảo sát Tuần 1.
- **Khóa định danh chính xác:** Bảng `lecturers` có cột `scopus_author_id` UNIQUE — khớp với 100% Author ID có trong dữ liệu thực tế.
- **Lưu vết bất biến (NFR-02):** Bảng `audit_log` được thiết kế chỉ-INSERT; quyền UPDATE/DELETE bị revoke trên role `app_backend` của PostgreSQL.
- **Phục vụ rollback (FR-20):** Cột `approved_at` ở `approval_queue` và `standardized_data.is_invalidated` hỗ trợ quy tắc rollback ≤ 72 giờ.
- **Cờ cảnh báo dữ liệu thực tế:** Hai cờ `is_retracted`, `is_erratum` ở `approval_queue` để cảnh báo 4 bản ghi đặc biệt (2 Retracted + 2 Erratum theo khảo sát Tuần 1) — không cho phép Phê duyệt.
- **Hỗ trợ fuzzy match (FR-11):** Index GIN kết hợp `pg_trgm` trên `lecturers.full_name_no_diacritics` để tăng tốc RapidFuzz.

---

## 2. SƠ ĐỒ ERD (ENTITY-RELATIONSHIP DIAGRAM)

Sơ đồ dưới đây mô tả **10 bảng** với các quan hệ một–nhiều (`||--o{`) và nhiều–một tùy chọn (`}o--o|`), được vẽ bằng cú pháp Mermaid `erDiagram` để nhúng trực tiếp vào tài liệu.

**[Chèn Hình 3.1: Sơ đồ ERD — Hệ thống Chuẩn hóa Hồ sơ Scopus ICTU]**

```mermaid
erDiagram
    raw_scopus_data ||--o{ author_extracted : "1 bài → N tác giả bóc tách"
    raw_scopus_data ||--o{ approval_queue : "1 bài → N dòng queue"
    author_extracted }o--|| lecturers : "N tác giả → 1 ứng viên GV"
    approval_queue   }o--o| lecturers : "N queue → 0..1 ứng viên gợi ý"
    approval_queue   ||--o| standardized_data : "1 queue → 0..1 dòng chuẩn hoá"
    standardized_data }o--|| lecturers : "N chuẩn → 1 GV ICTU chính thức"
    standardized_data ||--o{ audit_log : "1 chuẩn → N vết kiểm toán"
    export_audit_log }o--o| users : "0..1 người xuất"

    raw_scopus_data {
        bigserial id PK
        text eid UK
        text title
        text authors
        text author_ids
        smallint year
        integer cited_by
        text doi
        text publisher
        text document_type
        boolean duplicate
        text_array warning_reasons
        varchar processing_status
    }

    lecturers {
        bigserial id PK
        varchar lecturer_code UK
        text full_name
        text full_name_no_diacritics
        text scopus_author_id UK
        text affiliation
        varchar faculty
        boolean is_active
    }

    author_extracted {
        bigserial id PK
        bigint raw_id FK
        smallint author_index
        text normalized_name
        bigint lecturer_id FK
        numeric match_score
        jsonb evidence_payload
        boolean is_common_name
    }

    approval_queue {
        bigserial id PK
        bigint raw_id FK
        bigint author_extracted_id FK
        bigint lecturer_id FK
        varchar status
        numeric match_score
        jsonb evidence_payload
        boolean is_retracted
        boolean is_erratum
        timestamp approved_at
    }

    standardized_data {
        bigserial id PK
        bigint approval_queue_id FK_UK
        bigint lecturer_id FK
        numeric match_score
        jsonb evidence_payload
        timestamp approved_at
        boolean is_invalidated
        timestamp invalidated_at
    }

    audit_log {
        bigserial id PK
        varchar action
        bigint actor_id FK
        varchar target_table
        bigint target_id
        jsonb before_json
        jsonb after_json
        uuid correlation_id
        timestamp created_at
    }

    export_audit_log {
        bigserial id PK
        varchar file_format
        jsonb filter_criteria_json
        integer row_count
        char file_hash_sha256
        bigint actor_id FK
    }

    security_audit_log {
        bigserial id PK
        bigint actor_id FK
        varchar attempted_endpoint
        varchar required_permission
        smallint http_status
        inet ip_address
    }

    users {
        bigserial id PK
        varchar username UK
        varchar password_hash
        varchar role
        boolean is_active
    }

    upload_batches {
        bigserial id PK
        text file_name
        bigint operator_id FK
        integer total_rows
        uuid correlation_id UK
        varchar processing_status
    }
```

### 2.1. Thuyết minh các quan hệ chính

| Quan hệ | Loại | Ý nghĩa nghiệp vụ |
|---|---|---|
| `raw_scopus_data → author_extracted` | Một–Nhiều | Một bài báo có nhiều tác giả (trung bình 3–5, có bài ≥ 14 theo khảo sát) |
| `raw_scopus_data → approval_queue` | Một–Nhiều | Mỗi bài sinh nhiều dòng trong hàng đợi duyệt (1 dòng / 1 tác giả) |
| `author_extracted → lecturers` | Nhiều–Một (tùy chọn) | Nhiều tác giả Scopus có thể ánh xạ về 0 hoặc 1 GV ICTU |
| `approval_queue → lecturers` | Nhiều–Một (tùy chọn) | Queue có thể chưa gán được GV nào khi score < 50% |
| `approval_queue → standardized_data` | Một–Một (tùy chọn) | Mỗi queue sau khi duyệt sinh tối đa 1 dòng chuẩn hoá |
| `standardized_data → audit_log` | Một–Nhiều | Một bản ghi chuẩn có thể có nhiều vết kiểm toán (APPROVE, ROLLBACK...) |

---

## 3. TỪ ĐIỂN DỮ LIỆU (DATA DICTIONARY)

Từ điển dữ liệu dưới đây mô tả chi tiết **10 bảng** gồm: tên trường, kiểu PostgreSQL, khóa chính (PK), khóa ngoại (FK) và ý nghĩa nghiệp vụ.

### 3.1. Bảng `users` — Tài khoản người dùng

| Cột | Kiểu PostgreSQL | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính tự tăng |
| `username` | `VARCHAR(64)` | UNIQUE, NOT NULL | Tên đăng nhập |
| `password_hash` | `VARCHAR(255)` | NOT NULL | Mật khẩu đã hash (bcrypt/argon2) |
| `role` | `VARCHAR(16)` | NOT NULL, CHECK ∈ {'operator','admin'} | Phân quyền theo NFR-04 |
| `is_active` | `BOOLEAN` | NOT NULL DEFAULT TRUE | Tài khoản còn hiệu lực |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT NOW() | Thời điểm tạo |
| `last_login` | `TIMESTAMPTZ` | NULL | Lần đăng nhập gần nhất |

### 3.2. Bảng `upload_batches` — Lô nhập dữ liệu

| Cột | Kiểu PostgreSQL | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `file_name` | `TEXT` | NOT NULL | Tên tệp CSV (ví dụ `Truong_Quang_Huy_Scopus_Sep2026.csv`) |
| `operator_id` | `BIGINT` | FK → `users.id`, NOT NULL | Người upload |
| `uploaded_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT NOW() | Thời điểm upload |
| `total_rows`, `new_rows`, `duplicate_rows`, `warning_rows` | `INTEGER` | NOT NULL DEFAULT 0 | Số liệu tổng hợp lô |
| `correlation_id` | `UUID` | UNIQUE, NOT NULL | Mã liên kết log pipeline |
| `processing_status` | `VARCHAR(16)` | CHECK ∈ {PENDING, RUNNING, COMPLETED, FAILED} | Trạng thái xử lý |

### 3.3. Bảng `raw_scopus_data` — Dữ liệu CSV gốc 22 cột

| Cột | Kiểu PostgreSQL | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `eid` | `TEXT` | **UNIQUE NOT NULL** | Mã EID Scopus — khóa Idempotency theo NFR-01 |
| `title` | `TEXT` | NOT NULL | Tiêu đề bài báo |
| `authors`, `author_full_names`, `author_ids` | `TEXT` | NOT NULL | Chuỗi tác giả và chuỗi Scopus Author ID phân cách `;` |
| `year` | `SMALLINT` | NOT NULL, CHECK (year BETWEEN 1900 AND 2100) | Năm công bố (2008–2027 theo khảo sát) |
| `cited_by` | `INTEGER` | NOT NULL DEFAULT 0 | Tổng lượt trích dẫn (tổng 4 885 theo khảo sát) |
| `doi` | `TEXT` | NULL | DOI — khuyết 25/660 bản ghi |
| `publisher` | `TEXT` | NULL | Nhà xuất bản — khuyết 22/660 bản ghi |
| `document_type` | `TEXT` | NULL | Conference paper / Article / Retracted / Erratum |
| `open_access` | `TEXT` | NULL | Cờ OA — khuyết 487/660 bản ghi |
| `batch_id` | `BIGINT` | FK → `upload_batches.id`, NOT NULL | Lô nhập |
| `operator_id` | `BIGINT` | FK → `users.id`, NOT NULL | Người upload |
| `uploaded_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT NOW() | Thời điểm insert |
| `duplicate` | `BOOLEAN` | NOT NULL DEFAULT FALSE | Cờ trùng EID theo NFR-01 |
| `warning_reasons` | `TEXT[]` | NULL | Mảng lý do cảnh báo: MISSING_DOI, MISSING_YEAR... |
| `processing_status` | `VARCHAR(16)` | DEFAULT 'PENDING' | Trạng thái pipeline |

### 3.4. Bảng `lecturers` — Danh mục Giảng viên ICTU

| Cột | Kiểu PostgreSQL | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `lecturer_code` | `VARCHAR(16)` | UNIQUE, NOT NULL | Mã GV ICTU (L001, L002...) |
| `full_name` | `TEXT` | NOT NULL | Họ tên đầy đủ có dấu |
| `full_name_no_diacritics` | `TEXT` | NOT NULL | Họ tên không dấu — khóa fuzzy match (FR-08) |
| `scopus_author_id` | `TEXT` | UNIQUE, NULL | Scopus Author ID — khóa match chính xác (FR-10) |
| `affiliation`, `department`, `faculty` | `TEXT/VARCHAR` | NULL | Đơn vị công tác |
| `academic_rank`, `email` | `VARCHAR` | NULL | Học hàm / email |
| `is_active` | `BOOLEAN` | NOT NULL DEFAULT TRUE | Còn công tác hay đã nghỉ/chuyển |
| `created_at`, `updated_at` | `TIMESTAMPTZ` | DEFAULT NOW() | Theo dõi thời gian |

### 3.5. Bảng `author_extracted` — Tác giả bóc tách từ mỗi bài

| Cột | Kiểu PostgreSQL | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `raw_id` | `BIGINT` | FK → `raw_scopus_data.id`, NOT NULL | Bài báo gốc |
| `author_index` | `SMALLINT` | NOT NULL | Thứ tự tác giả trong bài (1, 2, 3...) |
| `lastname_initials` | `VARCHAR(64)` | NOT NULL | Họ + chữ cái đầu tên, ví dụ `Nguyen T.-T.` |
| `normalized_name` | `TEXT` | NOT NULL | Tên không dấu lower — khóa so khớp (FR-08) |
| `scopus_author_id` | `BIGINT` | NULL | ID Scopus sau parse (FR-07) |
| `lecturer_id` | `BIGINT` | FK → `lecturers.id`, NULL | GV ICTU ứng viên sau khi match |
| `match_score` | `NUMERIC(4,3)` | CHECK 0 ≤ score ≤ 1 | Match Score ∈ [0,1] (FR-12) |
| `evidence_payload` | `JSONB` | NULL | Căn cứ đối chiếu: `{author_id_match, name_score, affil_score, common_name_flag}` |
| `is_common_name` | `BOOLEAN` | NOT NULL DEFAULT FALSE | Cờ cảnh báo tên phổ biến (Nguyen/Tran/Le) |

### 3.6. Bảng `approval_queue` — Hàng đợi phê duyệt

| Cột | Kiểu PostgreSQL | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `raw_id` | `BIGINT` | FK → `raw_scopus_data.id`, NOT NULL | Bài báo |
| `author_extracted_id` | `BIGINT` | FK → `author_extracted.id`, NOT NULL | Tác giả bóc tách |
| `lecturer_id` | `BIGINT` | FK → `lecturers.id`, NULL | GV ICTU gợi ý |
| `status` | `VARCHAR(16)` | NOT NULL, CHECK ∈ {MATCH_HIGH, MATCH_MEDIUM, PENDING_VERIFY, APPROVED, PENDING} | Trạng thái duyệt (FR-13) |
| `match_score` | `NUMERIC(4,3)` | NOT NULL | Bản sao score (FR-12) |
| `evidence_payload` | `JSONB` | NOT NULL | Bằng chứng đính kèm Diff View (FR-16) |
| `is_retracted` | `BOOLEAN` | NOT NULL DEFAULT FALSE | Cờ Retracted (4/660 bản ghi theo khảo sát) |
| `is_erratum` | `BOOLEAN` | NOT NULL DEFAULT FALSE | Cờ Erratum |
| `approved_at` | `TIMESTAMPTZ` | NULL | Thời điểm duyệt — dùng cho rule 72h rollback (FR-20) |
| `approved_by` | `BIGINT` | FK → `users.id`, NULL | Admin duyệt |

### 3.7. Bảng `standardized_data` — Dữ liệu chuẩn hoá sau phê duyệt

| Cột | Kiểu PostgreSQL | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `approval_queue_id` | `BIGINT` | FK → `approval_queue.id`, **UNIQUE**, NOT NULL | Mỗi queue tối đa 1 dòng chuẩn |
| `lecturer_id` | `BIGINT` | FK → `lecturers.id`, NOT NULL | GV ICTU chính thức sau duyệt |
| `raw_id` | `BIGINT` | FK → `raw_scopus_data.id`, NOT NULL | Bài báo gốc |
| `match_score` | `NUMERIC(4,3)` | NOT NULL | Score tại thời điểm duyệt |
| `evidence_payload` | `JSONB` | NOT NULL | Lưu vết evidence (FR-21) |
| `approved_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT NOW() | Thời điểm duyệt |
| `approved_by` | `BIGINT` | FK → `users.id`, NOT NULL | Admin duyệt |
| `is_invalidated` | `BOOLEAN` | NOT NULL DEFAULT FALSE | Cờ rollback (FR-20) |
| `invalidated_at` | `TIMESTAMPTZ` | NULL | Thời điểm rollback |
| `invalidated_by` | `BIGINT` | FK → `users.id`, NULL | Admin rollback |
| `invalidation_reason` | `TEXT` | NULL | Lý do rollback (bắt buộc khi ROLLBACK) |

### 3.8. Bảng `audit_log` — Nhật ký kiểm toán (bất biến theo NFR-02)

| Cột | Kiểu PostgreSQL | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `action` | `VARCHAR(32)` | NOT NULL, CHECK ∈ {APPROVE, MANUAL_OVERRIDE, DEFER, ROLLBACK, LECTURER_CRUD} | Loại thao tác (FR-21) |
| `actor_id` | `BIGINT` | FK → `users.id`, NOT NULL | Người thực hiện |
| `target_table` | `VARCHAR(32)` | NOT NULL | Bảng bị tác động |
| `target_id` | `BIGINT` | NOT NULL | Khóa chính bản ghi bị tác động |
| `before_json`, `after_json` | `JSONB` | NULL | Trạng thái trước/sau |
| `reason` | `TEXT` | NULL | Lý do (bắt buộc cho ROLLBACK theo UC-04) |
| `correlation_id` | `UUID` | NOT NULL | Liên kết pipeline (ST-13) |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT NOW() | Thời điểm ghi — **không UPDATE/DELETE sau khi ghi** |

### 3.9. Bảng `export_audit_log` — Lưu vết xuất báo cáo

| Cột | Kiểu PostgreSQL | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `file_format` | `VARCHAR(8)` | NOT NULL, CHECK ∈ {xlsx, csv, json} | Định dạng xuất (FR-26/27/28) |
| `filter_criteria_json` | `JSONB` | NOT NULL | Bộ lọc đã dùng: `{year_from, year_to, faculty, publisher_family, status}` |
| `row_count` | `INTEGER` | NOT NULL, CHECK ≥ 0 | Số dòng dữ liệu xuất |
| `file_hash_sha256` | `CHAR(64)` | NOT NULL | Hash SHA-256 hex 64 ký tự (FR-29) |
| `file_name` | `TEXT` | NOT NULL | Tên file xuất |
| `actor_id` | `BIGINT` | FK → `users.id`, NOT NULL | Người xuất |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT NOW() | Thời điểm xuất |

### 3.10. Bảng `security_audit_log` — Lưu vết truy cập trái phép (NFR-04)

| Cột | Kiểu PostgreSQL | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `actor_id` | `BIGINT` | FK → `users.id`, NULL | Người dùng (NULL nếu chưa đăng nhập) |
| `attempted_endpoint` | `VARCHAR(255)` | NOT NULL | API/endpoint bị gọi trái phép |
| `required_permission` | `VARCHAR(64)` | NOT NULL | Quyền cần có (ví dụ `admin.rollback`) |
| `http_status` | `SMALLINT` | NOT NULL | Mã HTTP trả về (403) |
| `ip_address` | `INET` | NULL | IP client |
| `user_agent` | `TEXT` | NULL | Trình duyệt / client |
| `payload_json` | `JSONB` | NULL | Payload request |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT NOW() | Thời điểm |

---

## 4. MÃ NGUỒN SQL DDL (POSTGRESQL 15)

Script dưới đây được thiết kế để chạy migration từ đầu trên PostgreSQL 15+. Thứ tự tạo bảng tuân theo quan hệ khóa ngoại: bảng không phụ thuộc tạo trước.

### 4.1. Khởi tạo Schema và Extension

```sql
CREATE SCHEMA IF NOT EXISTS scopus_ictu;
SET search_path TO scopus_ictu, public;

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pg_trgm;     -- phục vụ FR-11 Fuzzy Match
```

### 4.2. Bảng Users và Upload Batches

```sql
CREATE TABLE scopus_ictu.users (
    id              BIGSERIAL    PRIMARY KEY,
    username        VARCHAR(64)  NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    role            VARCHAR(16)  NOT NULL
                    CHECK (role IN ('operator', 'admin')),
    is_active       BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    last_login      TIMESTAMPTZ
);

CREATE INDEX idx_users_role ON scopus_ictu.users(role);

CREATE TABLE scopus_ictu.upload_batches (
    id                  BIGSERIAL    PRIMARY KEY,
    file_name           TEXT         NOT NULL,
    operator_id         BIGINT       NOT NULL
                        REFERENCES scopus_ictu.users(id)
                        ON DELETE RESTRICT,
    uploaded_at         TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    total_rows          INTEGER      NOT NULL DEFAULT 0,
    new_rows            INTEGER      NOT NULL DEFAULT 0,
    duplicate_rows      INTEGER      NOT NULL DEFAULT 0,
    warning_rows        INTEGER      NOT NULL DEFAULT 0,
    correlation_id      UUID         NOT NULL UNIQUE
                        DEFAULT uuid_generate_v4(),
    processing_status   VARCHAR(16)  NOT NULL DEFAULT 'PENDING'
                        CHECK (processing_status IN
                               ('PENDING','RUNNING','COMPLETED','FAILED'))
);

CREATE INDEX idx_upload_batches_status
    ON scopus_ictu.upload_batches(processing_status, uploaded_at DESC);
```

### 4.3. Bảng Lecturers và Raw Scopus Data

```sql
CREATE TABLE scopus_ictu.lecturers (
    id                          BIGSERIAL    PRIMARY KEY,
    lecturer_code               VARCHAR(16)  NOT NULL UNIQUE,
    full_name                   TEXT         NOT NULL,
    full_name_no_diacritics     TEXT         NOT NULL,
    scopus_author_id            TEXT         UNIQUE,
    affiliation                 TEXT,
    department                  VARCHAR(64),
    faculty                     VARCHAR(64),
    academic_rank               VARCHAR(32),
    email                       VARCHAR(128),
    is_active                   BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at                  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at                  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_lecturers_name_trgm
    ON scopus_ictu.lecturers
    USING gin (full_name_no_diacritics gin_trgm_ops);

CREATE INDEX idx_lecturers_faculty_dept
    ON scopus_ictu.lecturers(faculty, department);

CREATE TABLE scopus_ictu.raw_scopus_data (
    id                  BIGSERIAL    PRIMARY KEY,
    eid                 TEXT         NOT NULL UNIQUE,
    title               TEXT         NOT NULL,
    authors             TEXT         NOT NULL,
    author_full_names   TEXT         NOT NULL,
    author_ids          TEXT         NOT NULL,
    year                SMALLINT     NOT NULL
                        CHECK (year BETWEEN 1900 AND 2100),
    source_title        TEXT,
    volume              TEXT,
    issue               TEXT,
    art_no              TEXT,
    page_start          TEXT,
    page_end            TEXT,
    cited_by            INTEGER      NOT NULL DEFAULT 0,
    doi                 TEXT,
    link                TEXT,
    abstract            TEXT,
    author_keywords     TEXT,
    publisher           TEXT,
    document_type       TEXT,
    publication_stage   TEXT,
    open_access         TEXT,
    source              TEXT,
    batch_id            BIGINT       NOT NULL
                        REFERENCES scopus_ictu.upload_batches(id)
                        ON DELETE RESTRICT,
    operator_id         BIGINT       NOT NULL
                        REFERENCES scopus_ictu.users(id)
                        ON DELETE RESTRICT,
    uploaded_at         TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    duplicate           BOOLEAN      NOT NULL DEFAULT FALSE,
    warning_reasons     TEXT[],
    processing_status   VARCHAR(16)  NOT NULL DEFAULT 'PENDING'
                        CHECK (processing_status IN
                               ('PENDING','PROCESSING','PROCESSED','FAILED'))
);

CREATE INDEX idx_raw_year_doctype
    ON scopus_ictu.raw_scopus_data(year, document_type);
CREATE INDEX idx_raw_batch
    ON scopus_ictu.raw_scopus_data(batch_id);
CREATE INDEX idx_raw_processing
    ON scopus_ictu.raw_scopus_data(processing_status)
    WHERE processing_status <> 'PROCESSED';
```

### 4.4. Bảng Author Extracted và Approval Queue

```sql
CREATE TABLE scopus_ictu.author_extracted (
    id                  BIGSERIAL      PRIMARY KEY,
    raw_id              BIGINT         NOT NULL
                        REFERENCES scopus_ictu.raw_scopus_data(id)
                        ON DELETE CASCADE,
    author_index        SMALLINT       NOT NULL,
    lastname_initials   VARCHAR(64)    NOT NULL,
    normalized_name     TEXT           NOT NULL,
    scopus_author_id    BIGINT,
    lecturer_id         BIGINT
                        REFERENCES scopus_ictu.lecturers(id)
                        ON DELETE SET NULL,
    match_score         NUMERIC(4,3)   CHECK (match_score BETWEEN 0 AND 1),
    evidence_payload    JSONB,
    is_common_name      BOOLEAN        NOT NULL DEFAULT FALSE,
    UNIQUE (raw_id, author_index)
);

CREATE INDEX idx_author_ext_lecturer
    ON scopus_ictu.author_extracted(lecturer_id);
CREATE INDEX idx_author_ext_score
    ON scopus_ictu.author_extracted(match_score DESC);

CREATE TABLE scopus_ictu.approval_queue (
    id                  BIGSERIAL      PRIMARY KEY,
    raw_id              BIGINT         NOT NULL
                        REFERENCES scopus_ictu.raw_scopus_data(id)
                        ON DELETE CASCADE,
    author_extracted_id BIGINT         NOT NULL
                        REFERENCES scopus_ictu.author_extracted(id)
                        ON DELETE CASCADE,
    lecturer_id         BIGINT
                        REFERENCES scopus_ictu.lecturers(id)
                        ON DELETE SET NULL,
    status              VARCHAR(16)    NOT NULL
                        CHECK (status IN
                               ('MATCH_HIGH','MATCH_MEDIUM',
                                'PENDING_VERIFY','APPROVED','PENDING')),
    match_score         NUMERIC(4,3)   NOT NULL
                        CHECK (match_score BETWEEN 0 AND 1),
    evidence_payload    JSONB          NOT NULL,
    document_type       VARCHAR(64),
    is_retracted        BOOLEAN        NOT NULL DEFAULT FALSE,
    is_erratum          BOOLEAN        NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ    NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ    NOT NULL DEFAULT NOW(),
    approved_at         TIMESTAMPTZ,
    approved_by         BIGINT
                        REFERENCES scopus_ictu.users(id)
                        ON DELETE SET NULL,
    UNIQUE (raw_id, author_extracted_id)
);

CREATE INDEX idx_queue_status_created
    ON scopus_ictu.approval_queue(status, created_at DESC);
CREATE INDEX idx_queue_lecturer
    ON scopus_ictu.approval_queue(lecturer_id);
CREATE INDEX idx_queue_approved_at
    ON scopus_ictu.approval_queue(approved_at)
    WHERE approved_at IS NOT NULL;
```

### 4.5. Bảng Standardized Data và các Audit Log

```sql
CREATE TABLE scopus_ictu.standardized_data (
    id                      BIGSERIAL      PRIMARY KEY,
    approval_queue_id       BIGINT         NOT NULL UNIQUE
                            REFERENCES scopus_ictu.approval_queue(id)
                            ON DELETE RESTRICT,
    lecturer_id             BIGINT         NOT NULL
                            REFERENCES scopus_ictu.lecturers(id)
                            ON DELETE RESTRICT,
    raw_id                  BIGINT         NOT NULL
                            REFERENCES scopus_ictu.raw_scopus_data(id)
                            ON DELETE RESTRICT,
    match_score             NUMERIC(4,3)   NOT NULL
                            CHECK (match_score BETWEEN 0 AND 1),
    evidence_payload        JSONB          NOT NULL,
    approved_at             TIMESTAMPTZ    NOT NULL DEFAULT NOW(),
    approved_by             BIGINT         NOT NULL
                            REFERENCES scopus_ictu.users(id)
                            ON DELETE RESTRICT,
    is_invalidated          BOOLEAN        NOT NULL DEFAULT FALSE,
    invalidated_at          TIMESTAMPTZ,
    invalidated_by          BIGINT
                            REFERENCES scopus_ictu.users(id)
                            ON DELETE SET NULL,
    invalidation_reason     TEXT
);

CREATE INDEX idx_std_lecturer_year
    ON scopus_ictu.standardized_data(lecturer_id, raw_id);
CREATE INDEX idx_std_active
    ON scopus_ictu.standardized_data(is_invalidated, approved_at DESC);

CREATE TABLE scopus_ictu.audit_log (
    id              BIGSERIAL    PRIMARY KEY,
    action          VARCHAR(32)  NOT NULL
                    CHECK (action IN
                           ('APPROVE','MANUAL_OVERRIDE',
                            'DEFER','ROLLBACK','LECTURER_CRUD')),
    actor_id        BIGINT       NOT NULL
                    REFERENCES scopus_ictu.users(id)
                    ON DELETE RESTRICT,
    target_table    VARCHAR(32)  NOT NULL,
    target_id       BIGINT       NOT NULL,
    before_json     JSONB,
    after_json      JSONB,
    reason          TEXT,
    correlation_id  UUID         NOT NULL,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_target
    ON scopus_ictu.audit_log(target_table, target_id);
CREATE INDEX idx_audit_actor_time
    ON scopus_ictu.audit_log(actor_id, created_at DESC);
CREATE INDEX idx_audit_correlation
    ON scopus_ictu.audit_log(correlation_id);

CREATE TABLE scopus_ictu.export_audit_log (
    id                      BIGSERIAL    PRIMARY KEY,
    file_format             VARCHAR(8)   NOT NULL
                            CHECK (file_format IN ('xlsx','csv','json')),
    filter_criteria_json    JSONB        NOT NULL,
    row_count               INTEGER      NOT NULL CHECK (row_count >= 0),
    file_hash_sha256        CHAR(64)     NOT NULL,
    file_name               TEXT         NOT NULL,
    actor_id                BIGINT       NOT NULL
                            REFERENCES scopus_ictu.users(id)
                            ON DELETE RESTRICT,
    created_at              TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_export_actor_time
    ON scopus_ictu.export_audit_log(actor_id, created_at DESC);

CREATE TABLE scopus_ictu.security_audit_log (
    id                      BIGSERIAL    PRIMARY KEY,
    actor_id                BIGINT
                            REFERENCES scopus_ictu.users(id)
                            ON DELETE SET NULL,
    attempted_endpoint      VARCHAR(255) NOT NULL,
    required_permission     VARCHAR(64)  NOT NULL,
    http_status             SMALLINT     NOT NULL,
    ip_address              INET,
    user_agent              TEXT,
    payload_json            JSONB,
    created_at              TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_security_actor_time
    ON scopus_ictu.security_audit_log(actor_id, created_at DESC);
```

### 4.6. Thực thi bất biến Audit Log (NFR-02)

```sql
-- Tạo role ứng dụng (nếu chưa có) — REVOKE UPDATE/DELETE trên audit_log
DO $$
BEGIN
    IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'app_backend') THEN
        CREATE ROLE app_backend LOGIN PASSWORD 'change_me_in_production';
    END IF;
END
$$;

GRANT USAGE ON SCHEMA scopus_ictu TO app_backend;

-- Các bảng nghiệp vụ: SELECT/INSERT/UPDATE
GRANT SELECT, INSERT, UPDATE ON scopus_ictu.users              TO app_backend;
GRANT SELECT, INSERT         ON scopus_ictu.upload_batches    TO app_backend;
GRANT SELECT, INSERT, UPDATE ON scopus_ictu.lecturers         TO app_backend;
GRANT SELECT, INSERT, UPDATE ON scopus_ictu.raw_scopus_data   TO app_backend;
GRANT SELECT, INSERT, UPDATE ON scopus_ictu.author_extracted  TO app_backend;
GRANT SELECT, INSERT, UPDATE ON scopus_ictu.approval_queue    TO app_backend;
GRANT SELECT, INSERT, UPDATE ON scopus_ictu.standardized_data TO app_backend;
GRANT SELECT, INSERT         ON scopus_ictu.export_audit_log  TO app_backend;
GRANT SELECT, INSERT         ON scopus_ictu.security_audit_log TO app_backend;

-- Riêng audit_log: chỉ INSERT + SELECT, KHÔNG UPDATE/DELETE
GRANT SELECT, INSERT ON scopus_ictu.audit_log TO app_backend;
```

### 4.7. View phục vụ báo cáo thống kê (FR-25)

```sql
CREATE OR REPLACE VIEW scopus_ictu.v_yearly_publication AS
SELECT
    EXTRACT(YEAR FROM r.uploaded_at)::SMALLINT AS report_year,
    r.year                                                    AS publication_year,
    COUNT(DISTINCT r.id)                                      AS total_papers,
    SUM(r.cited_by)                                           AS total_citations,
    COUNT(*) FILTER (WHERE r.document_type = 'Retracted')     AS retracted_count,
    COUNT(*) FILTER (WHERE r.document_type = 'Erratum')       AS erratum_count,
    COUNT(*) FILTER (WHERE r.doi IS NULL)                     AS missing_doi_count
FROM scopus_ictu.raw_scopus_data r
WHERE r.duplicate = FALSE
GROUP BY report_year, r.year
ORDER BY report_year DESC, r.year DESC;

COMMENT ON VIEW scopus_ictu.v_yearly_publication IS
    'Thống kê số bài / trích dẫn / cảnh báo theo năm công bố - phục vụ dashboard FR-25';
```

---

## 5. MA TRẬN TRUY VẾT BẢNG ↔ FR ↔ PI

| Bảng | FR đáp ứng | PI đáp ứng | Ghi chú |
|---|---|---|---|
| `users` | FR-01, FR-22 | PI 1.1, PI 3.3 | Nền tảng phân quyền theo NFR-04 |
| `upload_batches` | FR-04, FR-15 | PI 1.1, PI 2.1 | Truy vết nguồn lô nhập |
| `raw_scopus_data` | FR-02, FR-03, FR-04, FR-05, FR-06 | PI 1.1, PI 1.2 | Lưu 22 cột gốc + cờ Idempotency |
| `lecturers` | FR-08, FR-10, FR-11, FR-22 | PI 2.1, PI 3.3 | Master data, khóa match chính xác |
| `author_extracted` | FR-07, FR-08, FR-09, FR-10, FR-11, FR-12, FR-13 | PI 2.1 | Kết quả parse + Match Score + evidence |
| `approval_queue` | FR-13, FR-14, FR-16, FR-17, FR-18, FR-19 | PI 2.1, PI 3.3 | Hàng đợi phê duyệt + cờ Retracted/Erratum |
| `standardized_data` | FR-17, FR-18, FR-20 | PI 3.3, PI 3.4 | Dữ liệu chuẩn sau duyệt + cờ rollback |
| `audit_log` | FR-21, FR-22 | PI 3.3, PI 3.4 | Nhật ký kiểm toán bất biến (NFR-02) |
| `export_audit_log` | FR-29 | PI 3.4 | Lưu vết xuất báo cáo có ký số |
| `security_audit_log` | NFR-04 | PI 3.3 | Lưu vết truy cập trái phép |

**Khẳng định độ phủ:** 10 bảng CSDL đáp ứng toàn bộ **29 FR**, **5 NFR** và **5 PI** của đề cương; mỗi yêu cầu chức năng đều có ít nhất một bảng phụ trách, mỗi bảng đều có khóa chính, khóa ngoại rõ ràng, kèm chỉ mục phục vụ hiệu năng (NFR-03) và ràng buộc bất biến cho audit log (NFR-02).

---

## 6. KIỂM THỬ SCHEMA (SMOKE TEST)

Sau khi chạy migration, thực thi các truy vấn sau để xác nhận schema đã đúng:

```sql
-- 1. Kiểm tra tất cả bảng đã tạo
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'scopus_ictu'
ORDER BY table_name;

-- 2. Kiểm tra role app_backend không có quyền UPDATE/DELETE trên audit_log
SELECT grantee, privilege_type
FROM information_schema.role_table_grants
WHERE table_schema = 'scopus_ictu'
  AND table_name   = 'audit_log'
  AND grantee      = 'app_backend'
ORDER BY privilege_type;
-- Kỳ vọng: chỉ thấy INSERT và SELECT, KHÔNG có UPDATE/DELETE.

-- 3. Kiểm tra Idempotency: insert 2 lần cùng EID phải fail
INSERT INTO scopus_ictu.upload_batches (file_name, operator_id)
VALUES ('test.csv', 1) RETURNING id;          -- Lấy batch_id

INSERT INTO scopus_ictu.raw_scopus_data
    (eid, title, authors, author_full_names, author_ids, year,
     batch_id, operator_id)
VALUES
    ('TEST-EID-001','T1','A','A','1',2026,1,1);

INSERT INTO scopus_ictu.raw_scopus_data
    (eid, title, authors, author_full_names, author_ids, year,
     batch_id, operator_id)
VALUES
    ('TEST-EID-001','T1','A','A','1',2026,1,1);
-- Kỳ vọng: lỗi UNIQUE violation do eid='TEST-EID-001' đã tồn tại.
```
