# TUẦN 7 — BÁO CÁO TIẾN ĐỘ ĐỒ ÁN TỐT NGHIỆP

## **Hệ thống Quản lý và Chuẩn hóa Hồ sơ Scopus ICTU**

**Trường ĐH CNTT & Truyền thông Thái Nguyên — ICTU**
**Sinh viên:** Trương Quang Huy — MSSV: DTC235200899 — Lớp: CN CNTT K22A
**Giảng viên hướng dẫn:** TS. Nguyễn Thế Vinh — Bộ môn Công nghệ phần mềm, Khoa Công nghệ thông tin (ICTU)
**Tuần:** 7 / 10
**Ngày hoàn thành:** 27/09/2026
**Phạm vi:** Export Module & Analytics Dashboard (FR-23 → FR-29, NFR-03)

---

## §1. TỔNG QUAN KẾT QUẢ THỰC HIỆN TUẦN 7

### 1.1. Kết quả đạt được theo đề cương

| Hạng mục | Trạng thái | Ghi chú |
|---|---|---|
| **P1 — Export Module (Excel 4 sheets, CSV, JSON)** | ✅ Hoàn thành 100% | `backend/app/services/export_service.py` |
| **P2 — Export Audit (`export_audit_log` + SHA-256 NFR-03)** | ✅ Hoàn thành 100% | Header `X-File-SHA256`, audit trên mỗi lần xuất |
| **P3 — Analytics Dashboard (FR-29) — 5 endpoints KPI** | ✅ Hoàn thành 100% | `backend/app/services/analytics_service.py` |
| **P4 — Frontend `DashboardPage.jsx` & `ExportPage.jsx`** | ✅ Hoàn thành 100% | Recharts, modal SHA-256, lịch sử phân trang |
| **P5 — Navbar cập nhật (Dashboard, Báo cáo & Xuất)** | ✅ Hoàn thành 100% | Cả operator + admin |
| **P6 — Bộ kiểm thử Pytest (test_exports + test_analytics)** | ✅ Hoàn thành 100% | 73 test case mới, **160/160 tổng pass** |
| **P7 — Frontend `npm run build` 0 errors** | ✅ Hoàn thành 100% | 704 modules transformed |

### 1.2. Kết quả kiểm thử tổng hợp

| Test suite | Số test | Pass | Fail |
|---|---|---|---|
| test_exports.py | 38 | 38 | 0 |
| test_analytics.py | 35 | 35 | 0 |
| **Tổng Tuần 7** | **73** | **73** | **0** |
| Toàn bộ hệ thống (Tuần 4 → 7) | 160 | 160 | 0 |
| **Tổng thời gian pytest** | **~120 s** | | |

---

## §2. THIẾT KẾ CHI TIẾT CẤU TRÚC 4 SHEET EXCEL & CƠ CHẾ XÁC THỰC TOÀN VẸN BẰNG BĂM SHA-256 (NFR-03)

### 2.1. Cấu trúc 4 Sheet chuẩn hóa

Mỗi file `.xlsx` xuất ra đều có đúng **4 sheet** theo Data Exchange Spec ICTU:

#### **Sheet 1 — `Tổng quan` (Overview)**

| Hàng | Nội dung | Mục đích |
|---|---|---|
| A1 | "BÁO CÁO TỔNG HỢP HỒ SƠ SCOPUS — ICTU" | Tiêu đề chính, brand ICTU primary `#1F4E79` |
| A2 | "Trường ĐH CNTT & Truyền thông Thái Nguyên \| Xuất: dd/mm/yyyy HH:MM" | Subtitle, ngày xuất |
| A4–B4 | Header: "Chỉ số" \| "Giá trị" | Header bảng KPI |
| A5–B11 | 7 chỉ số KPI: Tổng công bố, Tổng GV có công bố, Phạm vi năm, Tỷ lệ chuẩn hóa, Retracted, Erratum, Thiếu DOI | Bảng tổng hợp |
| Sau bảng KPI | "Cơ cấu theo Khoa" — bảng 3 cột | Phân bổ theo đơn vị |

