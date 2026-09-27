# TUẦN 6 — BÁO CÁO TIẾN ĐỘ ĐỒ ÁN TỐT NGHIỆP

**Đề tài:** Hệ thống Quản lý và Chuẩn hóa Hồ sơ Scopus ICTU
**Giai đoạn:** Tuần 6 — Approval Workflow + Diff View + Audit bất biến + Rollback 72h
**Sinh viên:** Trương Quang Huy — MSSV: DTC235200899 — Lớp: CN CNTT K22A
**Giảng viên hướng dẫn:** TS. Nguyễn Thế Vinh — Bộ môn Công nghệ phần mềm, Khoa Công nghệ thông tin (ICTU)
**Trường:** Đại học CNTT & Truyền thông Thái Nguyên (ICTU)
**Ngày cập nhật:** 27/09/2026

> Tài liệu này tổng hợp toàn bộ kết quả Tuần 6: Approval Workflow end-to-end, Audit Service bất biến theo NFR-02, Diff View 2 cột, Rollback 72h theo FR-20, RBAC chặt chẽ và bộ test 87 tự động.

---

## §1. TỔNG QUAN KẾT QUẢ THỰC HIỆN TUẦN 6

### 1.1. Mục tiêu tuần 6 (theo đề cương)
> *"Hoàn thiện luồng nghiệp vụ Phê duyệt (Approval Workflow), Giao diện Diff View đối soát 2 cột, các thao tác Phê duyệt / Gán tay / Chờ xác minh / Hoàn tác Rollback trong 72 giờ và ghi nhật ký kiểm toán bất biến (NFR-02, FR-16 đến FR-21, UC-03, UC-04)."*

### 1.2. Bảng đối chiếu yêu cầu ↔ kết quả

| # | Hạng mục theo đề cương | Trạng thái | Đường dẫn file chính |
|---|---|---|---|
| 1.1 | `audit_service.py` — ghi audit bất biến | ✅ | `backend/app/services/audit_service.py` |
| 1.2 | `POST /approval-queue/{id}/approve` (Admin, chặn Retracted/Erratum) | ✅ | `backend/app/api/v1/approvals.py` |
| 1.3 | `POST /approval-queue/{id}/manual-assign` (Admin) | ✅ | `backend/app/api/v1/approvals.py` |
| 1.4 | `POST /approval-queue/{id}/defer` (Admin) | ✅ | `backend/app/api/v1/approvals.py` |
| 1.5 | `POST /standardized-data/{id}/rollback` (Admin, 72h rule) | ✅ | `backend/app/api/v1/approvals.py` |
| 1.6 | `GET /approval-queue` filter + phân trang | ✅ | `backend/app/api/v1/approvals.py` |
| 1.7 | `GET /approval-queue/{id}/diff` (FR-17) | ✅ | `backend/app/api/v1/approvals.py` |
| 1.8 | `GET /audit-log` viewer | ✅ | `backend/app/api/v1/approvals.py` |
| 2.1 | `QueuePage.jsx` nâng cấp + filter | ✅ | `frontend/src/pages/QueuePage.jsx` |
| 2.2 | `DiffViewModal.jsx` 2 cột + Action Bar | ✅ | `frontend/src/components/DiffViewModal.jsx` |
| 2.3 | Phím tắt A / J / K / Esc | ✅ | (trong `DiffViewModal.jsx`) |
| 2.4 | Badge điểm màu + cờ Retracted/Erratum | ✅ | (trong `DiffViewModal.jsx`) |
| 3.1 | `test_approvals.py` (16 tests) | ✅ 16/16 PASS | `backend/tests/test_approvals.py` |
| 3.2 | Audit log redaction (password_hash) | ✅ | `test_audit_sanitize_redacts_sensitive_keys` |
| 3.3 | RBAC: operator → 403 trên mutate | ✅ | 2 tests `test_operator_cannot_*` |
| 3.4 | `TUAN6.MD` báo cáo tổng thể | ✅ | `TUAN6.MD` |
| 3.5 | Smoke end-to-end `W6_smoke_approval.py` | ✅ | `backend/W6_smoke_approval.py` |

