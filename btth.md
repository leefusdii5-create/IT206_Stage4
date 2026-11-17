### Phần 1: Báo cáo thiết kế giải pháp (Architecture & Data Flow)

#### 1. Bài toán thực tế & Mục tiêu

Tự động hóa quy trình tiếp nhận, phân loại, kiểm tra chính sách và phản hồi khiếu nại đơn hàng cho một cửa hàng thương mại điện tử. Thay vì dùng một prompt khổng lồ dễ gây "ảo giác" (hallucination) hoặc tràn ngữ cảnh, hệ thống sử dụng kiến trúc **Prompt Chaining (Chuỗi Prompt)** gồm 3 mắt xích (Nodes) chạy tuần tự, tích hợp **Chain-of-Thought (CoT)** và **Xử lý đứt gãy (Fallback/Exception Handling)**.

#### 2. Kiến trúc luồng xử lý khép kín (Pipeline)

```text
[Raw Input] 
   │
   ▼
[Prompt 1: Trích xuất & Làm sạch (Extraction & Validation)]
   ├──► (Nếu thiếu dữ liệu / Dữ liệu dị biệt) ──► [Nhánh Đứt Gãy: Tạo câu hỏi bổ sung & Dừng chuỗi]
   │
   ▼ (Nếu dữ liệu hợp lệ)
[Prompt 2: Suy luận nghiệp vụ & Đối chiếu chính sách (CoT Reasoning)]
   │
   ▼ (JSON Quyết định nghiệp vụ)
[Prompt 3: Soạn thảo phản hồi & Định dạng hành động (Action & Response)]
   │
   ▼
[Clean JSON Output cho Backend]

```

#### 3. Cơ chế tối ưu Token & Chống Crash hệ thống

* **Tối ưu Token:** Mỗi Prompt chỉ nhận đúng phần dữ liệu cần thiết từ bước trước dưới dạng JSON rút gọn, không truyền lại toàn bộ lịch sử hội thoại hay văn bản thừa.
* **Chống Crash (Defensive Prompting):** Sử dụng schema JSON cố định làm giao thức giao tiếp giữa các mắt xích. Nếu gặp dữ liệu dị biệt (sai định dạng, ký tự rác, prompt injection, thông tin trống), hệ thống gắn cờ `"status": "HALT"` để ngắt chuỗi an toàn mà không gây lỗi parser ở backend.

---

### Phần 2: Hệ thống Prompt Thực thi (Prompt Chaining)

#### Mắt xích 1: Trích xuất & Kiểm tra tính toàn vẹn (Extraction & Validation)

*Nhiệm vụ: Lọc nhiễu, chuẩn hóa dữ liệu thô và phát hiện bẫy dữ liệu.*

```markdown
# Role
Bạn là Bộ lọc Dữ liệu Đầu vào (Data Sanitizer).

# Task
Trích xuất thông tin từ văn bản khách hàng và kiểm tra tính hợp lệ.

# Rules (Chống lỗi & Tối ưu token)
- Chỉ trả về JSON thuần túy, không giải thích.
- Các trường bắt buộc: `order_id` (định dạng #DH + số), `phone` (đủ 10 số, bắt đầu bằng 0), `issue_summary` (tóm tắt lỗi dưới 20 từ).
- Nếu bất kỳ trường nào thiếu, sai định dạng, hoặc chứa nội dung vô nghĩa/spam: đặt `"status": "HALT"` và liệt kê lỗi vào `missing_info`. Nếu đủ và đúng: đặt `"status": "PASS"`.

# Input
{{RAW_CUSTOMER_MESSAGE}}

# Output Schema
{
  "status": "PASS | HALT",
  "data": {
    "order_id": "string | null",
    "phone": "string | null",
    "issue_summary": "string | null"
  },
  "missing_info": ["string"]
}

```

---

#### Mắt xích 2: Suy luận Logic Nghiệp vụ (Chain-of-Thought Reasoning)

*Điều kiện kích hoạt: Chỉ chạy khi Mắt xích 1 trả về `"status": "PASS"`.*

```markdown
# Role
Bạn là Chuyên viên Thẩm định Khiếu nại.

# Policy Context
1. Lỗi sản xuất (rách, bung chỉ, sai mẫu): Đổi mới 100% miễn phí trong 7 ngày.
2. Lỗi do khách hàng (giặt sai cách, tự làm hỏng): Từ chối đổi trả, hỗ trợ voucher 10%.
3. Không xác định rõ nguyên nhân: Yêu cầu gửi ảnh/video thực tế.

# Chain-of-Thought Instructions
Suy luận từng bước trong trường `reasoning_trace` trước khi chốt phương án:
- Bước 1: Xác định nguyên nhân gốc rễ từ `issue_summary`.
- Bước 2: Đối chiếu với 3 quy tắc trong Policy Context.
- Bước 3: Chốt mã hành động (`APPROVE_EXCHANGE`, `REJECT_WITH_VOUCHER`, hoặc `REQUEST_MEDIA`).

# Input (Từ Mắt xích 1)
{{OUTPUT_PROMPT_1}}

# Output Schema
{
  "order_id": "string",
  "reasoning_trace": "Bước 1: ... Bước 2: ... Bước 3: ...",
  "decision_code": "APPROVE_EXCHANGE | REJECT_WITH_VOUCHER | REQUEST_MEDIA"
}

```

