# TIẾN ĐỘ TUẦN 2: PHÂN TÍCH YÊU CẦU VÀ ĐẶC TẢ CA SỬ DỤNG

> **Đề tài:** Xây dựng ứng dụng web quản lý và chuẩn hóa hồ sơ tác giả, công bố khoa học trên Scopus phục vụ Trường Đại học Công nghệ thông tin và Truyền thông (ICTU).
> **Tuần thực hiện:** Tuần 2 (26/09/2026)
> **Nhiệm vụ Tuần 2:** (1) Xác định Actor & Phân quyền (RBAC); (2) Đặc tả Yêu cầu Chức năng / Phi chức năng; (3) Vẽ Sơ đồ Ca sử dụng; (4) Đặc tả chi tiết 5 UC trọng tâm kèm Tiêu chí nghiệm thu.
> **Căn cứ:** Mục 7b — PI 1.1, PI 1.2, PI 2.1, PI 3.3, PI 3.4 trong đề cương đồ án; Kết quả khảo sát Tuần 1 từ 660 bản ghi Scopus.
> **Người soạn:** Trương Quang Huy — SV ngành CNTT, ICTU.

---

## 1. XÁC ĐỊNH VAI TRÒ NGƯỜI DÙNG VÀ PHÂN QUYỀN (ACTORS & RBAC)

Hệ thống có **ba tác nhân** (actor): hai người dùng trực tiếp (Operator, Admin) và một tác nhân kỹ thuật tự động (System Engine) chạy nền để xử lý pipeline. Phân định này khớp với ba swimlane trong sơ đồ Activity `scopus-ictu-swimlane.puml`.

### 1.1. Bảng tổng hợp Actor

| Mã | Tên Actor | Loại | Mô tả ngắn |
|---|---|---|---|
| **ACT-01** | Chuyên viên xử lý dữ liệu (ICTU Operator) | Người dùng | Đầu mối kỹ thuật Phòng Sau đại học / Phòng KHCN ICTU; chịu trách nhiệm tải file, theo dõi pipeline, xuất báo cáo |
| **ACT-02** | Quản trị viên / Cán bộ duyệt (ICTU Admin / Approver) | Người dùng | Cán bộ có thẩm quyền đối soát, phê duyệt, hoàn tác, quản lý Danh mục Giảng viên ICTU |
| **ACT-03** | Tác nhân hệ thống tự động (Background Worker / System Engine) | Hệ thống | Engine xử lý nền: parse, chuẩn hoá tên tiếng Việt, tính Match Score, ghi log, audit |

**Vai trò nghiệp vụ chi tiết:**

- **Operator** chuẩn bị và tải tệp CSV Scopus xuất từ cổng Scopus theo định kỳ (tháng/quý); theo dõi log pipeline để xác nhận lô nhập đã xử lý thành công; lọc và xuất báo cáo chuẩn hoá để nộp kiểm định; phối hợp với Admin khi phát sinh ngoại lệ.
- **Admin** đối soát, phê duyệt / gán tay / chuyển chờ xác minh cho từng bản ghi trong hàng đợi `APPROVAL_QUEUE`; hoàn tác (Rollback) khi phát hiện duyệt nhầm trong vòng 72 giờ; quản lý Danh mục Giảng viên ICTU (thêm, sửa, gộp, tách); xem toàn bộ `AUDIT_LOG` phục vụ kiểm toán nội bộ.
- **System Engine** tương tác qua message queue (RabbitMQ/Redis) và REST webhook; mỗi job phải idempotent ở cấp job (job_id + dedup_key) để tránh xử lý trùng khi retry; mọi log đều có correlation_id liên kết từ lúc upload đến lúc xuất báo cáo.

### 1.2. Ma trận phân quyền người dùng (RBAC Matrix)

| Quyền (Permission Code) | Mô tả | Operator | Admin |
|---|---|:---:|:---:|
| `op.upload` | Tải tệp Scopus CSV lên hệ thống | ✅ | ✅ |
| `op.view_raw` / `admin.view_raw` | Xem dữ liệu `RAW_DATA` và trạng thái xử lý | ✅ | ✅ |
| `op.filter` | Lọc dữ liệu theo Năm / Khoa / Publisher / Trạng thái | ✅ | ✅ |
| `op.export` / `admin.export` | Xuất báo cáo chuẩn hoá (Excel/CSV/JSON) | ✅ | ✅ |
| `op.audit_read` | Xem lịch sử xuất dữ liệu (`EXPORT_AUDIT_LOG`) của bản thân | ✅ | ✅ |
| `admin.approve` | Phê duyệt liên kết tác giả Scopus — giảng viên ICTU | ⛔ **403** | ✅ |
| `admin.manual_assign` | Gán thủ công giảng viên ICTU cho tác giả Scopus | ⛔ **403** | ✅ |
| `admin.defer` | Chuyển sang trạng thái Chờ xác minh | ⛔ **403** | ✅ |
| `admin.rollback` | Hoàn tác quyết định duyệt đã ban hành (≤ 72 giờ) | ⛔ **403** | ✅ |
| `admin.lecturer_crud` | CRUD Danh mục Giảng viên ICTU | ⛔ **403** | ✅ |
| `admin.audit_full_read` | Xem toàn bộ `AUDIT_LOG` (mọi actor) | ⛔ **403** | ✅ |
| `admin.user_mgmt` | Quản lý tài khoản Operator và Admin | ⛔ **403** | ✅ |

**Ghi chú:** Mọi lần Operator cố gọi API thuộc nhóm `admin.*` đều nhận mã **HTTP 403 Forbidden** và bị ghi vào `SECURITY_AUDIT_LOG` (IP, user_agent, payload) theo NFR-04.

---

## 2. YÊU CẦU CHỨC NĂNG (FR) VÀ YÊU CẦU PHI CHỨC NĂNG (NFR)

Yêu cầu chức năng được phân thành **bốn nhóm phân hệ** theo đúng cấu trúc bốn giai đoạn quy trình. Mỗi FR có mã định danh duy nhất, tên, mô tả tóm tắt và ánh xạ về PI trong Mục 7b.

### 2.1. Yêu cầu chức năng (Functional Requirements — FR)

#### 2.1.1. FR1 — Tiếp nhận và kiểm tra dữ liệu

| Mã FR | Tên yêu cầu | Mô tả tóm tắt | PI |
|---|---|---|---|
| FR-01 | Tải tệp Scopus CSV | Giao diện Upload cho phép Operator tải tệp `.csv` từ Scopus, dung lượng tối đa 50 MB. | PI 1.1 |
| FR-02 | Kiểm tra cấu trúc schema | Xác thực 22 trường bắt buộc theo đặc tả và encoding UTF-8. Sai cấu trúc → trả JSON lỗi chi tiết và dừng luồng. | PI 1.1, PI 1.2 |
| FR-03 | Kiểm tra Idempotency theo EID | Đối chiếu `(EID, Author ID)` với các lô nhập trước. Bản ghi đã tồn tại → đánh dấu `duplicate = true`, không insert lại. | PI 1.1 |
| FR-04 | Lưu trữ RAW_DATA | Mọi bản ghi hợp lệ được insert nguyên vẹn 22 trường vào bảng `RAW_DATA`, kèm `batch_id`, `operator_id`, `uploaded_at`. | PI 1.1 |
| FR-05 | Validation cấp dòng | Phát hiện dòng thiếu DOI / Author ID / Year / Affiliation, đánh dấu cảnh báo dòng. Hệ thống **không dừng luồng** vì lý do cảnh báo. | PI 1.2 |
| FR-06 | Trả về báo cáo lỗi cấu trúc | Khi schema sai, trả JSON chứa danh sách cột thiếu/sai tên; UI hiển thị popup chi tiết cho Operator. | PI 1.2 |

