# HƯỚNG DẪN SỬ DỤNG
## Hệ thống Quản lý và Chuẩn hóa Hồ sơ Công bố Khoa học Scopus — ICTU

> **Đồ án tốt nghiệp:** Xây dựng Hệ thống Quản lý và Chuẩn hóa Hồ sơ Công bố Khoa học Scopus cho Trường Đại học Công nghệ Thông tin và Truyền thông — Đại học Thái Nguyên (ICTU)
> **Sinh viên thực hiện:** Trương Quang Huy — MSSV: DTC235200899 — Lớp: CN CNTT K22A
> **Giảng viên hướng dẫn:** TS. Nguyễn Thế Vinh — Bộ môn Công nghệ phần mềm, Khoa Công nghệ thông tin (ICTU — vinhnt@ictu.edu.vn)
> **Phiên bản tài liệu:** 1.0 (Tuần 10) — đi kèm mã nguồn `scopus-ictu` phiên bản `0.4.0`
> **Trạng thái kiểm chứng:** ✅ Backend `pytest` **262/262 PASS** · Frontend `npm run build` **exit = 0**

---

## MỤC LỤC

| Phần | Nội dung |
|---|---|
| **§1** | Tổng quan hệ thống & kiến trúc (mục tiêu, sơ đồ, công nghệ, vai trò, cấu trúc thư mục) |
| **§2** | Danh mục yêu cầu chức năng FR/NFR — bảng đối chiếu yêu cầu ↔ API ↔ màn hình |
| **§3** | Cài đặt & vận hành (`manage.bat`, `.env`, cơ chế PostgreSQL ↔ SQLite, kiểm thử, xử lý sự cố) |
| **§4** | Quy trình nghiệp vụ 5 bước (end-to-end) + các chức năng nâng cao |
| **§5** | Kịch bản demo trước Hội đồng (timeline 10 phút, phím tắt, bằng chứng SHA-256, Q&A) |
| **Phụ lục A** | Bảng phân bổ 262 test case backend |
| **Phụ lục B** | Từ điển khái niệm & bảng tra lỗi nhanh |

---

## §1. TỔNG QUAN HỆ THỐNG & KIẾN TRÚC

### 1.1. Mục tiêu & phạm vi

Hệ thống số hóa toàn bộ quy trình **tiếp nhận — chuẩn hóa — đối soát — phê duyệt —
thống kê — xuất báo cáo** dữ liệu công bố khoa học Scopus của ICTU:

1. **Nạp dữ liệu (Stage 1):** Upload tệp CSV kết xuất từ Scopus (22 cột bắt buộc,
   tối đa 50 MB), kiểm tra schema, chống trùng lặp theo `EID` + `Author ID`.
2. **Chuẩn hóa & đối soát (Stage 2):** Parse chuỗi tác giả → chuẩn hóa tiếng Việt
   (bỏ dấu) → so khớp tuyệt đối theo Scopus Author ID → so khớp mờ
   (Jaro-Winkler) → tính **Match Score** có trọng số → phân nhóm
   `MATCH_HIGH / MATCH_MEDIUM / PENDING`.
3. **Phê duyệt có kiểm toán (Stage 3):** Hàng đợi phê duyệt + màn hình **Diff
   View**, 3 lựa chọn duyệt (Phê duyệt liên kết / Gán thủ công / Giữ chờ xác
   minh), hoàn tác trong 72 giờ; mọi thao tác ghi `AUDIT_LOG` chỉ-INSERT.
4. **Thống kê & xuất báo cáo (Stage 4):** Dashboard KPI, biểu đồ xu hướng,
   bộ lọc đa tiêu chí, tìm kiếm toàn văn, xuất **Excel (4 sheet) / CSV (UTF-8 BOM)
   / JSON**, mỗi file export được **ký số SHA-256** và ghi `EXPORT_AUDIT_LOG`.
5. **Xử lý bất đồng bộ (Tuần 9):** Các tác vụ nặng chạy qua **Background Task
   Queue** (`background_tasks`), giám sát tiến độ realtime qua **SSE**, có thể
   hủy cooperative / chạy lại / tải kết quả — HTTP request không bị block.

**Đối tượng sử dụng:** 2 vai trò — **Operator** (cán bộ khoa/phòng, người nạp và
tra cứu dữ liệu) và **Admin** (người duyệt, quản trị danh mục & kiểm toán).

### 1.2. Kiến trúc tổng thể

```mermaid
flowchart LR
  subgraph Browser["Trình duyệt (React 18 + Vite 5)"]
    UI["DashboardPage · UploadDashboard<br/>PublicationsPage · QueuePage<br/>ExportPage · LecturersPage<br/>AuditLogPage · TasksPage"]
    CTX["AuthContext · TaskMonitorContext · ToastContext"]
    UI --- CTX
  end

  subgraph API["FastAPI 0.110+ (uvicorn :8000)"]
    R1["/auth · /uploads · /pipeline"]
    R2["/approval-queue · /standardized-data"]
    R3["/lecturers · /exports · /analytics"]
    R4["/audit-logs · /tasks (SSE)"]
  end

  subgraph Worker["Background Worker (threadpool / celery)"]
    TM["TaskRepo + ProgressEventBus"]
    HD["Handler registry: CSV_INGESTION · MATCHING_PIPELINE<br/>AUTO_RECOMPUTE · BULK_EXPORT"]
    TM --> HD
  end

  subgraph Data["Database"]
    PG[("PostgreSQL 15 — schema scopus_ictu<br/>(fallback: SQLite scopus_dev.db)")]
    FS[/"_uploads · _exports"/]
  end

  CTX -->|"REST (JWT Bearer)"| API
  API --> TM
  TM --> PG
  HD --> PG
  HD --> FS
  TM -->|"SSE: snapshot/progress/status/close"| CTX
  API --> PG
```

**Nguyên tắc kiến trúc đang áp dụng trong mã nguồn:**

| Nguyên tắc | Biểu hiện trong dự án |
|---|---|
| *Áo trắng — áo vàng* (không hardcode) | Mọi cấu hình (port, DB URL, secret, trần upload) đọc từ `backend/.env` qua `pydantic-settings` (`backend/app/core/config.py`) |
| Một nguồn dữ liệu duy nhất cho UI | `TaskMonitorContext` là **nguồn duy nhất** cho trạng thái tác vụ nền (Navbar, Dashboard, TasksPage, Drawer, modal đều đọc chung) |
| Tách nghiệp vụ khỏi hạ tầng | Worker `threadpool (local)` ↔ `celery (production)` thay thế được mà không sửa handler |
| Kiểm toán bất biến | `audit_log` / `security_audit_log` / `export_audit_log` chỉ INSERT (NFR-02) |
| Đồng nhất giao diện | Toàn bộ Box/Grid dùng bộ class chuẩn khai báo tại `frontend/src/index.css`: `.app-container`, `.app-card`, `.app-kpi-card`, `.app-table-card`, `.app-chart-box`, … |

### 1.3. Cấu trúc thư mục dự án

```text
scopus-ictu/
├── manage.bat                     # Menu điều khiển: setup | run | test | help
├── package-lock.json              # Lock dependencies (gốc workspace)
├── TUAN5.MD … TUAN9.MD            # Báo cáo tiến độ theo tuần (5 → 9)
├── HUONG_DAN_SU_DUNG.MD           # ← Tài liệu này
│
├── backend/                       # FastAPI + SQLAlchemy
│   ├── .env / .env.example        # Cấu hình (DB, JWT, upload, seed user)
│   ├── pytest.ini                 # testpaths = tests
│   ├── requirements.txt           # fastapi, sqlalchemy, pandas, openpyxl, pytest…
│   ├── scopus_dev.db              # CSDL SQLite dùng khi fallback (dev)
│   ├── _uploads/                  # CSV gốc đã tải lên (lưu theo <năm>/<tháng>/<ngày>)
│   ├── _exports/                  # File báo cáo do task BULK_EXPORT sinh ra
│   ├── app/
│   │   ├── main.py                # Khởi tạo FastAPI, CORS, exception handler
│   │   ├── database.py            # Engine + auto-fallback PostgreSQL → SQLite
│   │   ├── core/                  # config.py (settings), security.py (JWT/bcrypt)
│   │   ├── models/                # 11 bảng: users, upload_batches, raw_scopus_data,
│   │   │                          # lecturers, author_extracted, approval_queue,
│   │   │                          # standardized_data, audit_log, export_audit_log,
│   │   │                          # security_audit_log, background_tasks
│   │   ├── schemas/               # Pydantic DTO (request/response)
│   │   ├── services/              # ingestion, author extractor, matching engine,
│   │   │                          # approval workflow, export service,
│   │   │                          # analytics, task_manager, task_handlers
│   │   └── api/v1/                # auth, uploads, pipeline, approvals, lecturers,
│   │                              # exports, analytics, audit, tasks
│   └── tests/                     # 13 file test — 262 test case
│
├── frontend/                      # React 18 + Vite 5 + TailwindCSS 3
│   ├── .env / .env.example        # VITE_API_BASE_URL, VITE_APP_NAME…
│   ├── package.json               # dev | build | preview
│   └── src/
│       ├── main.jsx               # Bọc ToastProvider → AuthProvider → TaskMonitorProvider
│       ├── App.jsx                # Router + PrivateRoute / AdminRoute + app-container
│       ├── index.css              # Tailwind + bộ class Box/Grid chuẩn dùng chung
│       ├── api/client.js          # Axios instance + toàn bộ API wrapper
│       ├── contexts/              # AuthContext, TaskMonitorContext, ToastContext
│       ├── hooks/useTaskStream.js # EventSource cho SSE
│       ├── components/            # Navbar, TaskProgressDrawer, BackgroundTasksCard,
│       │                          # DiffViewModal, DragDropUpload, Navbar…
│       └── pages/                 # LoginPage, DashboardPage, UploadDashboard,
│                                  # PublicationsPage, QueuePage, ExportPage,
│                                  # LecturersPage, AuditLogPage, TasksPage
│
├── scripts/dev_orchestrator.py    # Chạy song song Backend + Frontend trong 1 cửa sổ
└── test/                          # Dữ liệu & tài liệu phục vụ demo/đồ án
    ├── Truong_Quang_Huy_Scopus_Sep2026.csv    # Mẫu 660 bản ghi Scopus (1 header + 660 dòng)
    ├── docs/scopus-ictu/                      # SRS, schema.md, BPMN, TUAN3-4.MD
    └── imageUML/                              # Sơ đồ use case / quy trình nghiệp vụ
```