**Định dạng chuẩn:**
- Background header: `#1F4E79` (ICTU primary), white bold text size 14
- Subtitle: `#D6E4F0` light blue, italic size 10
- Body rows: alternating row colors `#F2F8FD` (zebra striping)
- Borders: thin `#AAAAAA`, căn lề center cho header / right cho số
- Column widths: A=40, B=20, C=18, D=18 (auto-fit by content)

#### **Sheet 2 — `Theo Giảng viên` (By Lecturer)**

| Cột | Header | Mô tả |
|---|---|---|
| A | Mã GV | `lecturer_code` (e.g., `GV001`) |
| B | Họ và tên | `full_name` |
| C | Học hàm / Học vị | `academic_rank` (PGS, GS, TS, ...) |
| D | Khoa / Bộ môn | `faculty` / `department` |
| E | Scopus Author ID | `scopus_author_id` |
| F | Tổng bài | `COUNT(standardized_data)` |
| G | Điểm TB | `AVG(match_score)` 3-decimal |
| H | Năm đầu | `MIN(year)` |
| I | Năm cuối | `MAX(year)` |

**Aggregation logic:**
```sql
SELECT lecturer_id, COUNT(*), AVG(match_score),
       MIN(year), MAX(year)
FROM standardized_data
JOIN raw_scopus_data ON ...
WHERE is_invalidated = false AND [filters]
GROUP BY lecturer_id
ORDER BY COUNT(*) DESC
```

#### **Sheet 3 — `Theo Bài báo` (By Publication)**

| Cột | Header | Mô tả |
|---|---|---|
| A | Năm | `raw_scopus_data.year` |
| B | Tiêu đề bài báo | `title` (truncate 200 chars) |
| C | DOI | `doi` |
| D | Tạp chí / Hội nghị | `source_title` |
| E | Tên GV ICTU | `lecturer.full_name` |
| F | Mã GV | `lecturer.lecturer_code` |
| G | Khoa | `lecturer.faculty` |
| H | Điểm Match | `match_score` 3-decimal |
| I | Trạng thái | `approval_queue.status` |
| J | Ngày duyệt | `standardized_data.approved_at` ISO-8601 |

#### **Sheet 4 — `Nhật ký Duyệt` (Audit Trail)**

| Cột | Header | Mô tả |
|---|---|---|
| A | Thời gian | `audit_log.created_at` |
| B | Hành động | APPROVE / MANUAL_OVERRIDE / DEFER / ROLLBACK / ROLLBACK_DENIED (translate tiếng Việt) |
| C | Bảng | `target_table` |
| D | ID bản ghi | `target_id` |
| E | Người thực hiện | `users.username` |
| F | Lý do / Ghi chú | `audit_log.reason` |
| G | Correlation ID | UUID liên kết batch |
| H | Chi tiết trước | `before_json` (truncate 200 chars) |

### 2.2. Cơ chế xác thực toàn vẹn bằng băm SHA-256 (NFR-03)

**Quy trình:**

1. **Sinh file trong memory:** Service tạo `io.BytesIO()` chứa toàn bộ nội dung file (Excel, CSV, JSON).
2. **Băm SHA-256:** `hashlib.sha256(content).hexdigest()` → chuỗi hex 64 ký tự.
3. **Gắn vào HTTP response headers:**
   - `Content-Disposition: attachment; filename="scopus_ictu_report_YYYYMMDD_HHMMSS.xlsx"`
   - `X-File-SHA256: <hash>` ← **Quan trọng NFR-03**
   - `X-File-Size-Bytes: <size>`
   - `X-Row-Count: <n>`