#### 2.1.2. FR2 — Bóc tách và tính điểm gợi ý chuẩn hoá

| Mã FR | Tên yêu cầu | Mô tả tóm tắt | PI |
|---|---|---|---|
| FR-07 | Parse chuỗi tác giả | Tách chuỗi `Authors` và `Author(s) ID` thành danh sách tác giả có cấu trúc `(lastname, given_name_initials, scopus_author_id)`. | PI 2.1 |
| FR-08 | Chuẩn hoá tên tiếng Việt | Bỏ dấu, lower, loại bỏ khoảng trắng thừa để tạo khóa so khớp mờ ổn định. | PI 2.1 |
| FR-09 | Đếm số tác giả / bài | Đếm chính xác số tác giả trên từng bài, lưu vào `AUTHOR_EXTRACTED` để phát hiện bất thường (bài ≥ 10 tác giả). | PI 2.1 |
| FR-10 | So khớp chính xác qua Scopus Author ID | Đối chiếu `scopus_author_id` của từng tác giả với `scopus_author_id` trong Danh mục GV ICTU (bước ưu tiên cao nhất). | PI 2.1 |
| FR-11 | So khớp mờ (Fuzzy Match) | Áp dụng thuật toán tương đồng (Jaro-Winkler / Levenshtein) trên tên không dấu kết hợp Affiliation để bắt các trường hợp Author ID không có. | PI 2.1 |
| FR-12 | Tính Match Score | Tính điểm ∈ [0, 1] với trọng số Author ID (0,6) + Tên (0,3) + Affiliation (0,1); sinh `evidence_payload` chứa căn cứ đối chiếu. | PI 2.1 |
| FR-13 | Phân loại ngưỡng tin cậy | HIGH (Score ≥ 85% + bằng chứng đơn vị) / MEDIUM (50%–85%) / PENDING (Score < 50% hoặc tên phổ biến thiếu bằng chứng). | PI 2.1 |
| FR-14 | Ghi hàng đợi phê duyệt | Mỗi kết quả gợi ý lưu vào `APPROVAL_QUEUE` với status tương ứng; **tuyệt đối không tự động gộp** khi status = PENDING. | PI 2.1, PI 3.3 |
| FR-15 | Ghi log pipeline | Mỗi bước pipeline ghi log gồm `correlation_id`, `log_level`, `raw_id`, `score` phục vụ debug và truy vết. | PI 2.1 |

#### 2.1.3. FR3 — Hàng đợi đối soát, phê duyệt và hoàn tác

| Mã FR | Tên yêu cầu | Mô tả tóm tắt | PI |
|---|---|---|---|
| FR-16 | Mở Approval Queue (Diff View) | UI hiển thị bản ghi chờ duyệt với Diff trước/sau chuẩn hoá và `evidence_payload` đính kèm. | PI 3.3 |
| FR-17 | Phê duyệt liên kết | Admin nhấn **[Phê duyệt liên kết]** → ghi vào `STANDARDIZED_DATA` và `AUDIT_LOG` (action=APPROVE). | PI 3.3 |
| FR-18 | Gán tay giảng viên | Admin chọn **[Gán thủ công]** → chọn GV ICTU khác; ghi `STANDARDIZED_DATA` + `AUDIT_LOG` (action=MANUAL_OVERRIDE). | PI 3.3 |
| FR-19 | Chuyển chờ xác minh | Admin nhấn **[Giữ chờ xác minh]** → cập nhật `APPROVAL_QUEUE.status = PENDING_VERIFY`, ghi `AUDIT_LOG` (action=DEFER). | PI 3.3 |
| FR-20 | Hoàn tác (Rollback) | Admin nhấn **[Rollback]** cho bản ghi vừa duyệt (trong vòng 72 giờ) → khôi phục `APPROVAL_QUEUE.status = PENDING`, ghi `AUDIT_LOG` (action=ROLLBACK). | PI 3.3 |
| FR-21 | Ghi nhật ký kiểm toán | Mọi thao tác duyệt/gán tay/defer/rollback đều ghi vào `AUDIT_LOG` với 7 trường bắt buộc; **không cho phép UPDATE/DELETE** sau khi ghi. | PI 3.3, PI 3.4 |
| FR-22 | Quản lý Danh mục Giảng viên ICTU | Admin CRUD Danh mục Giảng viên (thêm, sửa, gộp, tách); mỗi thao tác đều có audit riêng. | PI 3.3 |

#### 2.1.4. FR4 — Thống kê, tìm kiếm và kết xuất báo cáo

| Mã FR | Tên yêu cầu | Mô tả tóm tắt | PI |
|---|---|---|---|
| FR-23 | Lọc dữ liệu đa tiêu chí | Lọc `STANDARDIZED_DATA` theo Năm (đơn/khoảng), Khoa–Bộ môn, Publisher Family (Springer / IEEE / Elsevier / Khác), Trạng thái duyệt. | PI 3.4 |
| FR-24 | Tìm kiếm toàn văn | Tìm kiếm trên `Title`, `Author Keywords`, `Abstract` với full-text search, hỗ trợ tiếng Việt không dấu. | PI 3.4 |
| FR-25 | Thống kê tổng quan | Dashboard: tổng số bài theo năm, top tác giả, top publisher, tỉ lệ Match HIGH/MEDIUM/PENDING, tổng lượt trích dẫn. | PI 3.4 |
| FR-26 | Xuất báo cáo Excel | Kết xuất file `.xlsx` nhiều sheet (Summary, Raw, Standardized, Audit Trail) theo định dạng thống nhất với GVHD. | PI 3.4 |
| FR-27 | Xuất báo cáo CSV | Kết xuất file `.csv` với encoding UTF-8 (BOM) để tương thích Excel và các công cụ BI. | PI 3.4 |
| FR-28 | Xuất báo cáo JSON | Kết xuất file `.json` chuẩn JSON Schema đã công bố, phục vụ liên thông hệ thống chung. | PI 3.4 |
| FR-29 | Ký số và log export | Mỗi file xuất được ký số (SHA-256 + chữ ký số hệ thống); ghi vào `EXPORT_AUDIT_LOG`: `actor_id`, `timestamp`, `row_count`, `file_hash`. | PI 3.4 |

### 2.2. Yêu cầu phi chức năng (Non-Functional Requirements — NFR)

Yêu cầu phi chức năng mô tả **ràng buộc chất lượng** áp dụng xuyên suốt hệ thống, có **tiêu chí đo lường định lượng** để nghiệm thu khách quan.

