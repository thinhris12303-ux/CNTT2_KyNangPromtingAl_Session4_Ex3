# BÁO CÁO THIẾT KẾ GIẢI PHÁP PROMPTING ỨNG DỤNG THỰC TẾ
**Chủ đề:** Kỹ thuật Prompting Nâng cao (Logic & Chuỗi - Chain-of-Thought, Prompt Chaining & Xử lý đứt gãy)

---

## 1. MỤC TIÊU & BỐI CẢNH BÀI TOÁN
* **Mục tiêu:** Ứng dụng kỹ thuật lập luận chuỗi (**Chain-of-Thought - CoT**), chuỗi tác vụ (**Prompt Chaining**) và cơ chế xử lý ngoại lệ (**Fallback / Error Handling**) để tự động hóa quy trình nghiệp vụ thực tế, rèn luyện tư duy cấu trúc logic trước khi triển khai thành mã nguồn.
* **Bối cảnh thực tế:** Một cửa hàng thương mại điện tử nhận lượng lớn tin nhắn khiếu nại, yêu cầu đổi/trả hàng mỗi ngày từ khách hàng. Nội dung tin nhắn thường viết tự do, thiếu thông tin hoặc sai lệch dữ liệu đơn hàng. Nhân viên CSKH mất nhiều thời gian đối chiếu chính sách và tính toán mức hoàn tiền thủ công.
* **Giải pháp đề xuất:** Xây dựng hệ thống **3 mắt xích (Prompt Chain)** tự động hóa từ khâu bóc tách dữ liệu thô, kiểm tra tính hợp lệ, suy luận tính toán mức đền bù cho đến sinh phản hồi khách hàng.

---

## 2. BƯỚC 1: THIẾT KẾ GIẢI PHÁP (INPUT / OUTPUT & LUỒNG XỬ LÝ)

### 2.1. Xác định Input (Đầu vào)
1. **Tin nhắn khiếu nại thô (`raw_message`):** Văn bản tự nhiên từ khách hàng (có thể viết tắt, thiếu mã đơn, không đề cập bằng chứng).
2. **Dữ liệu hệ thống đối chiếu (`order_db`):** Bản ghi đơn hàng thực tế trên hệ thống (Mã đơn, Giá trị đơn hàng, Ngày nhận hàng thành công, Trạng thái đơn).
3. **Quy tắc nghiệp vụ cửa hàng (`policy_rules`):**
   * Thời hạn đổi/trả: Trong vòng **7 ngày** kể từ ngày nhận hàng thành công.
   * Lỗi do cửa hàng / vận chuyển (vỡ, hỏng, giao sai): Hoàn **100%** giá trị đơn, miễn phí vận chuyển hoàn.
   * Khách đổi ý / không vừa kích cỡ: Hoàn **80%** giá trị đơn (khấu trừ 20% phí lưu kho), khách chịu phí vận chuyển hoàn (`30.000 VNĐ`).
   * Điều kiện bắt buộc với khiếu nại vỡ/hỏng/thiếu hàng: Phải có **video mở hộp (unboxing)**.

### 2.2. Luồng xử lý & Cơ chế xử lý đứt gãy (Fallback)
* **Mắt xích 1 (Data Extraction):** Bóc tách dữ liệu từ tin nhắn và đối chiếu với `order_db` $\rightarrow$ Xuất JSON chuẩn hóa.
* **Mắt xích 2 (CoT Reasoning & Validation):** Kiểm tra các bẫy dữ liệu (thiếu trường thông tin, sai lệch mã đơn, quá hạn 7 ngày, thiếu video bằng chứng) $\rightarrow$ Đưa ra quyết định điều hướng và tính toán tài chính.
* **Mắt xích 3 (Response Generation):** Dựa trên mã trạng thái từ Mắt xích 2 để sinh câu trả lời phù hợp cho khách hàng.

### 2.3. Cấu trúc Output mong muốn
* **Output Mắt xích 1:** Đối tượng JSON chứa các trường thông tin đã chuẩn hóa và danh sách trường bị khuyết (`missing_fields`).
* **Output Mắt xích 2:** Khối lập luận từng bước (`<thinking>`) kèm JSON quyết định (`AUTO_APPROVE`, `NEED_INFO`, `ESCALATE_HUMAN`, hoặc `REJECT_EXPIRED`) và số tiền hoàn cụ thể.
* **Output Mắt xích 3:** Đoạn tin nhắn phản hồi khách hàng ngắn gọn, lịch sự, đúng quy định.

---

## 3. BƯỚC 2: HỆ THỐNG PROMPT CHUYÊN NGHIỆP

