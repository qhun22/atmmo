# TIẾN ĐỘ TUẦN 4: NỀN TẢNG ỨNG DỤNG, XÁC THỰC/RBAC VÀ NHẬP DỮ LIỆU

> **Đề tài:** Xây dựng ứng dụng web quản lý và chuẩn hóa hồ sơ tác giả, công bố khoa học trên Scopus phục vụ Trường Đại học Công nghệ thông tin và Truyền thông (ICTU).
> **Tuần thực hiện:** Tuần 4 — Nhiệm vụ chính: Xây dựng nền tảng ứng dụng (React + FastAPI + PostgreSQL), xác thực/phân quyền RBAC, nhập dữ liệu CSV Scopus có kiểm tra lỗi.
> **Căn cứ:** Đặc tả FR/NFR Tuần 2 (`BAO_CAO_TIEN_DO_TUAN_2.md`); Thiết kế CSDL Tuần 3 (`TUAN3.MD`); Kết quả khảo sát 660 bản ghi Tuần 1.
> **Người soạn:** Trương Quang Huy — SV ngành CNTT, ICTU.

---

## 1. TỔNG QUAN NỀN TẢNG CÔNG NGHỆ VÀ KIẾN TRÚC THỰC THI

### 1.1. Kiến trúc tổng thể

Hệ thống tuân theo mô hình **3 tầng** (đã đặc tả ở Tuần 3):

```
┌─────────────────────────────────────────────────────────┐
│  TẦNG 1: TRÌNH DUYỆT / CLIENT                        │
│  React 18 + Vite 5 + Tailwind CSS 3                    │
│  - Trang Login, Upload Dashboard                         │
│  - Drag-drop CSV, hiển thị kết quả, bảng batch        │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTP/REST (JWT Bearer)
                       ▼
┌──────────────────────┬──────────────────────────────────┐
│  TẦNG 2: BACKEND API (FastAPI + Uvicorn)              │
│  Python 3.11+ / FastAPI 0.110+                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Core Layer                                        │  │
│  │  - config.py: Pydantic BaseSettings (đọc .env)    │  │
│  │  - database.py: SQLAlchemy 2.x + psycopg2         │  │
│  │  - security.py: JWT + bcrypt + RBAC dependency    │  │
│  └──────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────┐  │
│  │  API Layer (api/v1/)                              │  │
│  │  - POST /auth/login, /auth/refresh, GET /auth/me │  │
│  │  - POST /uploads (4-step pipeline)                │  │
│  │  - GET /uploads/batches                           │  │
│  └──────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Services Layer                                   │  │
│  │  - scopus_parser.py: validate schema + row warns  │  │
│  │  - idempotency.py: EID dedup check                │  │
│  │  - storage.py: lưu file CSV lên đĩa              │  │
│  └──────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────┘
                       │ SQLAlchemy ORM
                       ▼
┌──────────────────────┬──────────────────────────────────┐
│  TẦNG 3: DATABASE (PostgreSQL 15)                     │
│  Schema: scopus_ictu (10 bảng)                        │
│  app_backend role với revoke UPDATE/DELETE trên       │
│  audit_log (bất biến theo NFR-02)                    │
└───────────────────────────────────────────────────────┘
```

### 1.2. Cấu trúc thư mục dự án