| Mã NFR | Tên | Mô tả | Tiêu chí đo lường (Metric) |
|---|---|---|---|
| **NFR-01** | Tính toàn vẹn & Idempotency | Không tạo bản ghi trùng lặp khi import lại cùng một lô EID. | Re-import **660 dòng 3 lần liên tiếp** → số bản ghi mới sinh = 0; số bản ghi duplicate flag = 660. |
| **NFR-02** | Khả năng kiểm toán (Auditability) | Mọi thao tác duyệt/hoàn tác phải lưu vết timestamp, user_id, dữ liệu trước/sau. | **100%** thao tác duyệt có `AUDIT_LOG` row; `AUDIT_LOG` **không UPDATE/DELETE** sau khi ghi (revoke quyền ở DB role). |
| **NFR-03** | Hiệu năng (Performance) | Parse và tính Match Score cho lô 660 bản ghi trong thời gian cho phép. | Tổng thời gian pipeline xử lý **≤ 10 giây** trên máy chủ 4 vCPU / 8 GB RAM / SSD NVMe / Python 3.11+. |
| **NFR-04** | Bảo mật & Phân quyền (Security & RBAC) | Phân định chặt chẽ quyền hạn giữa Operator và Admin; truy cập trái phép phải được ghi nhận. | Operator gọi API có `admin.*` → **HTTP 403 Forbidden**; mỗi attempt ghi 1 bản ghi vào `SECURITY_AUDIT_LOG`. |
| **NFR-05** | Tương thích trao đổi dữ liệu (Interoperability) | Kết xuất định dạng chuẩn dùng chung với GVHD và hệ thống liên thông. | File `.xlsx` mở được bằng **MS Excel 2019+**; file `.json` validate theo **JSON Schema công bố** PASS 100%; file `.csv` mở được bằng **Google Sheets + Excel**. |

**Ghi chú đo lường bổ sung:**
- **NFR-01** — 660/660 bản ghi thực tế từ `Truong_Quang_Huy_Scopus_Sep2026.csv` đều có EID đầy đủ, đảm bảo khóa định danh ổn định để thực hiện Idempotency.
- **NFR-03** — Trung bình 3–5 tác giả / bài trong lô khảo sát → số phép so khớp ước tính 2 640–3 300, khả thi với RapidFuzz trong ngưỡng cho phép.

---

## 3. SƠ ĐỒ CA SỬ DỤNG TỔNG QUAN (USE CASE DIAGRAM)

Sơ đồ ca sử dụng dưới đây mô tả toàn bộ phạm vi nghiệp vụ của hệ thống ở mức tổng quan, đặt trong phạm vi **Hệ thống Quản lý và Chuẩn hóa Hồ sơ Scopus ICTU**, với hai tác nhân chính (Operator và Admin) và năm ca sử dụng xương sống xuyên suốt bốn giai đoạn quy trình: tiếp nhận dữ liệu → bóc tách và chuẩn hoá → đối soát và phê duyệt → thống kê và xuất báo cáo. Sơ đồ được đặc tả đầy đủ trong file PlantUML `scopus-ictu-usecase.puml` và file đặc tả chi tiết `scopus-ictu-usecase-spec.md` kèm theo báo cáo này.

**[Chèn Hình 2.1: Sơ đồ ca sử dụng tổng quan — Hệ thống Quản lý và Chuẩn hóa Hồ sơ Scopus ICTU]**

### 3.1. Bảng thuyết minh các Use Case theo phân hệ

| Mã UC | Tên UC | Actor chính | Phân hệ | Mục đích nghiệp vụ |
|---|---|---|---|---|
| **UC-01** | Nhập và kiểm tra tính toàn vẹn dữ liệu Scopus (Import & Idempotency) | Operator | Tiếp nhận dữ liệu | Đưa tệp CSV Scopus thô vào hệ thống, đảm bảo cấu trúc 22 cột, không trùng EID với lô trước, lưu `RAW_DATA` nguyên vẹn. |
| **UC-02** | Bóc tách tác giả và tính điểm gợi ý chuẩn hoá (Parsing & Match Scoring) | System | Chuẩn hoá tự động | Parse tác giả, chuẩn hoá tên không dấu, đối soát với Danh mục GV ICTU, tính Match Score, phân nhóm HIGH/MEDIUM/PENDING. |
| **UC-03** | Đối soát và phê duyệt liên kết tác giả - giảng viên (Review & Approval) | Admin | Phê duyệt | Admin mở Diff View, chọn **[Phê duyệt]** / **[Gán tay]** / **[Chờ xác minh]**, ghi `STANDARDIZED_DATA` + `AUDIT_LOG`. |
| **UC-04** | Hoàn tác kết quả phê duyệt và ghi vết kiểm toán (Rollback & Audit Log) | Admin | Kiểm toán | Khi phát hiện duyệt nhầm trong vòng 72 giờ, Admin nhấn **[Rollback]**; hệ thống ghi `AUDIT_LOG.action = ROLLBACK`. |
| **UC-05** | Thống kê và kết xuất báo cáo chuẩn hoá (Reporting & Export) | Operator | Báo cáo | Lọc dữ liệu, xem dashboard, xuất Excel/CSV/JSON có ký số, ghi `EXPORT_AUDIT_LOG`. |

### 3.2. Diễn giải các quan hệ giữa các Use Case

| Quan hệ | Loại | Diễn giải ý nghĩa nghiệp vụ |
|---|---|---|
| UC-01 → UC-00 (Đăng nhập) | `<<include>>` | Phải đăng nhập mới được upload tệp. |
| UC-01 → Schema check | `<<include>>` | Nhập file mà không kiểm tra schema thì UC-01 chưa hoàn tất. |
| UC-02 → UC-00 | `<<include>>` | Bắt buộc xác thực trước khi vào pipeline. |
| UC-02 → UC-01 | `<<include>>` | Pipeline chỉ chạy khi `RAW_DATA` đã hợp lệ. |
| UC-03 → UC-00 | `<<include>>` | Bắt buộc xác thực trước khi duyệt. |
| UC-03 → UC-02 | `<<include>>` | Chỉ duyệt khi đã có gợi ý từ Match Score. |
| UC-04 → UC-00 | `<<include>>` | Bắt buộc xác thực. |
| UC-04 → UC-03 | `<<extend>>` | Rollback chỉ xảy ra khi UC-03 đã chạy và phát hiện duyệt nhầm. |
| UC-05 → UC-00 | `<<include>>` | Bắt buộc xác thực. |
| UC-05 → UC-03 | `<<include>>` | Chỉ xuất dữ liệu đã được chuẩn hoá (APPROVED). |
| UC-06 (Gán tay) → UC-03 | `<<extend>>` | Gán tay chỉ kích hoạt khi gợi ý tự động sai GV. |
| UC-07 (Chờ xác minh) → UC-03 | `<<extend>>` | Defer chỉ kích hoạt khi căn cứ chưa đủ. |

---

## 4. ĐẶC TẢ CHI TIẾT 5 CA SỬ DỤNG TRỌNG TÂM VÀ TIÊU CHÍ NGHIỆM THU

Mỗi ca sử dụng được đặc tả theo mẫu Cockburn kết hợp Given-When-Then cho tiêu chí nghiệm thu, đảm bảo hội đồng có thể đối chiếu và kiểm thử khách quan.

### 4.1. UC-01 — Nhập và kiểm tra tính toàn vẹn dữ liệu Scopus

**Ánh xạ PI:** PI 1.1, PI 1.2

#### 4.1.1. Thuộc tính tổng quan