### 1.4. Công nghệ sử dụng

| Lớp | Công nghệ | Phiên bản / ghi chú |
|---|---|---|
| Frontend | React + Vite | `react ^18.3.1`, `vite ^5.4.10` |
| | Định tuyến / UI | `react-router-dom ^6.27.0`, `tailwindcss ^3.4.14` |
| | Biểu đồ & HTTP | `recharts ^3.10.1`, `axios ^1.7.7`, `react-dropzone ^14.2.10` |
| | Realtime | `EventSource` (SSE) qua hook `useTaskStream` |
| Backend | FastAPI + Uvicorn | `fastapi>=0.110`, `uvicorn[standard]>=0.27` |
| | ORM / CSDL | `SQLAlchemy>=2.0.25`, `psycopg2-binary` (PostgreSQL 15, schema `scopus_ictu`), SQLite cho dev |
| | Xác thực | `python-jose[cryptography]` (JWT HS256) + `passlib[bcrypt]` |
| | Xử lý dữ liệu | `pandas`, `openpyxl` (Excel 4 sheet), `rapidfuzz` (Jaro-Winkler — FR-11) |
| | Cấu hình | `pydantic>=2.6` + `pydantic-settings` (đọc `backend/.env`) |
| Kiểm thử | `pytest>=8.0` + `httpx` (TestClient) | 262 test case / 13 file |
| Vận hành | `manage.bat`, `scripts/dev_orchestrator.py` | Windows 10/11, Python 3.11+ (đã kiểm chứng trên 3.14), Node 18+ |

### 1.5. Vai trò, quyền hạn & tài khoản demo

| Vai trò | Quyền hạn chính | Menu hiển thị |
|---|---|---|
| **Operator** | Upload CSV, xem Dashboard/Tra cứu, xuất báo cáo, theo dõi tác vụ nền **của mình** | Dashboard · Upload CSV · Báo cáo & Xuất dữ liệu · *Tra cứu & Đối soát* (Công bố, Hàng đợi) |
| **Admin** | Toàn bộ quyền của Operator + **duyệt / gán thủ công / hoàn tác**, CRUD danh mục giảng viên, xem Audit Log, xem & hủy **mọi** tác vụ nền | Thêm mục *Quản trị* (Giảng viên, Audit Log, Tiến trình nền) |

**Tài khoản seed sẵn (khai báo trong `backend/.env`):**

| Tài khoản | Mật khẩu | Vai trò | Dùng để demo |
|---|---|---|---|
| `qhun22` | `Quanghuyso1@` | Operator | Luồng nạp dữ liệu, tra cứu, xuất báo cáo |
| `admin` | `Quanghuyso1#` | Admin | Luồng phê duyệt, hoàn tác, kiểm toán, quản trị tác vụ |

> ⚠️ Hai tài khoản trên chỉ dùng cho **môi trường development/demo**. Khi triển khai
> thật phải đổi mật khẩu, thay `JWT_SECRET_KEY` và đặt `APP_ENV=production`.

---

## §2. DANH MỤC YÊU CẦU CHỨC NĂNG & ĐỐI CHIẾU TRIỂN KHAI

### 2.1. Yêu cầu chức năng (FR-01 → FR-29)

> **Chuẩn đối chiếu:** bộ số FR/NFR chính thức trong SRS
> (`test/docs/scopus-ictu/srs/scopus-ictu-requirements-actors.md`).
> Mã nguồn có vài chỗ ghi tag ngắn theo tuần triển khai (ví dụ `exports.py` ghi
> `FR-24..FR-27`, `analytics.py` ghi `FR-29`) — cột **“Module mã nguồn”** bên dưới
> giúp tra ngược chính xác vị trí cài đặt.

**Nhóm A — Nạp dữ liệu & kiểm soát chất lượng (Stage 1)**

| Mã | Tên yêu cầu | Màn hình / API | Module mã nguồn | TT |
|---|---|---|---|---|
| **FR-01** | Tải tệp Scopus CSV (kéo-thả hoặc chọn file, ≤ 50 MB) | `UploadDashboard` → `POST /api/v1/uploads` | `services/ingestion`, `components/DragDropUpload.jsx` | ✅ |
| **FR-02** | Kiểm tra cấu trúc 22 cột bắt buộc + encoding UTF-8 | `POST /api/v1/uploads` (422 JSON chi tiết) | `core/config.py::UPLOAD_REQUIRED_COLUMNS` | ✅ |
| **FR-03** | Kiểm tra Idempotency theo `EID` + `Author ID`, gắn `duplicate = true` | `POST /api/v1/uploads` (lần 2 trở đi) | `services/ingestion`, `models/uploads.py` | ✅ |
| **FR-04** | Lưu trữ `RAW_DATA` nguyên vẹn 22 trường + `batch_id`/`operator_id` | `GET /api/v1/uploads/batches` | `models/raw_scopus_data` | ✅ |
| **FR-05** | Validation cấp dòng: thiếu DOI / Author ID / Year / Affiliation → gắn cảnh báo, **không dừng luồng** | `UploadDashboard` (thẻ cảnh báo) + `GET /api/v1/pipeline/publications` | `services/ingestion` (`warning_breakdown`) | ✅ |
| **FR-06** | Trả JSON lỗi cấu trúc (danh sách cột thiếu/sai) để UI hiển thị | `POST /api/v1/uploads` → 422 | `api/v1/uploads.py`, `core/exception_handlers` | ✅ |

**Nhóm B — Chuẩn hóa & đối soát (Stage 2)**

| Mã | Tên yêu cầu | Màn hình / API | Module mã nguồn | TT |
|---|---|---|---|---|
| **FR-07** | Parse `Authors` + `Author(s) ID` thành danh sách tác giả có cấu trúc | `POST /api/v1/pipeline/process-batch/{batch_id}` | `services/author_extractor` | ✅ |
| **FR-08** | Chuẩn hóa tên tiếng Việt (bỏ dấu, lower, gộp khoảng trắng) | Pipeline nội bộ | `services/matching_engine` (`normalize_vietnamese`) | ✅ |
| **FR-09** | Đếm số tác giả/bài, lưu `AUTHOR_EXTRACTED` | `GET /api/v1/pipeline/status/{batch_id}` | `models/author_extracted` | ✅ |
| **FR-10** | So khớp tuyệt đối theo Scopus Author ID (ưu tiên cao nhất) | Pipeline nội bộ | `services/matching_engine` | ✅ |
| **FR-11** | So khớp mờ (Levenshtein / Jaro-Winkler) khi không có Author ID | Pipeline nội bộ | `services/matching_engine` + `rapidfuzz` | ✅ |
| **FR-12** | Tính Match Score ∈ [0,1] (Author ID 0,6 · Tên 0,3 · Affiliation 0,1) + `evidence_payload` | `QueuePage` (cột Score) | `services/matching_engine` | ✅ |
| **FR-13** | Phân nhóm `MATCH_HIGH` ≥ 85 % · `MATCH_MEDIUM` 50–85 % · `PENDING` < 50 % | `QueuePage` (badge HIGH/MEDIUM/PENDING) | `services/matching_engine` | ✅ |
| **FR-14** | Ghi hàng đợi phê duyệt `APPROVAL_QUEUE`, **không tự động gộp** khi PENDING | `GET /api/v1/approval-queue` | `models/approval_queue` | ✅ |
| **FR-15** | Ghi log pipeline (`correlation_id`, `log_level`, `raw_id`, `score`) | Log server + Audit Log | `services/task_manager` (`correlation_id`) | ✅ |

