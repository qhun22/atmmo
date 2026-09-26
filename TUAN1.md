# Bản khảo sát nguồn dữ liệu Scopus — 660 bản ghi

> **Đề tài:** Xây dựng ứng dụng web quản lý và chuẩn hóa hồ sơ tác giả, công bố khoa học trên Scopus phục vụ Trường Đại học Công nghệ thông tin và Truyền thông (ICTU).
> **Nguồn khảo sát:** `Truong_Quang_Huy_Scopus_Sep2026.csv` — xuất từ Scopus tháng 9/2026.
> **Người khảo sát:** Trương Quang Huy — SV ngành Công nghệ thông tin, ICTU.
> **Mục đích:** Cung cấp căn cứ thực tế cho Mục 2.1.1 *Hiện trạng dữ liệu* và Mục 7b (PI 1.1, 1.2) đồ án tốt nghiệp.

---

## 1. Tổng quan nguồn dữ liệu

| Tiêu chí | Giá trị khảo sát |
|---|---|
| Số bản ghi (công bố) | **660** |
| Số trường dữ liệu / bản ghi | **22** (Authors, Author full names, Author(s) ID, Title, Year, Source title, Volume, Issue, Art. No., Page start, Page end, Cited by, DOI, Link, Abstract, Author Keywords, Publisher, Document Type, Publication Stage, Open Access, Source, EID) |
| Khoảng thời gian công bố | **2008 – 2027** (1 bản ghi năm 2027 là bản in-press) |
| Tổng số lượt trích dẫn (Cited by) | **4 885** |
| Bản ghi có Author(s) ID đầy đủ | **660 / 660 (100%)** |
| Bản ghi có Year đầy đủ | **660 / 660 (100%)** |
| Bản ghi thiếu DOI | **25 / 660 (3,79%)** |

---

## 2. Phân bố theo năm công bố (Year)

| Năm | Số bài | Ghi chú |
|---|---:|---|
| 2008 | 5 | Dữ liệu lịch sử, tác giả có thể đã nghỉ/chuyển công tác |
| 2009 | 9 | |
| 2010 | 4 | |
| 2011 | 3 | |
| 2012 | 2 | |
| 2013 | 4 | |
| 2014 | 5 | |
| 2015 | 15 | |
| 2016 | 21 | |
| 2017 | 40 | |
| 2018 | 16 | |
| 2019 | 17 | |
| 2020 | 32 | Giai đoạn COVID-19 |
| 2021 | 43 | |
| 2022 | 45 | |
| 2023 | 68 | Bắt đầu tăng tốc |
| 2024 | 83 | |
| 2025 | 96 | |
| 2026 | **151** | Đỉnh cao nhất (tính đến tháng 9/2026) |
| 2027 | 1 | Bài in-press (Article in press) |

**Nhận xét (PI 1.1):** Xu hướng công bố tăng rõ rệt từ 2023, riêng 2026 đã có **151 bài** — cho thấy áp lực chuẩn hoá dữ liệu lớn nhất rơi vào giai đoạn hiện tại. Dữ liệu giai đoạn 2008-2014 cần đối chiếu đặc biệt vì tác giả có thể đã thay đổi đơn vị công tác.

---

## 3. Phân bố theo loại hình công bố (Document Type)

| Document Type | Số bài | Tỷ lệ |
|---|---:|---:|
| Conference paper | 367 | 55,6% |
| Article | 263 | 39,8% |
| Book chapter | 13 | 2,0% |
| Review | 11 | 1,7% |
| Retracted | **2** | 0,3% |
| Erratum | **2** | 0,3% |
| Editorial | 1 | 0,15% |
| Note | 1 | 0,15% |

**Nhận xét:**
- **Conference paper chiếm đa số** (≈56%) → yêu cầu chuẩn hoá dữ liệu hội nghị kỹ lưỡng, đặc biệt trường *Source title* và *Publisher*.
- Có **2 bài Retracted** và **2 Erratum** → hệ thống cần **cờ cảnh báo** để tránh đưa vào báo cáo kiểm định.

---

## 4. Tình trạng dữ liệu thiếu & ngoại lệ

| Vấn đề | Số bản ghi | Tỷ lệ | Hướng xử lý trong hệ thống |
|---|---:|---:|---|
| Thiếu DOI | 25 | 3,79% | Đánh dấu cảnh báo dòng, **không dừng luồng**; tra cứu thủ công qua EID/Liên kết Scopus |
| Retracted / Erratum | 4 | 0,61% | Hiển thị banner cảnh báo đỏ trong giao diện Diff; mặc định loại khỏi báo cáo kiểm định |
| Khuyết Open Access flag | 487 | 73,8% | Mặc định = Closed Access; xử lý sau trong báo cáo thống kê OA |
| Trống Publisher | 22 | 3,33% | Không ảnh hưởng nghiệp vụ chính (vẫn dùng được cho đếm bài) |

**Phát hiện quan trọng (PI 1.2):** *Mọi bản ghi đều có `Author(s) ID` và `Year` đầy đủ* — đây là điều kiện thuận lợi cho bước **đối soát chính xác qua Scopus Author ID** và **Idempotency qua EID**.