| Thuộc tính | Nội dung |
|---|---|
| Mã UC | UC-01 |
| Tên UC | Nhập và kiểm tra tính toàn vẹn dữ liệu Scopus (Import & Idempotency) |
| Actor chính | Operator |
| Actor phụ | System (Background Worker thực hiện validation) |
| Mô tả ngắn | Operator tải tệp CSV Scopus lên hệ thống. Hệ thống kiểm tra cấu trúc 22 cột, kiểm tra trùng lặp với các lô nhập trước (Idempotency theo EID), đánh dấu cảnh báo dòng thiếu trường nhưng vẫn tiếp nhận, sau đó lưu `RAW_DATA`. |
| FR Mapping | FR-01, FR-02, FR-03, FR-04, FR-05, FR-06 |
| NFR Mapping | NFR-01 (Idempotency) |
| Mức độ quan trọng | Cao — cổng vào duy nhất của dữ liệu |

#### 4.1.2. Điều kiện tiên quyết và Điều kiện kết thúc

| Điều kiện tiên quyết (Pre-conditions) | Điều kiện kết thúc (Post-conditions) |
|---|---|
| (a) Operator đã đăng nhập thành công vào hệ thống. (b) Operator có quyền `op.upload`. (c) Tệp CSV xuất từ Scopus có đúng schema 22 trường (theo kết quả khảo sát Tuần 1, 660/660 bản ghi tuân thủ schema). | **Thành công:** `RAW_DATA` có thêm bản ghi mới với `batch_id`, `operator_id`, `uploaded_at`; `APPROVAL_QUEUE` được ghi sau khi scoring; `log_pipeline` ghi nhận `correlation_id`. **Thất bại:** Schema sai → không insert bản ghi nào, UI hiển thị JSON lỗi chi tiết. Toàn bộ trùng EID → bản ghi insert với `duplicate = true`, không sinh bản ghi mới (giữ nguyên tắc Idempotency). |

#### 4.1.3. Luồng chính (Basic Flow)

| # | Actor | Hành động |
|---|---|---|
| 1 | Operator | Mở menu "Nhập dữ liệu Scopus", kéo-thả hoặc chọn file `.csv` từ máy trạm. |
| 2 | System | Validate định dạng (extension `.csv`, dung lượng ≤ 50 MB, encoding UTF-8). |
| 3 | System | Đọc 22 cột schema theo đặc tả. |
| 4 | System | Đối chiếu `(EID, Author ID)` với bảng `RAW_DATA` đã có → phân loại 3 nhóm: NEW / DUPLICATE / INVALID. |
| 5 | System | Insert bản ghi NEW vào `RAW_DATA` với `duplicate = false`; bản ghi DUPLICATE insert với `duplicate = true` để truy vết. |
| 6 | System | Chạy validation cấp dòng: phát hiện thiếu DOI / Author ID / Year / Affiliation → đánh dấu `warning = true` (không dừng luồng). |
| 7 | System | Trigger UC-02 để chạy pipeline Parsing & Scoring cho các bản ghi NEW. |
| 8 | System | Trả về báo cáo tổng hợp cho Operator: tổng dòng / dòng mới / dòng trùng / dòng cảnh báo. |

#### 4.1.4. Luồng rẽ nhánh / Ngoại lệ chính

| Mã | Tình huống | Xử lý |
|---|---|---|
| A1 | Schema không hợp lệ (thiếu/sai tên cột, sai encoding) | Dừng luồng, trả JSON `{ error: "SCHEMA_INVALID", missing_columns: [...], invalid_encoding: true }`. UI hiển thị popup chi tiết + link tải "Template CSV chuẩn ICTU". |
| A2 | Toàn bộ lô đều trùng EID với lô trước | Chèn 100% bản ghi với `duplicate = true`; trả thông báo "Lô đã tồn tại trong hệ thống, không sinh bản ghi mới"; không trigger UC-02. |
| A3 | **Phát hiện bài thiếu DOI (25/660 bản ghi theo khảo sát Tuần 1)** | Đánh dấu `warning = true` cho 25 dòng; pipeline vẫn tiếp tục → UC-02 vẫn chạy; Admin sẽ thấy cờ cảnh báo ở UC-03. |
| A4 | File không phải UTF-8 hoặc > 50 MB | Từ chối upload, trả mã lỗi `FILE_INVALID`; Operator chuẩn bị lại file. |

#### 4.1.5. Tiêu chí nghiệm thu (Given-When-Then)

| AC | Mô tả tiêu chí |
|---|---|
| AC-01.01 | **Given** Operator đã đăng nhập với quyền `op.upload` và chọn file `Truong_Quang_Huy_Scopus_Sep2026.csv` (660 dòng × 22 cột), **When** Operator nhấn "Tải lên", **Then** hệ thống insert đủ 660 bản ghi vào `RAW_DATA` với `duplicate = false` và trả về báo cáo "Tổng: 660 / Mới: 660 / Trùng: 0 / Cảnh báo: 25". |
| AC-01.02 | **Given** `RAW_DATA` đã có 660 bản ghi của batch A, **When** Operator tải lại đúng file đó, **Then** hệ thống insert 660 bản ghi với `duplicate = true`, **không sinh bản ghi mới**, và `log_pipeline` ghi nhận 1 event "RE-IMPORT_NO_DUPLICATE_INSERT". |
| AC-01.03 | **Given** Operator tải file thiếu cột `DOI`, **When** hệ thống đọc schema, **Then** hệ thống dừng luồng, trả JSON `{ error: "SCHEMA_INVALID", missing_columns: ["DOI"] }`, không insert bản ghi nào, và UI hiển thị popup lỗi chi tiết. |
| AC-01.04 | **Given** Operator tải file hợp lệ nhưng có 25 dòng thiếu `DOI`, **When** validation cấp dòng chạy, **Then** 25 dòng đó được insert với `warning = true` (lý do: "MISSING_DOI"), **không dừng luồng**, và pipeline UC-02 vẫn được trigger. |
| AC-01.05 | **Given** Operator không đăng nhập, **When** Operator cố truy cập trang Upload, **Then** hệ thống redirect về trang đăng nhập (kích hoạt UC-00). |
| AC-01.06 | **Given** Operator có role không có quyền `op.upload`, **When** Operator cố gọi API upload, **Then** API trả HTTP 403 và ghi 1 bản ghi vào `SECURITY_AUDIT_LOG` với `actor_id`, IP, user_agent, payload. |

---

### 4.2. UC-02 — Bóc tách tác giả và tính điểm gợi ý chuẩn hoá

**Ánh xạ PI:** PI 2.1

#### 4.2.1. Thuộc tính tổng quan

| Thuộc tính | Nội dung |
|---|---|
| Mã UC | UC-02 |
| Tên UC | Bóc tách tác giả và tính điểm gợi ý chuẩn hoá (Parsing & Match Scoring) |
| Actor chính | System (Background Worker) |
| Actor phụ | Operator (giám sát qua dashboard), Admin (xem kết quả trong UC-03) |
| Mô tả ngắn | Hệ thống tự động parse chuỗi tác giả, chuẩn hoá tên tiếng Việt không dấu, đối soát với Danh mục Giảng viên ICTU qua 2 bước (khớp chính xác Author ID + Fuzzy Match), tính Match Score và phân nhóm HIGH/MEDIUM/PENDING, ghi vào `APPROVAL_QUEUE`. |
| FR Mapping | FR-07, FR-08, FR-09, FR-10, FR-11, FR-12, FR-13, FR-14, FR-15 |
| NFR Mapping | NFR-03 (Performance ≤ 10s cho 660 bản ghi) |
| Mức độ quan trọng | Cao — lõi thuật toán, quyết định chất lượng dữ liệu |

#### 4.2.2. Điều kiện tiên quyết và Điều kiện kết thúc

