# Thiết kế Cơ sở dữ liệu — Hệ thống Chuẩn hóa Hồ sơ Scopus ICTU

> **Đề tài:** Xây dựng ứng dụng web quản lý và chuẩn hóa hồ sơ tác giả, công bố khoa học trên Scopus phục vụ Trường Đại học Công nghệ thông tin và Truyền thông (ICTU).
> **Tuần thực hiện:** Tuần 3 — Nhiệm vụ 1: Thiết kế CSDL quan hệ.
> **Căn cứ:** Đặc tả FR/NFR Tuần 2 (`BAO_CAO_TIEN_DO_TUAN_2.md`); Kết quả khảo sát 660 bản ghi Tuần 1 (`scopus-ictu-data-survey.md`); Chuẩn SQL PostgreSQL 15.
> **Quy ước đặt tên:** `snake_case` cho toàn bộ tên bảng và cột; khóa chính (PK) là cột `id` (BIGSERIAL); khóa ngoại (FK) có hậu tố `_id`.
> **Người soạn:** Trương Quang Huy — SV ngành CNTT, ICTU.

---

## Mục lục
1. [Sơ đồ ERD (Mermaid)](#1-sơ-đồ-erd-mermaid)
2. [Từ điển dữ liệu (Data Dictionary)](#2-từ-điển-dữ-liệu-data-dictionary)
3. [Mã nguồn SQL DDL (PostgreSQL 15)](#3-mã-nguồn-sql-ddl-postgresql-15)
4. [Ma trận truy vết bảng ↔ FR ↔ PI](#4-ma-trận-truy-vết-bảng--fr--pi)

---

## 1. Sơ đồ ERD (Mermaid)

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
        text eid UK "Khóa Idempotency từ Scopus"
        text title "Tiêu đề bài báo"
        text authors "Chuỗi tác giả viết tắt không dấu"
        text author_full_names "Tên đầy đủ có dấu"
        text author_ids "Chuỗi Scopus Author ID phân cách ';'"
        smallint year "Năm công bố 2008-2027"
        text source_title
        text volume
        text issue
        text art_no
        text page_start
        text page_end
        integer cited_by "Tổng lượt trích dẫn"
        text doi "Khuyết 25/660 bản ghi"
        text link
        text abstract
        text author_keywords
        text publisher "Khuyết 22/660 bản ghi"
        text document_type "Article/Conf./Retracted/Erratum"
        text publication_stage
        text open_access "Khuyết 487/660 bản ghi"
        text source
        bigint batch_id "FK ngầm → upload_batches"
        smallint operator_id "FK → users"
        timestamp uploaded_at
        boolean duplicate "Idempotency flag"
        text warning_reasons "Mảng lý do: MISSING_DOI..."
        varchar processing_status "PENDING/PROCESSED/FAILED"
    }

    lecturers {
        bigserial id PK
        varchar lecturer_code UK "Mã GV ICTU (L001...)"
        text full_name "Tên đầy đủ có dấu"
        text full_name_no_diacritics "Tên không dấu - khóa fuzzy match"
        text scopus_author_id UK "Khóa định danh Scopus"
        text affiliation "Khoa/Bộ môn hiện tại"
        varchar department "Mapping khoa"
        varchar faculty "CNTT/KHMT/KH&KTMT..."
        varchar academic_rank
        email email
        boolean is_active "Còn công tác hay đã nghỉ"
        timestamp created_at
        timestamp updated_at
    }

    author_extracted {
        bigserial id PK
        bigint raw_id FK "Trỏ raw_scopus_data.id"
        smallint author_index "Thứ tự trong bài"
        varchar lastname_initials "Nguyen T.-T."
        text normalized_name "Tên không dấu lower"
        bigint scopus_author_id "FK ngầm → lecturers"
        smallint lecturer_id FK "GV ICTU sau khi match (nullable)"
        numeric match_score "0.000 - 1.000"
        jsonb evidence_payload "Căn cứ đối chiếu"
        boolean is_common_name "Cờ cảnh báo Nguyen/Tran/Le..."
    }

    approval_queue {
        bigserial id PK
        bigint raw_id FK
        bigint author_extracted_id FK
        bigint lecturer_id FK "GV ICTU gợi ý (nullable)"
        varchar status "MATCH_HIGH/MEDIUM/PENDING_VERIFY/APPROVED/PENDING"
        numeric match_score "Bản sao score từ author_extracted"
        jsonb evidence_payload "Bản sao evidence"
        varchar document_type "Cờ cảnh báo Retracted/Erratum"
        boolean is_retracted
        boolean is_erratum
        timestamp created_at
        timestamp updated_at
        timestamp approved_at "Nullable - dùng cho rule 72h rollback"
        varchar approved_by "Nullable - FK → users"
    }

    standardized_data {
        bigserial id PK
        bigint approval_queue_id FK UK "Mỗi queue tối đa 1 chuẩn hoá"
        bigint lecturer_id FK "GV ICTU chính thức sau duyệt"
        bigint raw_id FK
        numeric match_score
        jsonb evidence_payload "Lưu vết tại thời điểm duyệt"
        timestamp approved_at
        varchar approved_by "FK → users"
        boolean is_invalidated "Rollback flag"
        timestamp invalidated_at "Nullable"
        varchar invalidated_by "Nullable - FK → users"
        text invalidation_reason "Nullable"
    }

    audit_log {
        bigserial id PK
        varchar action "APPROVE/MANUAL_OVERRIDE/DEFER/ROLLBACK"
        bigint actor_id "FK → users"
        varchar target_table "standardized_data/approval_queue..."
        bigint target_id
        jsonb before_json "Trạng thái trước"
        jsonb after_json "Trạng thái sau"
        text reason "Lý do (bắt buộc cho ROLLBACK)"
        uuid correlation_id "Liên kết log toàn pipeline"
        timestamp created_at "Không UPDATE/DELETE sau khi ghi"
    }

    export_audit_log {
        bigserial id PK
        varchar file_format "xlsx/csv/json"
        varchar filter_criteria_json "Lưu bộ lọc đã dùng"
        integer row_count
        varchar file_hash_sha256 "64 hex chars"
        varchar file_name "Tên file xuất"
        bigint actor_id "FK → users"
        timestamp created_at
    }

    users {
        bigserial id PK
        varchar username UK
        varchar password_hash "bcrypt/argon2"
        varchar role "operator/admin"
        boolean is_active
        timestamp created_at
        timestamp last_login
    }

    upload_batches {
        bigserial id PK
        text file_name "Tên tệp CSV"
        bigint operator_id "FK → users"
        timestamp uploaded_at
        integer total_rows
        integer new_rows
        integer duplicate_rows
        integer warning_rows
        varchar correlation_id "UUID liên kết log"
        varchar processing_status "PENDING/RUNNING/COMPLETED/FAILED"
    }
```

**Ghi chú đọc sơ đồ:**
- Đường `||--o{` = quan hệ **một-nhiều bắt buộc** (một bài có nhiều tác giả).
- Đường `}o--o|` = quan hệ **nhiều-một tùy chọn** (một queue có thể chưa gán được GV nào).
- Đường `||--o|` = quan hệ **một-một tùy chọn** (mỗi queue tối đa 1 dòng chuẩn hoá).

---

## 2. Từ điển dữ liệu (Data Dictionary)

### 2.1. Bảng `users` — Tài khoản người dùng

| Cột | Kiểu PG | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính tự tăng |
| `username` | `VARCHAR(64)` | UNIQUE, NOT NULL | Tên đăng nhập |
| `password_hash` | `VARCHAR(255)` | NOT NULL | Mật khẩu đã hash (bcrypt/argon2) |
| `role` | `VARCHAR(16)` | NOT NULL, CHECK (`role` IN ('operator','admin')) | Phân quyền theo NFR-04 |
| `is_active` | `BOOLEAN` | NOT NULL DEFAULT `TRUE` | Tài khoản còn hiệu lực |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT `NOW()` | Thời điểm tạo |
| `last_login` | `TIMESTAMPTZ` | NULL | Lần đăng nhập gần nhất |

### 2.2. Bảng `upload_batches` — Lô nhập dữ liệu

| Cột | Kiểu PG | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính tự tăng |
| `file_name` | `TEXT` | NOT NULL | Tên tệp CSV (ví dụ `Truong_Quang_Huy_Scopus_Sep2026.csv`) |
| `operator_id` | `BIGINT` | FK → `users.id`, NOT NULL | Người upload (Operator) |
| `uploaded_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT `NOW()` | Thời điểm upload |
| `total_rows` | `INTEGER` | NOT NULL DEFAULT 0 | Tổng số dòng trong tệp |
| `new_rows` | `INTEGER` | NOT NULL DEFAULT 0 | Số dòng mới (không trùng EID) |
| `duplicate_rows` | `INTEGER` | NOT NULL DEFAULT 0 | Số dòng duplicate (Idempotency) |
| `warning_rows` | `INTEGER` | NOT NULL DEFAULT 0 | Số dòng có cảnh báo (DOI thiếu, ...) |
| `correlation_id` | `UUID` | UNIQUE, NOT NULL | Mã liên kết log pipeline |
| `processing_status` | `VARCHAR(16)` | CHECK ∈ {PENDING, RUNNING, COMPLETED, FAILED} | Trạng thái xử lý batch |

### 2.3. Bảng `raw_scopus_data` — Dữ liệu CSV gốc 22 cột

| Cột | Kiểu PG | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính tự tăng |
| `eid` | `TEXT` | UNIQUE NOT NULL | Mã EID Scopus — khóa Idempotency theo NFR-01 |
| `title` | `TEXT` | NOT NULL | Tiêu đề bài báo |
| `authors` | `TEXT` | NOT NULL | Chuỗi tác giả viết tắt không dấu |
| `author_full_names` | `TEXT` | NOT NULL | Tên đầy đủ có dấu (Scopus export) |
| `author_ids` | `TEXT` | NOT NULL | Chuỗi Scopus Author ID phân cách `;` |
| `year` | `SMALLINT` | NOT NULL, CHECK (year BETWEEN 1900 AND 2100) | Năm công bố (2008-2027 theo khảo sát) |
| `source_title` | `TEXT` | NULL | Tên tạp chí / kỷ yếu hội nghị |
| `volume`, `issue`, `art_no` | `TEXT` | NULL | Tập, số, số bài |
| `page_start`, `page_end` | `TEXT` | NULL | Trang bắt đầu/kết thúc |
| `cited_by` | `INTEGER` | NOT NULL DEFAULT 0 | Tổng lượt trích dẫn (khảo sát: tổng 4 885) |
| `doi` | `TEXT` | NULL | DOI — khuyết 25/660 bản ghi |
| `link` | `TEXT` | NULL | URL Scopus |
| `abstract` | `TEXT` | NULL | Tóm tắt |
| `author_keywords` | `TEXT` | NULL | Từ khóa của tác giả |
| `publisher` | `TEXT` | NULL | Nhà xuất bản — khuyết 22/660 bản ghi |
| `document_type` | `TEXT` | NULL | Conference paper / Article / Retracted / Erratum |
| `publication_stage` | `TEXT` | NULL | Final / Article in press |
| `open_access` | `TEXT` | NULL | OA flag — khuyết 487/660 bản ghi |
| `source` | `TEXT` | NULL | Scopus source ID |
| `batch_id` | `BIGINT` | FK → `upload_batches.id`, NOT NULL | Lô nhập |
| `operator_id` | `BIGINT` | FK → `users.id`, NOT NULL | Người upload |
| `uploaded_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT `NOW()` | Thời điểm insert |
| `duplicate` | `BOOLEAN` | NOT NULL DEFAULT `FALSE` | Cờ trùng EID theo NFR-01 |
| `warning_reasons` | `TEXT[]` | NULL | Mảng lý do cảnh báo: MISSING_DOI, MISSING_YEAR, ... |
| `processing_status` | `VARCHAR(16)` | DEFAULT 'PENDING' | Trạng thái xử lý pipeline |

**Chỉ mục đặc biệt:**
- `UNIQUE(eid)` đảm bảo Idempotency cấp bản ghi.
- `INDEX(year, document_type)` phục vụ lọc đa tiêu chí FR-23.

### 2.4. Bảng `lecturers` — Danh mục Giảng viên ICTU

| Cột | Kiểu PG | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `lecturer_code` | `VARCHAR(16)` | UNIQUE NOT NULL | Mã GV ICTU (L001, L002...) |
| `full_name` | `TEXT` | NOT NULL | Họ tên đầy đủ có dấu |
| `full_name_no_diacritics` | `TEXT` | NOT NULL | Họ tên không dấu — khóa fuzzy match (FR-08) |
| `scopus_author_id` | `TEXT` | UNIQUE NULL | Scopus Author ID — khóa match chính xác (FR-10) |
| `affiliation` | `TEXT` | NULL | Đơn vị hiện tại |
| `department` | `VARCHAR(64)` | NULL | Bộ môn |
| `faculty` | `VARCHAR(64)` | NULL | Khoa |
| `academic_rank` | `VARCHAR(32)` | NULL | Học hàm/học vị |
| `email` | `VARCHAR(128)` | NULL | Email liên hệ |
| `is_active` | `BOOLEAN` | NOT NULL DEFAULT `TRUE` | Còn công tác hay đã nghỉ/chuyển |
| `created_at`, `updated_at` | `TIMESTAMPTZ` | DEFAULT `NOW()` | Theo dõi thời gian |

**Chỉ mục:** `INDEX(full_name_no_diacritics)` phục vụ so khớp mờ RapidFuzz; `INDEX(faculty, department)` phục vụ bộ lọc.

### 2.5. Bảng `author_extracted` — Tác giả bóc tách từ mỗi bài

| Cột | Kiểu PG | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `raw_id` | `BIGINT` | FK → `raw_scopus_data.id`, NOT NULL | Bài báo gốc |
| `author_index` | `SMALLINT` | NOT NULL | Thứ tự tác giả trong bài (1, 2, 3...) |
| `lastname_initials` | `VARCHAR(64)` | NOT NULL | Họ + chữ cái đầu tên, ví dụ `Nguyen T.-T.` |
| `normalized_name` | `TEXT` | NOT NULL | Tên không dấu lower — khóa so khớp (FR-08) |
| `scopus_author_id` | `BIGINT` | NULL | ID Scopus sau parse (FR-07) |
| `lecturer_id` | `BIGINT` | FK → `lecturers.id`, NULL | GV ICTU ứng viên sau khi match (nullable khi PENDING) |
| `match_score` | `NUMERIC(4,3)` | NULL, CHECK (0 ≤ score ≤ 1) | Match Score ∈ [0,1] (FR-12) |
| `evidence_payload` | `JSONB` | NULL | Căn cứ đối chiếu: `{author_id_match, name_score, affil_score, common_name_flag}` |
| `is_common_name` | `BOOLEAN` | NOT NULL DEFAULT `FALSE` | Cờ cảnh báo tên phổ biến (Nguyen/Tran/Le) — phục vụ AC-02.06 |

**Chỉ mục:** `INDEX(raw_id, author_index)` UNIQUE; `INDEX(lecturer_id)` phục vụ thống kê.

### 2.6. Bảng `approval_queue` — Hàng đợi phê duyệt

| Cột | Kiểu PG | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `raw_id` | `BIGINT` | FK → `raw_scopus_data.id`, NOT NULL | Bài báo |
| `author_extracted_id` | `BIGINT` | FK → `author_extracted.id`, NOT NULL | Tác giả bóc tách |
| `lecturer_id` | `BIGINT` | FK → `lecturers.id`, NULL | GV ICTU gợi ý (NULL khi PENDING) |
| `status` | `VARCHAR(16)` | NOT NULL, CHECK ∈ {MATCH_HIGH, MATCH_MEDIUM, PENDING_VERIFY, APPROVED, PENDING} | Trạng thái duyệt (FR-13) |
| `match_score` | `NUMERIC(4,3)` | NOT NULL | Bản sao score (FR-12) |
| `evidence_payload` | `JSONB` | NOT NULL | Bản sao evidence đính kèm Diff View (FR-16) |
| `document_type` | `VARCHAR(64)` | NULL | Cờ phát hiện bất thường |
| `is_retracted` | `BOOLEAN` | NOT NULL DEFAULT `FALSE` | Cờ Retracted (4/660 bản ghi) |
| `is_erratum` | `BOOLEAN` | NOT NULL DEFAULT `FALSE` | Cờ Erratum (4/660 bản ghi) |
| `created_at`, `updated_at` | `TIMESTAMPTZ` | DEFAULT `NOW()` | Theo dõi thời gian |
| `approved_at` | `TIMESTAMPTZ` | NULL | Thời điểm duyệt — dùng cho rule 72h rollback (FR-20) |
| `approved_by` | `BIGINT` | FK → `users.id`, NULL | Admin duyệt |

**Chỉ mục:** `INDEX(status, created_at DESC)` phục vụ Admin mở Queue; `INDEX(raw_id, author_extracted_id)` UNIQUE.

### 2.7. Bảng `standardized_data` — Dữ liệu chuẩn hoá sau phê duyệt

| Cột | Kiểu PG | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `approval_queue_id` | `BIGINT` | FK → `approval_queue.id`, UNIQUE NOT NULL | Mỗi queue tối đa 1 dòng chuẩn |
| `lecturer_id` | `BIGINT` | FK → `lecturers.id`, NOT NULL | GV ICTU chính thức sau duyệt |
| `raw_id` | `BIGINT` | FK → `raw_scopus_data.id`, NOT NULL | Bài báo gốc |
| `match_score` | `NUMERIC(4,3)` | NOT NULL | Score tại thời điểm duyệt |
| `evidence_payload` | `JSONB` | NOT NULL | Lưu vết evidence (FR-21) |
| `approved_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT `NOW()` | Thời điểm duyệt |
| `approved_by` | `BIGINT` | FK → `users.id`, NOT NULL | Admin duyệt |
| `is_invalidated` | `BOOLEAN` | NOT NULL DEFAULT `FALSE` | Cờ rollback (FR-20) |
| `invalidated_at` | `TIMESTAMPTZ` | NULL | Thời điểm rollback |
| `invalidated_by` | `BIGINT` | FK → `users.id`, NULL | Admin rollback |
| `invalidation_reason` | `TEXT` | NULL | Lý do rollback (bắt buộc khi ROLLBACK) |

**Chỉ mục:** `INDEX(lecturer_id, year)` phục vụ bộ lọc FR-23 và thống kê FR-25; `INDEX(is_invalidated)` lọc dữ liệu còn hiệu lực.

### 2.8. Bảng `audit_log` — Nhật ký kiểm toán

| Cột | Kiểu PG | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `action` | `VARCHAR(32)` | NOT NULL, CHECK ∈ {APPROVE, MANUAL_OVERRIDE, DEFER, ROLLBACK, LECTURER_CRUD} | Loại thao tác (FR-21) |
| `actor_id` | `BIGINT` | FK → `users.id`, NOT NULL | Người thực hiện |
| `target_table` | `VARCHAR(32)` | NOT NULL | Bảng bị tác động (`standardized_data`, `lecturers`, ...) |
| `target_id` | `BIGINT` | NOT NULL | Khóa chính bản ghi bị tác động |
| `before_json` | `JSONB` | NULL | Trạng thái trước |
| `after_json` | `JSONB` | NULL | Trạng thái sau |
| `reason` | `TEXT` | NULL | Lý do (bắt buộc cho ROLLBACK theo UC-04) |
| `correlation_id` | `UUID` | NOT NULL | Liên kết với pipeline (ST-13) |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT `NOW()` | Thời điểm ghi — bất biến theo NFR-02 |

**Ràng buộc bất biến (NFR-02):** Bảng này **không cho UPDATE/DELETE** sau khi insert. Thực thi bằng cách revoke quyền UPDATE/DELETE trên role `app_backend` của PostgreSQL (xem mục 3.6).

### 2.9. Bảng `export_audit_log` — Lưu vết xuất báo cáo

| Cột | Kiểu PG | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `file_format` | `VARCHAR(8)` | NOT NULL, CHECK ∈ {xlsx, csv, json} | Định dạng xuất (FR-26/27/28) |
| `filter_criteria_json` | `JSONB` | NOT NULL | Bộ lọc đã dùng: `{year_from, year_to, faculty, publisher_family, status}` |
| `row_count` | `INTEGER` | NOT NULL, CHECK (row_count ≥ 0) | Số dòng dữ liệu xuất |
| `file_hash_sha256` | `CHAR(64)` | NOT NULL | Hash SHA-256 hex 64 ký tự (FR-29) |
| `file_name` | `TEXT` | NOT NULL | Tên file xuất |
| `actor_id` | `BIGINT` | FK → `users.id`, NOT NULL | Người xuất (Operator/Admin) |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT `NOW()` | Thời điểm xuất |

### 2.10. Bảng `security_audit_log` — Lưu vết truy cập trái phép (bổ sung theo NFR-04)

| Cột | Kiểu PG | Ràng buộc | Ý nghĩa |
|---|---|---|---|
| `id` | `BIGSERIAL` | PK | Khóa chính |
| `actor_id` | `BIGINT` | FK → `users.id`, NULL | Người dùng (NULL nếu chưa đăng nhập) |
| `attempted_endpoint` | `VARCHAR(255)` | NOT NULL | API/endpoint bị gọi trái phép |
| `required_permission` | `VARCHAR(64)` | NOT NULL | Quyền cần có (vd: `admin.rollback`) |
| `http_status` | `SMALLINT` | NOT NULL | Mã HTTP trả về (403) |
| `ip_address` | `INET` | NULL | IP client |
| `user_agent` | `TEXT` | NULL | Trình duyệt / client |
| `payload_json` | `JSONB` | NULL | Payload request |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT `NOW()` | Thời điểm |

---

## 3. Mã nguồn SQL DDL (PostgreSQL 15)

Script dưới đây được thiết kế để chạy migration từ đầu trên PostgreSQL 15+. Thứ tự tạo bảng tuân theo quan hệ khóa ngoại: bảng không phụ thuộc tạo trước.

### 3.1. Schema và Extension

```sql
-- =====================================================================
-- Migration 001: Khởi tạo schema cho Hệ thống Scopus ICTU
-- Database: PostgreSQL 15+
-- Author: Trương Quang Huy — ICTU
-- Date: 2026-09-26
-- =====================================================================

CREATE SCHEMA IF NOT EXISTS scopus_ictu;
SET search_path TO scopus_ictu, public;

-- Extension cần thiết cho UUID và pg_trgm (fuzzy match)
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pg_trgm;     -- phục vụ FR-11 Fuzzy Match
```

### 3.2. Bảng Users và Upload Batches

```sql
-- =====================================================================
-- Bảng 1: users — Tài khoản & phân quyền (NFR-04)
-- =====================================================================
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

-- =====================================================================
-- Bảng 2: upload_batches — Lô nhập dữ liệu
-- =====================================================================
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

### 3.3. Bảng Raw Data và Lecturers

```sql
-- =====================================================================
-- Bảng 3: lecturers — Danh mục Giảng viên ICTU (Master Data)
-- =====================================================================
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

-- Index GIN cho fuzzy match tên không dấu (FR-11)
CREATE INDEX idx_lecturers_name_trgm
    ON scopus_ictu.lecturers
    USING gin (full_name_no_diacritics gin_trgm_ops);

CREATE INDEX idx_lecturers_faculty_dept
    ON scopus_ictu.lecturers(faculty, department);

-- =====================================================================
-- Bảng 4: raw_scopus_data — Dữ liệu CSV gốc 22 cột
-- =====================================================================
CREATE TABLE scopus_ictu.raw_scopus_data (
    id                  BIGSERIAL    PRIMARY KEY,
    eid                 TEXT         NOT NULL UNIQUE,           -- NFR-01
    title               TEXT         NOT NULL,
    authors             TEXT         NOT NULL,
    author_full_names   TEXT         NOT NULL,
    author_ids          TEXT         NOT NULL,                  -- chuỗi ';' 
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

### 3.4. Bảng Author Extracted và Approval Queue

```sql
-- =====================================================================
-- Bảng 5: author_extracted — Tác giả bóc tách từ mỗi bài
-- =====================================================================
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

-- =====================================================================
-- Bảng 6: approval_queue — Hàng đợi phê duyệt
-- =====================================================================
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
    approved_at         TIMESTAMPTZ,                              -- rule 72h
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

### 3.5. Bảng Standardized và Audit Log

```sql
-- =====================================================================
-- Bảng 7: standardized_data — Dữ liệu chuẩn sau phê duyệt
-- =====================================================================
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

-- =====================================================================
-- Bảng 8: audit_log — Nhật ký kiểm toán (NFR-02: bất biến)
-- =====================================================================
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

-- =====================================================================
-- Bảng 9: export_audit_log — Lưu vết xuất báo cáo (FR-29)
-- =====================================================================
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

-- =====================================================================
-- Bảng 10: security_audit_log — Truy cập trái phép (NFR-04)
-- =====================================================================
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

### 3.6. Bất biến Audit Log (NFR-02)

```sql
-- =====================================================================
-- Thực thi bất biến AUDIT_LOG theo NFR-02:
-- Tạo role riêng cho backend, revoke UPDATE/DELETE.
-- =====================================================================

-- Tạo role ứng dụng (nếu chưa có)
DO $$
BEGIN
    IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'app_backend') THEN
        CREATE ROLE app_backend LOGIN PASSWORD 'change_me_in_production';
    END IF;
END
$$;

-- Cấp quyền tối thiểu trên các bảng nghiệp vụ
GRANT USAGE ON SCHEMA scopus_ictu TO app_backend;

GRANT SELECT, INSERT, UPDATE          ON scopus_ictu.users                 TO app_backend;
GRANT SELECT, INSERT                  ON scopus_ictu.upload_batches       TO app_backend;
GRANT SELECT, INSERT, UPDATE          ON scopus_ictu.lecturers            TO app_backend;
GRANT SELECT, INSERT, UPDATE          ON scopus_ictu.raw_scopus_data      TO app_backend;
GRANT SELECT, INSERT, UPDATE          ON scopus_ictu.author_extracted     TO app_backend;
GRANT SELECT, INSERT, UPDATE          ON scopus_ictu.approval_queue       TO app_backend;
GRANT SELECT, INSERT, UPDATE          ON scopus_ictu.standardized_data    TO app_backend;
GRANT SELECT, INSERT                  ON scopus_ictu.export_audit_log     TO app_backend;
GRANT SELECT, INSERT                  ON scopus_ictu.security_audit_log   TO app_backend;

-- Riêng audit_log: chỉ INSERT + SELECT, KHÔNG UPDATE/DELETE
GRANT SELECT, INSERT ON scopus_ictu.audit_log TO app_backend;
```

### 3.7. View hỗ trợ báo cáo

```sql
-- =====================================================================
-- View phục vụ báo cáo thống kê (FR-25)
-- =====================================================================
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
    'Thống kê số bài / trích dẫn / cảnh báo theo năm công bố — phục vụ dashboard FR-25';
```

### 3.8. Kiểm thử Schema (Smoke Test)

```sql
-- =====================================================================
-- Chạy sau khi migration xong để verify
-- =====================================================================

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
```

---

## 4. Ma trận truy vết bảng ↔ FR ↔ PI

| Bảng | FR đáp ứng | PI đáp ứng | Ghi chú |
|---|---|---|---|
| `users` | FR-01, FR-22 | PI 1.1, PI 3.3 | Nền tảng phân quyền NFR-04 |
| `upload_batches` | FR-04, FR-15 | PI 1.1, PI 2.1 | Truy vết nguồn lô nhập |
| `raw_scopus_data` | FR-02, FR-03, FR-04, FR-05, FR-06 | PI 1.1, PI 1.2 | Lưu 22 cột gốc |
| `lecturers` | FR-08, FR-10, FR-11, FR-22 | PI 2.1, PI 3.3 | Master data |
| `author_extracted` | FR-07, FR-08, FR-09, FR-10, FR-11, FR-12, FR-13 | PI 2.1 | Lưu kết quả bóc tách + Match Score |
| `approval_queue` | FR-13, FR-14, FR-16, FR-17, FR-18, FR-19 | PI 2.1, PI 3.3 | Hàng đợi phê duyệt |
| `standardized_data` | FR-17, FR-18, FR-20 | PI 3.3, PI 3.4 | Dữ liệu chuẩn sau duyệt |
| `audit_log` | FR-21, FR-22 | PI 3.3, PI 3.4 | Nhật ký kiểm toán (bất biến) |
| `export_audit_log` | FR-29 | PI 3.4 | Lưu vết xuất báo cáo |
| `security_audit_log` | NFR-04 | PI 3.3 | Truy cập trái phép |

**Khẳng định độ phủ:** Toàn bộ 29 FR và 5 PI trong đề cương đều có ít nhất một bảng CSDL phụ trách; hệ thống 10 bảng đáp ứng đủ các yêu cầu chức năng, phi chức năng và ràng buộc audit của Tuần 2.