---

## 5. Phân bố số tác giả / bài

| Số tác giả / bài | Số bài |
|---:|---:|
| 1 | 26 |
| 2 | 128 |
| 3 | 129 |
| 4 | 117 |
| 5 | 110 |
| 6 | 78 |
| 7 | 30 |
| 8 | 17 |
| 9 | 9 |
| 10 | 12 |
| 11 | 1 |
| 12 | 2 |
| 14 | 1 |

**Nhận xét:** Trung bình khoảng **3-5 tác giả / bài** (chiếm 53,9%). Bài 1 tác giả (26 bài) là case đơn giản nhất cho đối soát. Bài ≥ 10 tác giả (15 bài) là case phức tạp nhất — dễ xảy ra trùng tên phổ biến, **bắt buộc qua duyệt Admin** ngay cả khi Match Score cao.

---

## 6. Top Nhà xuất bản (Publisher)

| Publisher | Số bài |
|---|---:|
| Springer Science and Business Media Deutschland GmbH | 245 |
| Institute of Electrical and Electronics Engineers Inc. (IEEE) | 75 |
| Springer Verlag | 34 |
| Springer | 27 |
| (trống) | 22 |
| Elsevier Ltd | 15 |
| Elsevier B.V. | 14 |
| Institute of Advanced Engineering and Science | 11 |
| IEEE Computer Society | 10 |
| European Alliance for Innovation | 9 |

**Nhận xét:** **Springer (306 bài gộp)** và **IEEE (85 bài gộp)** chiếm **~59% tổng số bài** → khi xây bộ lọc thống kê nên gom nhóm theo publisher family thay vì từng biến thể pháp nhân.

---

## 7. Đánh giá mức độ sẵn sàng cho chuẩn hoá tự động

| Khía cạnh | Đánh giá | Căn cứ |
|---|---|---|
| **Cấu trúc 22 trường** | ✅ Ổn định | Toàn bộ 660 dòng tuân thủ schema, 0 lỗi parse |
| **Khóa định danh tác giả** | ✅ Tuyệt vời | 100% có `Author(s) ID` |
| **Khóa định danh bài báo** | ✅ Tuyệt vời | 100% có EID; cho phép Idempotency |
| **Khóa định danh năm** | ✅ Tuyệt vời | 100% có Year |
| **DOI** | ⚠️ Thiếu 3,79% | Cần cơ chế cảnh báo dòng |
| **Open Access** | ⚠️ Thiếu 73,8% | Không ảnh hưởng nghiệp vụ chính |
| **Trường tên tiếng Việt** | ⚠️ Cần chuẩn hoá | Tên trong `Authors` viết tắt không dấu, cần parse + normalize |
| **Retracted / Erratum** | ⚠️ Cần cờ cảnh báo | 4 bản ghi phải đánh dấu để loại khỏi báo cáo kiểm định |

**Kết luận khảo sát:** Dữ liệu **đủ tốt để tự động hoá phần lớn** (Idempotency, parse tác giả, đối soát qua Author ID). Chỉ **4 trường hợp đặc biệt** cần xử lý riêng: 25 bài khuyết DOI, 4 bài Retracted/Erratum, 487 bài khuyết OA flag, 15 bài ≥ 10 tác giả — toàn bộ đều đã được tính trước trong **logic ngưỡng Match Score** (PI 2.1) và **giao diện Diff của Admin** (PI 3.3).

---

## 8. Đề xuất chức năng hệ thống từ kết quả khảo sát

| # | Chức năng đề xuất | Căn cứ thực tế | Mapping PI |
|---|---|---|---|
| 1 | **Idempotency check qua EID** trước khi insert RAW_DATA | 100% bản ghi có EID | PI 1.1 |
| 2 | **So khớp chính xác qua Author(s) ID** làm bước đầu tiên | 100% có Author ID | PI 2.1 |
| 3 | **Fuzzy match qua tên tiếng Việt không dấu + Affiliation** làm bước 2 | Tên `Authors` viết tắt không dấu | PI 2.1 |
| 4 | **Ngưỡng 85/50% + cờ cảnh báo Retracted/Erratum** trong giao diện duyệt | 4 bài bất thường + 15 bài ≥ 10 tác giả | PI 3.3 |
| 5 | **Bộ lọc thống kê theo publisher family** (Springer, IEEE, Elsevier, Khác) | 3 publisher gốc chiếm ~92% | PI 3.4 |
| 6 | **Cảnh báo dòng (không dừng luồng)** cho 25 bài khuyết DOI | 3,79% dữ liệu thiếu DOI | PI 1.2 |

---

*File sinh kèm: `scopus-ictu-overview-flow.md` (sơ đồ tổng quan Mermaid 6 bước vừa nửa trang Word).*
*Tham chiếu chéo: xem `scopus-ictu-swimlane.puml` (chi tiết 4 giai đoạn), `scopus-ictu-approval.ir.json` (BPMN 2.0), `scopus-ictu-process-spec.md` (đặc tả bước).*