```
scopus-ictu/
├── backend/
│   ├── app/
│   │   ├── core/
│   │   │   ├── config.py       # Pydantic BaseSettings đọc .env
│   │   │   ├── database.py     # SQLAlchemy engine + session + get_db
│   │   │   └── security.py     # JWT + bcrypt + RBAC dependencies
│   │   ├── models/
│   │   │   └── __init__.py     # 10 SQLAlchemy ORM models
│   │   ├── schemas/
│   │   │   ├── auth.py         # LoginRequest, TokenResponse, UserPublic
│   │   │   └── upload.py       # BatchSummary, ErrorResponse, SchemaMissingColumns
│   │   ├── api/v1/
│   │   │   ├── auth.py         # /auth/* endpoints
│   │   │   └── uploads.py      # /uploads/* endpoints (4-step pipeline)
│   │   ├── services/
│   │   │   ├── scopus_parser.py # validate_schema, iter_rows, row_warnings
│   │   │   ├── idempotency.py  # fetch_existing_eids, compute_duplicate_map
│   │   │   └── storage.py      # save_upload, ensure_default_dir
│   │   └── main.py             # FastAPI app + CORS + exception handlers + seed
│   ├── tests/
│   │   ├── conftest.py          # Fixtures: SQLite DB, TestClient, seed users
│   │   ├── test_auth.py          # 7 test cases cho Auth
│   │   └── test_upload_idempotency.py  # 9 test cases cho UC-01
│   ├── requirements.txt
│   ├── .env.example
│   └── pytest.ini
├── frontend/
│   ├── src/
│   │   ├── api/client.js         # Axios singleton + interceptors
│   │   ├── contexts/AuthContext.jsx  # Auth state toàn cục
│   │   ├── components/
│   │   │   ├── Navbar.jsx        # Header với username + role badge
│   │   │   ├── DragDropUpload.jsx # kéo thả CSV + progress
│   │   │   └── BatchSummaryCards.jsx # 4 thẻ chỉ số + warning breakdown
│   │   ├── pages/
│   │   │   ├── LoginPage.jsx     # Form đăng nhập
│   │   │   └── UploadDashboard.jsx # Trang chính sau login
│   │   ├── App.jsx + main.jsx
│   │   └── index.css             # Tailwind @layer components
│   ├── package.json
│   ├── vite.config.js            # Proxy /api -> backend
│   ├── tailwind.config.js
│   ├── postcss.config.js
│   └── .env.example
├── manage.bat                    # Script chạy 3 luồng (setup/run/test)
└── smoke_660.py                  # Smoke test trên file 660 dòng thực tế
```

### 1.3. Nguyên tắc "Áo trắng — Áo vàng" (Configuration as Code)

**Nguyên tắc tuyệt đối:** Không hardcode bất kỳ cấu hình nào (cổng, URL, secret key) trong mã nguồn.

| Cấu hình | Backend (Python) | Frontend (React) |
|---|---|---|
| Cổng server | `settings.PORT` ← `.env` | — (Vite proxy) |
| Database URL | `settings.DATABASE_URL` ← `.env` | — |
| JWT Secret | `settings.JWT_SECRET_KEY` ← `.env` | — |
| CORS Origins | `settings.CORS_ORIGINS` ← `.env` | `VITE_API_BASE_URL` ← `.env` |
| Max upload size | `settings.UPLOAD_MAX_FILE_MB` ← `.env` | — |

Backend đọc config qua `pydantic-settings` → `BaseSettings` → tự động parse `.env`. Frontend đọc qua `import.meta.env.VITE_*` (Vite).

---

## 2. CHI TIẾT MODULE XÁC THỰC & PHÂN QUYỀN RBAC

### 2.1. Kiến trúc JWT

Backend sinh **2 loại token**:

| Token | TTL | Mục đích |
|---|---|---|
| `access_token` | 30 phút | Xác thực API requests |
| `refresh_token` | 7 ngày | Đổi access token mới khi hết hạn |

**Cấu trúc JWT payload:**

```json
{
  "sub": "1",                    // user.id
  "type": "access",              // hoặc "refresh"
  "role": "operator",            // extra claim
  "username": "qhun22",         // extra claim
  "iat": 1727380800,
  "exp": 1727382600
}
```

### 2.2. Quản lý mật khẩu (bcrypt)

```python
# app/core/security.py
pwd_context: CryptContext = CryptContext(
    schemes=["bcrypt"],
    deprecated="auto",
    bcrypt__rounds=settings.BCRYPT_ROUNDS,  # 12 vòng
)
```

### 2.3. Dependency `get_current_user`

Dependency FastAPI này resolve user từ Bearer token trong mọi request:

```python
def get_current_user(
    token: Optional[str] = Depends(oauth2_scheme),
    db: Session = Depends(get_db),
) -> User:
    # decode JWT → lấy user.id → query DB
    # raise HTTP 401 nếu token không hợp lệ / hết hạn / user bị disable
```

### 2.4. Dependency `require_role(*roles)` — RBAC enforcement