| Điều kiện tiên quyết (Pre-conditions) | Điều kiện kết thúc (Post-conditions) |
|---|---|
| (a) UC-01 đã chạy thành công và `RAW_DATA` có bản ghi mới. (b) Danh mục Giảng viên ICTU đã được Admin seed (tối thiểu `lecturer_id`, `full_name_no_diacritics`, `scopus_author_id`, `affiliation`). (c) Message queue (RabbitMQ/Redis) hoạt động bình thường. | **Thành công:** `AUTHOR_EXTRACTED` populate đầy đủ; `APPROVAL_QUEUE` có bản ghi ứng với mỗi (bài, tác giả) với `status ∈ {MATCH_HIGH, MATCH_MEDIUM, PENDING_VERIFY}`; `log_pipeline` ghi nhận các event `ST-05 → ST-14` với `correlation_id`. **Thất bại:** Job fail giữa chừng → bản ghi dở dang rollback về trạng thái chưa xử lý (`processing_status = 'FAILED'`); Operator nhận cảnh báo "Pipeline lỗi tại bước X, lô Y". |

#### 4.2.3. Luồng chính (Basic Flow)

| # | Actor | Hành động |
|---|---|---|
| 1 | System | Trigger tự động từ UC-01; nhận danh sách `raw_id` cần xử lý. |
| 2 | System | Với mỗi `raw_id`: parse chuỗi `Authors` và `Author(s) ID` → danh sách `(lastname_initials, scopus_author_id)`. |
| 3 | System | Chuẩn hoá tên sang dạng không dấu, lower, loại bỏ khoảng trắng thừa → lưu vào `AUTHOR_EXTRACTED.normalized_name`. |
| 4 | System | Đếm số tác giả / bài, lưu `AUTHOR_EXTRACTED.author_count`. |
| 5 | System | **Bước 1 đối soát:** So khớp chính xác `scopus_author_id` với Danh mục GV ICTU. |
| 6 | System | **Bước 2 đối soát:** Với tác giả chưa khớp ở bước 5, áp dụng Fuzzy Match (Jaro-Winkler / Levenshtein) trên `normalized_name` + `Affiliation`. |
| 7 | System | Tính Match Score: `score = 0,6 × author_id + 0,3 × name + 0,1 × affiliation`, đính kèm `evidence_payload`. |
| 8 | System | Phân nhóm: Score ≥ 85% + bằng chứng đơn vị → MATCH_HIGH; 50%–85% → MATCH_MEDIUM; <50% hoặc tên phổ biến thiếu bằng chứng → PENDING_VERIFY. |
| 9 | System | Ghi từng kết quả vào `APPROVAL_QUEUE` với `status` tương ứng. |
| 10 | System | Ghi log pipeline (`log_level=INFO`, `raw_id`, `score`, `correlation_id`) cho từng bước. |
| 11 | System | Gửi notification tới Admin (WebSocket / email nội bộ) "Có N bản ghi mới cần duyệt". |

#### 4.2.4. Luồng rẽ nhánh / Ngoại lệ chính

| Mã | Tình huống | Xử lý |
|---|---|---|
| A1 | Tác giả đã có `scopus_author_id` khớp chính xác ở bước 5 | Gán `candidate_lecturer_id` ngay lập tức; `score = 0,6 + 0,3 × name_score + 0,1 × affil_score`. |
| A2 | Tên tác giả thuộc dạng phổ biến (Nguyen, Tran, Le, …) nhưng không có Author ID | Vẫn chạy fuzzy match để lấy top-3 candidate; nếu score > 85% nhưng **thiếu bằng chứng đơn vị** → PENDING_VERIFY (không tự động gộp). Ghi log "COMMON_NAME_LOW_EVIDENCE". |
| A3 | Chuỗi `Authors` chứa ký tự đặc biệt không chuẩn (ví dụ "Do Thanh T." thiếu dấu chấm cuối) | Hệ thống regex normalize: thêm dấu `.` cuối initials nếu thiếu. Nếu vẫn không parse được → ghi log `WARN_UNPARSEABLE_AUTHOR`, đánh dấu dòng cần Admin xử lý tay trong UC-03. |
| A4 | Hệ thống không có bất kỳ ứng viên GV ICTU nào ứng với tác giả | `candidate_lecturer_id = NULL`, status = PENDING_VERIFY; ghi log "NO_CANDIDATE_FOUND". |
| A5 | **Pipeline timeout vượt 10 giây (NFR-03)** | Ghi log `ERROR_PIPELINE_TIMEOUT`; chia nhỏ batch, retry từng phần; sau 3 lần retry vẫn fail → gửi cảnh báo tới Admin. |

#### 4.2.5. Tiêu chí nghiệm thu (Given-When-Then)

| AC | Mô tả tiêu chí |
|---|---|
| AC-02.01 | **Given** `RAW_DATA` có 660 bản ghi mới, **When** UC-02 chạy trên máy chủ 4 vCPU / 8 GB RAM, **Then** tổng thời gian từ lúc nhận job đến khi ghi `APPROVAL_QUEUE` xong **≤ 10 giây** (median của 5 lần chạy liên tiếp). |
| AC-02.02 | **Given** 1 bản ghi có `Authors = "Nguyen T.-T."`, `Author(s) ID = "58035626100"`, và Danh mục GV có `lecturer_id = L001` với `scopus_author_id = "58035626100"`, **When** UC-02 xử lý, **Then** hệ thống ghi vào `APPROVAL_QUEUE` với `candidate_lecturer_id = L001`, `score ≥ 0,85`, `status = MATCH_HIGH`. |
| AC-02.03 | **Given** 1 bản ghi có tác giả "Nguyen Van A" không có Author ID, và Danh mục GV có 3 người cùng tên nhưng affiliation khác nhau, **When** UC-02 chạy, **Then** hệ thống chọn candidate có affiliation trùng với bài báo nhưng score < 0,85 → status = MATCH_MEDIUM hoặc PENDING_VERIFY (không tự động gộp). |
| AC-02.04 | **Given** một tác giả không có trong Danh mục GV ICTU, **When** UC-02 chạy, **Then** `candidate_lecturer_id = NULL`, `status = PENDING_VERIFY`, ghi log "NO_CANDIDATE_FOUND". |
| AC-02.05 | **Given** UC-02 hoàn tất, **When** Admin mở Approval Queue, **Then** UI hiển thị đầy đủ `evidence_payload` cho mỗi bản ghi (gồm: Author ID match?, name score, affiliation match?, common_name_flag). |
| AC-02.06 | **Given** 1 tác giả tên "Nguyen" (tên phổ biến) có score = 0,92 nhưng affiliation không khớp, **When** UC-02 chạy, **Then** status = PENDING_VERIFY (không phải MATCH_HIGH), vì thiếu bằng chứng đơn vị. |

---

### 4.3. UC-03 — Đối soát và phê duyệt liên kết tác giả - giảng viên

**Ánh xạ PI:** PI 3.3

#### 4.3.1. Thuộc tính tổng quan