**Nhóm C — Phê duyệt có kiểm toán (Stage 3)**

| Mã | Tên yêu cầu | Màn hình / API | Module mã nguồn | TT |
|---|---|---|---|---|
| **FR-16** | Mở Approval Queue dạng **Diff View** (raw ↔ suggested + `evidence_payload`) | `QueuePage` → `GET /api/v1/approval-queue/{queue_id}/diff` | `components/DiffViewModal.jsx` | ✅ |
| **FR-17** | Phê duyệt liên kết → ghi `STANDARDIZED_DATA` + `AUDIT_LOG(action=APPROVE)` | Nút **[Phê duyệt liên kết]** / phím `A` → `POST …/approve` | `services/approval_workflow` | ✅ |
| **FR-18** | Gán tay giảng viên khác → `AUDIT_LOG(action=MANUAL_OVERRIDE, before/after)` | Nút **[Gán thủ công]** → `POST …/manual-assign` | `services/approval_workflow` | ✅ |
| **FR-19** | Giữ chờ xác minh → `APPROVAL_QUEUE.status = PENDING_VERIFY`, `action=DEFER` | Nút **[Giữ chờ xác minh]** → `POST …/defer` | `services/approval_workflow` | ✅ |
| **FR-20** | Hoàn tác (Rollback) trong cửa sổ **72 giờ** | Nút **[Rollback]** → `POST /standardized-data/{id}/rollback` | `APPROVAL_ROLLBACK_WINDOW_HOURS=72` | ✅ |
| **FR-21** | Ghi nhật ký kiểm toán bất biến (không UPDATE/DELETE) | `AuditLogPage` → `GET /api/v1/audit-logs`, `/audit-logs/security` | `models/audit_log`, `services/audit` | ✅ |
| **FR-22** | CRUD Danh mục Giảng viên ICTU (thêm/sửa/xóa, mỗi thao tác có audit) | `LecturersPage` → `GET/POST/PUT/DELETE /api/v1/lecturers` | `api/v1/lecturers.py`, `components/LecturerModal.jsx` | ✅ |

**Nhóm D — Lọc, tra cứu, thống kê & xuất báo cáo (Stage 4)**

| Mã | Tên yêu cầu | Màn hình / API | Module mã nguồn | TT |
|---|---|---|---|---|
| **FR-23** | Lọc dữ liệu đa tiêu chí (Năm, Khoa, Publisher Family, Trạng thái duyệt) | `PublicationsPage`/`QueuePage` → `GET /api/v1/pipeline/publications` | `api/v1/pipeline.py` | ✅ |
| **FR-24** | Tìm kiếm toàn văn trên Title / Keywords / Abstract (không dấu) | Ô “Tìm kiếm” trên `PublicationsPage`, `QueuePage` | `services/*` + điều kiện `ILIKE`/`q` | ✅ |
| **FR-25** | Dashboard thống kê tổng quan (KPI, xu hướng năm, phân bố Khoa/trạng thái, top GV) | `DashboardPage` → `GET /api/v1/analytics/*` | `services/analytics_service.py` (tag nội bộ `FR-29`) | ✅ |
| **FR-26** | Xuất Excel `.xlsx` **4 sheet** (Summary, Raw, Standardized, Audit Trail) | `ExportPage` → `POST /api/v1/exports/excel` | `services/export_service.py` (tag `FR-24`) | ✅ |
| **FR-27** | Xuất CSV **UTF-8 BOM** (mở được bằng Excel) | `POST /api/v1/exports/csv` | `services/export_service.py` (tag `FR-25`) | ✅ |
| **FR-28** | Xuất JSON theo JSON Schema (Draft 2020-12) | `POST /api/v1/exports/json` | `services/export_service.py` (tag `FR-26`) | ✅ |
| **FR-29** | **Ký số SHA-256 + log export** (`actor_id`, `timestamp`, `row_count`, `file_hash`) | `POST /api/v1/exports/task` + `GET /exports/task/{id}/download`, lịch sử `GET /exports/history` | `models/export_audit_log` | ✅ |

### 2.2. Yêu cầu phi chức năng & tiêu chí kiểm chứng

| Mã | Tên yêu cầu | Tiêu chí nghiệm thu | Cách kiểm chứng trong repo | TT |
|---|---|---|---|---|
| **NFR-01** | Toàn vẹn & Idempotency | Re-import cùng lô EID nhiều lần → **0 bản ghi mới**, toàn bộ bị gắn `duplicate` | `tests/test_upload_idempotency.py` (8 test) | ✅ |
| **NFR-02** | Khả năng kiểm toán | 100 % thao tác duyệt/gán/defer/rollback/export có row audit; audit chỉ INSERT | `tests/test_audit_viewer.py` (16), `tests/test_approvals.py` (16) | ✅ |
| **NFR-03** | Hiệu năng | Pipeline 660 bản ghi trong thời gian cho phép; tác vụ nặng chuyển sang nền | `tests/test_pipeline_api.py`, `tests/test_recompute.py` | ✅ |
| **NFR-04** | Bảo mật & phân quyền (RBAC) | Operator gọi API `admin.*` → **403** và ghi `security_audit_log` | `tests/test_security_rbac.py` (24) | ✅ |
| **NFR-05** | Tương thích trao đổi dữ liệu | `.xlsx` mở bằng Excel 2019+; `.csv` UTF-8 BOM; `.json` đúng schema | `tests/test_exports.py` (48) | ✅ |
| **NFR-06** | *Xử lý bất đồng bộ (Tuần 9)* | Endpoint nặng trả **202 + `task_id`** trong < 200 ms; tiến độ realtime qua SSE; hủy cooperative | `tests/test_tasks.py` (40) + `TasksPage`/`TaskProgressDrawer` | ✅ |

### 2.3. Ma trận màn hình ↔ chức năng ↔ quyền truy cập

| Route | Màn hình | Chức năng chính | Operator | Admin |
|---|---|---|---|---|
| `/login` | `LoginPage` | Đăng nhập JWT (access token 30 phút · refresh 7 ngày) | ✅ | ✅ |
| `/dashboard` | `DashboardPage` | FR-25: 4 thẻ KPI, biểu đồ xu hướng năm, phân bổ trạng thái, top GV, phân bổ Khoa + thẻ **Tác vụ nền** (tự làm mới 30 s) | ✅ | ✅ |
| `/upload` | `UploadDashboard` | FR-01→06: kéo-thả CSV, kiểm tra schema, cảnh báo dòng, chế độ “Nhập chạy nền” (task `CSV_INGESTION`) | ✅ | ✅ |
| `/publications` | `PublicationsPage` | FR-23/24: lọc đa tiêu chí + tìm kiếm, xem chi tiết bài báo, kích hoạt `MATCHING_PIPELINE` | ✅ | ✅ |
| `/queue` | `QueuePage` | FR-16→21: mở **Diff View**, duyệt / gán tay / giữ chờ / rollback | ➖ *(bị chuyển hướng)* | ✅ |
| `/exports` | `ExportPage` | FR-26→29: xuất Excel/CSV/JSON, xuất nền (`BULK_EXPORT`) và **tải file + kiểm tra SHA-256** | ✅ | ✅ |
| `/lecturers` | `LecturersPage` | FR-22: CRUD danh mục giảng viên ICTU | ➖ | ✅ |
| `/audit-log` | `AuditLogPage` | FR-21: tra cứu nhật ký nghiệp vụ + nhật ký bảo mật | ➖ | ✅ |
| `/tasks` | `TasksPage` | NFR-06: 5 thẻ KPI, bảng hiệu năng theo loại, filter, tự refresh 3 s, modal chi tiết có tiến độ live | ✅ *(chỉ task của mình)* | ✅ *(toàn bộ)* |

### 2.4. Bảng API chính (tiền tố `/api/v1`)