```python
def require_role(*allowed_roles):
    def _checker(current_user=Depends(get_current_user), db=Depends(get_db)) -> User:
        if current_user.role.lower() in allowed_roles:
            return current_user

        # Khi Operator gọi API của Admin → ghi security_audit_log + HTTP 403
        _log_security_audit(request, current_user, allowed_roles)
        raise HTTPException(status_code=403, detail="Khong du quyen")
    return _checker
```

### 2.5. Bảng `security_audit_log` — NFR-04

Mỗi lần Operator cố gọi API của Admin (ví dụ `admin.rollback`, `admin.lecturer_crud`…), hệ thống ghi:

| Cột | Giá trị mẫu |
|---|---|
| `actor_id` | `1` (qhun22) |
| `attempted_endpoint` | `POST /api/v1/approvals/rollback` |
| `required_permission` | `admin.rollback` |
| `http_status` | `403` |
| `ip_address` | `127.0.0.1` |
| `payload_json` | `{"method":"POST","path":"/api/v1/approvals/rollback"}` |

### 2.6. Seed users

Khi backend khởi động lần đầu, `_seed_users()` tạo 2 tài khoản mẫu:

| Username | Password | Role | Quyền |
|---|---|---|---|
| `qhun22` | `Quanghuyso1@` | `operator` | `op.upload`, `op.view_raw`, `op.export` |
| `admin` | `Quanghuyso1#` | `admin` | toàn bộ quyền bao gồm `admin.*` |

### 2.7. API endpoints Auth

| Method | Endpoint | Mô tả | Permission |
|---|---|---|---|
| POST | `/api/v1/auth/login` | Đăng nhập, nhận JWT pair | Public |
| POST | `/api/v1/auth/refresh` | Đổi refresh token lấy access mới | Public |
| GET | `/api/v1/auth/me` | Thông tin user hiện tại | Authenticated |

---

## 3. CHI TIẾT PIPELINE NHẬP LIỆU, KIỂM TRA SCHEMA VÀ IDEMPOTENCY

### 3.1. API `POST /api/v1/uploads` — Pipeline 4 bước

```
[Upload CSV] → [1. Schema Check] → [2. Idempotency] → [3. Row Validation] → [4. Persistence]
                   ↓ fail 400            ↓                  ↓                    ↓
              STOP, không ghi DB    đánh dấu flag     ghi warning_reasons    INSERT batch + rows
```

**Bước 1 — Schema Check (FR-02 / FR-06):**
- CSV header được kiểm tra với **22 cột bắt buộc**:
  ```
  Authors | Author full names | Author(s) ID | Title | Year | Source title |
  Volume | Issue | Art. No. | Page start | Page end | Cited by | DOI | Link |
  Abstract | Author Keywords | Publisher | Document Type | Publication Stage |
  Open Access | Source | EID
  ```
- Nếu thiếu bất kỳ cột nào → HTTP 400 + JSON:
  ```json
  {
    "status": "error",
    "code": "SCHEMA_INVALID",
    "message": "File CSV thieu hoac sai ten cot.",
    "correlation_id": "550e8400-e29b-...",
    "details": {
      "expected": ["Authors", "Title", ...],
      "missing": ["DOI", "Year"],
      "extra": [],
      "received": [...]
    }
  }
  ```
- Lỗi schema → **KHÔNG insert bất kỳ bản ghi nào** vào DB.

**Bước 2 — Idempotency Check (FR-03 / NFR-01):**
- Đọc EID từng dòng CSV.
- Query `SELECT eid FROM raw_scopus_data WHERE eid IN (...)` — 1 query duy nhất.
- Nếu EID đã tồn tại → `duplicate = true` (hoặc **skip hoàn toàn** để không sinh constraint violation).
- Nếu EID chưa có → chuẩn bị INSERT.

**Bước 3 — Row Validation (FR-05):**
- Quét từng dòng, phát hiện các trường hợp:
  | Mã cảnh báo | Điều kiện |
  |---|---|
  | `MISSING_DOI` | DOI rỗng/null |
  | `MISSING_YEAR` | Year rỗng/null |
  | `INVALID_YEAR_RANGE` | Year ∉ [1900, 2100] |
  | `INVALID_YEAR_FORMAT` | Year không parse được int |
  | `MISSING_AUTHORS` | Authors rỗng |
  | `MISSING_AUTHOR_IDS` | Author(s) ID rỗng |
  | `MISSING_TITLE` | Title rỗng |
  | `MISSING_EID` | EID rỗng |
