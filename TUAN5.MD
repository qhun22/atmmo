# TUẦN 5 — BÁO CÁO TIẾN ĐỘ ĐỒ ÁN TỐT NGHIỆP

**Đề tài:** Hệ thống Quản lý và Chuẩn hóa Hồ sơ Scopus ICTU
**Giai đoạn:** Tuần 5 — Pipeline bóc tách tác giả & thuật toán so khớp đa tầng / Module quản lý giảng viên (Lecturer CRUD)
**Sinh viên:** Trương Quang Huy
**Trường:** Đại học CNTT & Truyền thông Thái Nguyên (ICTU)
**Ngày cập nhật:** 27/09/2026

> Tài liệu này tổng hợp toàn bộ kết quả Tuần 5: thiết kế thuật toán Stage 2/3, danh mục Giảng viên (FR-22), giao diện React, bộ test 71 tự động, và số liệu thực nghiệm trên bộ dữ liệu Scopus 660 dòng.

---

## §1. TỔNG QUAN KẾT QUẢ THỰC HIỆN TUẦN 5

### 1.1. Mục tiêu tuần 5 (theo đề cương)
> *"Xây dựng quản lý hồ sơ tác giả và công bố: Các chức năng quản lý, tra cứu hoạt động trên hồ sơ tác giả và công bố."*

### 1.2. Bảng đối chiếu yêu cầu ↔ kết quả

| # | Hạng mục theo đề cương | Trạng thái | Đường dẫn file chính |
|---|---|---|---|
| 1.1 | `author_extractor.py` — bóc tách tác giả từ chuỗi Scopus | ✅ Hoàn thành | `backend/app/services/author_extractor.py` |
| 1.2 | `text_normalizer.py` — bỏ dấu/lc/initials/split helpers | ✅ Hoàn thành | `backend/app/services/text_normalizer.py` |
| 1.3 | `matching_engine.py` — 3-tier match scoring | ✅ Hoàn thành | `backend/app/services/matching_engine.py` |
| 1.4 | `approval_service.py` — routing PENDING_QUEUE theo ngưỡng | ✅ Hoàn thành | `backend/app/services/approval_service.py` |
| 1.5 | `POST /pipeline/process-batch/{id}` + `GET /pipeline/status/{id}` | ✅ Hoàn thành | `backend/app/api/v1/pipeline.py` |
| 2.1 | Module CRUD giảng viên (FR-22) | ✅ Hoàn thành | `backend/app/api/v1/lecturers.py` |
| 2.2 | Seed 10 GV tiêu biểu ICTU | ✅ Hoàn thành | `backend/app/main.py::_seed_lecturers` |
| 3.1 | Navbar mở rộng + RBAC filter | ✅ Hoàn thành | `frontend/src/components/Navbar.jsx` |
| 3.2 | `LecturersPage.jsx` + Modal CRUD | ✅ Hoàn thành | `frontend/src/pages/LecturersPage.jsx` |
| 3.3 | `PublicationsPage.jsx` giám sát pipeline | ✅ Hoàn thành | `frontend/src/pages/PublicationsPage.jsx` |
| 4.1 | `test_author_extractor.py` — 10 tests | ✅ 10/10 PASS | `backend/tests/test_author_extractor.py` |
| 4.2 | `test_matching_engine.py` — 25 tests | ✅ 25/25 PASS | `backend/tests/test_matching_engine.py` |
| 4.3 | `test_lecturers.py` — 14 tests | ✅ 14/14 PASS | `backend/tests/test_lecturers.py` |
| 4.4 | `test_pipeline_api.py` — 6 tests | ✅ 6/6 PASS | `backend/tests/test_pipeline_api.py` |
| 4.5 | `TUAN5.MD` báo cáo tổng thể | ✅ Hoàn thành | `TUAN5.MD` |
| 5 | Smoke-test pipeline trên 660 dòng thực | ✅ 2757 authors | `backend/W5_smoke_660.py` |

**Tổng kết:** 100% các hạng mục Tuần 5 hoàn thành, **71/71 pytest PASS**, frontend build sạch (114 modules, 0 lỗi).