| Nhóm | Endpoint | Mô tả | Quyền |
|---|---|---|---|
| **auth** | `POST /auth/login` · `POST /auth/refresh` · `GET /auth/me` | Cấp & làm mới JWT, lấy hồ sơ người dùng hiện tại | Login: public |
| **uploads** | `POST /uploads` | Nạp CSV đồng bộ (multipart) | Mọi user đã đăng nhập |
| | `POST /uploads/task` | Nạp CSV **bất đồng bộ** → `202 + task_id` | Mọi user đã đăng nhập |
| | `GET /uploads/batches` | Danh sách lô đã nạp + thống kê cảnh báo | Mọi user |
| **pipeline** | `POST /pipeline/process-batch/{batch_id}` · `POST /pipeline/process-batch/{batch_id}/async` | Chạy chuẩn hóa & đối soát (đồng bộ / nền) | Mọi user |
| | `GET /pipeline/status/{batch_id}` · `GET /pipeline/publications` · `GET /pipeline/publications/{raw_id}` | Tiến độ xử lý, danh sách & chi tiết công bố | Mọi user |
| | `POST /pipeline/recompute-pending` · `POST /pipeline/recompute-pending/async` | Tính lại các bản ghi PENDING (nền) | **Admin** |
| **approval** | `GET /approval-queue` | Danh sách hàng đợi (filter/paging) | **Admin** |
| | `GET /approval-queue/{queue_id}/diff` | Dữ liệu Diff View + `evidence_payload` | **Admin** |
| | `POST /approval-queue/{id}/approve` · `/manual-assign` · `/defer` | 3 lựa chọn duyệt | **Admin** |
| | `POST /standardized-data/{id}/rollback` | Hoàn tác trong 72 giờ | **Admin** |
| **lecturers** | `GET /lecturers` · `GET /lecturers/options` · `GET /lecturers/{id}` | Tra cứu danh mục giảng viên | Mọi user |
| | `POST /lecturers` · `PUT /lecturers/{id}` · `DELETE /lecturers/{id}` | Thêm / sửa / xóa giảng viên | **Admin** |
| **exports** | `POST /exports/excel` · `/csv` · `/json` | Xuất báo cáo trực tiếp (file trả về ngay) | Mọi user |
| | `POST /exports/task` · `GET /exports/task/{task_id}/download` | Xuất **nền** (`BULK_EXPORT`) + tải file kết quả (404/403/409/410) | Mọi user |
| | `GET /exports/history` | Lịch sử export kèm `file_hash` SHA-256 | Mọi user |
| **analytics** | `GET /analytics/overview` · `/trends-by-year` · `/distribution-by-faculty` · `/matching-distribution` · `/top-lecturers` | Dữ liệu Dashboard (FR-25) | Mọi user |
| **audit** | `GET /audit-logs` · `GET /audit-logs/security` | Nhật ký nghiệp vụ & nhật ký bảo mật (RBAC attempts) | **Admin** |
| **tasks** | `GET /tasks` · `GET /tasks/{task_id}` · `GET /tasks/stats/summary` | Danh sách / chi tiết / thống kê tác vụ nền | Xem theo phạm vi vai trò |
| | `POST /tasks/{task_id}/cancel` | Hủy cooperative | Chủ tác vụ hoặc Admin |
| | `GET /tasks/{task_id}/stream` | **SSE** realtime: `snapshot` · `progress` · `status` · `ping` · `close` | Token qua query `?token=` |

> 📚 **Tài liệu API tương tác:** mở `http://localhost:8000/docs` (Swagger UI) sau khi
> chạy backend — mọi endpoint đều có mô tả, schema request/response và nút “Try it out”.

---

## §3. CÀI ĐẶT & VẬN HÀNH

### 3.1. Yêu cầu môi trường

| Thành phần | Yêu cầu tối thiểu | Kiểm tra |
|---|---|---|
| Hệ điều hành | Windows 10/11 (script `manage.bat` dùng `cmd` + PowerShell) | — |
| Python | **3.11+** (đã kiểm chứng trên 3.14) | `python --version` |
| Node.js + npm | Node 18+ (khuyến nghị 20 LTS) | `node -v`, `npm -v` |
| PostgreSQL | 15 (tùy chọn — có thể chạy bằng SQLite fallback) | `psql --version` |
| Dung lượng | ~1 GB cho `node_modules` + `.venv` | — |

### 3.2. Cài đặt bằng `manage.bat`

Chạy **double-click** `manage.bat` tại thư mục gốc dự án để mở menu tương tác:

```text
[1] SETUP   - Khoi tao moi truong (venv, pip, npm install)
[2] RUN     - Chay backend (FastAPI) + frontend (Vite)
[3] TEST    - Chay pytest backend
[4] HELP    - Hien thi huong dan
[5] EXIT    - Thoat
```

Hoặc dùng tham số dòng lệnh (phù hợp khi thao tác nhanh trước buổi bảo vệ):

```bat
manage.bat setup     :: cài đặt môi trường
manage.bat run       :: chạy đồng thời Backend + Frontend
manage.bat test      :: chạy pytest
manage.bat help      :: xem hướng dẫn
```

**`setup` thực hiện tuần tự:**

1. **Backend:** tạo `backend\.venv` (nếu chưa có) → nâng cấp `pip` → cài
   `backend\requirements.txt` → cài thêm `httpx`, `pytest`.
2. **Frontend:** chạy `npm install` trong `frontend\` nếu chưa có `node_modules`.
3. **Cấu hình:** copy `backend\.env.example → backend\.env` và
   `frontend\.env.example → frontend\.env` nếu file `.env` chưa tồn tại
   (**không ghi đè** file đang có).

> 💡 Cài thủ công (khi không dùng `manage.bat`):
> ```bat
> cd backend
> python -m venv .venv
> .venv\Scripts\python.exe -m pip install -r requirements.txt
> cd ..\frontend
> npm install
> ```

### 3.3. Chạy hệ thống

```bat
manage.bat run
```

`manage.bat` gọi `scripts\dev_orchestrator.py`, chạy **cả backend và frontend trong
một cửa sổ CMD duy nhất** (bấm `Ctrl+C` để tắt cả hai tiến trình):

| Dịch vụ | Địa chỉ | Ghi chú |
|---|---|---|
| Backend API | `http://localhost:8000` | Uvicorn + `--reload` (chỉ dev) |
| Swagger UI | `http://localhost:8000/docs` | Xem & thử trực tiếp mọi endpoint |
| Frontend | `http://localhost:5173` | Vite dev server (HMR) |

Luồng khởi động của backend (`app/main.py`):

1. Đọc `backend/.env` → resolve URL CSDL (có probe PostgreSQL, xem §3.5).
2. `Base.metadata.create_all(engine)` — tự đồng bộ bảng (SQLite luôn chạy được;
   PostgreSQL cần database + schema `scopus_ictu` đã tồn tại, đặc tả DDL tham khảo
   `test/docs/scopus-ictu/database/schema.md`).
3. Seed 2 tài khoản `qhun22` / `admin` nếu CSDL trống.
4. Đăng ký router `/api/v1/*`, bật CORS cho `http://localhost:5173`.

### 3.4. Cấu hình tệp `.env`

**`backend/.env`** — các biến quan trọng nhất:

| Biến | Giá trị mẫu | Ý nghĩa |
|---|---|---|
| `APP_ENV` | `development` | `development` \| `staging` \| `production` — quyết định cơ chế fallback CSDL |
| `HOST` / `PORT` | `0.0.0.0` / `8000` | Địa chỉ & cổng backend |
| `JWT_SECRET_KEY` | `MUST_OVERRIDE_…` | **Bắt buộc đổi** khi deploy thật |
| `JWT_ACCESS_TOKEN_EXPIRE_MINUTES` | `30` | Hạn access token |
| `JWT_REFRESH_TOKEN_EXPIRE_DAYS` | `7` | Hạn refresh token |
| `DATABASE_URL` | `postgresql+psycopg2://app_backend:app_backend_pwd@localhost:5432/scopus_ictu_db` | Chuỗi kết nối PostgreSQL |
| `DATABASE_FALLBACK_SQLITE` | `True` | Cho phép tự chuyển sang SQLite khi PostgreSQL “chết” (chỉ ở `development`) |
| `SQLITE_FALLBACK_PATH` | `sqlite:///./scopus_dev.db` | CSDL SQLite dự phòng (`backend/scopus_dev.db`) |
| `UPLOAD_MAX_FILE_MB` | `50` | Trần dung lượng file CSV (FR-01) |
| `EXPORT_DIR` | `./_exports` | Nơi lưu file báo cáo của task `BULK_EXPORT`. Đường dẫn được `resolve()` theo **thư mục làm việc của backend** ⇒ mặc định là `backend\_exports` |
| `UPLOAD_DIR` *(biến môi trường tùy chọn)* | `./_uploads` | Nơi lưu file CSV gốc; mặc định `backend\_uploads\<năm>\<tháng>\<ngày>\` |
| `EXPORT_RETENTION_HOURS` | `72` | Số giờ giữ file export trước khi dọn |
| `TASK_SSE_KEEPALIVE_SECONDS` | `15` | Chu kỳ gửi `ping` giữ kết nối SSE |
| `APPROVAL_ROLLBACK_WINDOW_HOURS` | `72` | Cửa sổ cho phép Rollback (FR-20) |
| `CORS_ORIGINS` | `["http://localhost:5173", …]` | Danh sách origin được phép gọi API |
| `SEED_OPERATOR_USERNAME` / `SEED_OPERATOR_PASSWORD` | `qhun22` / `Quanghuyso1@` | Tài khoản Operator seed sẵn |
| `SEED_ADMIN_USERNAME` / `SEED_ADMIN_PASSWORD` | `admin` / `Quanghuyso1#` | Tài khoản Admin seed sẵn |

> 🔐 **Lưu ý bảo mật:** file `.env` thật **không được commit** (đã có trong
> `backend/.gitignore`). Khi chuyển sang máy khác chỉ cần copy `.env.example` rồi
> sửa lại các giá trị nhạy cảm.