**Tổng kết:** 100% các hạng mục Tuần 6 hoàn thành. **87/87 pytest PASS** (71 cũ + 16 mới). Frontend build sạch (115 modules, 0 lỗi).

### 1.3. Nguyên tắc thực thi
- **Áo trắng — Áo vàng:** 0 hardcode. Mọi threshold (72h rollback window) đi qua `settings.APPROVAL_ROLLBACK_WINDOW_HOURS`. Audit sanitize tự động áp dụng cho mọi entry qua `_sanitize_payload`.
- **PostgreSQL ↔ SQLite auto-fallback:** Đã smoke test trên cả hai mode.
- **Audit bất biến (NFR-02):** Bảng `audit_log` được khai báo với CHECK constraint cho phép action; service `log_action()` chỉ INSERT (không có API update/delete), correlation_id liên kết các entry trong cùng transaction.
- **RBAC chặt chẽ:** Mọi endpoint mutate (`approve` / `manual-assign` / `defer` / `rollback`) yêu cầu `require_role("admin")`; khi operator cố truy cập, FastAPI raise 403 + `security_audit_log` được ghi.

---

## §2. THIẾT KẾ KIẾN TRÚC LUỒNG PHÊ DUYỆT

### 2.1. Mục tiêu nghiệp vụ
- **FR-16:** Phê duyệt bản ghi HIGH/MEDIUM → tạo `standardized_data`.
- **FR-17:** Diff View 2 cột giúp Admin kiểm tra thông tin trước khi duyệt.
- **FR-18:** Manual Override — Admin gán GV bất kỳ cho queue (có lý do).
- **FR-19:** Auto-block khi tài liệu thuộc diện Retracted/Erratum.
- **FR-20:** Rollback trong vòng 72h kể từ `approved_at`.
- **FR-21:** Defer — chuyển về `PENDING_VERIFY` chờ xác minh.

### 2.2. Sơ đồ trạng thái

```
                  ┌─────────────────────┐
                  │   MATCH_HIGH        │
                  │   (score >= 0.85)   │ ─── approve ──┐
                  └─────────┬───────────┘               │
                            │                           ▼
                            │                ┌─────────────────────────┐
                  ┌─────────▼──────────┐     │ standardized_data      │
                  │   MATCH_MEDIUM     │ ─── approve ─┐           │
                  │ (0.50<=score<0.85) │              │           │
                  └─────────┬──────────┘              │           ▼
                            │                         │   is_invalidated=False
                            │ defer                   │
                            ▼                         │  ┌──────────────────┐
                  ┌─────────────────────┐             │  │   APPROVED       │
                  │   PENDING_VERIFY    │◄────────────┘  │                  │
                  │ (score < 0.50)      │                └────────┬─────────┘
                  └─────────┬───────────┘                         │
                            │                                     │ rollback
                            │ approve                             │ (within 72h)
                            │                                     │
                            └─────────────────────────────────────┘
                                                │
                                                ▼
                                       ┌──────────────────┐
                                       │  ROLLBACK        │
                                       │ is_invalidated=True│
                                       │ queue=PENDING    │
                                       └──────────────────┘
```

### 2.3. Vòng đời dữ liệu — Data Lifecycle

| Trạng thái | Bảng chính | Hành động | Audit action |
|---|---|---|---|
| Khởi tạo | `approval_queue.status='MATCH_HIGH'`/`MEDIUM`/`PENDING_VERIFY` | process_batch (Tuần 5) | — |
| Phê duyệt | → `approval_queue.status='APPROVED'` + `standardized_data` (mới) | `POST /approval-queue/{id}/approve` | `APPROVE` |
| Gán tay | → `approval_queue.lecturer_id` thay đổi + `standardized_data` (mới) | `POST /approval-queue/{id}/manual-assign` | `MANUAL_OVERRIDE` |
| Defer | → `approval_queue.status='PENDING_VERIFY'` | `POST /approval-queue/{id}/defer` | `DEFER` |
| Rollback OK | → `standardized_data.is_invalidated=True`, queue trở về PENDING | `POST /standardized-data/{id}/rollback` | `ROLLBACK` |
| Rollback quá hạn | Không thay đổi, log lý do từ chối | (raise 409) | `ROLLBACK_DENIED` |