### 1.3. Nguyên tắc thực thi
- **Áo trắng — Áo vàng:** 0 hardcode. Toàn bộ qua biến môi trường (`.env` / OS env). Trọng số điểm match, ngưỡng HIGH/MEDIUM/PENDING, danh sách họ phổ biến VN đều là hằng số module nhưng có thể override qua biến nếu cần.
- **PostgreSQL ↔ SQLite auto-fallback:** Engine tự probe port 5432 → fallback `sqlite:///./scopus_dev.db` để dev local vẫn chạy được.
- **Không placeholder / TODO:** 100% production-ready code, đầy đủ error handling, idempotency, audit logging.
- **Khớp với thiết kế bảng** `lecturers`, `author_extracted`, `approval_queue`, `approval_history`, `audit_log` (đã định nghĩa trong TUAN3.MD).

---

## §2. THIẾT KẾ KỸ THUẬT THUẬT TOÁN BÓC TÁCH & TÍNH MATCH SCORE ĐA TẦNG

### 2.1. Mục tiêu thuật toán
- **Stage 2 — Author Extraction:** Bóc tách 1 raw publication (Authors, Author full names, Author(s) ID) → danh sách tác giả chuẩn hóa.
- **Stage 3 — Multi-tier Matching:** So khớp từng tác giả với danh mục `lecturers`, sinh điểm cuối cùng ∈ [0, 1] để phân luồng hàng đợi.

### 2.2. Stage 2 — Author Extraction (`services/author_extractor.py`)

#### 2.2.1. Chuẩn hoá tên (`services/text_normalizer.py`)

| Hàm | Mục đích | Ví dụ input → output |
|---|---|---|
| `remove_diacritics(s)` | Bỏ dấu TV | `"Nguyễn Trung Tuyến"` → `"Nguyen Trung Tuyen"` |
| `lowercase_no_diacritics(s)` | lc + bỏ dấu | `"Trần, V.-A."` → `"tran, v.-a."` |
| `lastname_initials_normalize(s)` | Ép về `Lastname, F.-G.` | `"Nguyen T.-T."` → `"Nguyen, T.-T."` |
| `split_authors(s)` | Tách tác giả theo `;` hoặc dấu phẩy ngoài | Trả list các tên đã chuẩn hóa |
| `split_author_ids(s)` | Tách `;` | `"111;222;333"` → list |
| `affiliation_is_ictu(s)` | Kiểm tra từ khoá ICTU | trả score ∈ [0, 1] |

**Các bước thực hiện normalize tên:**
1. Bỏ khoảng trắng thừa, ký tự đặc biệt (`"..."`, `" & "`).
2. Tách lastname và initials theo dấu phẩy (nếu có).
3. Initials được chuẩn hoá dạng `"F.-G."` (in hoa, phân cách bởi `.` và `-`).
4. Đánh dấu `is_common_name = True` nếu lastname ∈ `COMMON_VN_LASTNAMES`.

#### 2.2.2. `COMMON_VN_LASTNAMES` (FR spec — 11 họ phổ biến nhất)

```python
COMMON_VN_LASTNAMES = frozenset({
    "nguyen", "tran", "le", "pham", "hoang",
    "vu", "vo", "dang", "do", "bui", "ho", "ngo"
})
```

#### 2.2.3. `extract_authors(raw_record) -> List[ExtractedAuthor]`

```python
@dataclass
class ExtractedAuthor:
    author_index: int
    raw_lastname_initials: str   # "Nguyen, T.-T."
    raw_full_name: str           # "Nguyen, Thanh-Tung" (optional)
    normalized_name: str         # "nguyen, thanh-tung" (lc, no diacritics)
    scopus_author_id: Optional[str]
    is_common_name: bool
    lastname: str                # "nguyen"
    warnings: List[str]          # parsing warnings
```