### 3.1. Mắt xích 1: Trích xuất dữ liệu & Phát hiện bẫy dữ liệu trống/sai lệch
```markdown
# ROLE
Bạn là Hệ thống Trích xuất Dữ liệu Đơn hàng (Data Extractor). Nhiệm vụ của bạn là đọc tin nhắn khiếu nại thô của khách hàng và đối chiếu với dữ liệu hệ thống để xuất ra JSON chuẩn xác.

# INPUT DATA
- Tin nhắn khách hàng: {{raw_message}}
- Dữ liệu hệ thống: {{order_db}}

# CONSTRAINTS & ERROR HANDLING (XỬ LÝ BẪY DỮ LIỆU)
1. KHÔNG tự ý suy diễn hoặc bịa đặt thông tin không có trong Input. Nếu thiếu trường nào, gán giá trị `null`.
2. Nếu `ma_don` trong tin nhắn khác với `order_db`, đánh dấu `"data_mismatch": true`.
3. Kiểm tra các trường bắt buộc: `ma_don`, `ly_do_khieu_nai`, `co_video_unboxing` (true/false/null). Liệt kê các trường bị `null` vào mảng `missing_fields`.
4. Chỉ trả về duy nhất định dạng JSON hợp lệ, không kèm văn bản giải thích.

# OUTPUT SCHEMA
{
  "ma_don": "string | null",
  "ly_do_khieu_nai": "string | null",
  "phan_loai_loi": "LOI_SHOP | DOI_Y | KHONG_XAC_DINH",
  "co_video_unboxing": "boolean | null",
  "gia_tri_don_hang": "number | null",
  "so_ngay_tu_khi_nhan": "number | null",
  "data_mismatch": "boolean",
  "missing_fields": ["array of strings"]
}
```

### 3.2. Mắt xích 2: Lập luận Logic (Chain-of-Thought) & Xử lý đứt gãy
```markdown
# ROLE
Bạn là Chuyên viên Thẩm định Khiếu nại Tự động. Dựa trên JSON từ Mắt xích 1, hãy suy luận từng bước (Chain-of-Thought) để đưa ra quyết định xử lý.

# INPUT
- JSON từ Mắt xích 1: {{step1_json}}
- Ngày hiện tại: 2026-11-17

# CHAIN-OF-THOUGHT INSTRUCTIONS (BẮT BUỘC SUY LUẬN TUẦN TỰ)
Hãy trình bày suy luận theo đúng 4 bước sau trong thẻ `<thinking>`:
- Bước 1 (Kiểm tra toàn vẹn dữ liệu - Xử lý đứt gãy): Kiểm tra `data_mismatch` và `missing_fields`. Nếu `data_mismatch == true`, dừng tính toán và gắn cờ `ESCALATE_HUMAN` (Lý do: Sai lệch mã đơn). Nếu `missing_fields` không rỗng, gắn cờ `NEED_INFO` (Liệt kê thông tin cần xin thêm).
- Bước 2 (Kiểm tra thời hạn 7 ngày): Nếu `so_ngay_tu_khi_nhan > 7`, từ chối hoàn trả (`REJECT_EXPIRED`).
- Bước 3 (Kiểm tra bằng chứng): Nếu `phan_loai_loi == "LOI_SHOP"` (vỡ, hỏng, thiếu hàng) mà `co_video_unboxing != true`, kích hoạt xử lý đứt gãy `ESCALATE_HUMAN` (Lý do: Thiếu video mở hộp cần quản lý duyệt ngoại lệ).
- Bước 4 (Tính toán tài chính):
  * Nếu `LOI_SHOP` và đủ điều kiện: `tien_hoan = gia_tri_don_hang * 100%`, `phi_ship = 0`.
  * Nếu `DOI_Y` và đủ điều kiện: `tien_hoan = gia_tri_don_hang * 80%`, `phi_ship = 30000` (khách chịu).

# OUTPUT FORMAT
<thinking>
[Ghi rõ lập luận từng bước ở đây]
</thinking>
```json
{
  "decision_status": "AUTO_APPROVE | NEED_INFO | ESCALATE_HUMAN | REJECT_EXPIRED",
  "reason_code": "string",
  "tien_hoan_du_kien": "number",
  "phi_ship_khach_chiu": "number",
  "thong_tin_can_bo_sung": ["array of strings"]
}
```
```

### 3.3. Mắt xích 3: Điều hướng & Sinh phản hồi khách hàng
```markdown
# ROLE
Bạn là Nhân viên CSKH tận tâm của Cửa hàng. Hãy viết tin nhắn phản hồi khách hàng dựa trên kết quả thẩm định từ Mắt xích 2.

# INPUT
- Kết quả Mắt xích 2: {{step2_json}}

# RULES
- Nếu `decision_status == "NEED_INFO"`: Lịch sự xin lỗi vì sự bất tiện và đề nghị khách cung cấp đúng các mục trong `thong_tin_can_bo_sung`.
- Nếu `decision_status == "ESCALATE_HUMAN"`: Báo khách rằng trường hợp đang được chuyển gấp cho Quản lý cấp cao kiểm tra và sẽ phản hồi trong 15 phút (tuyệt đối không tự ý từ chối hay hứa hẹn số tiền).
- Nếu `decision_status == "AUTO_APPROVE"`: Thông báo duyệt yêu cầu, nêu rõ số tiền hoàn (`tien_hoan_du_kien`) và phí ship (`phi_ship_khach_chiu`).
- Nếu `decision_status == "REJECT_EXPIRED"`: Từ chối khéo léo do đã quá hạn 7 ngày theo chính sách.
- Giữ độ dài dưới 120 từ, thân thiện, rõ ràng.
```