4. **Trả về metadata qua Pydantic:** `ExportResultSchema` chứa `file_hash_sha256`, `file_size_bytes`, `row_count`, `file_name`, `exported_at`.
5. **Ghi `export_audit_log`:**
   - `file_format`: xlsx / csv / json
   - `filter_criteria_json`: {year_from, year_to, faculty, lecturer_id, status, ...}
   - `row_count`, `file_hash_sha256`, `file_name`
   - `actor_id`: người thực hiện xuất
   - `created_at`: thời điểm xuất (timestamp UTC)

**Code chính (`export_service.py`):**

```python
def sha256_bytes(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()

def log_export(db, *, actor_id, format_type, row_count,
               file_hash, file_size_bytes, flt):
    entry = ExportAuditLog(
        file_format=format_type.value,
        filter_criteria_json=asdict(flt),
        row_count=row_count,
        file_hash_sha256=file_hash,
        file_name=f"scopus_ictu_report_{now_str()}.{format_type.value}",
        actor_id=actor_id,
    )
    db.add(entry); db.commit()
```

**Kiểm toán viên có thể đối soát:**
- Tải file `.xlsx` từ server.
- Chạy `sha256sum file.xlsx` (Linux) hoặc `certUtil -hashfile file.xlsx SHA256` (Windows).
- So sánh với giá trị `X-File-SHA256` trong response header hoặc cột `file_hash_sha256` trong bảng `export_audit_log`.
- Nếu khớp → file toàn vẹn.

### 2.3. CSV UTF-8 BOM & JSON Draft 2020-12

**CSV:** Mỗi file `.csv` được encode UTF-8 với BOM (`\ufeff` → `0xEF 0xBB 0xBF` ở đầu file). Điều này đảm bảo Excel tự nhận diện encoding UTF-8 và hiển thị đúng tiếng Việt có dấu (Nghuyễn, Trần, ...).

**JSON:** Tuân thủ JSON Schema Draft 2020-12:
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "meta": {
    "exported_at": "2026-09-27T07:48:08+00:00",
    "application": "Scopus ICTU",
    "version": "1.0",
    "total_records": 660,
    "filters_applied": { "year_from": 2010, "year_to": 2026, ... }
  },
  "publications": [
    {
      "id": 1, "title": "...", "year": 2023, "doi": "10.1234/...",
      "lecturer": { "code": "GV001", "name": "...", "faculty": "..." },
      "match_score": 0.923, "approval_status": "APPROVED"
    }
  ]
}
```

---

## §3. HỆ THỐNG CHỈ SỐ KPI VÀ BIỂU ĐỒ PHÂN TÍCH DASHBOARD

### 3.1. 5 Endpoint KPI (FR-29)

| Endpoint | Trả về | Mục đích |
|---|---|---|
| `GET /api/v1/analytics/overview` | 8 chỉ số KPI | Tổng quan cấp quản lý |
| `GET /api/v1/analytics/trends-by-year` | `[{year, count}]` | Biểu đồ cột theo năm |
| `GET /api/v1/analytics/distribution-by-faculty` | `[{faculty, lecturer_count, publication_count, percentage}]` | Biểu đồ cơ cấu theo Khoa |
| `GET /api/v1/analytics/matching-distribution` | `{distribution: {status: {count, percentage}}, total}` | Biểu đồ tròn trạng thái |
| `GET /api/v1/analytics/top-lecturers?limit=10` | `[{lecturer_id, code, name, faculty, rank, publication_count, avg_match_score}]` | Bảng top giảng viên |

### 3.2. Cấu trúc `OverviewKPISchema`

```json
{
  "total_raw_publications": 660,
  "total_extracted_authors": 2757,
  "total_standardized": 658,
  "standardization_rate": 23.86,
  "total_retracted": 12,
  "total_erratum": 8,
  "total_missing_doi": 45,
  "total_lecturers_active": 87
}
```

**Công thức tính:**
- `standardization_rate = total_standardized / total_extracted_authors × 100` (làm tròn 2 chữ số thập phân)
- `total_retracted / total_erratum / total_missing_doi` — đếm trực tiếp từ bảng `approval_queue` và `raw_scopus_data`

### 3.3. Dashboard Frontend (`DashboardPage.jsx`)

**Layout 3 hàng:**

**Hàng trên — 4 Thẻ KPI:**
1. `Tổng công bố gốc` (blue)
2. `Đã chuẩn hóa` (green) — sub-label: "trên X lượt tác giả"
3. `Tỷ lệ hoàn thành` (indigo) — sub-label: "Đạt chuẩn ICTU"
4. `Giảng viên tham gia` (purple)

**Cảnh báo dữ liệu** (chỉ hiển thị khi có):
- Retracted > 0
- Erratum > 0
- Thiếu DOI > 0

**Hàng giữa:**
- Biểu đồ cột `Xu hướng công bố theo năm` (BarChart, recharts) — width 2/3
- Biểu đồ tròn `Phân bổ trạng thái` (PieChart) — width 1/3, màu:
  - APPROVED: `#10B981` (green)
  - PENDING_VERIFY: `#F59E0B` (amber)
  - MATCH_HIGH: `#3B82F6` (blue)
  - MATCH_MEDIUM: `#A78BFA` (purple)
  - PENDING: `#9CA3AF` (gray)