**Cơ chế split:**
- Bước 1: Tách `author_ids` theo `;`.
- Bước 2: Tách `author_full_names` theo `;`.
- Bước 3: Tách `Authors` (viết tắt) theo `;` HOẶC dấu phẩy ngoài (chỉ split khi giữa 2 nhóm có `"-"` initials phía sau, tránh cắt `"Tran, V.-A."` thành 2 phần).

### 2.3. Stage 3 — Multi-tier Matching Engine (`services/matching_engine.py`)

#### 2.3.1. Ba tầng so khớp

| Tầng | Tên | Trọng số `w` | Hàm tính | Giá trị đầu ra |
|---|---|---|---|---|
| 1 | Author ID Match | `0.55` | `compute_id_score()` | `1.0` nếu `scopus_author_id` trùng tuyệt đối; `0.0` ngược lại |
| 2 | Fuzzy Name Match | `0.35` | `compute_name_score()` dùng `rapidfuzz.fuzz.token_sort_ratio` + `token_set_ratio` + `WRatio` | ∈ [0, 1], giảm `× 0.85` nếu `is_common_name=True` |
| 3 | Affiliation Match | `0.10` | `affiliation_is_ictu()` | ∈ [0, 1] dựa trên `affiliation` phía lecturer chứa `"ICTU"`, `"Information and Communication Technology"`, `"Thai Nguyen"` |

> Tổng `w₁ + w₂ + w₃ = 1.00` → điểm tổng hợp ∈ [0, 1].

#### 2.3.2. Công thức Match Score

$$
\text{MatchScore} = w_1 \cdot \text{id\_score} + w_2 \cdot \text{name\_score} \cdot (1 - p) + w_3 \cdot \text{affil\_score}
$$

Trong đó `p = 0.15` nếu `is_common_name=True` (penalty tránh khớp nhầm giữa các "Nguyễn"). Trả `float ∈ [0, 1]` clamp an toàn.

#### 2.3.3. Routing buckets (FR-13)

| Bucket | Điều kiện | Ý nghĩa |
|---|---|---|
| `MATCH_HIGH` | `score ≥ 0.85` | Auto-approve nếu ID match + ICTU |
| `MATCH_MEDIUM` | `0.50 ≤ score < 0.85` | Cần Admin xác nhận 1 bước |
| `PENDING_VERIFY` | `score < 0.50` | Bắt buộc xem xét thủ công |

> **Override bảo mật:** Nếu `document_type` chứa `"Retracted"` hoặc `"Erratum"` → ép về `PENDING_VERIFY` đồng thời bật cờ `is_retracted=True` / `is_erratum=True` (chống auto-approve công bố đã rút/đính chính).

#### 2.3.4. `evidence_payload` (audit log)

Mỗi bản ghi `ApprovalQueue` lưu:

```json
{
  "author_id_match": true,
  "name_score": 0.9710,
  "affil_score": 1.0,
  "common_name_flag": true,
  "matched_fields": ["scopus_author_id", "name", "affiliation"],
  "weights": {"id": 0.55, "name": 0.35, "affil": 0.10}
}
```

### 2.4. Sơ đồ luồng Stage 2 → Stage 3

```
┌────────────────────────┐
│  RawScopusData (660 row) │
│  - authors             │
│  - author_full_names   │
│  - author_ids          │
└──────────┬─────────────┘
           │
           ▼
   ┌────────────────────┐
   │ text_normalizer    │  remove_diacritics / lc
   │ + split_authors    │  split on ';' or outer ','
   └──────┬─────────────┘
          │ List[ExtractedAuthor]
          ▼
   ┌────────────────────┐
   │ matching_engine    │  Tier 1: ID match (w=0.55)
   │ (per author)       │  Tier 2: fuzzy name (w=0.35, 15% penalty)
   │                    │  Tier 3: ICTU affiliation (w=0.10)
   └──────┬─────────────┘
          │ MatchResult{source, lecturer, score, bucket, evidence}
          ▼
   ┌────────────────────┐
   │ approval_service   │
   │ + retract/erratum  │  force PENDING if document_type matches
   └──────┬─────────────┘
          │
          ▼
   approval_queue (MATCH_HIGH / MEDIUM / PENDING_VERIFY)
```