| Thuộc tính | Nội dung |
|---|---|
| Mã UC | UC-03 |
| Tên UC | Đối soát và phê duyệt liên kết tác giả - giảng viên (Review & Approval) |
| Actor chính | Admin |
| Actor phụ | System (ghi `STANDARDIZED_DATA` + `AUDIT_LOG`) |
| Mô tả ngắn | Admin mở Approval Queue, xem Diff trước/sau chuẩn hoá, quyết định: (a) Phê duyệt liên kết gợi ý; (b) Gán tay giảng viên khác; (c) Chuyển chờ xác minh. Hệ thống ghi `STANDARDIZED_DATA` + `AUDIT_LOG`. |
| FR Mapping | FR-16, FR-17, FR-18, FR-19, FR-21 |
| NFR Mapping | NFR-02 (Auditability), NFR-04 (RBAC) |
| Mức độ quan trọng | Rất cao — điểm kiểm soát duy nhất trước khi dữ liệu thành "chuẩn hoá" |

#### 4.3.2. Điều kiện tiên quyết và Điều kiện kết thúc

| Điều kiện tiên quyết (Pre-conditions) | Điều kiện kết thúc (Post-conditions) |
|---|---|
| (a) Admin đã đăng nhập với role có quyền `admin.approve`. (b) `APPROVAL_QUEUE` có bản ghi `status ∈ {MATCH_HIGH, MATCH_MEDIUM, PENDING_VERIFY}`. | **Thành công:** `STANDARDIZED_DATA` được insert bản ghi mới với `canonical_author_id`, `score`, `evidence`; `APPROVAL_QUEUE.status` chuyển sang `APPROVED` hoặc `PENDING_VERIFY`; `AUDIT_LOG` có bản ghi mới với `action` tương ứng (`APPROVE` / `MANUAL_OVERRIDE` / `DEFER`). **Thất bại:** Không thay đổi `STANDARDIZED_DATA`; `APPROVAL_QUEUE.status` giữ nguyên; Admin nhận thông báo lỗi. |

#### 4.3.3. Luồng chính (Basic Flow — Phê duyệt gợi ý)

| # | Actor | Hành động |
|---|---|---|
| 1 | Admin | Mở menu "Approval Queue"; hệ thống hiển thị danh sách bản ghi chờ duyệt, sắp xếp theo `status` (HIGH → MEDIUM → PENDING) và `created_at`. |
| 2 | Admin | Chọn 1 bản ghi → UI hiển thị Diff: Tác giả Scopus (trước) ↔ Giảng viên ICTU gợi ý (sau) + `evidence_payload`. |
| 3 | Admin | Kiểm tra `evidence_payload`: Author ID match?, name score, affiliation match?, cờ Retracted/Erratum (nếu có). |
| 4 | Admin | Nếu đồng ý với căn cứ → nhấn nút **[Phê duyệt liên kết]**. |
| 5 | System | Insert bản ghi vào `STANDARDIZED_DATA` với `canonical_author_id = candidate_lecturer_id`, `score`, `evidence`. |
| 6 | System | Cập nhật `APPROVAL_QUEUE.status = APPROVED`. |
| 7 | System | Ghi `AUDIT_LOG`: `action = APPROVE`, `actor_id`, `before`, `after`, `timestamp`. |
| 8 | Admin | UI chuyển sang bản ghi tiếp theo. |

#### 4.3.4. Luồng rẽ nhánh / Ngoại lệ chính

| Mã | Tình huống | Xử lý |
|---|---|---|
| A1 | Admin thấy gợi ý sai GV (extend bởi UC-06 Gán tay) | Nhấn **[Gán thủ công]** → UI mở danh sách GV ICTU → chọn GV khác; ghi `STANDARDIZED_DATA` với GV được chọn, `AUDIT_LOG.action = MANUAL_OVERRIDE` (lưu `before.candidate → after.selected`). |
| A2 | Admin thấy căn cứ chưa đủ (extend bởi UC-07 Chờ xác minh) | Nhấn **[Giữ chờ xác minh]** → cập nhật `APPROVAL_QUEUE.status = PENDING_VERIFY`, ghi `AUDIT_LOG.action = DEFER`. |
| **A3** | **Phát hiện bài Retracted hoặc Erratum (4/660 bản ghi theo khảo sát)** | UI hiển thị banner cảnh báo đỏ "DOI: RETRACTED" hoặc "DOI: ERRATUM". Hệ thống mặc định **không cho phép Phê duyệt**; chỉ cho phép Gán tay sang GV có cờ "Loại khỏi báo cáo" hoặc Defer. |
| E1 | Mất kết nối database tại bước 5 | Rollback transaction; `APPROVAL_QUEUE.status` giữ nguyên; UI hiển thị "Lỗi hệ thống, vui lòng thử lại"; ghi log `ERROR_DB_CONNECTION_LOST`. |
| E2 | `evidence_payload` rỗng | Cảnh báo "Không tìm thấy bằng chứng đối chiếu"; nút **[Phê duyệt]** bị disable; Admin chỉ có thể chọn Defer. |

#### 4.3.5. Tiêu chí nghiệm thu (Given-When-Then)

| AC | Mô tả tiêu chí |
|---|---|
| AC-03.01 | **Given** Admin đã đăng nhập và `APPROVAL_QUEUE` có 1 bản ghi `status = MATCH_HIGH`, `candidate_lecturer_id = L001`, **When** Admin nhấn **[Phê duyệt liên kết]**, **Then** `STANDARDIZED_DATA` có 1 row mới với `canonical_author_id = L001`, `APPROVAL_QUEUE.status = APPROVED`, và `AUDIT_LOG` có 1 row với `action = APPROVE`, đầy đủ `before`/`after`/`timestamp`. |
| AC-03.02 | **Given** Admin thấy gợi ý sai GV (gợi ý L001 nhưng đúng là L007), **When** Admin chọn **[Gán thủ công]** rồi chọn L007, **Then** `STANDARDIZED_DATA.canonical_author_id = L007`, `AUDIT_LOG.action = MANUAL_OVERRIDE`, `AUDIT_LOG.after.candidate = L001`, `AUDIT_LOG.after.selected = L007`. |
| AC-03.03 | **Given** Admin thấy căn cứ chưa đủ, **When** Admin nhấn **[Giữ chờ xác minh]**, **Then** `APPROVAL_QUEUE.status = PENDING_VERIFY`, không tạo `STANDARDIZED_DATA`, ghi `AUDIT_LOG.action = DEFER`. |
| AC-03.04 | **Given** 1 bản ghi có `Document Type = "Retracted"`, **When** Admin mở bản ghi, **Then** UI hiển thị banner cảnh báo đỏ, nút **[Phê duyệt]** bị disable. |
| AC-03.05 | **Given** Operator (không phải Admin) cố gọi API approve, **When** request được gửi, **Then** API trả HTTP 403, không thay đổi dữ liệu, và `SECURITY_AUDIT_LOG` ghi nhận attempt. |
| AC-03.06 | **Given** UI hiển thị 1 bản ghi có `evidence_payload` rỗng, **When** Admin mở, **Then** nút **[Phê duyệt]** bị disable, chỉ cho phép Defer. |
| AC-03.07 | **Given** Admin duyệt thành công, **When** Admin xem `AUDIT_LOG`, **Then** bản ghi có đủ 7 trường: `id`, `actor_id`, `action`, `before_json`, `after_json`, `timestamp`, `correlation_id` (đúng NFR-02). |

---

### 4.4. UC-04 — Hoàn tác kết quả phê duyệt và ghi vết kiểm toán

**Ánh xạ PI:** PI 3.3, PI 3.4

#### 4.4.1. Thuộc tính tổng quan