**Hàng dưới:**
- Bảng `Top giảng viên có nhiều công bố nhất` (width 2/3) — 7 cột
- Biểu đồ cột ngang `Phân bổ theo Khoa` (width 1/3)

**Công nghệ:** `recharts` (^2.x) — thư viện nhẹ, declarative, tương thích React 18.

### 3.4. Trang Xuất Báo cáo (`ExportPage.jsx`)

**Form filter (7 trường):**
1. Từ năm (2000–2030)
2. Đến năm (2000–2030)
3. Khoa / Đơn vị (text)
4. Bộ môn (text)
5. Giảng viên cụ thể (dropdown — load từ `/v1/lecturers/options`)
6. Trạng thái (dropdown: APPROVED / PENDING_VERIFY / MATCH_HIGH / MATCH_MEDIUM)
7. Checkbox "Bao gồm bản ghi đã vô hiệu"

**3 Nút hành động xuất:**
- `[Xuất Excel Đầy đủ (4 Sheets)]` — primary blue
- `[Xuất CSV Nhanh]` — secondary
- `[Xuất JSON API]` — secondary

**Modal kết quả sau xuất** (`ResultModal`):
- Icon check + "Đã ghi nhận bản ghi vào `export_audit_log`"
- Grid 2 cột: Tên file, Định dạng, Số bản ghi, Dung lượng
- **Ô SHA-256 đặc biệt** với nút `Copy` — dùng `navigator.clipboard.writeText()` với fallback `document.execCommand("copy")`
- Toast xác nhận "✓ Đã copy" trong 2 giây

**Bảng lịch sử xuất (`/exports/history`):**
- 7 cột: ID, Thời gian, Người xuất, Định dạng, Số dòng, File, SHA-256 (rút gọn 12 ký tự + Copy)
- Pagination (10 records/page)

---

## §4. ĐẶC TẢ KIỂM TOÁN XUẤT DỮ LIỆU (`export_audit_log`)

### 4.1. Schema bảng (Tuần 6 đã tạo, Tuần 7 active)

```sql
CREATE TABLE export_audit_log (
  id BIGSERIAL PRIMARY KEY,
  file_format VARCHAR(8) NOT NULL
    CHECK (file_format IN ('xlsx','csv','json')),
  filter_criteria_json JSONB NOT NULL,
  row_count INTEGER NOT NULL CHECK (row_count >= 0),
  file_hash_sha256 CHAR(64) NOT NULL,
  file_name TEXT NOT NULL,
  actor_id BIGINT NOT NULL
    REFERENCES scopus_ictu.users(id) ON DELETE RESTRICT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_export_actor_time ON export_audit_log(actor_id, created_at);
```

### 4.2. Mỗi lần xuất đều ghi 1 record