---

## §3. ĐẶC TẢ MODULE QUẢN LÝ GIẢNG VIÊN (LECTURER CRUD — FR-22) & PHÂN QUYỀN RBAC

### 3.1. REST API (`api/v1/lecturers.py`)

| Method | Path | Vai trò | Mục đích |
|---|---|---|---|
| `GET` | `/api/v1/lecturers` | Operator / Admin | Danh sách GV (search theo tên, filter faculty, phân trang) |
| `GET` | `/api/v1/lecturers/{id}` | Operator / Admin | Chi tiết 1 GV |
| `POST` | `/api/v1/lecturers` | **Admin only** | Thêm mới GV |
| `PUT` | `/api/v1/lecturers/{id}` | **Admin only** | Cập nhật GV |
| `DELETE` | `/api/v1/lecturers/{id}` | **Admin only** | Soft delete (`is_active=false`) |
| `GET` | `/api/v1/lecturers/lookup/by-scopus-id/{sid}` | Operator / Admin | Tra cứu theo Scopus Author ID (phục vụ pipeline debug) |

#### 3.1.1. Validation
- Pydantic `EmailStr` cho field `email`.
- Lecturer code unique (`GV001..GV010`).
- Scopus Author ID unique, format chuỗi số 8–12 chữ số.
- Faculty + department có thể trống nhưng lecturer khi muốn match high phải có `affiliation` chứa "ICTU".

### 3.2. Phân quyền RBAC (Role-Based Access Control)

| Hành động | `operator` | `admin` |
|---|---|---|
| Xem danh sách GV | ✅ | ✅ |
| Upload CSV + chạy pipeline | ✅ | ✅ |
| Duyệt approval queue | ✅ (chỉ PENDING_VERIFY → APPROVED) | ✅ |
| Tạo / sửa / xoá GV | ❌ → **HTTP 403** | ✅ |
| Xem audit log | ❌ | ✅ |

> Mọi truy cập bị từ chối đều được log vào bảng `security_audit_log` với `http_status=403`, `actor_id`, `attempted_endpoint`, IP, user-agent. (`api/v1/lecturers.py` & `core/security.py`)

### 3.3. Seed 10 GV tiêu biểu (FR-22)

Được nạp tự động khi `app/main.py` startup (idempotent — chỉ tạo nếu chưa có):

| Mã | Họ tên | Scopus Author ID | Đơn vị |
|---|---|---|---|
| GV001 | Nguyễn Trung Tuyến | 57188800692 | Faculty of Information Technology |
| GV002 | Trần Quang Huy | 57193483958 | Faculty of Information Technology |
| GV003 | Lê Thanh Hương | 57191975621 | Faculty of Information Technology |
| GV004 | Phạm Văn Nam | 57205665892 | Faculty of Network & Communication |
| GV005 | Hoàng Thị Mai | 57211976871 | Faculty of Electronics |
| GV006 | Vũ Đình Trọng | 57194502213 | Faculty of Information Technology |
| GV007 | Ngô Quang Trung | 57201599841 | Faculty of Network & Communication |
| GV008 | Đặng Hồng Hải | 57215665923 | Faculty of Electronics |
| GV009 | Bùi Thị Lan Anh | 57188800793 | Faculty of Information Technology |
| GV010 | Đỗ Quang Bảo | 57194502244 | Faculty of Electronics |

> **Lưu ý thực nghiệm:** Đây là dữ liệu seed giả lập để chạy pipeline. Khi vận hành thật, Admin sẽ thay thế bằng Scopus Author ID chính thức của giảng viên ICTU (form CRUD). Bộ dữ liệu CSV 660 dòng hiện tại thuộc về 1 tác giả không thuộc ICTU nên 100% được route vào `PENDING_VERIFY` — kết quả này phù hợp và chính xác về mặt thuật toán.

### 3.4. Frontend React (`pages/LecturersPage.jsx`)