### 2.4. Quy tắc Rollback 72h (FR-20)

```python
window_hours = settings.APPROVAL_ROLLBACK_WINDOW_HOURS  # default 72
hours_elapsed = (utcnow - std.approved_at).total_seconds() / 3600

if hours_elapsed > window_hours:
    log_action(ROLLBACK_DENIED, reason="Xh > window Yh")
    raise RollbackWindowExpiredError  # HTTP 409
```

Cấu hình qua biến môi trường `APPROVAL_ROLLBACK_WINDOW_HOURS`, mặc định 72h.

### 2.5. Các exception domain → HTTP mapping

| Exception | HTTP | Code | Ý nghĩa |
|---|---|---|---|
| `RetractedOrErratumError` | 400 | `DOCUMENT_BLOCKED` | Bài bị Retracted/Erratum, không được duyệt |
| `QueueNotFoundError` | 404 | `QUEUE_NOT_FOUND` | Queue ID không tồn tại |
| `StandardizedNotFoundError` | 404 | `STANDARDIZED_NOT_FOUND` | Std ID không tồn tại |
| `LecturerNotFoundError` | 404 | `LECTURER_NOT_FOUND` | GV không tồn tại / đã inactive |
| `InvalidStateTransitionError` | 409 | `INVALID_STATE` | Double-approve hoặc defer khi đã PENDING |
| `AlreadyInvalidatedError` | 409 | `ALREADY_INVALIDATED` | Rollback khi `is_invalidated=True` |
| `RollbackWindowExpiredError` | 409 | `ROLLBACK_WINDOW_EXPIRED` | Quá 72h |
| `ApprovalError` (base) | 400 | tuỳ `code` | Lỗi chung |

### 2.6. Sơ đồ dòng dữ liệu — Approve

```
Operator/Admin ─── POST /approval-queue/{id}/approve ───▶ ApproveRequest
                                                        │
                                                        ▼
                                         require_admin (RBAC layer)
                                                        │
                                                        ▼
                                    approval_workflow.approve_queue()
                                                        │
                                  ┌─────────────────────┼─────────────────────┐
                                  │                     │                     │
                       is_retracted?         status != APPROVED       OK flow
                          │ ERR                    │ OK                     │
                          ▼                        ▼                        ▼
            RetractedOrErratumError(400)  q.status="APPROVED"          q.status ← "APPROVED"
                                        approved_at ← utcnow()          q.approved_by ← actor.id
                                        approved_by ← actor.id          standardized_data ← NEW
                                                                             │
                                                                             ▼
                                                       audit_service.log_action(APPROVE)
                                                                             │
                                                                             ▼
                                                                         db.commit()
                                                                             │
                                                                             ▼
                                                          ApprovalActionResponse
```

---

## §3. CƠ CHẾ KIỂM TOÁN BẤT BIẾN (`audit_log` NFR-02) & BẢO MẬT RBAC

### 3.1. Nguyên tắc bất biến (NFR-02)

- Bảng `audit_log` chỉ cho phép **INSERT** thông qua `audit_service.log_action()`. Không có API update/delete.
- Mỗi record có `correlation_id` (UUID) liên kết các entry trong cùng 1 transaction nghiệp vụ (ví dụ: 1 lần Approve tạo ra 2 entry: 1 cho `approval_queue`, 1 cho `standardized_data`).
- Mọi hành động nghiệp vụ đều ghi audit:
  - `LECTURER_CREATE` / `LECTURER_UPDATE` / `LECTURER_DELETE`
  - `APPROVE`
  - `MANUAL_OVERRIDE`
  - `DEFER`
  - `ROLLBACK` / `ROLLBACK_DENIED`
- CHECK constraint trên cột `action` đảm bảo chỉ chấp nhận 13 action hợp lệ (xem `models/__init__.py::audit_log_action_check`).

### 3.2. Audit Sanitization

`_sanitize_payload()` tự động loại bỏ các key nhạy cảm trước khi ghi:

```python
_SENSITIVE_KEYS = frozenset({
    "password", "password_hash", "new_password", "old_password",
    "access_token", "refresh_token", "jwt", "secret",
    "secret_key", "credit_card",
})
```

Nếu dict chứa key nhạy cảm → thay bằng `[REDACTED]`. Test `test_audit_sanitize_redacts_sensitive_keys` xác nhận điều này.

### 3.3. Phân quyền RBAC

| Endpoint | Operator | Admin |
|---|:---:|:---:|
| `GET /approval-queue` | ✅ | ✅ |
| `GET /approval-queue/{id}/diff` | ✅ | ✅ |
| `POST /approval-queue/{id}/approve` | ❌ → **403** | ✅ |
| `POST /approval-queue/{id}/manual-assign` | ❌ → **403** | ✅ |
| `POST /approval-queue/{id}/defer` | ❌ → **403** | ✅ |
| `POST /standardized-data/{id}/rollback` | ❌ → **403** | ✅ |
| `GET /audit-log` | ❌ → **403** | ✅ |
| Lecturer CRUD (Tuần 5) | ❌ → **403** | ✅ |

Khi operator cố truy cập admin endpoint:
1. `require_role("admin")` raise `HTTPException(403)`.
2. Trước khi raise, `security_audit_log` được ghi với `http_status=403`, `required_permission="admin"`, `actor_id`, IP, User-Agent.

### 3.4. Correlation ID pattern

```python
correlation_id = str(uuid4())  # Sinh 1 lần cho mỗi transaction nghiệp vụ

# Approve tạo 2 audit entry với CÙNG correlation_id:
log_action(action=APPROVE, target_table=approval_queue, target_id=q.id, ...)
log_action(action=APPROVE, target_table=standardized_data, target_id=std.id, ...)

# Khi audit-log viewer filter theo correlation_id → truy ngược đầy đủ lịch sử 1 thao tác
```

---

## §4. MÔ TẢ GIAO DIỆN DIFF VIEW, PHÍM TẮT & TRẢI NGHIỆM NGƯỜI DÙNG

### 4.1. Layout Diff View

`DiffViewModal.jsx` chia màn hình thành 2 cột rõ ràng:

#### Cột trái — Scopus (dữ liệu gốc)
- **Header badge:** Trạng thái (HIGH / MEDIUM / PENDING / APPROVED) + cờ Retracted/Erratum.
- **Tiêu đề bài báo** (title), **Năm**, **Cited by**, **Publisher**, **DOI** (kèm link `doi.org/<doi>`).
- **Tác giả Scopus** dạng monospace; highlight vị trí tác giả đang xét (`#2/4`).
- **Author full names** và **Author(s) ID** ở dạng monospace để dễ đối chiếu.

#### Cột phải — Lecturer ICTU (đề xuất)
- **Card thông tin GV:** Họ tên, Mã GV (`GV001..GV010`), Khoa/Bộ môn, Scopus Author ID, Email.
- **Score Bar** với 3 thanh tiến trình:
  - **Tổng hợp** (màu green ≥85%, amber ≥50%, slate <50%)
  - **Name (fuzzy)** — RapidFuzz
  - **Affiliation ICTU**
- **Trọng số hiển thị:** `w₁=0.55 (ID), w₂=0.35 (Name), w₃=0.10 (Affil)`.
- **Bằng chứng đối chiếu (Evidence Payload)** dạng checklist:
  - ✅ Khớp Author ID Scopus
  - ✅ Khớp tên mờ (X%) + ghi chú penalty nếu họ phổ biến
  - ✅ Khớp đơn vị ICTU
  - ✅ Cờ họ phổ biến VN
  - ✅/❌ Cờ an toàn (Retracted/Erratum)
- **Thông tin rollback** nếu đã duyệt: "Đã duyệt Xh trước · còn <deadline> deadline rollback".

### 4.2. Action Bar

| Nút | Màu | Phím tắt | Điều kiện kích hoạt |
|---|---|---|---|
| **Phê duyệt** | 🟢 Xanh | `A` | Admin + `can_approve=true` + không bị Retracted/Erratum |
| **Gán tay GV khác** | 🔵 Lam | — | Admin (mọi trạng thái trừ Retracted/Erratum) |
| **Chờ xác minh** | 🟡 Vàng | — | Admin (chuyển queue về PENDING_VERIFY) |
| **Rollback** | 🟠 Cam | — | Admin + `can_rollback=true` (còn trong 72h) |