```python
entry = ExportAuditLog(
    file_format=format_type.value,           # "xlsx" / "csv" / "json"
    filter_criteria_json={
        "year_from": 2010, "year_to": 2026,
        "faculty": "Khoa CNTT", "lecturer_id": 12,
        "status": "APPROVED", "include_invalidated": False
    },
    row_count=658,
    file_hash_sha256="a3f5b9c1d4e6...",     # 64 hex chars
    file_name="scopus_ictu_report_20260927_074808.xlsx",
    actor_id=2,                              # user.id
)
db.add(entry); db.commit()
```

### 4.3. Truy vết & đối soát

| Câu hỏi | Cách trả lời |
|---|---|
| Ai đã xuất báo cáo Excel tháng 9? | `SELECT * FROM export_audit_log WHERE file_format='xlsx' AND created_at BETWEEN ...` |
| Có bao nhiêu lần xuất CSV trong năm 2026? | `SELECT COUNT(*) FROM export_audit_log WHERE file_format='csv' AND EXTRACT(YEAR FROM created_at)=2026` |
| File `scopus_ictu_report_...xlsx` có toàn vẹn? | `sha256sum file.xlsx` rồi so với `file_hash_sha256` trong audit log |
| Lần xuất gần nhất của operator `qhun22`? | Join `export_audit_log` với `users` WHERE `users.username='qhun22'` ORDER BY `created_at DESC LIMIT 1` |

---

## §5. TIÊU CHÍ NGHIỆM THU KỸ THUẬT TUẦN 7 & KẾT QUẢ KIỂM THỬ PYTEST

### 5.1. Tiêu chí nghiệm thu

| # | Tiêu chí | Trạng thái | Bằng chứng |
|---|---|---|---|
| **AC-7.1** | Excel xuất ra đúng **4 sheets** chuẩn hóa (Tổng quan, Theo GV, Theo Bài báo, Nhật ký Duyệt) | ✅ | `test_excel_report_has_4_sheets` |
| **AC-7.2** | Mỗi sheet có header ICTU brand màu primary, border, alignment chuẩn | ✅ | Visual inspection + `test_excel_overview_sheet_has_title` |
| **AC-7.3** | File Excel mở được bằng openpyxl, đọc được tất cả cells | ✅ | `test_excel_publication_sheet_has_data`, `test_excel_lecturer_sheet_has_lecturer` |
| **AC-7.4** | CSV có BOM UTF-8, mở trong Excel hiển thị đúng tiếng Việt | ✅ | `test_csv_has_bom` |
| **AC-7.5** | JSON tuân thủ Draft 2020-12 với `$schema` key | ✅ | `test_json_has_draft_schema` |
| **AC-7.6** | Mỗi file xuất ra có mã SHA-256 hex 64 chars | ✅ | `test_excel_sha256_matches_content`, `test_csv_sha256_matches_content`, `test_json_sha256_matches_content` |
| **AC-7.7** | Hash khớp 100% với bytes của file (verify được) | ✅ | `test_sha256_bytes_matches_hashlib` |
| **AC-7.8** | Mỗi lần xuất ghi 1 bản ghi vào `export_audit_log` | ✅ | `test_log_export_creates_record`, `test_log_export_filters_json_saved` |
| **AC-7.9** | API `GET /exports/history` trả về lịch sử có phân trang, có SHA-256 | ✅ | `test_export_history_returns_paginated_list`, `test_export_history_pagination` |
| **AC-7.10** | KPI overview tính đúng `standardization_rate` (chuẩn hoá / trích xuất × 100) | ✅ | `test_kpis_rate_calculation` |
| **AC-7.11** | Trends by year aggregate đúng, ordered ascending | ✅ | `test_trends_aggregates_correctly`, `test_trends_ordered_ascending` |
| **AC-7.12** | Faculty distribution có đầy đủ: faculty / lecturer_count / publication_count / percentage | ✅ | `test_distribution_has_lecturer_count`, `test_distribution_percentage_sums_to_100` |
| **AC-7.13** | Matching distribution trả về 4 bucket (APPROVED, PENDING_VERIFY, MATCH_HIGH, MATCH_MEDIUM) | ✅ | `test_matching_statuses_present` |
| **AC-7.14** | Top lecturers ordered desc theo publication_count | ✅ | `test_top_lecturers_ordered_by_count` |
| **AC-7.15** | **Cả Admin và Operator đều có quyền xuất báo cáo** (FR-23) | ✅ | `test_operator_can_export`, `test_operator_can_view_analytics` |
| **AC-7.16** | Endpoints analytics trả về 401 nếu chưa auth | ✅ | `test_overview_endpoint_requires_auth` × 5 |
| **AC-7.17** | Dashboard là trang mặc định sau login | ✅ | `App.jsx` `<Route path="/" element={<Navigate to="/dashboard" replace />} />` |
| **AC-7.18** | Navbar có `[Dashboard]` và `[Báo cáo & Xuất dữ liệu]` | ✅ | `Navbar.jsx` `MENU_ITEMS` |