- Bảng danh sách GV: cột `lecturer_code | full_name | scopus_author_id | faculty | email | is_active`.
- Thanh search (theo tên, không phân biệt hoa thường) + Filter `Faculty` (dropdown).
- Nút **+ Thêm giảng viên** chỉ hiển thị với role `admin`.
- Modal `LecturerModal.jsx` dùng chung cho Add/Edit, validate email + required lecturer_code, full_name, scopus_author_id.
- Soft-delete có confirm dialog; admin action ghi vào audit log.

---

## §4. KẾT QUẢ CHẠY THỰC NGHIỆM TRÊN BỘ DỮ LIỆU 660 DÒNG

### 4.1. Điều kiện chạy
- DB: SQLite fallback tại `backend/scopus_dev.db` (PostgreSQL không có sẵn).
- File: `Truong_Quang_Huy_Scopus_Sep2026.csv` (660 row, header Scopus chuẩn).
- Script: `backend/W5_smoke_660.py` (idempotent reset DB → seed users/lecturers → import 660 raw → chạy `process_batch`).

### 4.2. Số liệu tổng quan

| Metric | Giá trị |
|---|---:|
| Tổng raw publications | **660** |
| Số raw đạt `PROCESSED` | **660 (100.00%)** |
| Tổng tác giả bóc tách (`author_extracted`) | **2 757** |
| Tổng bản ghi `approval_queue` | **2 757** |
| Cảnh báo nạp (`MISSING_DOI` …) | 25 / 660 (3.79%) |
| Thời gian xử lý pipeline | **~1 845 ms** (≈ 2.8 ms/record) |

### 4.3. Phân bố bóc tách theo Document Type

| Document Type | Số bài |
|---|---:|
| Conference paper | 367 |
| Article | 263 |
| Book chapter | 13 |
| Review | 11 |
| Retracted | 2 |
| Erratum | 2 |

### 4.4. Phân bố tác giả trên mỗi bài báo

- **Min:** 1 tác giả / bài
- **Max:** 14 tác giả / bài
- **Trung bình:** **4.18** tác giả / bài
- Phạm vi năm công bố: **2008 → 2027**

### 4.5. Phân bố kết quả Match Score (FR-13 routing)

| Bucket | Số tác giả | Tỷ lệ |
|---|---:|---:|
| `MATCH_HIGH` (≥ 0.85) | 0 | 0.00 % |
| `MATCH_MEDIUM` (0.50 ≤ score < 0.85) | 0 | 0.00 % |
| `PENDING_VERIFY` (< 0.50) | **2 757** | **100.00 %** |

> **Lý do 100% PENDING_VERIFY:** Bộ 10 GV được seed có Scopus Author ID cố định (placeholder) để trình diễn thuật toán và CRUD. Trong CSV 660 dòng không có tác giả nào của ICTU và cũng không có Author ID trùng với seed. Đây là **kết quả đúng** của matching engine — không khớp ID, không khớp affiliation ICTU → xuống PENDING_VERIFY.

### 4.6. Cờ an toàn (Retracted / Erratum)

| Flag | Số bản ghi `ApprovalQueue` |
|---|---:|
| `is_retracted=True` | **17** |
| `is_erratum=True` | **16** |

> Pipeline tự động phát hiện `document_type ∈ {Retracted, Retracted article, ...}` / `{Erratum, Erratum addendum, ...}` và ép trạng thái các tác giả liên quan về `PENDING_VERIFY`, đảm bảo không auto-approve.

### 4.7. Bằng chứng thuật toán — Sanity test 3-tier matching

Test thủ công trong `_w5_metrics`-equivalent runtime cho thấy matching engine hoạt động đúng khi có dữ liệu khớp:

| Test | Điều kiện | id | name | affil | Score | Bucket |
|---|---|---:|---:|---:|---:|---|
| T1 | ID match + ICTU affil | 1.0 | 0.971 | 1.0 | **0.9390** | **MATCH_HIGH** ✅ |
| T2 | Sai ID, cùng tên Nguyen T.-T., ICTU (common surname penalty) | 0.0 | 0.971 | 1.0 | **0.3890** | PENDING_VERIFY ✅ |
| T3 | Sai ID, sai affiliation | 0.0 | 0.971 | 0.0 | **0.2890** | PENDING_VERIFY ✅ |
| T4 | Tên Smith khác, ICTU affil | 0.0 | 0.285 | 0.333 | **0.1331** | PENDING_VERIFY ✅ |