Mỗi button đều có **tooltip** giải thích lý do disable. Ví dụ:
- `"Bài bị Retracted/Erratum — không thể duyệt"`
- `"Đã quá 72h hoặc bản ghi chưa APPROVED"`

### 4.3. Phím tắt

| Phím | Hành động |
|---|---|
| `A` | Approve (chỉ Admin) |
| `J` | Next record (queue tiếp theo) |
| `K` | Previous record |
| `Esc` | Đóng Diff Modal |

> Phím tắt KHÔNG hoạt động khi focus đang ở `<input>` / `<textarea>` / `<select>`.

### 4.4. Dialog phụ (Sub-dialogs)

- **Manual Assign Dialog:** search GV theo tên/mã/khoa, chọn từ dropdown, bắt buộc nhập lý do ≥ 3 ký tự.
- **Defer Dialog:** textarea lý do ≥ 3 ký tự.
- **Rollback Dialog:** cảnh báo deadline + textarea lý do ≥ 3 ký tự.

### 4.5. Filter & Search (`QueuePage.jsx`)

| Filter | Loại |
|---|---|
| Trạng thái | Select: Tất cả / MATCH_HIGH / MATCH_MEDIUM / PENDING_VERIFY / APPROVED |
| Retracted | Select: Tất cả / Có / Không |
| Erratum | Select: Tất cả / Có / Không |
| Tìm kiếm | Input text: title / tên tác giả / GV |

Phân trang: 15 dòng / trang, nút `← Trước` / `Sau →`.

### 4.6. Bảng danh sách

Mỗi dòng hiển thị:
- Queue ID (monospace)
- Title + EID + năm
- Tác giả (lastname initials + normalized_name)
- GV gợi ý (tên + mã GV)
- Score %
- Badge trạng thái (màu theo bucket)
- Cờ ⚠R / ⚠E (Retracted / Erratum)
- Nút "Xem diff →"

Click vào dòng → mở Diff Modal. Nút "Xem diff" stop-propagation để tránh conflict.

---

## §5. TIÊU CHÍ NGHIỆM THU KỸ THUẬT & KẾT QUẢ KIỂM THỬ

### 5.1. Bảng tiêu chí nghiệm thu

| # | Tiêu chí (AC / FR / NFR) | Phương pháp kiểm | Kết quả |
|---|---|---|---|
| AC-06.1 | Approve HIGH tạo `standardized_data` + 2 audit entry | `test_approve_high_creates_standardized_and_audit` | ✅ |
| AC-06.2 | Approve bị chặn nếu retracted (HTTP 400) | `test_approve_retracted_returns_400` | ✅ |
| AC-06.3 | Approve bị chặn nếu erratum (HTTP 400) | `test_approve_erratum_returns_400` | ✅ |
| AC-06.4 | Manual assign đổi lecturer + audit MANUAL_OVERRIDE | `test_manual_assign_changes_lecturer_and_logs_audit` | ✅ |
| AC-06.5 | Defer → PENDING_VERIFY + audit DEFER | `test_defer_returns_to_pending` | ✅ |
| AC-06.6 | Rollback trong 72h → is_invalidated=True | `test_rollback_within_72h_succeeds` | ✅ |
| AC-06.7 | Rollback quá 72h → HTTP 409 + ROLLBACK_DENIED audit | `test_rollback_after_72h_returns_409` | ✅ |
| AC-06.8a | Operator không thể approve (HTTP 403) | `test_operator_cannot_approve` | ✅ |
| AC-06.8b | Operator không thể rollback (HTTP 403) | `test_operator_cannot_rollback` | ✅ |
| AC-06.9 | Diff view payload đầy đủ 2 cột + capability flags | `test_diff_view_payload_complete` | ✅ |
| AC-06.10 | List queue filter (status, batch_id, q, is_retracted) | `test_list_queue_filters` | ✅ |
| AC-06.11 | Audit log viewer cho admin | `test_audit_log_viewer` | ✅ |
| AC-06.12 | Audit sanitize (password_hash redaction) | `test_audit_sanitize_redacts_sensitive_keys` | ✅ |
| AC-06.13 | Double-approve → HTTP 409 INVALID_STATE | `test_double_approve_returns_409` | ✅ |
| AC-06.14 | Manual assign missing reason → 422 | `test_manual_assign_missing_reason_returns_422` | ✅ |
| AC-06.15 | Operator không truy cập audit-log | `test_audit_log_viewer_forbidden_for_operator` | ✅ |
| FR-16 | Approve tạo standardized_data | (kết hợp AC-06.1) | ✅ |
| FR-17 | Diff View 2 cột | `test_diff_view_payload_complete` | ✅ |
| FR-18 | Manual Override | `test_manual_assign_*` | ✅ |
| FR-19 | Block Retracted/Erratum | `test_approve_*_returns_400` | ✅ |
| FR-20 | Rollback 72h rule | `test_rollback_*` | ✅ |
| FR-21 | Defer chuyển về PENDING | `test_defer_returns_to_pending` | ✅ |
| NFR-02 | Audit log bất biến + sanitize | `test_audit_sanitize_*` + model CHECK constraint | ✅ |
| RBAC | 4 endpoint admin có require_role | 2 RBAC tests | ✅ |

