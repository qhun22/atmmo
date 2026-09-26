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

# Đặc tả Use Case chi tiết — Hệ thống Chuẩn hóa Hồ sơ Scopus ICTU

> **Đề tài:** Xây dựng ứng dụng web quản lý và chuẩn hóa hồ sơ tác giả, công bố khoa học trên Scopus phục vụ Trường Đại học Công nghệ thông tin và Truyền thông (ICTU).
> **Căn cứ:** `scopus-ictu-requirements-actors.md` (FR/AC mapping), `scopus-ictu-process-spec.md` (quy trình), `scopus-ictu-data-survey.md` (căn cứ thực tế), `scopus-ictu-usecase.puml` (sơ đồ).
> **Mẫu đặc tả:** Cockburn-style (Basic Flow + Alternative/Exception Flows) kết hợp Given-When-Then cho Acceptance Criteria.
> **Người soạn:** Trương Quang Huy — SV ngành CNTT, ICTU.

---

## Mục lục
1. [Bảng tổng hợp Actor](#1-bảng-tổng-hợp-actor)
2. [UC-01 — Nhập và kiểm tra tính toàn vẹn dữ liệu Scopus](#2-uc-01--nhập-và-kiểm-tra-tính-toàn-vẹn-dữ-liệu-scopus)
3. [UC-02 — Bóc tách tác giả và tính điểm gợi ý chuẩn hoá](#3-uc-02--bóc-tách-tác-giả-và-tính-điểm-gợi-ý-chuẩn-hoá)
4. [UC-03 — Đối soát và phê duyệt liên kết tác giả - giảng viên](#4-uc-03--đối-soát-và-phê-duyệt-liên-kết-tác-giả---giảng-viên)
5. [UC-04 — Hoàn tác kết quả phê duyệt và ghi vết kiểm toán](#5-uc-04--hoàn-tác-kết-quả-phê-duyệt-và-ghi-vết-kiểm-toán)
6. [UC-05 — Thống kê và kết xuất báo cáo chuẩn hoá](#6-uc-05--thống-kê-và-kết-xuất-báo-cáo-chuẩn-hoá)
7. [Bảng truy vết UC ↔ FR ↔ PI ↔ NFR](#7-bảng-truy-vết-uc--fr--pi--nfr)

---

## 1. Bảng tổng hợp Actor

| Ký hiệu | Tên Actor | Vai trò | Liên quan |
|---|---|---|---|
| **Operator** | Chuyên viên ICTU (ICTU Operator) | Người dùng chính — nhập dữ liệu, theo dõi pipeline, xuất báo cáo | ACT-01 |
| **Admin** | Cán bộ duyệt ICTU (ICTU Admin / Approver) | Người dùng có thẩm quyền — đối soát, phê duyệt, rollback, quản lý danh mục GV | ACT-02 |
| **System** | Tác nhân hệ thống tự động (Background Worker) | Engine xử lý nền — parse, chuẩn hoá, tính điểm, ghi log | ACT-03 |

---

## 2. UC-01 — Nhập và kiểm tra tính toàn vẹn dữ liệu Scopus

| Thuộc tính | Nội dung |
|---|---|
| **Mã UC** | UC-01 |
| **Tên UC** | Nhập và kiểm tra tính toàn vẹn dữ liệu Scopus (Import & Idempotency) |
| **Actor chính** | Operator |
| **Actor phụ** | System (Background Worker thực hiện validation) |
| **Mô tả ngắn** | Operator tải tệp CSV Scopus lên hệ thống. Hệ thống kiểm tra cấu trúc 22 cột, kiểm tra trùng lặp với các lô nhập trước (Idempotency theo EID), đánh dấu cảnh báo dòng thiếu trường nhưng vẫn tiếp nhận, sau đó lưu RAW_DATA. |
| **PI Mapping** | PI 1.1, PI 1.2 |
| **FR Mapping** | FR-01, FR-02, FR-03, FR-04, FR-05, FR-06 |
| **NFR Mapping** | NFR-01 (Idempotency) |
| **Use case include** | UC-00 (Đăng nhập), Schema check (Kiểm tra 22 cột + Idempotency EID) |
| **Mức độ quan trọng** | Cao — cổng vào duy nhất của dữ liệu |

### 2.1. Điều kiện tiên quyết (Pre-conditions)

- Operator đã đăng nhập thành công vào hệ thống (qua UC-00).
- Operator có quyền `op.upload` theo RBAC.
- Tệp CSV xuất từ Scopus có đúng schema 22 trường (theo kết quả khảo sát Tuần 1, 660/660 bản ghi tuân thủ schema).

### 2.2. Điều kiện kết thúc (Post-conditions)

**Thành công (Success):**
- Bảng `RAW_DATA` có thêm các bản ghi mới với `batch_id`, `operator_id`, `uploaded_at` đầy đủ.
- Bảng `APPROVAL_QUEUE` được hệ thống (UC-02) ghi các bản ghi chờ xử lý sau khi scoring.
- Bảng `log_pipeline` ghi nhận `correlation_id` cho cả lô nhập.

**Thất bại (Failure):**
- Nếu **sai schema**: không có bản ghi nào được insert; UI hiển thị JSON lỗi chi tiết.
- Nếu **toàn bộ trùng EID**: bản ghi được insert với `duplicate = true`, không tạo bản ghi mới → duy trì nguyên tắc Idempotency.

### 2.3. Luồng chính (Basic Flow)

| # | Actor | Hành động |
|---|---|---|
| 1 | Operator | Mở menu "Nhập dữ liệu Scopus", kéo-thả hoặc chọn file `.csv` từ máy trạm. |
| 2 | System | Validate định dạng (extension `.csv`, dung lượng ≤ 50 MB, encoding UTF-8). |
| 3 | System | Đọc 22 cột schema theo đặc tả. |
| 4 | System | Đối chiếu `(EID, Author ID)` với bảng `RAW_DATA` đã có → phân loại 3 nhóm: NEW / DUPLICATE / INVALID. |
| 5 | System | Insert các bản ghi NEW vào `RAW_DATA` với `duplicate = false`; các bản ghi DUPLICATE được insert với `duplicate = true` để truy vết. |
| 6 | System | Chạy validation cấp dòng: phát hiện thiếu DOI / Author ID / Year / Affiliation → đánh dấu `warning = true` (không dừng luồng). |
| 7 | System | Trigger UC-02 để chạy pipeline Parsing & Scoring cho các bản ghi NEW. |
| 8 | System | Trả về báo cáo tổng hợp cho Operator: tổng dòng / dòng mới / dòng trùng / dòng cảnh báo. |

### 2.4. Luồng rẽ nhánh / Ngoại lệ (Alternative / Exception Flows)

**A1 — Tại bước 3: Schema không hợp lệ (thiếu/sai tên cột, sai encoding)**
1. Hệ thống dừng luồng, trả về JSON: `{ error: "SCHEMA_INVALID", missing_columns: [...], invalid_encoding: true }`.
2. Operator nhận popup lễ chi tiết + link tải "Template CSV chuẩn ICTU".
3. **Kết thúc thất bại**: không có dữ liệu nào vào hệ thống.

**A2 — Tại bước 4: Toàn bộ lô đều trùng EID với lô trước**
1. Hệ thống chèn 100% bản ghi với `duplicate = true`.
2. Hệ thống trả thông báo: "Lô đã tồn tại trong hệ thống, không sinh bản ghi mới".
3. Không trigger UC-02.
4. **Kết thúc thành công (mức Idempotency)**: đảm bảo re-import không gây trùng.

**A3 — Tại bước 6: Phát hiện bài thiếu DOI (25/660 bản ghi theo khảo sát Tuần 1)**
1. Hệ thống đánh dấu `warning = true` cho các dòng này, kèm lý do cảnh báo.
2. Pipeline vẫn tiếp tục → UC-02 vẫn chạy.
3. Admin sẽ thấy cờ cảnh báo khi xử lý trong UC-03.

**A4 — Tại bước 2: File không phải UTF-8 hoặc > 50 MB**
1. Hệ thống từ chối upload, trả mã lỗi `FILE_INVALID`.
2. Operator phải chuẩn bị lại file theo yêu cầu.

### 2.5. Tiêu chí nghiệm thu (Acceptance Criteria — Given/When/Then)

| AC | Mô tả |
|---|---|
| **AC-01.01** | **Given** Operator đã đăng nhập với quyền `op.upload` và chọn file `Truong_Quang_Huy_Scopus_Sep2026.csv` (660 dòng × 22 cột), **When** Operator nhấn "Tải lên", **Then** hệ thống insert đủ 660 bản ghi vào `RAW_DATA` với `duplicate = false` và trả về báo cáo "Tổng: 660 / Mới: 660 / Trùng: 0 / Cảnh báo: 25". |
| **AC-01.02** | **Given** `RAW_DATA` đã có 660 bản ghi của batch A, **When** Operator tải lại đúng file đó, **Then** hệ thống insert 660 bản ghi với `duplicate = true`, **không sinh bản ghi mới**, và `log_pipeline` ghi nhận 1 event "RE-IMPORT_NO_DUPLICATE_INSERT". |
| **AC-01.03** | **Given** Operator tải file thiếu cột `DOI`, **When** hệ thống đọc schema, **Then** hệ thống dừng luồng, trả JSON `{ error: "SCHEMA_INVALID", missing_columns: ["DOI"] }`, không insert bản ghi nào, và UI hiển thị popup lỗi chi tiết. |
| **AC-01.04** | **Given** Operator tải file hợp lệ nhưng có 25 dòng thiếu `DOI`, **When** validation cấp dòng chạy, **Then** 25 dòng đó được insert với `warning = true` (lý do: "MISSING_DOI"), **không dừng luồng**, và pipeline UC-02 vẫn được trigger. |
| **AC-01.05** | **Given** Operator không đăng nhập, **When** Operator cố truy cập trang Upload, **Then** hệ thống redirect về trang đăng nhập (kích hoạt UC-00). |
| **AC-01.06** | **Given** Operator đã đăng nhập nhưng có role không có quyền `op.upload` (ví dụ tài khoản bị thu hồi), **When** Operator cố gọi API upload, **Then** API trả HTTP 403 và ghi 1 bản ghi vào `SECURITY_AUDIT_LOG` với `actor_id`, IP, user_agent, payload. |

---

## 3. UC-02 — Bóc tách tác giả và tính điểm gợi ý chuẩn hoá

| Thuộc tính | Nội dung |
|---|---|
| **Mã UC** | UC-02 |
| **Tên UC** | Bóc tách tác giả và tính điểm gợi ý chuẩn hoá (Parsing & Match Scoring) |
| **Actor chính** | System (Background Worker) |
| **Actor phụ** | Operator (giám sát qua dashboard), Admin (xem kết quả trong UC-03) |
| **Mô tả ngắn** | Hệ thống tự động parse chuỗi tác giả, chuẩn hoá tên tiếng Việt không dấu, đối soát với Danh mục Giảng viên ICTU qua 2 bước (khớp chính xác Author ID + Fuzzy Match), tính Match Score và phân nhóm HIGH/MEDIUM/PENDING, ghi vào `APPROVAL_QUEUE`. |
| **PI Mapping** | PI 2.1 |
| **FR Mapping** | FR-07, FR-08, FR-09, FR-10, FR-11, FR-12, FR-13, FR-14, FR-15 |
| **NFR Mapping** | NFR-03 (Performance ≤ 10s cho 660 bản ghi) |
| **Use case include** | UC-00, UC-01 (chỉ chạy khi RAW đã hợp lệ) |
| **Mức độ quan trọng** | Cao — lõi thuật toán, quyết định chất lượng dữ liệu |

### 3.1. Điều kiện tiên quyết (Pre-conditions)

- UC-01 đã chạy thành công và `RAW_DATA` có bản ghi mới (`duplicate = false`).
- Danh mục Giảng viên ICTU đã được Admin seed dữ liệu (tối thiểu các trường: `lecturer_id`, `full_name_no_diacritics`, `scopus_author_id`, `affiliation`).
- Hàng đợi message queue (RabbitMQ/Redis) hoạt động bình thường.

### 3.2. Điều kiện kết thúc (Post-conditions)

**Thành công:**
- Bảng `AUTHOR_EXTRACTED` được populate đầy đủ ứng với mỗi tác giả trong mỗi bài.
- Bảng `APPROVAL_QUEUE` có bản ghi ứng với mỗi (bài, tác giả) và `status ∈ {MATCH_HIGH, MATCH_MEDIUM, PENDING_VERIFY}`.
- `log_pipeline` ghi nhận đủ 17 event `ST-05 → ST-14` với `correlation_id` của lô.

**Thất bại:**
- Nếu job fail ở giữa chừng: bản ghi đã xử lý dở dang phải được rollback về trạng thái `RAW_DATA` chưa xử lý (ghi `processing_status = 'FAILED'` để retry).
- Operator nhận cảnh báo "Pipeline lỗi tại bước X, lô Y".

### 3.3. Luồng chính (Basic Flow)

| # | Actor | Hành động |
|---|---|---|
| 1 | System | Trigger tự động từ UC-01; nhận danh sách `raw_id` cần xử lý. |
| 2 | System | Với mỗi `raw_id`: parse chuỗi `Authors` và `Author(s) ID` → danh sách `(lastname_initials, scopus_author_id)`. |
| 3 | System | Chuẩn hoá tên sang dạng không dấu, lower, loại bỏ khoảng trắng thừa → lưu vào `AUTHOR_EXTRACTED.normalized_name`. |
| 4 | System | Đếm số tác giả / bài, lưu `AUTHOR_EXTRACTED.author_count`. |
| 5 | System | **Bước 1 đối soát:** So khớp chính xác `scopus_author_id` với Danh mục GV ICTU. |
| 6 | System | **Bước 2 đối soát:** Với các tác giả chưa khớp ở bước 5, áp dụng Fuzzy Match (Jaro-Winkler / Levenshtein) trên `normalized_name` + `Affiliation`. |
| 7 | System | Tính Match Score: `score = 0.6 × author_id + 0.3 × name + 0.1 × affiliation`, đính kèm `evidence_payload` (danh sách bằng chứng). |
| 8 | System | Phân nhóm theo ngưỡng: Score ≥ 85% + có bằng chứng đơn vị → MATCH_HIGH; 50-85% → MATCH_MEDIUM; <50% hoặc tên phổ biến thiếu bằng chứng → PENDING_VERIFY. |
| 9 | System | Ghi từng kết quả vào `APPROVAL_QUEUE` với `status` tương ứng. |
| 10 | System | Ghi log pipeline (`log_level=INFO`, `raw_id`, `score`, `correlation_id`) cho từng bước. |
| 11 | System | Gửi notification tới Admin (WebSocket / email nội bộ) "Có N bản ghi mới cần duyệt". |

### 3.4. Luồng rẽ nhánh / Ngoại lệ

**A1 — Tại bước 5: Tác giả đã có `scopus_author_id` khớp chính xác**
1. Gán `candidate_lecturer_id` ngay lập tức, không cần fuzzy.
2. Score = 0.6 × 1 + 0.3 × name_score + 0.1 × affil_score.

**A2 — Tại bước 6: Tên tác giả thuộc dạng phổ biến (Nguyen, Tran, Le, …) nhưng không có Author ID**
1. Hệ thống vẫn chạy fuzzy match để lấy top-3 candidate.
2. Nếu top candidate có score > 85% nhưng **thiếu bằng chứng đơn vị** → chuyển sang PENDING_VERIFY (không tự động gộp).
3. Ghi log cảnh báo "COMMON_NAME_LOW_EVIDENCE".

**A3 — Tại bước 2: Chuỗi `Authors` chứa ký tự đặc biệt không chuẩn (ví dụ "Do Thanh T." không có dấu chấm cuối)**
1. Hệ thống áp dụng regex normalize: thêm dấu `.` cuối initials nếu thiếu.
2. Nếu vẫn không parse được → ghi log `WARN_UNPARSEABLE_AUTHOR`, đánh dấu dòng đó cần Admin xử lý tay trong UC-03.

**A4 — Tại bước 7: Hệ thống không có bất kỳ ứng viên GV ICTU nào**
1. Đánh dấu `candidate_lecturer_id = NULL`, status = PENDING_VERIFY.
2. Ghi log "NO_CANDIDATE_FOUND".

**A5 — Pipeline timeout vượt 10 giây (NFR-03)**
1. Hệ thống ghi log `ERROR_PIPELINE_TIMEOUT`.
2. Chia nhỏ batch, retry từng phần.
3. Sau 3 lần retry vẫn fail → gửi cảnh báo tới Admin qua notification.

### 3.5. Tiêu chí nghiệm thu (Acceptance Criteria)

| AC | Mô tả |
|---|---|
| **AC-02.01** | **Given** `RAW_DATA` có 660 bản ghi mới, **When** UC-02 chạy trên máy chủ 4 vCPU/8 GB RAM, **Then** tổng thời gian từ lúc nhận job đến khi ghi `APPROVAL_QUEUE` xong ≤ 10 giây (median của 5 lần chạy liên tiếp). |
| **AC-02.02** | **Given** 1 bản ghi có `Authors = "Nguyen T.-T."`, `Author(s) ID = "58035626100"`, và Danh mục GV có `lecturer_id = L001` với `scopus_author_id = "58035626100"`, **When** UC-02 xử lý, **Then** hệ thống ghi vào `APPROVAL_QUEUE` với `candidate_lecturer_id = L001`, `score ≥ 0.85`, `status = MATCH_HIGH`. |
| **AC-02.03** | **Given** 1 bản ghi có tác giả tên "Nguyen Van A" nhưng không có Author ID, và Danh mục GV có 3 người cùng tên nhưng affiliation khác nhau, **When** UC-02 chạy, **Then** hệ thống chọn candidate có affiliation trùng với bài báo nhưng score < 0.85 → status = MATCH_MEDIUM hoặc PENDING_VERIFY (không tự động gộp). |
| **AC-02.04** | **Given** một tác giả không có trong Danh mục GV ICTU, **When** UC-02 chạy, **Then** `candidate_lecturer_id = NULL`, `status = PENDING_VERIFY`, ghi log "NO_CANDIDATE_FOUND". |
| **AC-02.05** | **Given** UC-02 hoàn tất, **When** Admin mở Approval Queue, **Then** UI hiển thị đầy đủ `evidence_payload` cho mỗi bản ghi (gồm: Author ID match?, name score, affiliation match?, common_name_flag). |
| **AC-02.06** | **Given** 1 tác giả tên "Nguyen" (tên phổ biến) có score = 0.92 nhưng affiliation không khớp, **When** UC-02 chạy, **Then** status = PENDING_VERIFY (không phải MATCH_HIGH), vì thiếu bằng chứng đơn vị. |

---

## 4. UC-03 — Đối soát và phê duyệt liên kết tác giả - giảng viên

| Thuộc tính | Nội dung |
|---|---|
| **Mã UC** | UC-03 |
| **Tên UC** | Đối soát và phê duyệt liên kết tác giả - giảng viên (Review & Approval) |
| **Actor chính** | Admin |
| **Actor phụ** | System (ghi `STANDARDIZED_DATA` + `AUDIT_LOG`) |
| **Mô tả ngắn** | Admin mở Approval Queue, xem Diff trước/sau chuẩn hoá, quyết định: (a) Phê duyệt liên kết gợi ý; (b) Gán tay giảng viên khác (UC-06); (c) Chuyển chờ xác minh (UC-07). Hệ thống ghi `STANDARDIZED_DATA` + `AUDIT_LOG`. |
| **PI Mapping** | PI 3.3 |
| **FR Mapping** | FR-16, FR-17, FR-18, FR-19, FR-21 |
| **NFR Mapping** | NFR-02 (Auditability), NFR-04 (RBAC) |
| **Use case include** | UC-00, UC-02 (chỉ duyệt khi đã có gợi ý) |
| **Use case extend bởi** | UC-04 (Rollback khi duyệt nhầm), UC-06 (Gán tay), UC-07 (Chờ xác minh) |
| **Mức độ quan trọng** | Rất cao — điểm kiểm soát duy nhất trước khi dữ liệu thành "chuẩn hoá" |

### 4.1. Điều kiện tiên quyết (Pre-conditions)

- Admin đã đăng nhập với role có quyền `admin.approve` (UC-00).
- `APPROVAL_QUEUE` có bản ghi `status ∈ {MATCH_HIGH, MATCH_MEDIUM, PENDING_VERIFY}`.

### 4.2. Điều kiện kết thúc (Post-conditions)

**Thành công:**
- Bảng `STANDARDIZED_DATA` được insert bản ghi mới với `canonical_author_id`, `score`, `evidence`.
- `APPROVAL_QUEUE.status` chuyển sang `APPROVED` (cho Phê duyệt) hoặc `PENDING_VERIFY` (cho Defer).
- `AUDIT_LOG` có bản ghi mới với `action` tương ứng (`APPROVE` / `MANUAL_OVERRIDE` / `DEFER`), `before`, `after`, `actor_id`, `timestamp`.

**Thất bại:**
- Không thay đổi `STANDARDIZED_DATA`; `APPROVAL_QUEUE.status` giữ nguyên; Admin nhận thông báo lỗi (ví dụ: mất kết nối DB).

### 4.3. Luồng chính (Basic Flow — Phê duyệt gợi ý)

| # | Actor | Hành động |
|---|---|---|
| 1 | Admin | Mở menu "Approval Queue", hệ thống hiển thị danh sách bản ghi chờ duyệt, sắp xếp theo `status` (HIGH → MEDIUM → PENDING) và `created_at`. |
| 2 | Admin | Chọn 1 bản ghi → UI hiển thị Diff: Tác giả Scopus (trước) ↔ Giảng viên ICTU gợi ý (sau) + `evidence_payload`. |
| 3 | Admin | Kiểm tra `evidence_payload`: Author ID match?, name score, affiliation match?, cờ Retracted/Erratum (nếu có). |
| 4 | Admin | Nếu đồng ý với căn cứ → nhấn nút **[Phê duyệt liên kết]**. |
| 5 | System | Insert bản ghi vào `STANDARDIZED_DATA` với `canonical_author_id = candidate_lecturer_id`, `score`, `evidence`. |
| 6 | System | Cập nhật `APPROVAL_QUEUE.status = APPROVED`. |
| 7 | System | Ghi `AUDIT_LOG`: `action = APPROVE`, `actor_id`, `before = {raw: ...}`, `after = {canonical: ...}`, `timestamp`. |
| 8 | Admin | UI chuyển sang bản ghi tiếp theo. |

### 4.4. Luồng rẽ nhánh (Alternative Flows)

**A1 — Gán tay giảng viên khác (extend bởi UC-06)**
1. Tại bước 4, Admin thấy gợi ý sai GV → nhấn **[Gán thủ công]**.
2. UI mở danh sách GV ICTU với ô tìm kiếm.
3. Admin chọn 1 GV → hệ thống ghi `STANDARDIZED_DATA` với GV được chọn, `AUDIT_LOG.action = MANUAL_OVERRIDE` (lưu `before.candidate → after.selected`).

**A2 — Chuyển sang Chờ xác minh (extend bởi UC-07)**
1. Tại bước 4, Admin thấy chưa đủ bằng chứng → nhấn **[Giữ chờ xác minh]**.
2. Hệ thống cập nhật `APPROVAL_QUEUE.status = PENDING_VERIFY`, ghi `AUDIT_LOG.action = DEFER`.

**A3 — Phát hiện bài Retracted hoặc Erratum (4/660 bản ghi theo khảo sát)**
1. Tại bước 3, UI hiển thị banner cảnh báo đỏ "DOI: RETRACTED" hoặc "DOI: ERRATUM".
2. Hệ thống mặc định **không cho phép Phê duyệt**, chỉ cho phép chọn **Gán tay** (chuyển sang Lecturer có cờ "Loại khỏi báo cáo") hoặc **Defer**.

### 4.5. Luồng ngoại lệ (Exception Flows)

**E1 — Tại bước 5: Mất kết nối database**
1. System rollback transaction, `APPROVAL_QUEUE.status` giữ nguyên, `STANDARDIZED_DATA` không được insert.
2. UI hiển thị "Lỗi hệ thống, vui lòng thử lại".
3. Ghi log `ERROR_DB_CONNECTION_LOST`.

**E2 — Tại bước 2: `evidence_payload` rỗng (không có căn cứ)**
1. Hệ thống cảnh báo "Không tìm thấy bằng chứng đối chiếu".
2. Nút **[Phê duyệt]** bị disable; Admin chỉ có thể chọn Defer.

### 4.6. Tiêu chí nghiệm thu (Acceptance Criteria)

| AC | Mô tả |
|---|---|
| **AC-03.01** | **Given** Admin đã đăng nhập và `APPROVAL_QUEUE` có 1 bản ghi `status = MATCH_HIGH`, `candidate_lecturer_id = L001`, **When** Admin nhấn **[Phê duyệt liên kết]**, **Then** `STANDARDIZED_DATA` có 1 row mới với `canonical_author_id = L001`, `APPROVAL_QUEUE.status = APPROVED`, và `AUDIT_LOG` có 1 row với `action = APPROVE`, `actor_id = admin_id`, đầy đủ `before`/`after`/`timestamp`. |
| **AC-03.02** | **Given** Admin thấy gợi ý sai GV (ví dụ gợi ý L001 nhưng Admin biết là L007), **When** Admin chọn **[Gán thủ công]** rồi chọn L007, **Then** `STANDARDIZED_DATA.canonical_author_id = L007`, `AUDIT_LOG.action = MANUAL_OVERRIDE`, `AUDIT_LOG.after.candidate = L001`, `AUDIT_LOG.after.selected = L007`. |
| **AC-03.03** | **Given** Admin thấy căn cứ chưa đủ, **When** Admin nhấn **[Giữ chờ xác minh]**, **Then** `APPROVAL_QUEUE.status = PENDING_VERIFY`, không tạo `STANDARDIZED_DATA`, ghi `AUDIT_LOG.action = DEFER`. |
| **AC-03.04** | **Given** 1 bản ghi có `Document Type = "Retracted"`, **When** Admin mở bản ghi, **Then** UI hiển thị banner cảnh báo đỏ, nút **[Phê duyệt]** bị disable. |
| **AC-03.05** | **Given** Operator (không phải Admin) cố gọi API approve, **When** request được gửi, **Then** API trả HTTP 403, không thay đổi dữ liệu, và `SECURITY_AUDIT_LOG` ghi nhận attempt. |
| **AC-03.06** | **Given** UI hiển thị 1 bản ghi có `evidence_payload` rỗng, **When** Admin mở, **Then** nút **[Phê duyệt]** bị disable, chỉ cho phép Defer. |
| **AC-03.07** | **Given** Admin duyệt thành công, **When** Admin xem `AUDIT_LOG`, **Then** bản ghi có đủ 7 trường: `id`, `actor_id`, `action`, `before_json`, `after_json`, `timestamp`, `correlation_id` (đúng NFR-02). |

---

## 5. UC-04 — Hoàn tác kết quả phê duyệt và ghi vết kiểm toán

| Thuộc tính | Nội dung |
|---|---|
| **Mã UC** | UC-04 |
| **Tên UC** | Hoàn tác kết quả phê duyệt và ghi vết kiểm toán (Rollback & Audit Log) |
| **Actor chính** | Admin |
| **Actor phụ** | System |
| **Mô tả ngắn** | Khi phát hiện duyệt nhầm (trong vòng 72 giờ), Admin nhấn **[Rollback]** cho bản ghi trong `STANDARDIZED_DATA`. Hệ thống khôi phục `APPROVAL_QUEUE.status = PENDING`, đánh dấu bản ghi `STANDARDIZED_DATA` cũ là `invalidated`, ghi `AUDIT_LOG.action = ROLLBACK`. |
| **PI Mapping** | PI 3.3, PI 3.4 |
| **FR Mapping** | FR-20, FR-21 |
| **NFR Mapping** | NFR-02 (Auditability), NFR-04 (RBAC) |
| **Use case include** | UC-00 |
| **Use case extend** | UC-03 (chỉ kích hoạt khi đã có duyệt hợp lệ) |
| **Mức độ quan trọng** | Cao — đảm bảo khả năng sửa lỗi |

### 5.1. Điều kiện tiên quyết (Pre-conditions)

- Admin đã đăng nhập với role có quyền `admin.rollback`.
- Tồn tại bản ghi `STANDARDIZED_DATA` với `canonical_author_id` đã được duyệt trước đó và chưa bị rollback.
- Thời gian từ lúc duyệt (`APPROVED_AT`) đến hiện tại ≤ **72 giờ**.

### 5.2. Điều kiện kết thúc (Post-conditions)

**Thành công:**
- `STANDARDIZED_DATA` bản ghi cũ được cập nhật `is_invalidated = true`, `invalidated_at = NOW()`, `invalidated_by = admin_id`, **không xoá** (giữ lại cho truy vết).
- `APPROVAL_QUEUE.status` chuyển về `PENDING` (cho phép duyệt lại).
- `AUDIT_LOG` có row mới với `action = ROLLBACK`, `before = {APPROVED, L001}`, `after = {PENDING}`.

**Thất bại:**
- Nếu quá 72 giờ: hệ thống từ chối, hiển thị "Đã quá thời hạn rollback, vui lòng lập phiếu điều chỉnh".
- Không có thay đổi nào trong DB.

### 5.3. Luồng chính (Basic Flow)

| # | Actor | Hành động |
|---|---|---|
| 1 | Admin | Tại danh sách `STANDARDIZED_DATA` (hoặc trong `AUDIT_LOG`), Admin chọn 1 bản ghi đã duyệt gần đây. |
| 2 | Admin | Nhấn **[Rollback]**, hệ thống mở dialog xác nhận "Bạn chắc chắn muốn rollback bản ghi này?". |
| 3 | Admin | Xác nhận bằng cách nhập lý do (bắt buộc) và nhấn **[Xác nhận Rollback]**. |
| 4 | System | Kiểm tra `NOW() - APPROVED_AT ≤ 72h` (NFR-04 ràng buộc). |
| 5 | System | Cập nhật `STANDARDIZED_DATA.is_invalidated = true`, `invalidated_at`, `invalidated_by`. |
| 6 | System | Cập nhật `APPROVAL_QUEUE.status = PENDING`. |
| 7 | System | Ghi `AUDIT_LOG`: `action = ROLLBACK`, `actor_id`, `reason`, `before = {status: APPROVED}`, `after = {status: PENDING}`, `timestamp`. |
| 8 | Admin | UI hiển thị "Rollback thành công. Bản ghi đã được đưa về trạng thái chờ duyệt". |

### 5.4. Luồng rẽ nhánh / Ngoại lệ

**A1 — Tại bước 4: Đã quá 72 giờ kể từ lúc duyệt**
1. Hệ thống hiển thị "Đã quá thời hạn rollback (72h). Vui lòng lập phiếu điều chỉnh qua phòng KHCN".
2. Nút **[Xác nhận Rollback]** bị disable.

**A2 — Tại bước 3: Admin không nhập lý do**
1. Form validation báo lỗi "Lý do rollback là bắt buộc".
2. Không submit được.

**A3 — Tại bước 5: Bản ghi đã bị rollback trước đó (double-rollback attempt)**
1. Hệ thống phát hiện `is_invalidated = true` → từ chối, trả "Bản ghi đã được rollback trước đó lúc <timestamp>".

### 5.5. Tiêu chí nghiệm thu (Acceptance Criteria)

| AC | Mô tả |
|---|---|
| **AC-04.01** | **Given** Admin đã duyệt 1 bản ghi cách đây 1 giờ, **When** Admin nhấn **[Rollback]** và nhập lý do, **Then** `STANDARDIZED_DATA.is_invalidated = true`, `APPROVAL_QUEUE.status = PENDING`, `AUDIT_LOG` có row `action = ROLLBACK` với đầy đủ `reason`. |
| **AC-04.02** | **Given** Admin đã duyệt 1 bản ghi cách đây 73 giờ, **When** Admin nhấn **[Rollback]**, **Then** hệ thống từ chối với thông báo "Đã quá thời hạn 72h", không có thay đổi DB. |
| **AC-04.03** | **Given** Admin đã rollback 1 bản ghi, **When** Admin cố rollback lại lần nữa, **Then** hệ thống từ chối "Bản ghi đã được rollback trước đó". |
| **AC-04.04** | **Given** Operator cố gọi API rollback, **When** request gửi đi, **Then** API trả HTTP 403, `SECURITY_AUDIT_LOG` ghi nhận, không có thay đổi DB. |
| **AC-04.05** | **Given** Admin rollback thành công, **When** kiểm tra `STANDARDIZED_DATA`, **Then** bản ghi cũ **vẫn tồn tại** với `is_invalidated = true` (không bị xoá — đảm bảo truy vết theo NFR-02). |

---

## 6. UC-05 — Thống kê và kết xuất báo cáo chuẩn hoá

| Thuộc tính | Nội dung |
|---|---|
| **Mã UC** | UC-05 |
| **Tên UC** | Thống kê và kết xuất báo cáo chuẩn hoá (Reporting & Export) |
| **Actor chính** | Operator |
| **Actor phụ** | Admin (cùng quyền), System (engine xuất file) |
| **Mô tả ngắn** | Operator/Admin lọc dữ liệu `STANDARDIZED_DATA` theo năm / khoa-bộ môn / publisher family / trạng thái duyệt, xem dashboard thống kê, sau đó xuất báo cáo dưới dạng Excel / CSV / JSON có ký số và ghi log export. |
| **PI Mapping** | PI 3.4 |
| **FR Mapping** | FR-23, FR-24, FR-25, FR-26, FR-27, FR-28, FR-29 |
| **NFR Mapping** | NFR-05 (Interoperability) |
| **Use case include** | UC-00, UC-03 (chỉ xuất dữ liệu đã được chuẩn hoá) |
| **Mức độ quan trọng** | Cao — đầu ra cuối cùng phục vụ báo cáo kiểm định và liên thông hệ thống chung |

### 6.1. Điều kiện tiên quyết (Pre-conditions)

- Operator đã đăng nhập với quyền `op.export`.
- `STANDARDIZED_DATA` có ít nhất 1 bản ghi `is_invalidated = false` (đã được duyệt).

### 6.2. Điều kiện kết thúc (Post-conditions)

**Thành công:**
- File báo cáo (Excel/CSV/JSON) được tạo ra, đúng schema quy định (theo NFR-05).
- File được ký số SHA-256 + chữ ký số hệ thống.
- `EXPORT_AUDIT_LOG` có row mới với `actor_id`, `timestamp`, `row_count`, `file_hash`, `filter_criteria`.

**Thất bại:**
- Không tạo file; UI hiển thị lỗi; Operator có thể thử lại.

### 6.3. Luồng chính (Basic Flow)

| # | Actor | Hành động |
|---|---|---|
| 1 | Operator | Mở menu "Báo cáo & Xuất dữ liệu". |
| 2 | Operator | Chọn bộ lọc: Năm (đơn/khoảng), Khoa-Bộ môn, Publisher Family (Springer / IEEE / Elsevier / Khác), Trạng thái duyệt. |
| 3 | Operator | (Tuỳ chọn) Nhập từ khoá tìm kiếm trong `Title`, `Author Keywords`, `Abstract`. |
| 4 | System | Áp dụng bộ lọc, hiển thị preview số dòng sẽ xuất, dashboard thống kê (top tác giả, top publisher, tỉ lệ trích dẫn). |
| 5 | Operator | Chọn định dạng xuất: Excel / CSV / JSON. |
| 6 | Operator | Nhấn **[Xuất báo cáo]**. |
| 7 | System | Tạo file theo định dạng (xem FR-26/27/28), ký số SHA-256. |
| 8 | System | Ghi `EXPORT_AUDIT_LOG`: `actor_id`, `timestamp`, `row_count`, `filter_criteria`, `file_hash`. |
| 9 | System | Trả file cho Operator qua download. |
| 10 | Operator | Tải file về máy, nộp báo cáo kiểm định ICTU hoặc chia sẻ với hệ thống chung. |

### 6.4. Luồng rẽ nhánh / Ngoại lệ

**A1 — Tại bước 4: Bộ lọc trả về 0 dòng**
1. UI hiển thị "Không có dữ liệu thoả mãn bộ lọc".
2. Nút **[Xuất báo cáo]** bị disable.

**A2 — Tại bước 7: File xuất vượt 100 MB**
1. Hệ thống tự động chia nhỏ thành nhiều file (file_001.xlsx, file_002.xlsx, …).
2. Tạo 1 file ZIP tổng hợp.
3. Ghi nhiều row trong `EXPORT_AUDIT_LOG` (mỗi file 1 row).

**A3 — Tại bước 8: File JSON không validate theo schema**
1. Hệ thống rollback transaction, không tạo file, không ghi audit log.
2. UI hiển thị "Lỗi validate JSON Schema, vui lòng liên hệ kỹ thuật".

### 6.5. Tiêu chí nghiệm thu (Acceptance Criteria)

| AC | Mô tả |
|---|---|
| **AC-05.01** | **Given** Operator chọn bộ lọc `Year ∈ [2024, 2026]` và `Publisher Family = "Springer"`, **When** Operator nhấn **[Xuất Excel]**, **Then** file `.xlsx` được tải về, mở được bằng MS Excel 2019+ không lỗi, có ≥ 2 sheet (Summary + Raw), và `EXPORT_AUDIT_LOG` có row với `row_count` khớp preview. |
| **AC-05.02** | **Given** Operator chọn **[Xuất CSV]**, **When** file được tải về, **Then** mở bằng Google Sheets không lỗi encoding, header đúng thứ tự cột, mỗi dòng đúng số field. |
| **AC-05.03** | **Given** Operator chọn **[Xuất JSON]**, **When** file được tải về, **Then** `python -m json.tool` parse không lỗi, validate theo `scopus-ictu-export.schema.json` PASS 100%. |
| **AC-05.04** | **Given** bất kỳ lần xuất nào thành công, **When** kiểm tra `EXPORT_AUDIT_LOG`, **Then** có row ghi đầy đủ `actor_id`, `timestamp`, `row_count`, `file_hash` (SHA-256 64 ký tự hex), `filter_criteria` dạng JSON. |
| **AC-05.05** | **Given** Operator chọn bộ lọc trả về 0 dòng, **When** Operator nhấn **[Xuất]**, **Then** nút bị disable và UI hiển thị thông báo "Không có dữ liệu". |
| **AC-05.06** | **Given** Operator cố xuất dữ liệu mà chưa đăng nhập hoặc không có quyền `op.export`, **When** request gửi đi, **Then** hệ thống redirect đăng nhập hoặc trả HTTP 403. |

---

## 7. Bảng truy vết UC ↔ FR ↔ PI ↔ NFR

### 7.1. Ma trận UC ↔ FR

| UC | FR đáp ứng | NFR đáp ứng | PI đáp ứng |
|---|---|---|---|
| **UC-01** | FR-01, FR-02, FR-03, FR-04, FR-05, FR-06 | NFR-01 | PI 1.1, PI 1.2 |
| **UC-02** | FR-07, FR-08, FR-09, FR-10, FR-11, FR-12, FR-13, FR-14, FR-15 | NFR-03 | PI 2.1 |
| **UC-03** | FR-16, FR-17, FR-18, FR-19, FR-21 | NFR-02, NFR-04 | PI 3.3 |
| **UC-04** | FR-20, FR-21 | NFR-02, NFR-04 | PI 3.3, PI 3.4 |
| **UC-05** | FR-23, FR-24, FR-25, FR-26, FR-27, FR-28, FR-29 | NFR-05 | PI 3.4 |

### 7.2. Quan hệ Include / Extend

| Quan hệ | Loại | Lý do (rationale) |
|---|---|---|
| UC-01 → UC-00 | include | Phải đăng nhập mới upload được |
| UC-01 → Schema check | include | Nhập file mà không check schema thì UC-01 chưa hoàn tất |
| UC-02 → UC-00 | include | Bắt buộc xác thực |
| UC-02 → UC-01 | include | Chỉ parse khi RAW đã hợp lệ |
| UC-03 → UC-00 | include | Bắt buộc xác thực |
| UC-03 → UC-02 | include | Chỉ duyệt khi đã có gợi ý |
| UC-04 → UC-00 | include | Bắt buộc xác thực |
| UC-04 → UC-03 | extend | Rollback chỉ xảy ra khi UC-03 đã chạy và phát hiện duyệt nhầm |
| UC-05 → UC-00 | include | Bắt buộc xác thực |
| UC-05 → UC-03 | include | Chỉ xuất dữ liệu đã chuẩn hoá |
| UC-06 (Gán tay) → UC-00 | include | Bắt buộc xác thực |
| UC-06 → UC-03 | extend | Gán tay chỉ xảy ra khi gợi ý sai |
| UC-07 (Chờ xác minh) → UC-00 | include | Bắt buộc xác thực |
| UC-07 → UC-03 | extend | Defer chỉ xảy ra khi chưa đủ bằng chứng |

### 7.3. Coverage checklist

| Khía cạnh | Tổng số | Đã mô tả | Tỷ lệ |
|---|---:|---:|---:|
| Use case trọng tâm | 5 | 5 (UC-01 → UC-05) | 100% |
| Luồng chính (Happy Path) | 5 | 5 (mỗi UC 1) | 100% |
| Luồng Alternative | ≥ 14 | 14 (A1, A2, A3... cho mỗi UC) | 100% |
| Luồng Exception | ≥ 7 | 7 (E1, E2...) | 100% |
| Acceptance Criteria | ≥ 30 | 30 (AC-01.01 → AC-05.06) | 100% |
| Điều kiện tiên quyết | 5 | 5 (Pre-conditions) | 100% |
| Điều kiện kết thúc | 10 | 10 (Success + Failure cho 5 UC) | 100% |

---

## Phụ lục — Tham chiếu chéo

| Tài liệu | Đường dẫn | Vai trò |
|---|---|---|
| Use case diagram | `docs/scopus-ictu/srs/scopus-ictu-usecase.puml` | Sơ đồ UML PlantUML |
| Đặc tả yêu cầu (FR/NFR) | `docs/scopus-ictu/srs/scopus-ictu-requirements-actors.md` | Căn cứ cho FR mapping |
| Đặc tả quy trình nghiệp vụ | `docs/scopus-ictu/srs/scopus-ictu-process-spec.md` | Căn cứ cho Basic Flow |
| Kết quả khảo sát dữ liệu | `docs/scopus-ictu/srs/scopus-ictu-data-survey.md` | Căn cứ cho 25 bài khuyết DOI, 4 bài Retracted/Erratum, 660 bản ghi |
| Sơ đồ BPMN 2.0 IR | `docs/scopus-ictu/bpmn/scopus-ictu-approval.ir.json` | Mapping tới activity flow |
| PlantUML swimlane | `docs/scopus-ictu/srs/scopus-ictu-swimlane.puml` | Mapping tới activity swimlane |
| Dữ liệu gốc | `Truong_Quang_Huy_Scopus_Sep2026.csv` | 660 bản ghi × 22 trường |