### 5.2. Bảng tổng hợp test suite

| File test | Class | Số test | Pass |
|---|---|---|---|
| `test_exports.py` | TestSHA256Helper | 3 | 3 |
| | TestExportFilter | 3 | 3 |
| | TestGenerateExcelReport | 8 | 8 |
| | TestGenerateCSVReport | 5 | 5 |
| | TestGenerateJSONReport | 6 | 6 |
| | TestExportAuditLog | 2 | 2 |
| | TestExportAPI | 11 | 11 |
| `test_analytics.py` | TestOverviewKPIs | 3 | 3 |
| | TestTrendsByYear | 4 | 4 |
| | TestDistributionByFaculty | 5 | 5 |
| | TestMatchingDistribution | 4 | 4 |
| | TestTopLecturers | 5 | 5 |
| | TestAnalyticsAPI | 14 | 14 |
| **Tổng Tuần 7** | | **73** | **73** |
| **Tổng cộng dự án** | | **160** | **160** |

### 5.3. Kết quả Pytest

```bash
$ python -m pytest
============================= 160 passed, 5 warnings in 119.62s (0:01:59) =============================
```

### 5.4. Kết quả Frontend Build

```bash
$ npm run build
✓ 704 modules transformed.
dist/index.html                   0.51 kB │ gzip:   0.37 kB
dist/assets/index-DPv1K4BQ.css   28.08 kB │ gzip:   5.19 kB
dist/assets/index-Xx_QBW_p.js   775.57 kB │ gzip: 228.76 kB │ map: 3,528.06 kB
✓ built in 5.81s
```

> **Ghi chú:** Bundle size lớn (775 KB) do bao gồm `recharts`. Có thể tối ưu bằng code-splitting trong tương lai (Tuần 9/10).

---

## §6. CẤU TRÚC FILE MÃ NGUỒN ĐÃ THAY ĐỔI / TẠO MỚI

### 6.1. Backend

| File | Loại | Mô tả |
|---|---|---|
| `backend/requirements.txt` | Sửa | Thêm `pandas`, `openpyxl` |
| `backend/app/services/export_service.py` | **Mới** | 4-sheet Excel, CSV BOM, JSON Draft 2020-12, SHA-256, audit log |
| `backend/app/services/analytics_service.py` | **Mới** | 5 functions: overview, trends, faculty_dist, matching_dist, top_lecturers |
| `backend/app/schemas/export.py` | **Mới** | Pydantic: ExportFilterSchema, ExportRequest, ExportHistoryItem, ExportResultSchema |
| `backend/app/schemas/analytics.py` | **Mới** | Pydantic: OverviewKPISchema, YearTrendItem, FacultyDistItem, MatchingDistributionSchema, TopLecturerItem |
| `backend/app/schemas/__init__.py` | Sửa | Export thêm analytics + export schemas |
| `backend/app/services/__init__.py` | Sửa | Export thêm export_service + analytics_service |
| `backend/app/api/v1/exports.py` | **Mới** | Router `prefix="/exports"` với 4 endpoints |
| `backend/app/api/v1/analytics.py` | **Mới** | Router `prefix="/analytics"` với 5 endpoints |
| `backend/app/api/v1/__init__.py` | Sửa | Mount exports + analytics routers |
| `backend/tests/conftest.py` | Sửa | Thêm fixtures `db`, `admin_user`, `operator_user` |
| `backend/tests/test_exports.py` | **Mới** | 38 test cases |
| `backend/tests/test_analytics.py` | **Mới** | 35 test cases |