**`frontend/.env`** — chỉ các biến có tiền tố `VITE_`:

| Biến | Giá trị mẫu | Ý nghĩa |
|---|---|---|
| `VITE_API_BASE_URL` | `http://localhost:8000` | Base URL của backend |
| `VITE_APP_NAME` | `Scopus ICTU` | Tên hiển thị trên Navbar/tiêu đề |
| `VITE_HTTP_TIMEOUT_MS` | `30000` | Timeout HTTP (ms) |

### 3.5. Cơ chế PostgreSQL ↔ SQLite (tự động dự phòng)

Được cài trong `backend/app/database.py::_resolve_database_url()`:

| `DATABASE_URL` | `APP_ENV` | Trạng thái PostgreSQL | Kết quả |
|---|---|---|---|
| bắt đầu bằng `sqlite` | bất kỳ | — | Dùng **SQLite** ngay (chế độ test) |
| PostgreSQL | `test` | — | Dùng đúng URL trong `.env`, **không probe** |
| PostgreSQL | `development` | Probe **thành công** (`SELECT 1`) | Dùng **PostgreSQL** (production-grade) |
| PostgreSQL | `development` | Probe **thất bại** + `DATABASE_FALLBACK_SQLITE=True` | **Tự chuyển SQLite** `backend/scopus_dev.db`, ghi log `WARNING` |
| PostgreSQL | `development` | Probe thất bại + `DATABASE_FALLBACK_SQLITE=False` | `RuntimeError` — dừng khởi động |
| PostgreSQL | `staging`/`production` | Không kết nối được | `RuntimeError` **FATAL** — tuyệt đối không fallback |

**Các tình huống sử dụng thực tế:**

* **Bảo vệ đồ án không có PostgreSQL** (máy hội đồng không cài PostgreSQL): giữ
  nguyên `APP_ENV=development` — hệ thống tự chạy trên `backend\scopus_dev.db`,
  banner log sẽ hiện `[DB] PostgreSQL unreachable ... FALLBACK to SQLite file: …`.
* **Ép buộc dùng SQLite** (kể cả khi PostgreSQL đang chạy): sửa
  `DATABASE_URL=sqlite:///./scopus_dev.db`.
* **Ép buộc dùng PostgreSQL** (muốn phát hiện lỗi cấu hình ngay): đặt
  `DATABASE_FALLBACK_SQLITE=False`.
* Khi chạy SQLite, hệ thống tự **bỏ tiền tố schema** (`scopus_ictu.`) khỏi mọi
  bảng vì SQLite không hỗ trợ multi-schema; dữ liệu và API hoàn toàn giống nhau.

> ⚠️ **Khác biệt cần biết khi demo:** SQLite lưu 1 file duy nhất nên xóa
> `backend\scopus_dev.db` là “reset sạch” toàn bộ dữ liệu — hữu ích khi muốn diễn
> lại kịch bản từ đầu, nhưng cũng có nghĩa dữ liệu demo sẽ mất.

### 3.6. Chạy kiểm thử (bằng chứng chất lượng)

**Backend — 262 test case:**

```bat
manage.bat test
```

hoặc chạy trực tiếp (khuyến nghị, tránh lệ thuộc vào Python hệ thống):

```bat
cd backend
.venv\Scripts\python.exe -m pytest tests -v --tb=short
```

Kết quả mong đợi:

```text
262 passed in ~365s
```

Bằng chứng thô lấy từ `pytest --junitxml`:

```xml
<testsuite name="pytest" errors="0" failures="0" skipped="0" tests="262" ... />
```

**Frontend — build production:**

```bat
cd frontend
npm run build
```

Kết quả mong đợi: `vite v5.x building for production... ✓ 715 modules transformed` và
**exit code = 0**, các chunk `DashboardPage`, `QueuePage`, `TasksPage`,
`UploadDashboard`… được tách lazy-load. Muốn xem thử bản build:
`npm run preview`.

> 💡 Khi máy trạm gặp lỗi quyền với thư mục Temp của Windows, chạy lại pytest với
> thư mục tạm nằm trong dự án:
> ```bat
> .venv\Scripts\python.exe -m pytest tests -q --tb=short -p no:cacheprovider --basetemp "%CD%\_pytest_tmp"
> ```

### 3.7. Xử lý sự cố thường gặp

| Hiện tượng | Nguyên nhân | Cách khắc phục |
|---|---|---|
| `No module named pytest` / `'pytest' is not recognized` | Đang gọi Python hệ thống thay vì venv | Dùng `backend\.venv\Scripts\python.exe -m pytest tests` |
| `PermissionError: [WinError 5] Access is denied: …\Temp\pytest-of-<user>` | Thư mục Temp Windows bị khóa/không đủ quyền | Thêm `--basetemp "%CD%\_pytest_tmp"` như mục §3.6 (đây là lỗi môi trường, không phải lỗi nghiệp vụ) |
| `Port 8000 already in use` / `Port 5173 is in use` | Tiến trình cũ chưa tắt | `netstat -ano \| findstr :8000` → `taskkill /PID <pid> /F`; hoặc đổi `PORT` trong `.env` / `vite --port 5174` |
| `CORS policy: No 'Access-Control-Allow-Origin'` | Frontend chạy ở origin khác `CORS_ORIGINS` | Thêm origin đang dùng vào `CORS_ORIGINS` trong `backend/.env`, khởi động lại backend |
| Đăng nhập xong bị trả về `/login` liên tục | Token cũ/hết hạn trong `localStorage` | DevTools → Application → Local Storage → xóa `access_token`, `refresh_token` → đăng nhập lại |
| `database is locked` (SQLite) | Một tiến trình/DB browser khác đang giữ `scopus_dev.db` | Đóng công cụ đang mở file DB, hoặc tắt backend cũ rồi chạy lại |
| Log `[DB] PostgreSQL unreachable ... FALLBACK to SQLite file` | PostgreSQL chưa chạy | Bình thường trong dev — hệ thống vẫn chạy trên SQLite. Muốn dùng PostgreSQL: khởi động dịch vụ PostgreSQL 15 rồi chạy lại |
| Upload báo `422 – thiếu cột …` | File CSV không đúng cấu trúc xuất từ Scopus | Dùng đúng file kết xuất Scopus (22 cột, có UTF-8); xem danh sách cột bắt buộc trong `UPLOAD_REQUIRED_COLUMNS` |
| Nút upload trả `413/400` | File > 50 MB hoặc sai định dạng | Giảm dung lượng/kích thước, chỉ nhận `.csv` (điều chỉnh `UPLOAD_MAX_FILE_MB` nếu cần) |
| Không thấy tiến độ realtime (SSE) | Kết nối SSE bị proxy/extension chặn | Panel vẫn tự cập nhật bằng polling dự phòng 2,5 s; kiểm tra tab Network xem request `/tasks/{id}/stream` |
| Trang trắng sau `npm run dev` | Lỗi runtime frontend hoặc thiếu `.env` | Mở Console (F12) xem lỗi; tạo `frontend\.env` từ `.env.example` rồi chạy lại |
| `npm install` thất bại do mạng | Kết nối npm registry chập chờn | `npm cache clean --force` rồi `npm install` lại; hoặc dùng `npm ci` khi có `package-lock.json` |

---

## §4. QUY TRÌNH NGHIỆP VỤ 5 BƯỚC (END-TO-END)

```mermaid
flowchart LR
  S1["① ĐĂNG NHẬP<br/>qhun22 / admin"] --> S2["② NẠP CSV SCOPUS<br/>/upload"]
  S2 --> S3["③ CHUẨN HÓA & ĐỐI SOÁT<br/>/publications → matching"]
  S3 --> S4["④ DUYỆT HÀNG ĐỢI<br/>/queue → Diff View"]
  S4 --> S5["⑤ THỐNG KÊ & XUẤT BÁO CÁO<br/>/dashboard · /exports"]
  S5 -.->|"kiểm chứng SHA-256"| S5
```

### Bước 1 — Đăng nhập & định vị giao diện

| # | Thao tác | Kết quả mong đợi |
|---|---|---|
| 1 | Mở `http://localhost:5173` | Trang `LoginPage` hiển thị form đăng nhập |
| 2 | Nhập `qhun22` / `Quanghuyso1@` (Operator) | Chuyển tới `/dashboard`, Navbar hiện avatar + huy hiệu vai trò **Operator** |
| 3 | Quan sát Navbar | **1 hàng duy nhất**: Dashboard · Upload CSV · Báo cáo & Xuất dữ liệu · dropdown *Tra cứu & Đối soát* · nút **Tiến trình nền** · menu người dùng |
| 4 | Thu nhỏ cửa sổ xuống dưới `1024 px` | Navbar tự chuyển sang nút **☰ hamburger**; menu mở dạng ngăn kéo trượt từ phải |
| 5 | Bấm `Esc` hoặc bấm ra ngoài ngăn kéo | Ngăn kéo/dropdown đóng lại ngay |