---

#### Mắt xích 3: Tổng hợp Phản hồi & Xử lý Đứt gãy (Response & Fallback Generator)

*Nhiệm vụ: Nhận đầu ra từ Mắt xích 1 (nếu `HALT`) hoặc Mắt xích 2 (nếu `PASS`) để tạo câu trả lời chuẩn mực cho khách và lệnh thực thi cho hệ thống.*

```markdown
# Role
Bạn là Trợ lý Chăm sóc Khách hàng Tự động.

# Instructions
Dựa vào dữ liệu đầu vào:
- Nếu `status == "HALT"`: Viết lời nhắn lịch sự xin khách bổ sung chính xác các thông tin trong `missing_info`. Gán `system_action = "WAIT_FOR_USER"`.
- Nếu có `decision_code`: Viết lời nhắn thông báo kết quả xử lý theo đúng mã quyết định, giọng văn chuyên nghiệp, ngắn gọn. Gán `system_action = decision_code`.

# Input
{{PREVIOUS_STEP_JSON}}

# Output Schema
{
  "system_action": "string",
  "customer_reply": "string"
}

```

---

### Phần 3: Kết quả Chạy thử nghiệm (End-to-End Execution)

#### Kịch bản A: Dữ liệu bị đứt gãy / Dị biệt (Kiểm tra cơ chế bắt lỗi)

* **Input thô:** *"Shop làm ăn kiểu gì vậy, cái áo mới mua hôm qua mặc vào đã rách toạc đường chỉ ở nách rồi!! Đổi cái khác ngay cho tôi, số đt 0912345xx, mã đơn quên mất rồi."*
* **Kết quả Mắt xích 1:**

```json
{
  "status": "HALT",
  "data": {
    "order_id": null,
    "phone": null,
    "issue_summary": "Áo mới mua bị rách đường chỉ ở nách"
  },
  "missing_info": [
    "Mã đơn hàng (định dạng #DH...)",
    "Số điện thoại hợp lệ (đủ 10 chữ số)"
  ]
}

```

*(Hệ thống phát hiện `status: "HALT"`, lập tức bỏ qua Mắt xích 2 để tiết kiệm token và nhảy thẳng sang Mắt xích 3).*

* **Kết quả Mắt xích 3 (Đầu ra cuối cùng):**

```json
{
  "system_action": "WAIT_FOR_USER",
  "customer_reply": "Chào bạn, cửa hàng rất xin lỗi về sự cố áo bị rách đường chỉ ở nách. Để hệ thống kiểm tra và hỗ trợ đổi mới ngay cho bạn, bạn vui lòng cung cấp giúp mình Mã đơn hàng (ví dụ: #DH1234) và Số điện thoại đặt hàng đầy đủ 10 số nhé!"
}

```

---

#### Kịch bản B: Dữ liệu chuẩn (Kiểm tra luồng Chain-of-Thought)

* **Input thô:** *"Chào shop, mình nhận đơn #DH8842 hôm qua nhưng mở ra thấy áo bị bung chỉ ở tay áo. SĐT mình 0909123456, shop hỗ trợ đổi giúp mình nhé."*
* **Kết quả Mắt xích 1:**

```json
{
  "status": "PASS",
  "data": {
    "order_id": "#DH8842",
    "phone": "0909123456",
    "issue_summary": "Áo bị bung chỉ ở tay áo khi mới nhận"
  },
  "missing_info": []
}

```

* **Kết quả Mắt xích 2 (CoT hoạt động):**

```json
{
  "order_id": "#DH8842",
  "reasoning_trace": "Bước 1: Sản phẩm bị bung chỉ ở tay áo ngay khi mở hộp, xác định là lỗi từ khâu sản xuất. Bước 2: Đối chiếu chính sách số 1 (lỗi sản xuất được đổi mới 100% trong 7 ngày). Bước 3: Chốt duyệt đổi hàng.",
  "decision_code": "APPROVE_EXCHANGE"
}

```

* **Kết quả Mắt xích 3 (Đầu ra cuối cùng):**

```json
{
  "system_action": "APPROVE_EXCHANGE",
  "customer_reply": "Chào bạn, cửa hàng rất tiếc về lỗi bung chỉ ở tay áo của đơn hàng #DH8842. Yêu cầu đổi sản phẩm mới 100% miễn phí của bạn đã được duyệt. Nhân viên giao nhận sẽ liên hệ qua số 0909123456 để hỗ trợ đổi hàng trong thời gian sớm nhất ạ!"
}