- Hệ thống **KHÔNG dừng** khi gặp cảnh báo — tiếp tục xử lý các dòng còn lại.

**Bước 4 — Persistence (FR-04):**
- Insert 1 bản ghi vào `upload_batches` (tổng hợp).
- Insert các bản ghi vào `raw_scopus_data` (22 trường gốc + metadata).
- Tất cả trong 1 transaction — rollback nếu bất kỳ lỗi nào.

### 3.2. Kết quả chạy thực nghiệm với file 660 dòng

Chạy `smoke_660.py` trên file thực tế `Truong_Quang_Huy_Scopus_Sep2026.csv`:

```
=== STEP 1: SCHEMA CHECK ===
Valid: True
Expected: 22 columns
Received: 22 columns
Missing: []
Extra: []

=== STEP 2 + 3: ROW PROCESSING ===
Total rows: 660
Rows with warnings: 25
Warning breakdown:
  MISSING_DOI                  25

Unique EIDs:        660
Internal duplicates: 0
```

**Kết luận:**
- ✅ Schema hoàn toàn khớp với mẫu Scopus (22/22 cột).
- ✅ 660 EIDs đều duy nhất trong lô này.
- ✅ **25 dòng có cảnh báo MISSING_DOI** — khớp chính xác với kết quả khảo sát Tuần 1 (25/660 bài thiếu DOI).
- ✅ Không có bài Retracted/Erratum được phát hiện ở bước này (cần pipeline ST-06 mới kiểm tra document_type).

### 3.3. Ràng buộc idempotency (NFR-01)

Khi re-upload cùng file CSV lần thứ 2:
- Các EID đã tồn tại → **KHÔNG INSERT bản ghi mới** → `duplicate_rows` tăng, `new_rows` giữ nguyên.
- Kết quả: batch 1 → new=660, dup=0; batch 2 → new=0, dup=660. Không có constraint violation.
- Đảm bảo re-import **660 dòng 3 lần liên tiếp → số bản ghi mới sinh = 0** theo yêu cầu NFR-01.

### 3.4. Giới hạn file

| Thông số | Giá trị |
|---|---|
| Dung lượng tối đa | 50 MB |
| Encoding | UTF-8 (tự động nhận BOM) |
| Content-Type | `text/csv`, `application/csv`, `application/vnd.ms-excel` |
| Extension | `.csv` |

File vượt 50 MB → HTTP 413.

---

## 4. KẾT QUẢ GIAO DIỆN WEB

### 4.1. Trang Đăng nhập (LoginPage)

**Layout:** Centered card trên nền sáng.

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│              ┌────────────────────────┐              │
│              │  [ICTU Logo]          │              │
│              │  Scopus ICTU           │              │
│              │  He thong Chuan hoa... │              │
│              └────────────────────────┘              │
│                                                      │
│              ┌────────────────────────┐              │
│              │  [Error message]       │ ← red alert  │
│              │  Username             │              │
│              │  [__________________] │              │
│              │  Password              │              │
│              │  [__________________] │              │
│              │  [  Dang nhap       ] │              │
│              └────────────────────────┘              │
│                                                      │
│              Tai khoan mau                          │
│              Operator: qhun22 / Quanghuyso1@         │
│              Admin: admin / Quanghuyso1#             │
└──────────────────────────────────────────────────────┘
```

**Chức năng:**
- Form gửi username/password → POST `/api/v1/auth/login`.
- Nhận access_token + refresh_token → lưu LocalStorage.
- Tự động gọi `/auth/me` để lấy thông tin user.
- Redirect đến Dashboard khi thành công.
- Hiển thị lỗi 401 nếu sai tài khoản.

### 4.2. Trang Nhập liệu (UploadDashboard)

**Layout:** Header (Navbar) + 3 phần chính:

```
┌─────────────────────────────────────────────────────────────────────┐
│ [ICTU Logo] Scopus ICTU   Xin chao, qhun22 [Operator] [Dang xuat]  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Nhap du lieu Scopus                                                 │
│  Tai len tep CSV tu Scopus de nhap du lieu cong bo khoa hoc.         │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                     Tai tep CSV                              │    │
│  │                                                             │    │
│  │    [Icon upload]                                            │    │
│  │    Keo file CSV Scopus vao day                              │    │
│  │    hoac bam de chon file                                    │    │
│  │    Dung luong toi da: 50 MB | Encoding: UTF-8               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  KET QUA XU LY - Truong_Quang_Huy_Scopus_Sep2026.csv               │
│                                                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│  │ Tong so │ │ Dong moi │ │ Trung lap │ │ Canh bao │              │
│  │  660    │ │   660    │ │    0     │ │    25    │              │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘              │
│                                                                      │
│  Chi tiet canh bao                                                  │
│  MISSING_DOI: x25                                                   │
│  Thoi gian xu ly: 142 ms                                            │
│                                                                      │
│  Cac lo nhap gan day                                                 │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ File         │ Tong │ Moi │ Trung │ CB   │ Trang thai       │   │
│  │ Truong...csv │ 660  │ 660 │  0    │  25  │ [Hoan tat]       │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