### 5.2. Kết quả chạy pytest toàn bộ

```
============================= test session starts =============================
platform win32 -- Python 3.14.0, pytest-9.1.1, pluggy-1.6.1
rootdir: D:\DataFolder\ProjectIctu\DoAn\scopus-ictu\backend
configfile: pytest.ini
plugins: anyio-4.15.1
collected 87 items

tests\test_approvals.py ................                                 [ 18%]
tests\test_auth.py ........                                              [ 27%]
tests\test_author_extractor.py ..........                                [ 39%]
tests\test_lecturers.py ..............                                   [ 55%]
tests\test_matching_engine.py .........................                  [ 83%]
tests\test_pipeline_api.py ......                                        [ 90%]
tests\test_upload_idempotency.py ........                                [100%]

======================= 87 passed, 4 warnings in 14.45s =======================
```

### 5.3. Smoke test end-to-end (`W6_smoke_approval.py`)

```
=== STEP 0: DB check ===  engine=sqlite url=sqlite:///./scopus_dev.db
=== STEP 1: Doc file ===  rows=660, batch_id=1
=== STEP 2: Run pipeline ===
  total_authors=2757, HIGH=0, MEDIUM=0, PENDING=2757
  retracted=17, erratum=16
  duration=1856 ms

=== STEP 3: Demo Approval Workflow ===
  [OK] approve queue #1 -> std_id=1, lecturer_id=1
  [OK] manual_assign queue #3 -> GV "Lê Thanh Hương", std_id=2
  [OK] rollback quá 72h bị chặn: RollbackWindowExpiredError (73.0h > 72h)
  [OK] rollback hợp lệ: is_invalidated=True, queue_status=PENDING_VERIFY

=== STEP 4: Final statistics ===
  audit_log total: 7
  audit_log by action: {'APPROVE': 2, 'MANUAL_OVERRIDE': 2, 'ROLLBACK_DENIED': 1, 'ROLLBACK': 2}
  standardized_data: total=2, invalidated=1
  APPROVED queue count: 1
```

### 5.4. Vấn đề đã xử lý trong tuần 6

| # | Vấn đề | Xử lý |
|---|---|---|
| 1 | CHECK constraint `audit_log.action` quá hẹp | Mở rộng whitelist trong `models/__init__.py` thêm `ROLLBACK_DENIED`, `LECTURER_CREATE/UPDATE/DELETE`, `PIPELINE_PROCESS`, `EXPORT`, `LOGIN_*` |
| 2 | `evidence_payload` key không nhất quán giữa Stage 3 (`author_id_match`) và DiffView schema (`scopus_author_id_match`) | `get_diff_view()` chấp nhận cả 2 key qua `ev.get("scopus_author_id_match") or ev.get("author_id_match")` |
| 3 | `correlation_id` collision khi nhiều test dùng cùng string | Helper `_setup_batch` dùng `uuid.uuid4()` |
| 4 | Audit sanitize test inject thẳng `AuditLog` (bypass sanitize) | Test đổi sang gọi qua `log_action()` để verify pipeline sanitize thực sự |
| 5 | Frontend thiếu RBAC check trong DiffViewModal | Mọi button đều có `disabled={!isAdmin || ...}` |