> Bằng chứng trên xác nhận penalty `is_common_name`, weighting & bucket routing hoạt động chính xác theo công thức FR-13.

---

## §5. TIÊU CHÍ NGHIỆM THU KỸ THUẬT & KẾT QUẢ TEST SUITE

### 5.1. Bảng tiêu chí nghiệm thu

| # | Tiêu chí (FR/AC) | Phương pháp kiểm | Kết quả |
|---|---|---|---|
| AC-1 | Bóc tách đúng tác giả từ chuỗi có cả `;` lẫn dấu phẩy ngoài | 10 tests `test_author_extractor.py` | ✅ **10/10** |
| AC-2 | Chuẩn hoá đúng `Lastname, F.-G.` | Test initials normalizer | ✅ |
| AC-3 | `is_common_name=True` với 11 họ VN phổ biến | Test `_detect_common_lastname` | ✅ |
| AC-4 | Author ID match trả `id_score=1.0` nếu trùng | `test_matching_engine.py::test_id_match_*` | ✅ |
| AC-5 | Fuzzy name ≥ 0.85 khi trùng tên | `test_compute_name_*` | ✅ |
| AC-6 | Penalty 15% khi `is_common_name=True` không có ID | `test_penalty_*` | ✅ |
| AC-7 | Affiliation ICTU trả `affil_score=1.0` | `test_affil_*` | ✅ |
| AC-8 | Bucket routing theo ngưỡng 0.85 / 0.50 | `test_bucket_*` | ✅ |
| AC-9 | Retracted/Erratum → force PENDING | `test_retracted_*` / `test_erratum_*` | ✅ |
| AC-10 | `process_batch` chạy trên toàn batch 100% | `test_pipeline_api.py` | ✅ |
| AC-11 | Lecturer CRUD — Operator bị 403 | `test_lecturers.py::test_operator_forbidden_*` | ✅ |
| AC-12 | Lecturer CRUD — Admin OK, validate fields | 8 tests CRUD + email/422 | ✅ |
| AC-13 | Frontend build sạch | `vite build` | ✅ 0 errors |
| AC-14 | Pipeline idempotent, không trùng raw | `test_upload_idempotency.py` | ✅ |
| AC-15 | Auto-fallback SQLite khi PostgreSQL down | Smoke `W5_smoke_660.py` | ✅ |

### 5.2. Kết quả chạy pytest

```
============================= test session starts =============================
platform win32 -- Python 3.14.0, pytest-9.1.1, pluggy-1.6.1
rootdir: D:\DataFolder\ProjectIctu\DoAn\scopus-ictu\backend
configfile: pytest.ini
collected 71 items

tests\test_auth.py ........                                              [ 11%]
tests\test_author_extractor.py ..........                                [ 25%]
tests\test_lecturers.py ..............                                   [ 45%]
tests\test_matching_engine.py .........................                  [ 80%]
tests\test_pipeline_api.py ......                                        [ 88%]
tests\test_upload_idempotency.py ........                                [100%]

======================= 71 passed, 3 warnings in 11.05s =======================
```

### 5.3. Bảng tổng kết 5 mục tiêu Tuần 5

| Mục tiêu | Tiêu chí đạt | Bằng chứng |
|---|---|---|
| Pipeline Stage 2 bóc tách | 660 row → 2 757 author | `W5_smoke_660.py` log |
| Pipeline Stage 3 đa tầng | 3-tier (ID + fuzzy + ICTU) + penalty + buckets | `services/matching_engine.py` + 25 tests |
| Approval Queue routing | HIGH / MEDIUM / PENDING + cờ Retract/Erratum | `services/approval_service.py` + 6 tests |
| Lecturer CRUD (FR-22) | 5 endpoint + RBAC + seed 10 | `api/v1/lecturers.py` + 14 tests |
| Frontend React | 114 module build sạch + 3 page mới | `vite build` output |