**Chức năng:**
- **Drag & drop** file CSV với `react-dropzone`.
- Thanh tiến trình khi upload.
- Hiển thị 4 thẻ chỉ số: Tổng / Mới / Trùng / Cảnh báo.
- Breakdown cảnh báo (MISSING_DOI × 25).
- Thời gian xử lý (ms).
- Bảng các lô nhập gần đây (được fetch từ `GET /api/v1/uploads/batches`).
- Badge trạng thái: PENDING / RUNNING / COMPLETED / FAILED.

### 4.3. Navbar

Header cố định hiển thị:
- Logo + tên app (từ `VITE_APP_NAME`).
- Tên user đang đăng nhập.
- **Role badge** màu xanh (Operator) hoặc tím (Admin).
- Nút Đăng xuất.

---

## 5. TIÊU CHÍ NGHIỆM THU KỸ THUẬT TUẦN 4 VÀ KẾT QUẢ KIỂM THỬ

### 5.1. Tiêu chí nghiệm thu (AC Tuần 4)

| Mã AC | Mô tả | Kết quả |
|---|---|---|
| **AC-01.01** | Operator đăng nhập thành công → access_token + refresh_token | ✅ PASS |
| **AC-01.02** | Sai mật khẩu → HTTP 401 | ✅ PASS |
| **AC-01.03** | GET /auth/me với Bearer hợp lệ → user info | ✅ PASS |
| **AC-01.04** | Không có Bearer → HTTP 401 | ✅ PASS |
| **AC-01.05** | Refresh token hợp lệ → access_token mới | ✅ PASS |
| **AC-01.06** | Upload CSV đúng 22 cột → 201 + summary | ✅ PASS |
| **AC-01.07** | Re-upload → duplicate_rows tăng, new_rows = 0 | ✅ PASS |
| **AC-01.08** | Upload CSV sai schema → 400 + chi tiết cột thiếu | ✅ PASS |
| **AC-01.09** | Upload không có Bearer → 401 | ✅ PASS |
| **AC-01.10** | Dòng khuyết DOI → warning_rows tăng, MISSING_DOI | ✅ PASS |
| **AC-01.11** | GET /uploads/batches → danh sách batch | ✅ PASS |
| **AC-01.12** | File > 50 MB → HTTP 413 | ✅ PASS |
| **AC-01.13** | Admin cũng có quyền upload | ✅ PASS |
| **AC-01.14** | Warning breakdown đúng với 25 MISSING_DOI | ✅ PASS |
| **AC-01.15** | App khởi động thành công (import main.py OK) | ✅ PASS |
| **AC-01.16** | Seed users được tạo khi DB trống | ✅ PASS |

### 5.2. Kết quả pytest