---

## §PHỤ LỤC

### A. Cấu trúc file mới / cập nhật trong tuần 6

```
backend/
├── app/
│   ├── services/
│   │   ├── audit_service.py          [NEW]  log_action + sanitize + actions
│   │   └── approval_workflow.py      [NEW]  approve/manual/defer/rollback/list/diff
│   ├── schemas/
│   │   ├── approval.py               [NEW]  DiffView/QueueList/Approve/ManualAssign/Defer/Rollback/AuditLog
│   │   └── __init__.py               [UPDATED] export approval schemas
│   ├── api/v1/
│   │   ├── approvals.py              [NEW]  7 endpoint với RBAC
│   │   ├── lecturers.py              [UPDATED] + audit logging
│   │   └── __init__.py               [UPDATED] mount approvals router
│   ├── core/
│   │   └── config.py                 [UPDATED] + APPROVAL_ROLLBACK_WINDOW_HOURS, AUDIT_INSERT_ONLY_ENFORCED
│   └── models/
│       └── __init__.py               [UPDATED] expanded audit_log.action CHECK
├── tests/
│   └── test_approvals.py             [NEW] 16 tests
└── W6_smoke_approval.py              [NEW] end-to-end smoke

frontend/src/
├── api/
│   └── client.js                     [UPDATED] + approvalsApi
├── components/
│   └── DiffViewModal.jsx             [NEW] 2-cột + action bar + phím tắt
└── pages/
    └── QueuePage.jsx                 [REWRITE] filter + table + diff trigger

TUAN6.MD                              [NEW] This document
```

### B. Hướng dẫn chạy thực nghiệm

```bash
# 1. Chạy approval smoke end-to-end
cd backend
Remove-Item scopus_dev.db -ErrorAction SilentlyContinue
$env:APP_ENV = 'development'
$env:DATABASE_FALLBACK_SQLITE = 'True'
$env:DATABASE_URL = 'sqlite:///./scopus_dev.db'
.\.venv\Scripts\python.exe W6_smoke_approval.py

# 2. Chạy toàn bộ 87 tests
$env:APP_ENV = 'test'
.\.venv\Scripts\python.exe -m pytest tests/ -v

# 3. Build frontend
cd ../frontend
npm run build
```

### C. Tổng kết pipeline end-to-end (Tuần 5 + Tuần 6)

```
  Upload CSV  ─▶  raw_scopus_data  ─▶  process_batch (Stage 2+3)
                                              │
                                              ▼
                                       author_extracted (2 757)
                                              │
                                              ▼
                                       approval_queue (2 757)
                                              │
                          ┌───────────────────┼───────────────────┐
                          ▼                   ▼                   ▼
                    Admin xem Diff       Approve / Manual      Defer
                    (DiffViewModal)      Override (Tuần 6)
                                              │
                                              ▼
                                    standardized_data
                                              │
                                              ▼
                                    Rollback (72h, Tuần 6)
                                              │
                                              ▼
                                     audit_log (NFR-02)
```

### D. Lộ trình Tuần 7 (preview)

- Module xuất CSV / Excel báo cáo cho nhà trường (`POST /exports/...` + `export_audit_log`).
- Bộ lọc nâng cao trên QueuePage (lọc theo khoảng thời gian `approved_at`, theo lecturer, theo batch).
- Dashboard thống kê tổng quan (HIGH/MEDIUM/PENDING theo năm, theo khoa).
- Re-process pipeline khi Lecturer CRUD thay đổi (auto-recompute match score).
- Tích hợp OpenAlex API để enrich affiliation cho các publication không có ICTU keyword.

---

**Ký tên:** `backend/`
**Phiên bản báo cáo:** v1.0 (Tuần 6)