> 💡 Đăng nhập `admin` / `Quanghuyso1#` để thấy thêm nhóm menu **Quản trị**
> (Giảng viên · Audit Log · Tiến trình nền).

### Bước 2 — Nạp dữ liệu Scopus (FR-01 → FR-06)

| # | Thao tác | Kết quả mong đợi |
|---|---|---|
| 1 | Vào **Upload CSV** | Vùng kéo-thả (`DragDropUpload`) + nút chọn file |
| 2 | Kéo file `test\Truong_Quang_Huy_Scopus_Sep2026.csv` (**660 bản ghi**) vào vùng kéo-thả | Tên file + dung lượng hiển thị, hệ thống kiểm tra đuôi `.csv` |
| 3 | (Tùy chọn) Bật công tắc **“Nhập chạy nền”** | Hệ thống trả `202 + task_id`, thanh tiến độ xuất hiện trong panel **Tiến trình nền** |
| 4 | Bấm **Tải lên & xử lý** | Thẻ kết quả: `inserted`, `duplicate`, `warning_breakdown` (thiếu DOI / Author ID / Year / Affiliation) |
| 5 | Bấm **Tải lên lại đúng file đó** | Toàn bộ 660 dòng bị gắn `duplicate = true`, **không sinh bản ghi mới** ⇒ chứng minh **NFR-01 Idempotency** |

> ❗ Nếu file sai cấu trúc: API trả `422` kèm danh sách cột thiếu — UI hiển thị popup
> chi tiết (FR-06) và **không ghi gì vào CSDL**.

### Bước 3 — Chuẩn hóa & đối soát (FR-07 → FR-15)

| # | Thao tác | Kết quả mong đợi |
|---|---|---|
| 1 | Vào **Publications** (`/publications`) | Bảng công bố RAW + bộ lọc Năm / Khoa / Publisher / trạng thái, ô tìm kiếm |
| 2 | Chọn lô (batch) vừa nạp → bấm **Chạy chuẩn hóa & đối soát** | Tạo tác vụ `MATCHING_PIPELINE`; panel tiến trình hiện % và thông điệp từng bước |
| 3 | Theo dõi ở **Tiến trình nền** (badge trên Navbar) hoặc **TasksPage** | Tiến độ realtime qua SSE (`realtime` nhấp nháy), thời lượng tăng theo giây |
| 4 | Mở **Tra cứu & Đối soát → Hàng đợi** (`/queue`) | Bảng `APPROVAL_QUEUE` với cột Score (%) và badge `HIGH / MEDIUM / PENDING` |
| 5 | Lọc `Trạng thái = MATCH_HIGH` | Chỉ hiện các gợi ý có Match Score ≥ 85 % kèm bằng chứng đối chiếu |

> 💡 Có thể thử **tính lại PENDING** (Admin) để chạy lại các bản ghi điểm thấp sau khi
> danh mục giảng viên được cập nhật.

### Bước 4 — Phê duyệt hàng đợi với Diff View (FR-16 → FR-21)

> ⚠️ Bước này cần đăng nhập bằng tài khoản **admin** (Operator sẽ bị `AdminRoute`
> chuyển hướng khỏi `/queue`).

| # | Thao tác | Kết quả mong đợi |
|---|---|---|
| 1 | Vào **Tra cứu & Đối soát → Hàng đợi** | Bảng gồm: Queue, Bài báo (title/EID/năm), Tác giả, GV gợi ý, Score, Trạng thái (HIGH/MEDIUM/APPROVED/PENDING), Cờ `⚠ R` (Retracted) / `⚠ E` (Erratum) |
| 2 | Bấm vào một dòng bất kỳ (hoặc nút **Xem diff →**) | Mở **Diff View**: bên trái dữ liệu Scopus gốc, bên phải dữ liệu sau chuẩn hóa, kèm khối **`evidence_payload`** (căn cứ tính điểm) và mốc `rollback_deadline` |
| 3 | Bấm **[Phê duyệt liên kết]** (hoặc phím `A`) | Hộp thoại nhập ghi chú (tùy chọn) → ghi `STANDARDIZED_DATA` + `AUDIT_LOG(action=APPROVE)`, banner xanh xác nhận |
| 4 | Bấm **[Gán thủ công]** → chọn giảng viên khác từ danh mục → lưu | Cập nhật canonical author + ghi audit `MANUAL_OVERRIDE` (lưu cả **giá trị cũ → mới**) |
| 5 | Bấm **[Giữ chờ xác minh]** → nhập lý do (≥ 3 ký tự) | Bản ghi chuyển `PENDING_VERIFY`, ghi audit `DEFER` |
| 6 | Bấm **[⟲ Rollback]** (chỉ hiện khi `can_rollback = true`, trong 72 h) → nhập lý do | Hoàn tác về trạng thái trước khi duyệt, ghi audit `ROLLBACK` |
| 7 | Mở **Quản trị → Audit Log** | Thấy đầy đủ các dòng audit vừa sinh: `actor_id`, `action`, `before`, `after`, `timestamp` — **không có nút sửa/xóa** (bất biến) |

**Trạng thái chân trang của Diff View** hiển thị `hours_since_approved` và hạn
rollback còn lại, giúp hội đồng thấy rõ ràng buộc nghiệp vụ 72 giờ (FR-20).

### Bước 5 — Thống kê, xuất báo cáo & kiểm chứng chữ ký số (FR-25 → FR-29)

| # | Thao tác | Kết quả mong đợi |
|---|---|---|
| 1 | Vào **Dashboard** | 4 thẻ KPI (Tổng công bố gốc · Đã chuẩn hóa · Tỷ lệ hoàn thành · Giảng viên tham gia), biểu đồ xu hướng theo năm, phân bổ trạng thái, top giảng viên, phân bổ theo Khoa |
| 2 | Vào **Báo cáo & Xuất dữ liệu** (`/exports`) | Khối bộ lọc (năm, khoa, publisher, trạng thái…) và 3 nút xuất: **Xuất Excel Đầy đủ (4 Sheets)** · **Xuất CSV Nhanh** · **Xuất JSON API** |
| 3 | Bấm một nút xuất (đồng bộ) | Modal kết quả hiển thị số dòng, tên file và **`file_hash_sha256`** kèm nút **Copy**; file được tải về máy |
| 4 | Hoặc bật **“Xuất chạy nền”** (`BULK_EXPORT`) | Hệ thống trả `202 + task_id`; theo dõi ở panel **Tiến trình nền**; khi trạng thái `SUCCESS` bấm **Tải file** |
| 5 | Xem bảng **Lịch sử xuất báo cáo** (`GET /exports/history`) | Mỗi dòng có `actor`, `timestamp`, `row_count` và **`file_hash` SHA-256** |
| 6 | Mở PowerShell tại thư mục chứa file vừa tải và chạy `Get-FileHash` | Chuỗi hash **trùng khớp** với `file_hash` trong lịch sử ⇒ chứng minh FR-29 (ký số & toàn vẹn dữ liệu) |

```powershell
# Lấy hash SHA-256 của file báo cáo đã tải về
Get-FileHash -Algorithm SHA256 "$HOME\Downloads\scopus_ictu_excel_20260928.xlsx" |
  Format-List Algorithm, Hash, Path
```

### 4.6. Các chức năng nâng cao cần nắm

| Chức năng | Truy cập | Điểm đáng chú ý khi demo |
|---|---|---|
| **Navbar 1 hàng & responsive** | Mọi trang | ≥ 1024 px: 1 hàng gọn với dropdown nhóm; < 1024 px: hamburger + ngăn kéo trượt, có avatar/vai trò/nút Đăng xuất |
| **Panel Tiến trình nền** | Nút *Tiến trình nền* trên Navbar (badge số tác vụ đang chạy) | Ngăn kéo trượt từ phải, chứa danh sách tác vụ theo dõi; **chỉ có 1 điểm mở duy nhất**, không còn nút nổi che nội dung |
| **TasksPage** | *Quản trị → Tiến trình nền* | 5 thẻ KPI, bảng hiệu năng theo loại, filter loại/trạng thái/actor, phân trang, tự refresh 3 s, modal chi tiết có tiến độ live |
| **Theo dõi tác vụ** | Nút *Theo dõi* ở từng dòng task | Danh sách theo dõi lưu tại `localStorage["scopus_ictu_tracked_tasks"]`; polling 2,5 s và stats 5 s khi SSE bị chặn |
| **Hủy tác vụ** | Nút *Hủy* trong panel/chi tiết task | Hủy **cooperative** tại checkpoint — dữ liệu không bị hỏng giữa chừng |
| **Audit & RBAC** | *Quản trị → Audit Log* | Tab nhật ký nghiệp vụ + nhật ký bảo mật (ghi lại cả lần bị từ chối 403) |
| **Danh mục giảng viên** | *Quản trị → Giảng viên* | CRUD + tìm kiếm; mỗi thao tác sinh audit riêng (FR-22) |

---

## §5. KỊCH BẢN DEMO TRƯỚC HỘI ĐỒNG