| Thuộc tính | Nội dung |
|---|---|
| Mã UC | UC-04 |
| Tên UC | Hoàn tác kết quả phê duyệt và ghi vết kiểm toán (Rollback & Audit Log) |
| Actor chính | Admin |
| Actor phụ | System |
| Mô tả ngắn | Khi phát hiện duyệt nhầm (trong vòng 72 giờ), Admin nhấn **[Rollback]** cho bản ghi trong `STANDARDIZED_DATA`. Hệ thống khôi phục `APPROVAL_QUEUE.status = PENDING`, đánh dấu bản ghi `STANDARDIZED_DATA` cũ là `invalidated`, ghi `AUDIT_LOG.action = ROLLBACK`. |
| FR Mapping | FR-20, FR-21 |
| NFR Mapping | NFR-02 (Auditability), NFR-04 (RBAC) |
| Mức độ quan trọng | Cao — đảm bảo khả năng sửa lỗi và truy vết |

#### 4.4.2. Điều kiện tiên quyết và Điều kiện kết thúc

| Điều kiện tiên quyết (Pre-conditions) | Điều kiện kết thúc (Post-conditions) |
|---|---|
| (a) Admin đã đăng nhập với role có quyền `admin.rollback`. (b) Tồn tại bản ghi `STANDARDIZED_DATA` đã được duyệt trước đó và chưa bị rollback. (c) **Thời gian từ lúc duyệt (`APPROVED_AT`) đến hiện tại ≤ 72 giờ.** | **Thành công:** `STANDARDIZED_DATA.is_invalidated = true`, `invalidated_at = NOW()`, `invalidated_by = admin_id`, **không xoá** (giữ truy vết); `APPROVAL_QUEUE.status` chuyển về `PENDING`; `AUDIT_LOG` có row mới với `action = ROLLBACK`, `before`, `after`, `reason`. **Thất bại:** Nếu quá 72 giờ → hệ thống từ chối, hiển thị "Đã quá thời hạn rollback, vui lòng lập phiếu điều chỉnh"; không có thay đổi nào trong DB. |

#### 4.4.3. Luồng chính (Basic Flow)

| # | Actor | Hành động |
|---|---|---|
| 1 | Admin | Tại danh sách `STANDARDIZED_DATA` (hoặc trong `AUDIT_LOG`), Admin chọn 1 bản ghi đã duyệt gần đây. |
| 2 | Admin | Nhấn **[Rollback]**, hệ thống mở dialog xác nhận "Bạn chắc chắn muốn rollback bản ghi này?". |
| 3 | Admin | Xác nhận bằng cách nhập **lý do (bắt buộc)** và nhấn **[Xác nhận Rollback]**. |
| 4 | System | Kiểm tra `NOW() - APPROVED_AT ≤ 72h`. |
| 5 | System | Cập nhật `STANDARDIZED_DATA.is_invalidated = true`, `invalidated_at`, `invalidated_by`. |
| 6 | System | Cập nhật `APPROVAL_QUEUE.status = PENDING`. |
| 7 | System | Ghi `AUDIT_LOG`: `action = ROLLBACK`, `actor_id`, `reason`, `before = {status: APPROVED}`, `after = {status: PENDING}`, `timestamp`. |
| 8 | Admin | UI hiển thị "Rollback thành công. Bản ghi đã được đưa về trạng thái chờ duyệt". |

#### 4.4.4. Luồng rẽ nhánh / Ngoại lệ chính

| Mã | Tình huống | Xử lý |
|---|---|---|
| **A1** | **Đã quá 72 giờ kể từ lúc duyệt** | Hệ thống hiển thị "Đã quá thời hạn rollback (72h). Vui lòng lập phiếu điều chỉnh qua phòng KHCN". Nút **[Xác nhận Rollback]** bị disable. |
| A2 | Admin không nhập lý do | Form validation báo lỗi "Lý do rollback là bắt buộc"; không submit được. |
| A3 | Double-rollback attempt | Hệ thống phát hiện `is_invalidated = true` → từ chối, trả "Bản ghi đã được rollback trước đó lúc <timestamp>". |

#### 4.4.5. Tiêu chí nghiệm thu (Given-When-Then)

| AC | Mô tả tiêu chí |
|---|---|
| AC-04.01 | **Given** Admin đã duyệt 1 bản ghi cách đây 1 giờ, **When** Admin nhấn **[Rollback]** và nhập lý do, **Then** `STANDARDIZED_DATA.is_invalidated = true`, `APPROVAL_QUEUE.status = PENDING`, `AUDIT_LOG` có row `action = ROLLBACK` với đầy đủ `reason`. |
| AC-04.02 | **Given** Admin đã duyệt 1 bản ghi cách đây 73 giờ, **When** Admin nhấn **[Rollback]**, **Then** hệ thống từ chối với thông báo "Đã quá thời hạn 72h", không có thay đổi DB. |
| AC-04.03 | **Given** Admin đã rollback 1 bản ghi, **When** Admin cố rollback lại lần nữa, **Then** hệ thống từ chối "Bản ghi đã được rollback trước đó". |
| AC-04.04 | **Given** Operator cố gọi API rollback, **When** request gửi đi, **Then** API trả HTTP 403, `SECURITY_AUDIT_LOG` ghi nhận, không có thay đổi DB. |
| AC-04.05 | **Given** Admin rollback thành công, **When** kiểm tra `STANDARDIZED_DATA`, **Then** bản ghi cũ **vẫn tồn tại** với `is_invalidated = true` (không bị xoá — đảm bảo truy vết theo NFR-02). |

---

### 4.5. UC-05 — Thống kê và kết xuất báo cáo chuẩn hoá

**Ánh xạ PI:** PI 3.4

#### 4.5.1. Thuộc tính tổng quan

| Thuộc tính | Nội dung |
|---|---|
| Mã UC | UC-05 |
| Tên UC | Thống kê và kết xuất báo cáo chuẩn hoá (Reporting & Export) |
| Actor chính | Operator |
| Actor phụ | Admin (cùng quyền), System (engine xuất file) |
| Mô tả ngắn | Operator/Admin lọc dữ liệu `STANDARDIZED_DATA` theo năm / khoa-bộ môn / publisher family / trạng thái duyệt, xem dashboard thống kê, sau đó xuất báo cáo dưới dạng Excel / CSV / JSON có ký số và ghi log export. |
| FR Mapping | FR-23, FR-24, FR-25, FR-26, FR-27, FR-28, FR-29 |
| NFR Mapping | NFR-05 (Interoperability) |
| Mức độ quan trọng | Cao — đầu ra cuối cùng phục vụ báo cáo kiểm định và liên thông hệ thống chung |

#### 4.5.2. Điều kiện tiên quyết và Điều kiện kết thúc

| Điều kiện tiên quyết (Pre-conditions) | Điều kiện kết thúc (Post-conditions) |
|---|---|
| (a) Operator đã đăng nhập với quyền `op.export`. (b) `STANDARDIZED_DATA` có ít nhất 1 bản ghi `is_invalidated = false` (đã được duyệt). | **Thành công:** File báo cáo (Excel/CSV/JSON) được tạo ra đúng schema quy định (NFR-05); file được ký số SHA-256 + chữ ký số hệ thống; `EXPORT_AUDIT_LOG` có row mới với `actor_id`, `timestamp`, `row_count`, `file_hash`, `filter_criteria`. **Thất bại:** Không tạo file; UI hiển thị lỗi; Operator có thể thử lại. |