---

## 4. KẾT QUẢ CHẠY THỬ (TEST CASE CÓ BẪY DỮ LIỆU)

### 4.1. Dữ liệu đầu vào giả lập (Input)
* **`raw_message`:** *"Shop ơi hôm qua mình mới nhận đơn #DH882 mà mở ra thấy bình giữ nhiệt bị móp nắp rồi. Hoàn tiền cho mình nhé."*
  *(Bẫy dữ liệu: Khách hàng không cung cấp thông tin có quay video mở hộp hay không).*
* **`order_db`:** `{"ma_don": "DH882", "gia_tri": 250000, "ngay_nhan": "2026-11-16", "trang_thai": "DA_GIAO"}`

### 4.2. Phản hồi của AI qua từng mắt xích (Output)

#### Kết quả Mắt xích 1 (Trích xuất JSON)
```json
{
  "ma_don": "DH882",
  "ly_do_khieu_nai": "Bình giữ nhiệt bị móp nắp",
  "phan_loai_loi": "LOI_SHOP",
  "co_video_unboxing": null,
  "gia_tri_don_hang": 250000,
  "so_ngay_tu_khi_nhan": 1,
  "data_mismatch": false,
  "missing_fields": ["co_video_unboxing"]
}
```

#### Kết quả Mắt xích 2 (Lập luận CoT & Bắt bẫy dữ liệu thiếu)
```xml
<thinking>
- Bước 1 (Kiểm tra toàn vẹn dữ liệu): `data_mismatch` là false (mã đơn DH882 khớp hệ thống). Tuy nhiên, mảng `missing_fields` đang chứa `"co_video_unboxing"`. Do thiếu thông tin bắt buộc để xét duyệt lỗi móp méo từ shop/vận chuyển, quy trình tự động tạm dừng ở trạng thái `NEED_INFO` thay vì tự động duyệt tiền hoàn.
- Bước 2 (Kiểm tra thời hạn 7 ngày): `so_ngay_tu_khi_nhan` = 1 <= 7 ngày (Đạt điều kiện thời gian).
- Bước 3 (Kiểm tra bằng chứng): Chưa thể kết luận vì `co_video_unboxing` đang là `null`.
- Bước 4 (Tính toán tài chính): Tạm thời để `tien_hoan_du_kien` = 0 cho đến khi nhận đủ bằng chứng.
</thinking>
```
```json
{
  "decision_status": "NEED_INFO",
  "reason_code": "MISSING_UNBOXING_VIDEO",
  "tien_hoan_du_kien": 0,
  "phi_ship_khach_chiu": 0,
  "thong_tin_can_bo_sung": ["Video quay lại quá trình mở hộp (unboxing) sản phẩm bị móp nắp"]
}
```

#### Kết quả Mắt xích 3 (Tin nhắn gửi khách hàng)
> Dạ Cửa hàng rất xin lỗi bạn về trải nghiệm không mong muốn với sản phẩm bình giữ nhiệt ở đơn hàng **#DH882** ạ! 
> 
> Để bộ phận kỹ thuật có thể hỗ trợ duyệt hoàn tiền 100% và miễn phí vận chuyển cho mình nhanh nhất, bạn vui lòng gửi giúp shop **video quay lại quá trình mở hộp (unboxing)** thấy rõ tình trạng nắp bình bị móp nhé. Ngay khi nhận được video, shop sẽ xử lý hoàn tiền cho bạn ngay lập tức ạ!

---

## 5. TỔNG KẾT & ĐÁNH GIÁ GIẢI PHÁP
1. **Kiểm soát ảo giác (Anti-Hallucination):** Mắt xích 1 ép kiểu JSON và quy tắc gán `null` ngăn AI tự suy diễn thông tin chưa được xác thực.
2. **Minh bạch hóa lập luận:** Thẻ `<thinking>` ở Mắt xích 2 hiển thị rõ luồng suy luận từng bước trước khi xuất quyết định tài chính, giúp dễ dàng kiểm tra và gỡ lỗi (debug).
3. **An toàn vận hành (Fail-safe):** Các trường hợp đứt gãy dữ liệu (thiếu video, sai lệch mã đơn) được chặn lại bằng cờ điều hướng (`NEED_INFO`, `ESCALATE_HUMAN`) thay vì đưa ra cam kết sai chính sách.