### 5.1. Checklist chuẩn bị (làm trước 10 phút)

- [ ] `manage.bat setup` đã hoàn tất (`backend\.venv` + `frontend\node_modules` sẵn sàng).
- [ ] Chạy trước `manage.bat run`, xác nhận cả `:8000` và `:5173` phản hồi.
- [ ] Cửa sổ trình duyệt **≥ 1280 px**, zoom **100 %**, đăng nhập sẵn 2 tab:
      tab 1 = `qhun22` (Operator), tab 2 = `admin` (Admin).
- [ ] Mở sẵn thêm 2 tab: `http://localhost:8000/docs` và thư mục `backend\_exports`
      (nơi task `BULK_EXPORT` ghi file báo cáo; file xuất trực tiếp rơi vào `Downloads`).
- [ ] File CSV mẫu `test\Truong_Quang_Huy_Scopus_Sep2026.csv` đặt ở Desktop để kéo-thả.
- [ ] Cửa sổ **PowerShell** đã mở sẵn tại `%USERPROFILE%\Downloads` (dùng cho `Get-FileHash`).
- [ ] Bật **Focus Assist / Tắt thông báo Windows** và đặt điện thoại im lặng.
- [ ] Bản dự phòng: ảnh chụp các màn hình chính + `npm run build` → `npm run preview`.
- [ ] (Khuyến nghị) Ghi lại phiên demo thành video để phòng sự cố mạng/điện.

### 5.2. Timeline 10 phút

| Thời lượng | Nội dung | Thao tác cụ thể | Kết quả trình diễn |
|---|---|---|---|
| **0:00 – 0:45** | Giới thiệu & khởi động | Chỉ vào log `manage.bat run` → mở `:5173`, `:8000/docs` | Thấy backend + frontend chạy song song; Swagger liệt kê đủ nhóm API |
| **0:45 – 1:45** | Kiến trúc & Navbar responsive | Kéo cạnh cửa sổ từ rộng → hẹp dưới 1024 px rồi kéo lại | Navbar **1 hàng** không vỡ; chuyển thành **☰ hamburger + ngăn kéo trượt**; badge *Tiến trình nền* vẫn hiển thị |
| **1:45 – 3:15** | Nạp CSV 660 dòng ở chế độ **chạy nền** | `/upload` → kéo file CSV → bật “Nhập chạy nền” → Tải lên | API trả `202 + task_id`; panel tiến trình nhảy %; HTTP không bị chặn (vẫn bấm sang trang khác được) |
| **3:15 – 4:00** | Chứng minh **Idempotency (NFR-01)** | Tải lên **lại đúng file đó** | `inserted = 0`, `duplicate = 660` → không sinh bản ghi trùng |
| **4:00 – 5:30** | Chuẩn hóa & đối soát | `/publications` → chạy *Chuẩn hóa & đối soát* → `/queue` xem Score | Phân nhóm `HIGH ≥ 85 %`, `MEDIUM 50–85 %`, `PENDING < 50 %`; bảng có cờ Retracted/Erratum |
| **5:30 – 7:30** | Phê duyệt & hoàn tác (tài khoản **admin**) | Mở Diff View → duyệt 2 bản ghi (phím `A`) → chuyển bản ghi bằng phím `J` → **Rollback** 1 bản ghi | Cột Trạng thái đổi sang `APPROVED`; sau rollback quay lại đúng trạng thái cũ |
| **7:30 – 8:30** | Xuất báo cáo & **chữ ký số** | `/exports` → *Xuất Excel Đầy đủ (4 Sheets)* → Copy `file_hash_sha256` → chạy `Get-FileHash` trong PowerShell | **Hash trùng khớp tuyệt đối** ⇒ dữ liệu toàn vẹn (FR-29) |
| **8:30 – 9:15** | Kiểm toán & phân quyền | *Quản trị → Audit Log*; thử đăng nhập Operator rồi mở `/lecturers` hoặc gọi API admin | Audit ghi đủ `APPROVE / ROLLBACK`; operator nhận **403** và bị chặn ở UI |
| **9:15 – 10:00** | Tổng kết chất lượng | Dashboard + `TasksPage`; nêu `pytest 262/262 PASS`, `npm run build` exit 0 | Hội đồng thấy bằng chứng kiểm thử & kiến trúc tác vụ nền |

> 🎯 **Điểm nhấn nên nói rõ:** (1) Idempotency theo EID; (2) Match Score có
> `evidence_payload` — mọi quyết định duyệt đều có căn cứ; (3) Audit bất biến;
> (4) 202 + SSE thay vì chờ đồng bộ; (5) RBAC chặt giữa Operator và Admin.

### 5.3. Phím tắt & mẹo thao tác nhanh

| Phím | Ngữ cảnh | Tác dụng |
|---|---|---|
| `A` | **Diff View** khi đã đăng nhập `admin` và bản ghi cho phép duyệt | **Phê duyệt liên kết** ngay (hệ thống hỏi ghi chú rồi lưu) |
| `J` | **Diff View** | Chuyển sang bản ghi **kế tiếp** trong hàng đợi (rất tiện khi duyệt liên tiếp) |
| `K` | **Diff View** | Quay lại bản ghi **trước đó** |
| `Esc` | Diff View · ngăn kéo mobile · dropdown Navbar · menu người dùng | Đóng ngay lớp đang mở |
| `Tab` / `Shift+Tab` rồi `Enter` | Toàn hệ thống | Điều hướng và kích hoạt hoàn toàn bằng bàn phím |
| `Ctrl + Shift + R` | Trình duyệt | Tải lại cứng (loại bỏ cache UI cũ nếu vừa build lại) |
| `F12` → tab **Network** | Trình duyệt | Chứng minh trực tiếp: `202 Accepted` khi enqueue, `text/event-stream` cho SSE, `403` khi Operator gọi API admin |

> 📝 **Lưu ý:** phím tắt được **bỏ qua khi con trỏ đang nằm trong ô nhập liệu**
> (input/textarea/select) để tránh việc gõ lý do defer/rollback bị hiểu thành lệnh.
> **Mẹo demo:** mở sẵn Diff View cho bản ghi đầu tiên trước khi bắt đầu nói, sau đó
> dùng `J` để “duyệt nhanh” — hội đồng sẽ thấy rõ tốc độ thao tác.

### 5.4. Chứng minh chữ ký số SHA-256 (FR-29)

```powershell
# BƯỚC 1: lấy giá trị file_hash_sha256 hiển thị ở modal kết quả xuất (nút Copy)
#         hoặc ở cột hash trong bảng "Lịch sử xuất báo cáo" trên trang /exports

# BƯỚC 2: tính hash của file vừa tải về máy
$file = Get-ChildItem "$HOME\Downloads\*.xlsx","$HOME\Downloads\*.csv","$HOME\Downloads\*.json" |
        Sort-Object LastWriteTime -Descending | Select-Object -First 1
Write-Host "File   :" $file.Name
Write-Host "SHA-256:" (Get-FileHash -Algorithm SHA256 $file.FullName).Hash

# BƯỚC 3: dán hash ở BƯỚC 1 vào biến dưới đây để so sánh tự động
$hashHeThong = "<dán file_hash_sha256 lấy từ giao diện>"
$hashTrenDia = (Get-FileHash -Algorithm SHA256 $file.FullName).Hash
if ($hashHeThong.ToUpper() -eq $hashTrenDia.ToUpper()) {
    "✅ KHỚP - file toàn vẹn, không bị sửa đổi sau khi xuất"
} else {
    "❌ KHÔNG KHỚP - file đã bị thay đổi"
}
```

**Cách hệ thống tạo bằng chứng:** ngay khi sinh file, service export tính **SHA-256**
trên toàn bộ nội dung file và ghi một bản ghi vào `export_audit_log` gồm
`actor_id`, `timestamp`, `row_count`, `file_hash` (và `format`). Nếu file trên đĩa bị
sửa dù chỉ 1 byte, hash tính lại sẽ khác hoàn toàn ⇒ phát hiện được thay đổi trái phép.
Đây chính là tiêu chí của **FR-29 / NFR-05**.

### 5.5. Phương án dự phòng khi sự cố

| Tình huống | Xử lý nhanh ngay tại buổi bảo vệ |
|---|---|
| Máy hội đồng **không có PostgreSQL** | Không cần làm gì — hệ thống tự fallback SQLite; chỉ vào log chỉ rõ dòng `[DB] … FALLBACK to SQLite file: …` để giải thích cơ chế |
| Cổng `8000`/`5173` bị chiếm | `netstat -ano \| findstr :8000` → `taskkill /PID <pid> /F`; hoặc chạy `npx vite --port 5174` và thêm origin mới vào `CORS_ORIGINS` |
| Backend treo giữa demo | Mở task `Ctrl+C` ở cửa sổ `manage.bat run` rồi chạy lại; dữ liệu vẫn còn (SQLite file hoặc PostgreSQL) |
| Dữ liệu demo đã bị “bẩn” bởi lần chạy trước | Xóa `backend\scopus_dev.db` → chạy lại backend (reset sạch) rồi nạp lại CSV |
| Frontend trắng trang / UI cũ | `Ctrl+Shift+R`; kiểm tra `frontend\.env`; nếu cần thì `npm run build` + `npm run preview` |
| Mất điện / mất mạng đột ngột | Chiếu video hoặc ảnh chụp các màn hình đã chuẩn bị; mở Swagger `:8000/docs` và file test để chứng minh chức năng |
| Hết thời gian, chưa kịp chạy hết luồng | Ưu tiên: Idempotency → Diff View phê duyệt → SHA-256; các phần còn lại nói nhanh kèm số liệu `262/262 test` |