#### 4.5.3. Luồng chính (Basic Flow)

| # | Actor | Hành động |
|---|---|---|
| 1 | Operator | Mở menu "Báo cáo & Xuất dữ liệu". |
| 2 | Operator | Chọn bộ lọc: Năm (đơn/khoảng), Khoa–Bộ môn, Publisher Family (Springer / IEEE / Elsevier / Khác), Trạng thái duyệt. |
| 3 | Operator | (Tuỳ chọn) Nhập từ khoá tìm kiếm trong `Title`, `Author Keywords`, `Abstract`. |
| 4 | System | Áp dụng bộ lọc, hiển thị preview số dòng sẽ xuất, dashboard thống kê (top tác giả, top publisher, tỉ lệ trích dẫn). |
| 5 | Operator | Chọn định dạng xuất: Excel / CSV / JSON. |
| 6 | Operator | Nhấn **[Xuất báo cáo]**. |
| 7 | System | Tạo file theo định dạng (FR-26/27/28), ký số SHA-256. |
| 8 | System | Ghi `EXPORT_AUDIT_LOG`: `actor_id`, `timestamp`, `row_count`, `filter_criteria`, `file_hash`. |
| 9 | System | Trả file cho Operator qua download. |
| 10 | Operator | Tải file về máy, nộp báo cáo kiểm định ICTU hoặc chia sẻ với hệ thống chung. |

#### 4.5.4. Luồng rẽ nhánh / Ngoại lệ chính

| Mã | Tình huống | Xử lý |
|---|---|---|
| A1 | Bộ lọc trả về 0 dòng | UI hiển thị "Không có dữ liệu thoả mãn bộ lọc"; nút **[Xuất báo cáo]** bị disable. |
| A2 | File xuất vượt 100 MB | Hệ thống tự động chia nhỏ thành nhiều file (file_001.xlsx, file_002.xlsx, …) + 1 file ZIP tổng hợp; ghi nhiều row trong `EXPORT_AUDIT_LOG` (mỗi file 1 row). |
| A3 | File JSON không validate theo schema | Rollback transaction; không tạo file; không ghi audit log; UI hiển thị "Lỗi validate JSON Schema, vui lòng liên hệ kỹ thuật". |

#### 4.5.5. Tiêu chí nghiệm thu (Given-When-Then)

| AC | Mô tả tiêu chí |
|---|---|
| AC-05.01 | **Given** Operator chọn bộ lọc `Year ∈ [2024, 2026]` và `Publisher Family = "Springer"`, **When** Operator nhấn **[Xuất Excel]**, **Then** file `.xlsx` được tải về, mở được bằng MS Excel 2019+ không lỗi, có ≥ 2 sheet (Summary + Raw), và `EXPORT_AUDIT_LOG` có row với `row_count` khớp preview. |
| AC-05.02 | **Given** Operator chọn **[Xuất CSV]**, **When** file được tải về, **Then** mở bằng Google Sheets không lỗi encoding, header đúng thứ tự cột, mỗi dòng đúng số field. |
| AC-05.03 | **Given** Operator chọn **[Xuất JSON]**, **When** file được tải về, **Then** `python -m json.tool` parse không lỗi, validate theo `scopus-ictu-export.schema.json` PASS 100%. |
| AC-05.04 | **Given** bất kỳ lần xuất nào thành công, **When** kiểm tra `EXPORT_AUDIT_LOG`, **Then** có row ghi đầy đủ `actor_id`, `timestamp`, `row_count`, `file_hash` (SHA-256 64 ký tự hex), `filter_criteria` dạng JSON. |
| AC-05.05 | **Given** Operator chọn bộ lọc trả về 0 dòng, **When** Operator nhấn **[Xuất]**, **Then** nút bị disable và UI hiển thị thông báo "Không có dữ liệu". |
| AC-05.06 | **Given** Operator cố xuất dữ liệu mà chưa đăng nhập hoặc không có quyền `op.export`, **When** request gửi đi, **Then** hệ thống redirect đăng nhập hoặc trả HTTP 403. |

---

## 5. MA TRẬN TRUY VẾT YÊU CẦU (TRACEABILITY MATRIX)

### 5.1. Ma trận FR ↔ PI

Bảng dưới đây tổng hợp toàn bộ **29 FR ánh xạ về 5 PI** trong Mục 7b của đề cương, giúp hội đồng đối chiếu nhanh khi nghiệm thu.

| PI | Mô tả PI | FR đáp ứng | Actor thực hiện |
|---|---|---|---|
| **PI 1.1** | Import CSV Scopus & lưu `RAW_DATA` giữ nguyên gốc | FR-01, FR-02, FR-03, FR-04 | Operator → System |
| **PI 1.2** | Validation cấu trúc & cấp dòng (Idempotency, warning không dừng luồng) | FR-02, FR-05, FR-06 (NFR-01) | Operator, System |
| **PI 2.1** | Parse, chuẩn hoá tiếng Việt, đếm tác giả, đối soát GV ICTU, Match Score, phân nhóm HIGH/MEDIUM/PENDING | FR-07, FR-08, FR-09, FR-10, FR-11, FR-12, FR-13, FR-14, FR-15 (NFR-03) | System |
| **PI 3.3** | Approval Queue + Diff, 3 lựa chọn duyệt, Rollback, ghi `STANDARDIZED_DATA` + `AUDIT_LOG` | FR-16, FR-17, FR-18, FR-19, FR-20, FR-21, FR-22 (NFR-02, NFR-04) | Admin, System |
| **PI 3.4** | Lọc, tìm kiếm, thống kê, xuất Excel/CSV/JSON, ký số, log export | FR-23, FR-24, FR-25, FR-26, FR-27, FR-28, FR-29 (NFR-05) | Operator, Admin, System |

### 5.2. Ma trận UC ↔ FR ↔ NFR ↔ PI

Bảng này khẳng định **độ phủ 100% mục tiêu đề cương**: mỗi UC gắn với FR cụ thể, được kiểm soát bởi NFR phù hợp và phục vụ ít nhất một PI của Mục 7b.

| UC | FR đáp ứng | NFR đáp ứng | PI đáp ứng |
|---|---|---|---|
| **UC-01** Nhập và kiểm tra tính toàn vẹn dữ liệu | FR-01, FR-02, FR-03, FR-04, FR-05, FR-06 | NFR-01 | PI 1.1, PI 1.2 |
| **UC-02** Bóc tách tác giả và tính điểm gợi ý | FR-07, FR-08, FR-09, FR-10, FR-11, FR-12, FR-13, FR-14, FR-15 | NFR-03 | PI 2.1 |
| **UC-03** Đối soát và phê duyệt liên kết | FR-16, FR-17, FR-18, FR-19, FR-21 | NFR-02, NFR-04 | PI 3.3 |
| **UC-04** Hoàn tác và ghi vết kiểm toán | FR-20, FR-21 | NFR-02, NFR-04 | PI 3.3, PI 3.4 |
| **UC-05** Thống kê và kết xuất báo cáo | FR-23, FR-24, FR-25, FR-26, FR-27, FR-28, FR-29 | NFR-05 | PI 3.4 |

**Khẳng định độ phủ:** Toàn bộ 5 PI trong Mục 7b của đề cương đều có ít nhất một UC và FR tương ứng trong bảng trên; toàn bộ 29 FR, 5 NFR, 5 UC đều đã được mô tả trong các mục §1 – §4 của báo cáo này.