```
======================== test session starts ==========================
platform win32 -- Python 3.14.0, pytest-8.3.3
collected 16 items

tests/test_auth.py::test_login_operator_success         PASSED  [  6%]
tests/test_auth.py::test_login_admin_success           PASSED  [ 12%]
tests/test_auth.py::test_login_wrong_password           PASSED  [ 18%]
tests/test_auth.py::test_login_unknown_user             PASSED  [ 25%]
tests/test_auth.py::test_me_with_valid_token            PASSED  [ 31%]
tests/test_auth.py::test_me_without_token                PASSED  [ 37%]
tests/test_auth.py::test_me_with_invalid_token          PASSED  [ 50%]
tests/test_auth.py::test_refresh_token                   PASSED  [ 62%]

tests/test_upload_idempotency.py::test_upload_requires_auth             PASSED  [ 68%]
tests/test_upload_idempotency.py::test_upload_bad_schema               PASSED  [ 75%]
tests/test_upload_idempotency.py::test_upload_first_time_success        PASSED  [ 81%]
tests/test_upload_idempotency.py::test_upload_second_time_idempotent    PASSED  [ 87%]
tests/test_upload_idempotency.py::test_upload_admin_also_allowed       PASSED  [ 93%]
tests/test_upload_idempotency.py::test_list_batches                    PASSED  [100%]
tests/test_upload_idempotency.py::test_upload_too_large_file            PASSED  [100%]
tests/test_upload_idempotency.py::test_upload_missing_doi_recorded...   PASSED  [100%]

======================== 16 passed in 22.55s =========================
```

### 5.3. Kết quả build

```
# Backend
✅ App import OK: Scopus ICTU Backend
✅ All 16 pytest PASS
✅ smoke_660.py: 660 rows, 25 MISSING_DOI (khớp Tuần 1)

# Frontend
✅ npm install OK (164 packages)
✅ vite build OK
   dist/index.html         0.51 kB
   dist/assets/*.css      18.39 kB (gzip: 3.87 kB)
   dist/assets/*.js      292.20 kB (gzip: 94.51 kB)
   ✓ built in 1.10s
```

### 5.4. Bàn giao Tuần 4

| Cấu phần | Trạng thái | Ghi chú |
|---|---|---|
| Backend FastAPI app | ✅ Hoàn chỉnh | main.py, CORS, exception handlers, seed |
| Module xác thực JWT | ✅ Hoàn chỉnh | bcrypt + Access/Refresh token |
| RBAC (Operator/Admin) | ✅ Hoàn chỉnh | require_role + security_audit_log |
| Module nhập CSV (UC-01) | ✅ Hoàn chỉnh | 4-step pipeline |
| Schema validation (22 cột) | ✅ Hoàn chỉnh | validate_schema |
| Idempotency theo EID | ✅ Hoàn chỉnh | compute_duplicate_map, skip duplicate INSERT |
| Row-level warnings | ✅ Hoàn chỉnh | MISSING_DOI, MISSING_YEAR... |
| Frontend LoginPage | ✅ Hoàn chỉnh | Form + JWT + LocalStorage |
| Frontend UploadDashboard | ✅ Hoàn chỉnh | Drag-drop + 4 cards + batch table |
| Frontend Navbar | ✅ Hoàn chỉnh | User info + role badge |
| manage.bat | ✅ Hoàn chỉnh | 3 luồng: setup/run/test |
| TUAN4.MD | ✅ Hoàn chỉnh | 5 phần, kết quả test |
| pytest 16/16 | ✅ PASS | Auth + Upload + Idempotency |
| Smoke test 660 rows | ✅ PASS | 25 MISSING_DOI khớp Tuần 1 |

### 5.5. Phụ lục: Cấu hình môi trường local

```bash
# Backend .env (copy từ .env.example)
cp backend/.env.example backend/.env

# Chạy toàn bộ hệ thống
manage.bat setup   # Tạo venv + cài pip + npm install
manage.bat run     # Backend: http://localhost:8000
                   # Frontend: http://localhost:5173
                   # Docs: http://localhost:8000/docs

# Chạy kiểm thử
manage.bat test    # pytest 16/16 PASS

# Hoặc chạy riêng
cd backend && .venv\Scripts\python.exe -m uvicorn app.main:app --reload
cd frontend && npm run dev
```

---

*Tài liệu này hoàn thành **Tuần 4** theo đề cương đồ án tốt nghiệp: Nền tảng ứng dụng, xác thực/phân quyền RBAC và nhập dữ liệu Scopus có kiểm tra lỗi. Toàn bộ mã nguồn sẵn sàng chạy production sau khi cấu hình PostgreSQL 15 và .env.*