### 5.6. Câu hỏi thường gặp của Hội đồng & gợi ý trả lời

| Câu hỏi | Gợi ý trả lời (kèm bằng chứng trong repo) |
|---|---|
| Vì sao chọn **SSE** mà không dùng WebSocket? | Nhu cầu chỉ là **đẩy tiến độ một chiều** server → client; SSE chạy trên HTTP thường, tự động kết nối lại, đơn giản hơn về hạ tầng; vẫn có **polling dự phòng 2,5 s** (`TaskMonitorContext`) nếu SSE bị chặn |
| Làm sao chắc chắn **không trùng dữ liệu** khi nạp lại? | Khóa nghiệp vụ `(EID, Author ID)` + cờ `duplicate`; 8 test trong `tests/test_upload_idempotency.py` mô phỏng đúng việc nạp lại file 660 dòng |
| **Match Score** được tính thế nào, có minh bạch không? | Trọng số Author ID 0,6 · Tên 0,3 · Affiliation 0,1, kèm `evidence_payload` lưu lại căn cứ; ngưỡng phân nhóm 85 % / 50 % được cấu hình rõ trong `services/matching_engine` |
| Nếu hai giảng viên **trùng tên** thì sao? | Hệ thống không tự gộp: trường hợp thiếu bằng chứng rơi vào `PENDING`; Admin đối chiếu Affiliation và dùng **Gán thủ công** (có audit `MANUAL_OVERRIDE`) |
| Nhật ký kiểm toán có thể bị sửa không? | `audit_log` chỉ **INSERT** (NFR-02), không có endpoint UPDATE/DELETE; ngoài ra còn `security_audit_log` ghi cả các lần bị từ chối quyền |
| Tác vụ nền **hủy giữa chừng** có làm hỏng dữ liệu? | Hủy **cooperative** tại checkpoint — worker tự dừng ở điểm an toàn, dữ liệu đã ghi vẫn toàn vẹn; trạng thái cuối là `CANCELLED` |
| Bảo mật ở mức nào? | JWT access 30 phút + refresh 7 ngày, mật khẩu băm **bcrypt 12 rounds**, RBAC chặn ở tầng API (403 + audit), CORS whitelist, toàn bộ cấu hình nhạy cảm nằm trong `.env` không commit |
| Có thể **mở rộng** thế nào khi dữ liệu lớn? | Worker đổi `threadpool → celery`, `ProgressEventBus` hỗ trợ Redis pub/sub, CSDL PostgreSQL nhiều schema, frontend lazy-load theo route, export/analytics tách service riêng |

---

## PHỤ LỤC A — PHÂN BỔ 262 TEST CASE BACKEND

| File test | Số test | Nội dung kiểm chứng |
|---|---|---|
| `tests/test_analytics.py` | 34 | KPI tổng quan, xu hướng theo năm, phân bố theo Khoa, phân bố trạng thái matching, top giảng viên + yêu cầu xác thực của từng endpoint |
| `tests/test_approvals.py` | 16 | Luồng approve / manual-assign / defer / rollback và các ràng buộc lỗi (422, thiếu lý do, sai trạng thái) |
| `tests/test_audit_viewer.py` | 16 | Tra cứu nhật ký nghiệp vụ & nhật ký bảo mật, lọc theo action, phân quyền truy cập |
| `tests/test_auth.py` | 8 | Đăng nhập, refresh token, `/auth/me`, sai mật khẩu, token không hợp lệ |
| `tests/test_author_extractor.py` | 10 | Parse `Authors` và `Author(s) ID`, chuẩn hóa tên tiếng Việt, đếm số tác giả/bài |
| `tests/test_exports.py` | 48 | Xuất Excel (4 sheet), CSV (UTF-8 BOM), JSON + lịch sử export + task `BULK_EXPORT` (403/404/409/410) |
| `tests/test_lecturers.py` | 14 | CRUD danh mục giảng viên, validate email, RBAC, audit kèm theo |
| `tests/test_matching_engine.py` | 25 | So khớp Author ID, so khớp mờ, trọng số Match Score, ngưỡng HIGH/MEDIUM/PENDING |
| `tests/test_pipeline_api.py` | 6 | Chạy pipeline theo batch, trạng thái xử lý, danh sách công bố |
| `tests/test_recompute.py` | 13 | Tính lại các bản ghi PENDING (đồng bộ + bất đồng bộ) |
| `tests/test_security_rbac.py` | 24 | Operator gọi API admin → 403, ghi `security_audit_log`, phạm vi dữ liệu theo vai trò |
| `tests/test_tasks.py` | 40 | Task manager, cập nhật tiến độ, cancel cooperative, RBAC theo chủ tác vụ, API list/detail, **SSE**, stats, xác thực token qua query, CSV ingestion bất đồng bộ |
| `tests/test_upload_idempotency.py` | 8 | Upload hợp lệ, schema sai, **idempotency khi nạp lại**, file quá lớn, danh sách batch |
| **TỔNG** | **262** | `errors=0 · failures=0 · skipped=0` |

## PHỤ LỤC B — TỪ ĐIỂN KHÁI NIỆM & TRA CỨU NHANH

| Khái niệm | Giải thích |
|---|---|
| `RAW_DATA` | Bản ghi công bố nguyên gốc 22 trường từ CSV Scopus (kèm `batch_id`, `operator_id`, `uploaded_at`) |
| `AUTHOR_EXTRACTED` | Kết quả tách tác giả từng bài: `lastname`, `given_name_initials`, `scopus_author_id` |
| `MATCHING` / Match Score | Điểm tương đồng tác giả ∈ [0,1] = Author ID (0,6) + Tên (0,3) + Affiliation (0,1) |
| `MATCH_HIGH / MATCH_MEDIUM / PENDING` | Nhóm tin cậy theo ngưỡng ≥ 85 % · 50–85 % · < 50 % |
| `evidence_payload` | Bằng chứng đối chiếu được lưu lại cho mỗi gợi ý (phục vụ kiểm toán) |
| `APPROVAL_QUEUE` | Hàng đợi chờ duyệt với `bucket`, `match_score`, cờ `is_retracted`/`is_erratum` |
| `STANDARDIZED_DATA` | Kết quả đã duyệt: `canonical_author_id`, `score`, `evidence` |
| `AUDIT_LOG` | Nhật ký bất biến các thao tác duyệt/gán tay/defer/rollback |
| `EXPORT_AUDIT_LOG` | Lịch sử export: `actor_id`, `timestamp`, `row_count`, `file_hash` (SHA-256) |
| `background_tasks` | Bảng quản lý tác vụ nền: `PENDING → RUNNING → SUCCESS/FAILED/CANCELLED` + % tiến độ |
| Idempotency | Nạp lại cùng lô dữ liệu không sinh bản ghi mới (khóa `EID` + `Author ID`) |
| Cooperative cancel | Hủy tác vụ tại điểm kiểm soát an toàn thay vì kill cưỡng bức |
| SSE | *Server-Sent Events* — kênh một chiều đẩy tiến độ realtime (`/tasks/{id}/stream`) |
| RBAC | Phân quyền theo vai trò: Operator (nghiệp vụ) và Admin (quản trị + kiểm toán) |

**Lệnh kiểm chứng lại toàn bộ hệ thống (chạy trước khi nộp/bảo vệ):**

```bat
:: 1) Backend: 262 test case
cd backend
.venv\Scripts\python.exe -m pytest tests -q --tb=short

:: 2) Frontend: build production phải trả exit code 0
cd ..\frontend
npm run build
```

| Hạng mục | Kết quả ghi nhận |
|---|---|
| Backend pytest | **262 passed** (13 file test; `errors=0`, `failures=0`, `skipped=0`) |
| Frontend build | **exit = 0**, `715 modules transformed`, không lỗi/không cảnh báo chặn build |
| Đối chiếu yêu cầu | **FR-01 → FR-29** và **NFR-01 → NFR-06** đều đã triển khai và có test/kiểm chứng tương ứng |

> Tài liệu này được viết cho phiên bản mã nguồn `0.4.0` (Tuần 10). Khi có thay đổi về
> cấu hình, endpoint hay quy trình, vui lòng cập nhật lại đúng các mục **§2.4**, **§3.4**,
> **§3.5** và **Phụ lục B** để tài liệu luôn khớp với mã nguồn.