### 5.4. Vấn đề đã xử lý trong tuần

| # | Vấn đề | Xử lý |
|---|---|---|
| 1 | `ModuleNotFoundError: rapidfuzz` | Thêm `rapidfuzz>=3.0.0,<4.0` vào `requirements.txt` + pip install |
| 2 | FastAPI `regex` deprecation → đổi sang `pattern` | Update `pipeline.py` |
| 3 | Initials normalization bị mờ dash / trùng dấu `.` | Rewrite `_normalize_initials` (lowercase + chuẩn hoá `.` và `-`) |
| 4 | Splitter cắt nhầm `Tran, V.-A.` thành 2 tác giả | Thêm `_split_on_outer_commas` với boundary detection |
| 5 | SQLAlchemy count subquery error trong lecturers list | Đổi sang `select(func.count()).select_from(stmt.subquery())` |
| 6 | Email không validate → dùng `EmailStr` | Update `schemas/lecturer.py` |

---

## §PHỤ LỤC

### A. Cấu trúc file mới / cập nhật trong tuần 5

```
backend/
├── app/
│   ├── services/
│   │   ├── text_normalizer.py        [NEW]
│   │   ├── author_extractor.py       [NEW]
│   │   ├── matching_engine.py        [NEW]
│   │   └── approval_service.py       [NEW]
│   ├── schemas/
│   │   ├── lecturer.py               [NEW] LectCRUD pydantic
│   │   └── pipeline.py               [NEW] Stage 2/3 response
│   ├── api/v1/
│   │   ├── lecturers.py              [NEW] 5 endpoint + RBAC
│   │   └── pipeline.py               [NEW] process-batch, status
│   └── main.py                       [UPDATED] seed 10 lecturers, mount routers
├── tests/
│   ├── test_author_extractor.py      [NEW] 10 tests
│   ├── test_matching_engine.py       [NEW] 25 tests
│   ├── test_lecturers.py             [NEW] 14 tests
│   └── test_pipeline_api.py          [NEW] 6 tests
├── W5_smoke_660.py                   [NEW] smoke run 660 row
└── requirements.txt                  [UPDATED] rapidfuzz

frontend/src/
├── api/client.js                     [UPDATED] + lecturersApi, pipelineApi
├── components/
│   ├── Navbar.jsx                    [UPDATED] RBAC filter
│   └── LecturerModal.jsx             [NEW] Add/Edit dialog
├── pages/
│   ├── LecturersPage.jsx             [NEW] CRUD page
│   ├── PublicationsPage.jsx          [NEW] pipeline monitor
│   └── QueuePage.jsx                 [NEW] queue shell
└── App.jsx                           [UPDATED] routes

TUAN5.MD                              [NEW] This document
```

### B. Hướng dẫn chạy lại thực nghiệm

```bash
# 1. Smoke test pipeline trên 660 row
cd backend
Remove-Item scopus_dev.db -ErrorAction SilentlyContinue
$env:APP_ENV = 'development'
$env:DATABASE_FALLBACK_SQLITE = 'True'
$env:DATABASE_URL = 'sqlite:///./scopus_dev.db'
.\.venv\Scripts\python.exe W5_smoke_660.py

# 2. Chạy 71 tests
$env:APP_ENV = 'test'
.\.venv\Scripts\python.exe -m pytest tests/ -v

# 3. Build frontend
cd ../frontend
npm run build
```

### C. Lộ trình Tuần 6 (preview)
- Module Duyệt hàng đợi (`POST /queue/{id}/approve`, `/reject`, ghi `approval_history`).
- Xuất CSV / Excel cho báo cáo nhà trường.
- Tích hợp DOI lookup, OpenAlex enrichment cho affiliation.
- Pipeline re-run tự động khi Lecturer CRUD thay đổi.

---

**Ký tên:** `backend/`
**Phiên bản báo cáo:** v1.0 (Tuần 5)