### 6.2. Frontend

| File | Loại | Mô tả |
|---|---|---|
| `frontend/package.json` | Sửa | Thêm `recharts` dependency |
| `frontend/src/api/client.js` | Sửa | Thêm `exportsApi` & `analyticsApi` |
| `frontend/src/pages/DashboardPage.jsx` | **Mới** | 4 KPI cards + BarChart + PieChart + Top lecturers + Faculty bar |
| `frontend/src/pages/ExportPage.jsx` | **Mới** | Form filter + 3 nút export + ResultModal với SHA-256 copy + History table |
| `frontend/src/components/Navbar.jsx` | Sửa | Thêm menu `[Dashboard]` và `[Báo cáo & Xuất dữ liệu]` |
| `frontend/src/App.jsx` | Sửa | Thêm routes `/dashboard`, `/exports`; default redirect → `/dashboard` |

---

## §7. CAM KẾT "ÁO TRẮNG — ÁO VÀNG" TUẦN 7

✅ **Không hardcode cấu hình:**
- File format được quyết định bởi enum `ExportFormat` (không string magic).
- Màu ICTU brand là hằng số ở đầu file `export_service.py` (`ICTU_PRIMARY`, `ICTU_SECONDARY`).
- Pagination defaults lấy từ Pydantic Query schema.

✅ **PostgreSQL & SQLite tương thích:**
- Không sử dụng SQL phụ thuộc dialect (chỉ dùng SQLAlchemy 2.0 `select()` + `func.*`).
- Schema bảng `export_audit_log` đã có sẵn constraint `file_format IN (...)`, `row_count >= 0`.
- Test chạy thành công trên SQLite in-memory trong < 2 phút.

✅ **100% production-ready:**
- Không có chỗ nào ghi `// TODO` hoặc placeholder.
- Toàn bộ 38 + 35 = 73 test case mới đều pass, không có skip hoặc xfail.
- Error handling đầy đủ cho từng endpoint (HTTPException 401, 422, 500).

---

## §8. KẾT LUẬN & HƯỚNG TUẦN 8

### 8.1. Tổng kết Tuần 7

Tuần 7 đã hoàn thành xuất sắc **toàn bộ 4 phần** của đề cương:
- Module xuất báo cáo đa định dạng (Excel 4 sheets, CSV BOM, JSON Draft 2020-12)
- Cơ chế đảm bảo toàn vẹn dữ liệu bằng SHA-256 (NFR-03)
- Ghi vết kiểm toán xuất (`export_audit_log`)
- Dashboard BI với 5 KPI endpoints và frontend trực quan

**Số liệu đạt được:**
- 73 test case mới, 100% pass
- 160/160 tests toàn hệ thống
- Frontend build 0 errors, 704 modules transformed
- 100% production-ready code (không có `// TODO`)

### 8.2. Hướng Tuần 8 (dự kiến)

- **Tính năng phân quyền chi tiết:** Operator chỉ xem báo cáo của GV mình phụ trách.
- **Tích hợp email notification:** Gửi email cho admin khi có xuất báo cáo mới.
- **Tối ưu bundle size:** Code-splitting cho recharts (giảm 775 KB).
- **Audit log viewer UI:** Trang quản lý log audit cho admin.
- **Performance:** Index trên `standardized_data(raw_id)` để query nhanh hơn.

---

**Báo cáo Tuần 7 — Hết.**
*ICTU Scopus Profile Management & Standardization System — Graduation Thesis 2026*
