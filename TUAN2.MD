# Đặc tả Yêu cầu — Hệ thống Chuẩn hóa Hồ sơ Scopus ICTU
## Nhiệm vụ 1, Tuần 2 — Actors, Functional & Non-Functional Requirements

> **Đề tài:** Xây dựng ứng dụng web quản lý và chuẩn hóa hồ sơ tác giả, công bố khoa học trên Scopus phục vụ Trường Đại học Công nghệ thông tin và Truyền thông (ICTU).
> **Căn cứ:** Mục 7b — PI 1.1, 1.2, 2.1, 3.3, 3.4 trong đề cương đồ án; kết quả khảo sát Tuần 1 (`scopus-ictu-data-survey.md`); đặc tả quy trình (`scopus-ictu-process-spec.md`); sơ đồ BPMN/PlantUML cùng thư mục.
> **Người soạn:** Trương Quang Huy — SV ngành CNTT, ICTU.
> **Ngày:** 26/09/2026.

---

## Mục lục
1. [Xác định vai trò người dùng & tác nhân hệ thống](#1-xác-dịnh-vai-trò-người-dùng--tác-nhân-hệ-thống)
2. [Yêu cầu chức năng — Functional Requirements (FR)](#2-yêu-cầu-chức-năng--functional-requirements-fr)
3. [Yêu cầu phi chức năng — Non-Functional Requirements (NFR)](#3-yêu-cầu-phi-chức-năng--non-functional-requirements-nfr)
4. [Ma trận truy vết yêu cầu → PI](#4-ma-trận-truy-vết-yêu-cầu--pi)

---

## 1. Xác định vai trò người dùng & tác nhân hệ thống

Hệ thống có **3 tác nhân** (actor) theo đúng swimlane trong `scopus-ictu-swimlane.puml`:
1 người dùng chuyên trách (Operator), 1 người dùng có thẩm quyền (Admin/Approver), 1 tác nhân kỹ thuật tự động (Background Worker).

### 1.1. Bảng tổng hợp Actor

| Mã | Tên Actor | Loại | Swimlane | Mô tả ngắn |
|---|---|---|---|---|
| **ACT-01** | Chuyên viên xử lý dữ liệu (ICTU Operator) | Người dùng | Chuyên viên ICTU | Người trực tiếp vận hành: tải file, theo dõi pipeline, xuất báo cáo |
| **ACT-02** | Quản trị viên / Cán bộ duyệt (ICTU Admin / Approver) | Người dùng | Cán bộ duyệt | Người có thẩm quyền phê duyệt ánh xạ tác giả - giảng viên, quản lý danh mục |
| **ACT-03** | Tác nhân hệ thống tự động (Background Worker / System Engine) | Hệ thống | Hệ thống Python | Engine xử lý nền: parse, chuẩn hoá, tính Match Score, ghi log |

---

### 1.2. ACT-01 — Chuyên viên xử lý dữ liệu (ICTU Operator)

**Vai trò nghiệp vụ:** Đầu mối kỹ thuật của Phòng Sau đại học / Phòng Khoa học Công nghệ ICTU; chịu trách nhiệm đưa dữ liệu Scopus thô vào hệ thống và kết xuất báo cáo chuẩn.

**Trách nhiệm chính:**
- Chuẩn bị và tải tệp CSV Scopus xuất từ cổng Scopus theo định kỳ (tháng/quý).
- Theo dõi log pipeline để xác nhận lô nhập đã xử lý thành công.
- Lọc và trích xuất dữ liệu chuẩn hoá để nộp báo cáo kiểm định hoặc chia sẻ với các đơn vị liên quan.
- Phối hợp với Admin khi phát sinh ngoại lệ dữ liệu không tự động xử lý được.

**Quyền hạn (RBAC):**
| Mã quyền | Tên quyền | Phạm vi |
|---|---|---|
| `op.upload` | Tải tệp CSV lên hệ thống | Toàn hệ thống |
| `op.view_raw` | Xem dữ liệu RAW_DATA và trạng thái xử lý | Toàn hệ thống |
| `op.filter` | Lọc dữ liệu theo Năm / Khoa-Bộ môn / Publisher / Trạng thái | Toàn hệ thống |
| `op.export` | Xuất báo cáo (Excel/CSV/JSON) | Standardized Data |
| `op.audit_read` | Xem lịch sử xuất dữ liệu (EXPORT_AUDIT_LOG) | Bản thân |
| `op.approve` | **KHÔNG được phép** | — |
| `op.rollback` | **KHÔNG được phép** | — |
| `op.lecturer_crud` | **KHÔNG được phép** | — |

**Các tác vụ chính trên hệ thống (Use case liên kết):**
- `UC-01` Tải tệp Scopus CSV lên hệ thống.
- `UC-04` Theo dõi trạng thái xử lý lô nhập (dashboard).
- `UC-09` Lọc dữ liệu theo tiêu chí.
- `UC-10` Xuất báo cáo chuẩn hoá.

**SLA / Giới hạn xử lý:**
- Tối đa **3 lần re-import cùng một lô EID** trong vòng 24 giờ — quá số lần hệ thống sẽ tự chặn để bảo vệ Idempotency (NFR-01).

---

### 1.3. ACT-02 — Quản trị viên / Cán bộ duyệt (ICTU Admin / Approver)

**Vai trò nghiệp vụ:** Cán bộ phòng ban có thẩm quyền xác nhận ánh xạ giữa tác giả Scopus và giảng viên ICTU; chịu trách nhiệm pháp lý về tính chính xác của dữ liệu chuẩn hoá cuối cùng.

**Trách nhiệm chính:**
- Đối soát, phê duyệt / gán tay / chuyển chờ xác minh cho từng bản ghi trong hàng đợi `APPROVAL_QUEUE`.
- Hoàn tác (Rollback) khi phát hiện duyệt nhầm.
- Quản lý Danh mục Giảng viên ICTU (thêm, sửa, gộp, tách bản ghi giảng viên).
- Xem toàn bộ `AUDIT_LOG` để phục vụ kiểm toán nội bộ và thanh tra.

**Quyền hạn (RBAC):**
| Mã quyền | Tên quyền | Phạm vi |
|---|---|---|
| `admin.approve` | Phê duyệt liên kết tác giả Scopus - giảng viên ICTU | Standardized Data |
| `admin.manual_assign` | Gán thủ công giảng viên ICTU cho tác giả Scopus | Standardized Data |
| `admin.defer` | Chuyển sang trạng thái Chờ xác minh | Approval Queue |
| `admin.rollback` | Hoàn tác quyết định duyệt đã ban hành | Standardized Data (≤ 72 giờ) |
| `admin.lecturer_crud` | CRUD Danh mục Giảng viên ICTU | Master Data |
| `admin.audit_full_read` | Xem toàn bộ `AUDIT_LOG` (mọi actor) | Toàn hệ thống |
| `admin.view_raw` | Xem dữ liệu RAW_DATA + warning | Toàn hệ thống |
| `admin.export` | Xuất báo cáo chuẩn hoá | Standardized Data |
| `admin.user_mgmt` | Quản lý tài khoản Operator và Admin | RBAC |

**Các tác vụ chính trên hệ thống (Use case liên kết):**
- `UC-05` Mở Approval Queue, xem Diff trước/sau chuẩn hoá.
- `UC-06` Phê duyệt / Gán tay / Chờ xác minh.
- `UC-07` Rollback bản ghi đã duyệt.
- `UC-08` Quản lý Danh mục Giảng viên ICTU.
- `UC-09` Lọc dữ liệu (cùng Operator).
- `UC-10` Xuất báo cáo chuẩn hoá (cùng Operator).

**Ràng buộc nghiệp vụ:**
- Mọi thao tác phê duyệt phải có **bằng chứng đối chiếu** (`evidence_payload` từ bước tính Match Score) đính kèm `STANDARDIZED_DATA` — NFR-02.
- Rollback chỉ khả dụng trong **72 giờ** kể từ lúc duyệt — sau thời hạn phải lập phiếu điều chỉnh riêng.

---

### 1.4. ACT-03 — Tác nhân hệ thống tự động (Background Worker / System Engine)

**Vai trò nghiệp vụ:** Engine xử lý nền chạy bất đồng bộ (background job), thực thi toàn bộ pipeline xử lý dữ liệu mà không cần can thiệp thủ công.

**Các tác vụ ngầm tự động (System Tasks):**

| Mã | Tác vụ | Mô tả | Trigger |
|---|---|---|---|
| `ST-01` | Schema validation | Kiểm tra 22 cột theo đặc tả, encoding UTF-8 | Ngay khi Operator upload |
| `ST-02` | Idempotency check | Đối chiếu EID + Author ID với lô trước; đánh dấu DUPLICATE | Sau schema OK |
| `ST-03` | Raw data persistence | Insert/Upsert vào bảng `RAW_DATA` | Sau idempotency |
| `ST-04` | Row-level validation | Phát hiện & đánh dấu dòng thiếu DOI / Author ID / Year / Affiliation | Sau persistence |
| `ST-05` | Author parsing | Tách chuỗi `Authors` và `Author(s) ID` thành danh sách tác giả có cấu trúc | Sau validation |
| `ST-06` | Name normalization | Chuẩn hoá tiếng Việt không dấu, lower, loại bỏ khoảng trắng thừa | Sau parsing |
| `ST-07` | Author counting | Đếm số tác giả / bài, lưu `AUTHOR_EXTRACTED` | Sau normalization |
| `ST-08` | Exact match | So khớp chính xác qua `Scopus Author ID` với Danh mục GV ICTU | Song song với ST-09 |
| `ST-09` | Fuzzy match | So khớp mờ qua tên không dấu + Affiliation | Song song với ST-08 |
| `ST-10` | Score computation | Tính Match Score ∈ [0,1], sinh `evidence_payload` | Sau ST-08 + ST-09 |
| `ST-11` | Score classification | Phân nhóm HIGH/MEDIUM/PENDING theo ngưỡng 85/50% | Sau ST-10 |
| `ST-12` | Queue persistence | Ghi `APPROVAL_QUEUE` với status tương ứng | Sau ST-11 |
| `ST-13` | Pipeline logging | Ghi `log_pipeline` (log_level, raw_id, score) | Xuyên suốt |
| `ST-14` | Admin notification | Gửi notification (WebSocket / email nội bộ) khi có queue mới | Sau ST-12 |
| `ST-15` | Audit log persistence | Ghi `AUDIT_LOG` cho mọi thao tác của Admin | Khi Admin duyệt |
| `ST-16` | Rollback executor | Khôi phục `APPROVAL_QUEUE.status = PENDING`, ghi AUDIT_LOG | Khi Admin rollback |
| `ST-17` | Export engine | Kết xuất Excel/CSV/JSON chuẩn, ký số, ghi `EXPORT_AUDIT_LOG` | Khi Operator export |

**Đặc điểm:**
- Không có giao diện người dùng; tương tác qua **message queue** (RabbitMQ/Redis) và **REST webhook**.
- Mỗi job phải **idempotent ở cấp job** (job_id + dedup_key) — tránh xử lý trùng khi retry.
- Mọi log đều có **correlation_id** liên kết từ lúc upload đến lúc xuất báo cáo.

---

## 2. Yêu cầu chức năng — Functional Requirements (FR)

Yêu cầu chức năng được phân thành **4 nhóm phân hệ** theo đúng cấu trúc 4 giai đoạn quy trình đã đặc tả trong `scopus-ictu-process-spec.md`. Mỗi FR có mã định danh duy nhất, tên, mô tả tóm tắt, và ánh xạ về PI của Mục 7b.

### 2.1. FR1 — Tiếp nhận và kiểm tra dữ liệu (Import & Validation)

| Mã FR | Tên yêu cầu | Mô tả tóm tắt | PI |
|---|---|---|---|
| **FR-01** | Tải tệp Scopus CSV | Hệ thống cung cấp giao diện Upload (kéo-thả hoặc chọn file) cho phép Operator tải tệp `.csv` xuất từ Scopus với dung lượng tối đa 50 MB. | PI 1.1 |
| **FR-02** | Kiểm tra cấu trúc schema | Hệ thống xác thực 22 trường bắt buộc theo đặc tả (`Authors`, `Author full names`, `Author(s) ID`, `Title`, `Year`, `DOI`, `EID`, …) và encoding UTF-8. Sai cấu trúc → trả JSON lỗi chi tiết + dừng luồng. | PI 1.1, PI 1.2 |
| **FR-03** | Kiểm tra Idempotency theo EID | Trước khi insert, đối chiếu `(EID, Author ID)` với các lô nhập trước. Bản ghi đã tồn tại → đánh dấu `duplicate = true`, không insert lại. | PI 1.1 |
| **FR-04** | Lưu trữ RAW_DATA | Mọi bản ghi hợp lệ được insert nguyên vẹn 22 trường vào bảng `RAW_DATA`, kèm `batch_id`, `operator_id`, `uploaded_at` phục vụ truy vết nguồn. | PI 1.1 |
| **FR-05** | Validation cấp dòng (row-level) | Phát hiện và đánh dấu cảnh báo các dòng thiếu DOI / Author ID / Year / Affiliation. Hệ thống **không dừng luồng** vì lý do cảnh báo dòng — chỉ flag để xử lý sau. | PI 1.2 |
| **FR-06** | Trả về báo cáo lỗi cấu trúc | Khi schema sai, hệ thống trả về JSON chứa danh sách cột thiếu/sai tên; UI hiển thị popup chi tiết cho Operator. | PI 1.2 |

### 2.2. FR2 — Bóc tách và tính điểm gợi ý chuẩn hoá (Smart Normalization & Scoring)

| Mã FR | Tên yêu cầu | Mô tả tóm tắt | PI |
|---|---|---|---|
| **FR-07** | Parse chuỗi tác giả | Tách chuỗi `Authors` (viết tắt không dấu) và `Author(s) ID` (chuỗi số phân cách bằng `;`) thành danh sách tác giả có cấu trúc `(lastname, given_name_initials, scopus_author_id)`. | PI 2.1 |
| **FR-08** | Chuẩn hoá tên tiếng Việt | Bỏ dấu, lower, loại bỏ khoảng trắng thừa để tạo khóa so khớp mờ ổn định. | PI 2.1 |
| **FR-09** | Đếm số tác giả / bài | Đếm chính xác số lượng tác giả trên từng bài, lưu vào `AUTHOR_EXTRACTED` để phục vụ thống kê và phát hiện bất thường (bài ≥ 10 tác giả). | PI 2.1 |
| **FR-10** | So khớp chính xác qua Scopus Author ID | Đối chiếu `scopus_author_id` của từng tác giả với `scopus_author_id` trong Danh mục Giảng viên ICTU; đây là bước ưu tiên cao nhất. | PI 2.1 |
| **FR-11** | So khớp mờ (Fuzzy Match) | Sử dụng thuật toán tương đồng (Levenshtein / Jaro-Winkler) trên tên không dấu kết hợp Affiliation để bắt các trường hợp Author ID không có. | PI 2.1 |
| **FR-12** | Tính Match Score (%) | Tính điểm tương đồng ∈ [0, 1] kết hợp trọng số Author ID (0,6) + Tên (0,3) + Affiliation (0,1); sinh `evidence_payload` chứa căn cứ đối chiếu. | PI 2.1 |
| **FR-13** | Phân loại ngưỡng tin cậy | Phân nhóm gợi ý: **HIGH** (Score ≥ 85% + bằng chứng đơn vị), **MEDIUM** (50% ≤ Score < 85%), **PENDING** (Score < 50% hoặc tên phổ biến thiếu bằng chứng). | PI 2.1 |
| **FR-14** | Ghi hàng đợi phê duyệt | Mỗi kết quả gợi ý được lưu vào `APPROVAL_QUEUE` với status tương ứng; **tuyệt đối không tự động gộp** khi status = PENDING. | PI 2.1, PI 3.3 |
| **FR-15** | Ghi log pipeline | Mỗi bước pipeline ghi log gồm `correlation_id`, `log_level`, `raw_id`, `score` phục vụ debug và truy vết. | PI 2.1 |

### 2.3. FR3 — Hàng đợi đối soát, phê duyệt và hoàn tác (Review, Approval & Audit)

| Mã FR | Tên yêu cầu | Mô tả tóm tắt | PI |
|---|---|---|---|
| **FR-16** | Mở Approval Queue (Diff View) | Giao diện hiển thị bản ghi chờ duyệt với view so sánh Diff trước/sau chuẩn hoá (raw ↔ suggested mapping) và `evidence_payload` đính kèm. | PI 3.3 |
| **FR-17** | Phê duyệt liên kết | Admin bấm **[Phê duyệt liên kết]** → hệ thống ghi vào `STANDARDIZED_DATA` (canonical_author_id, score, evidence) và ghi `AUDIT_LOG` (action=APPROVE). | PI 3.3 |
| **FR-18** | Gán tay giảng viên | Admin chọn **[Gán thủ công]** → chọn giảng viên ICTU khác từ danh mục; hệ thống ghi `STANDARDIZED_DATA` + `AUDIT_LOG` (action=MANUAL_OVERRIDE, lưu GV cũ → GV mới). | PI 3.3 |
| **FR-19** | Chuyển chờ xác minh | Admin chọn **[Giữ chờ xác minh]** → cập nhật `APPROVAL_QUEUE.status = PENDING_VERIFY`, ghi `AUDIT_LOG` (action=DEFER). | PI 3.3 |
| **FR-20** | Hoàn tác (Rollback) | Admin bấm **[Rollback]** cho bản ghi vừa duyệt (trong vòng 72 giờ) → khôi phục `APPROVAL_QUEUE.status = PENDING`, ghi `AUDIT_LOG` (action=ROLLBACK). | PI 3.3 |
| **FR-21** | Ghi nhật ký kiểm toán | Mọi thao tác duyệt/gán tay/defer/rollback đều ghi vào `AUDIT_LOG`: `actor_id`, `action`, `before`, `after`, `timestamp`, `correlation_id`. **Không cho phép UPDATE/DELETE** bản ghi audit sau khi ghi. | PI 3.3, PI 3.4 |
| **FR-22** | Quản lý Danh mục Giảng viên ICTU | Admin CRUD Danh mục Giảng viên: thêm mới, sửa thông tin, gộp bản ghi trùng, tách bản ghi nếu cần. Mỗi thao tác đều có audit riêng. | PI 3.3 |

### 2.4. FR4 — Thống kê, tìm kiếm và kết xuất báo cáo (Statistics, Search & Export)

| Mã FR | Tên yêu cầu | Mô tả tóm tắt | PI |
|---|---|---|---|
| **FR-23** | Lọc dữ liệu đa tiêu chí | Operator/Admin lọc `STANDARDIZED_DATA` theo: Năm (đơn/trong khoảng), Khoa-Bộ môn (mapping từ Affiliation), Publisher Family (Springer/IEEE/Elsevier/Khác), Trạng thái duyệt. | PI 3.4 |
| **FR-24** | Tìm kiếm toàn văn | Tìm kiếm trên `Title`, `Author Keywords`, `Abstract` với full-text search, hỗ trợ tiếng Việt không dấu. | PI 3.4 |
| **FR-25** | Thống kê tổng quan | Dashboard hiển thị: tổng số bài theo năm, top tác giả, top publisher, tỉ lệ Match HIGH/MEDIUM/PENDING, tổng lượt trích dẫn. | PI 3.4 |
| **FR-26** | Xuất báo cáo Excel | Kết xuất file `.xlsx` theo định dạng thống nhất với GVHD, nhiều sheet (Summary, Raw, Standardized, Audit Trail). | PI 3.4 |
| **FR-27** | Xuất báo cáo CSV | Kết xuất file `.csv` với encoding UTF-8 (BOM) để tương thích Excel và các công cụ BI. | PI 3.4 |
| **FR-28** | Xuất báo cáo JSON | Kết xuất file `.json` chuẩn JSON Schema đã công bố, phục vụ liên thông hệ thống chung. | PI 3.4 |
| **FR-29** | Ký số và log export | Mỗi file xuất được ký số (SHA-256 + chữ ký số hệ thống); ghi vào `EXPORT_AUDIT_LOG`: `actor_id`, `timestamp`, `row_count`, `file_hash`. | PI 3.4 |

---

## 3. Yêu cầu phi chức năng — Non-Functional Requirements (NFR)

Yêu cầu phi chức năng mô tả **ràng buộc chất lượng** áp dụng xuyên suốt hệ thống. Mỗi NFR có **tiêu chí đo lường** (metric) để có thể kiểm thử/khảo sát khi nghiệm thu.

### 3.1. Bảng tổng hợp NFR

| Mã NFR | Tên | Mô tả | Tiêu chí đo lường |
|---|---|---|---|
| **NFR-01** | Tính toàn vẹn & Idempotency | Không tạo bản ghi trùng lặp khi import lại cùng một lô EID | Re-import 660 dòng 3 lần liên tiếp → số bản ghi mới sinh = 0; số bản ghi duplicate flag = 660 |
| **NFR-02** | Khả năng kiểm toán (Auditability) | Mọi thao tác duyệt/hoàn tác phải lưu vết timestamp, user_id, dữ liệu trước/sau | 100% thao tác duyệt có `AUDIT_LOG` row; `AUDIT_LOG` không UPDATE/DELETE sau khi ghi |
| **NFR-03** | Hiệu năng (Performance) | Parse và tính Match Score cho lô 660 bản ghi trong thời gian cho phép | Tổng thời gian `ST-05 → ST-12` ≤ **10 giây** trên máy chủ 4 vCPU/8 GB RAM |
| **NFR-04** | Bảo mật & Phân quyền (Security & RBAC) | Phân định chặt chẽ quyền hạn giữa Operator và Admin | Operator gọi API có `admin.*` → **403 Forbidden**; Audit trail ghi nhận attempt |
| **NFR-05** | Tương thích trao đổi dữ liệu (Interoperability) | Kết xuất định dạng chuẩn dùng chung | File `.xlsx` mở được bằng MS Excel 2019+; `.json` validate theo JSON Schema công bố; `.csv` mở được bằng Excel + Google Sheets |

### 3.2. Chi tiết từng NFR

#### NFR-01 — Tính toàn vẹn & Idempotency

- **Mô tả đầy đủ:** Khi Operator upload lại một tệp CSV có chứa toàn bộ EID đã có trong hệ thống (toàn bộ hoặc một phần), hệ thống phải **không sinh thêm bản ghi mới** trong `RAW_DATA`. Các bản ghi trùng được đánh dấu `duplicate = true` để truy vết.
- **Căn cứ khảo sát:** 660/660 bản ghi có `EID` đầy đủ → đảm bảo khoá định danh ổn định để thực hiện Idempotency.
- **Áp dụng:** Bước 1.7 trong `scopus-ictu-process-spec.md`; ST-02 trong ACT-03.
- **Phương pháp kiểm thử:** Re-import cùng một file CSV 3 lần liên tiếp; đếm số bản ghi `RAW_DATA` mới sinh (kỳ vọng = 0) và số bản ghi `duplicate = true` (kỳ vọng = 660).

#### NFR-02 — Khả năng kiểm toán (Auditability)

- **Mô tả đầy đủ:** Mọi hành động có ảnh hưởng đến `STANDARDIZED_DATA` (phê duyệt, gán tay, hoàn tác, defer) phải sinh ít nhất một bản ghi `AUDIT_LOG` chứa: `id`, `actor_id`, `action` (APPROVE / MANUAL_OVERRIDE / DEFER / ROLLBACK), `before_json`, `after_json`, `timestamp`, `correlation_id`.
- **Bất biến audit:** Sau khi ghi, bản ghi `AUDIT_LOG` **không được UPDATE/DELETE**; chỉ cho phép INSERT. Thực thi ở cả tầng ứng dụng và database (revoke quyền UPDATE/DELETE trên role service account của backend).
- **Áp dụng:** FR-17 → FR-21; FR-29.
- **Phương pháp kiểm thử:** Thực hiện 1 phiên duyệt + 1 rollback; truy vấn `AUDIT_LOG` đảm bảo có đủ 2 row với `before`/`after` khớp trạng thái.

#### NFR-03 — Hiệu năng (Performance)

- **Mô tả đầy đủ:** Toàn bộ pipeline xử lý từ khi Operator upload thành công đến khi `APPROVAL_QUEUE` được ghi xong (gồm 13 tác vụ ngầm ST-05 → ST-12 trong ACT-03) phải hoàn tất trong **≤ 10 giây** với lô 660 bản ghi.
- **Môi trường đo lường:** Máy chủ 4 vCPU, 8 GB RAM, SSD NVMe, Python 3.11+, Pandas + RapidFuzz, PostgreSQL 15.
- **Áp dụng:** ST-05 → ST-12 trong ACT-03.
- **Phương pháp kiểm thử:** Đo thời gian pipeline bằng `time.perf_counter()` ở đầu và cuối hàm `process_batch()`; log kết quả vào metric system. Chạy 5 lần liên tiếp, lấy median.
- **Kết quả khảo sát Tuần 1:** Trung bình 3-5 tác giả / bài → số lượng phép so khớp ~ 2 640-3 300 → khả thi với RapidFuzz trong ngưỡng cho phép.

#### NFR-04 — Bảo mật & Phân quyền (Security & RBAC)

- **Mô tả đầy đủ:** Hệ thống phải phân định rõ 2 vai trò (Operator, Admin) và từng quyền được liệt kê trong ACT-01 và ACT-02. Mọi API phải kiểm tra quyền ở middleware trước khi vào controller.
- **Xác thực:** Sử dụng session-based với CSRF token cho giao diện web, JWT cho API (nếu có).
- **Ghi nhận truy cập trái phép:** Mọi lần gọi API không đủ quyền phải ghi vào `SECURITY_AUDIT_LOG` kèm IP, user_agent, payload để phát hiện tấn công.
- **Áp dụng:** FR-17 → FR-22; FR-29.
- **Phương pháp kiểm thử:** Dùng tài khoản Operator gọi các endpoint `admin.approve`, `admin.rollback`, `admin.lecturer_crud` → kỳ vọng 403 + log trong `SECURITY_AUDIT_LOG`.

#### NFR-05 — Tương thích trao đổi dữ liệu (Interoperability)

- **Mô tả đầy đủ:** Các file xuất phải tuân thủ định dạng thống nhất với GVHD (Bộ môn/Khoa) và các hệ thống liên thông:
  - **Excel (.xlsx):** Cấu trúc nhiều sheet, header cố định, có Data Validation, không chứa macro.
  - **CSV (.csv):** Encoding UTF-8 with BOM, delimiter `,`, quote `"`, line ending CRLF.
  - **JSON (.json):** Tuân thủ JSON Schema `scopus-ictu-export.schema.json` (đặc tả kèm theo hệ thống).
- **Định danh phiên bản:** Mỗi file xuất có trường `schema_version` ở metadata để hỗ trợ tương thích ngược khi JSON Schema thay đổi.
- **Áp dụng:** FR-26 → FR-29.
- **Phương pháp kiểm thử:** Xuất file mẫu, mở bằng MS Excel 2019, Google Sheets, `pandas.read_json` để xác nhận không lỗi encoding/parse.

---

## 4. Ma trận truy vết yêu cầu → PI

Bảng dưới đây tổng hợp toàn bộ FR ánh xạ về các chỉ tiêu PI trong Mục 7b của đề cương, giúp hội đồng đối chiếu nhanh khi nghiệm thu.

### 4.1. Ma trận FR ↔ PI

| PI | Mô tả PI | FR đáp ứng | Actor thực hiện |
|---|---|---|---|
| **PI 1.1** | Import CSV Scopus & lưu RAW_DATA giữ nguyên gốc | FR-01, FR-02, FR-03, FR-04 | ACT-01 → ACT-03 |
| **PI 1.2** | Validation cấu trúc & cấp dòng (Idempotency, warning không dừng luồng) | FR-02, FR-05, FR-06, NFR-01 | ACT-01, ACT-03 |
| **PI 2.1** | Parse, chuẩn hoá tiếng Việt, đếm tác giả, đối soát GV ICTU, Match Score, phân nhóm HIGH/MEDIUM/PENDING | FR-07, FR-08, FR-09, FR-10, FR-11, FR-12, FR-13, FR-14, FR-15, NFR-03 | ACT-03 |
| **PI 3.3** | Approval Queue + Diff, 3 lựa chọn duyệt, Rollback, ghi STANDARDIZED_DATA + AUDIT_LOG | FR-16, FR-17, FR-18, FR-19, FR-20, FR-21, FR-22, NFR-02, NFR-04 | ACT-02, ACT-03 |
| **PI 3.4** | Lọc, tìm kiếm, thống kê, xuất Excel/CSV/JSON, ký số, log export | FR-23, FR-24, FR-25, FR-26, FR-27, FR-28, FR-29, NFR-05 | ACT-01, ACT-02, ACT-03 |

### 4.2. Ma trận NFR ↔ FR

| NFR | FR phụ thuộc trực tiếp | Ghi chú |
|---|---|---|
| NFR-01 | FR-02, FR-03, FR-04 | Liên quan bước Idempotency ở Stage 1 |
| NFR-02 | FR-17, FR-18, FR-19, FR-20, FR-21, FR-29 | Tất cả thao tác duyệt/rollback/export đều phải audit |
| NFR-03 | FR-07 → FR-15 | Toàn bộ pipeline Stage 2 |
| NFR-04 | FR-16 → FR-22, FR-29 | Mọi API admin đều kiểm tra quyền |
| NFR-05 | FR-23, FR-25, FR-26, FR-27, FR-28 | Toàn bộ stage thống kê & xuất báo cáo |

### 4.3. Coverage checklist (tự đánh giá)

| Khía cạnh | Tổng số | Đã mô tả | Tỷ lệ |
|---|---:|---:|---:|
| Actor | 3 | 3 (ACT-01, ACT-02, ACT-03) | 100% |
| System Task | 17 | 17 (ST-01 → ST-17) | 100% |
| Functional Requirement | 29 | 29 (FR-01 → FR-29) | 100% |
| Non-Functional Requirement | 5 | 5 (NFR-01 → NFR-05) | 100% |
| PI Mục 7b | 5 | 5 (PI 1.1, 1.2, 2.1, 3.3, 3.4) | 100% |

---

## Phụ lục — Tham chiếu chéo

| Tài liệu | Đường dẫn | Vai trò |
|---|---|---|
| Kết quả khảo sát Tuần 1 | `docs/scopus-ictu/srs/scopus-ictu-data-survey.md` | Căn cứ thực tế cho FR/NFR |
| Đặc tả quy trình nghiệp vụ | `docs/scopus-ictu/srs/scopus-ictu-process-spec.md` | Bước nghiệp vụ tương ứng với FR |
| Sơ đồ BPMN 2.0 IR | `docs/scopus-ictu/bpmn/scopus-ictu-approval.ir.json` | Luồng nghiệp vụ tổng thể |
| PlantUML swimlane (chi tiết) | `docs/scopus-ictu/srs/scopus-ictu-swimlane.puml` | Sơ đồ Activity 4 giai đoạn |
| Sơ đồ tổng quan (6 bước) | `docs/scopus-ictu/srs/scopus-ictu-overview-flow.md` | Sơ đồ trình bày hội đồng |
| Dữ liệu gốc | `Truong_Quang_Huy_Scopus_Sep2026.csv` | 660 bản ghi × 22 trường |
