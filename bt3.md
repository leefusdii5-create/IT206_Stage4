Chào bạn, dưới đây là thiết kế giải pháp Prompting nâng cao (Logic & Chuỗi) hoàn chỉnh cho bài toán tự động hóa xử lý thông tin tại cửa hàng, đáp ứng đầy đủ các yêu cầu về Chain-of-Thought (chuỗi tư duy), Prompt Chaining (chuỗi prompt) và cơ chế xử lý đứt gãy/lỗi dữ liệu.

---

## Bước 1: Thiết kế giải pháp (I/O)

### 1. Input (Dữ liệu đầu vào)

Hệ thống nhận vào một đoạn văn bản hoặc dữ liệu thô (đơn hàng, phản hồi khách hàng, hoặc báo cáo kiểm kê) có thể chứa các lỗi phổ biến như thiếu trường thông tin, sai định dạng, hoặc thông tin mâu thuẫn.

* **Ví dụ Input:** Khách hàng gửi qua chat: *"Tôi muốn đổi trả đơn hàng #DH9823 mua hôm qua. Sản phẩm bị lỗi rách đường chỉ ở tay áo. Gửi bù cho tôi cái mới hoặc hoàn tiền gấp nhé, sđt 0901234xxx."* (Lưu ý: Số điện thoại bị che mờ/thiếu số, thiếu tên khách hàng).

### 2. Output (Cấu trúc kết quả mong muốn)

Kết quả trả về theo định dạng JSON hoặc cấu trúc tường minh, chia rõ các bước xử lý logic:

* **Bước 1 - Phân tích dữ liệu & Kiểm tra lỗi:** Phát hiện các trường thiếu/sai (ví dụ: Số điện thoại không hợp lệ, thiếu tên).
* **Bước 2 - Chuỗi tư duy (Chain-of-Thought):** Suy luận logic từng bước để đưa ra phương án xử lý dựa trên chính sách cửa hàng.
* **Bước 3 - Xử lý đứt gãy (Fallback/Error Handling):** Đề xuất câu hỏi/thông báo bổ sung nếu dữ liệu không đủ để xử lý.
* **Bước 4 - Kết quả hành động cuối cùng:** Quyết định duyệt đổi trả, hoàn tiền hay yêu cầu khách bổ sung thông tin.

---

## Bước 2: Viết Prompt Hoàn Chỉnh (Áp dụng Kỹ thuật Nâng cao)

Bạn có thể sử dụng cấu trúc prompt chuyên nghiệp sau đây để đưa vào hệ thống AI:

```markdown
# Role & Objective
Bạn là Trợ lý Vận hành Thông minh chuyên tự động hóa quy trình xử lý đơn hàng và khiếu nại cho cửa hàng thời trang. Nhiệm vụ của bạn là áp dụng tư duy logic theo chuỗi (Chain-of-Thought) để phân tích yêu cầu từ khách hàng và xử lý các tình huống dữ liệu bị đứt gãy hoặc thiếu sót.

# Context & Rules
- Hãy thực hiện lần lượt các bước suy luận được định nghĩa bên dưới. KHÔNG bỏ qua bất kỳ bước nào.
- **Quy tắc xử lý lỗi (Ràng buộc dữ liệu):** Nếu phát hiện thông tin đầu vào bị thiếu (ví dụ: thiếu tên, số điện thoại không đủ 10 chữ số, thiếu mã đơn hàng) hoặc có dấu hiệu gian lận/mâu thuẫn, bạn PHAI chuyển sang chế độ xử lý đứt gãy (Fallback), từ chối đưa ra quyết định tự động và tạo câu hỏi yêu cầu khách hàng bổ sung.

# Input Data
{{USER_INPUT}}

# Execution Steps (Chain-of-Thought)

## Bước 1: Kiểm tra tính hợp lệ của dữ liệu (Validation)
- Trích xuất các trường: [Mã đơn hàng, Tên khách hàng, Số điện thoại, Nội dung khiếu nại].
- Kiểm tra xem có trường nào bị trống, thiếu hoặc sai định dạng không. Liệt kê rõ các lỗi phát hiện (nếu có).

## Bước 2: Suy luận logic nghiệp vụ (Logical Reasoning)
- Dựa trên nội dung khiếu nại, phân loại mức độ sự cố (Lỗi sản phẩm, Sai size, Giao thiếu, v.v.).
- Đối chiếu với chính sách đổi trả (Đổi trả trong 7 ngày đối với lỗi từ nhà sản xuất).

## Bước 3: Xử lý ngoại lệ & Đứt gãy (Fallback Handling)
- Nếu Bước 1 phát hiện lỗi dữ liệu: Xác định rõ thông tin nào đang thiếu và soạn sẵn một câu phản hồi lịch sự yêu cầu khách cung cấp bổ sung.
- Nếu dữ liệu hoàn toàn hợp lệ: Bỏ qua bước này.

## Bước 4: Đề xuất hành động cuối cùng (Final Action)
- Đưa ra quyết định cuối cùng (Ví dụ: Duyệt đổi mới, Yêu cầu gửi thêm hình ảnh lỗi, hoặc Yêu cầu bổ sung thông tin liên hệ).

# Output Format (JSON Structure)
{
  "validation_result": {
    "is_valid": true/false,
    "missing_or_invalid_fields": ["danh sách trường lỗi hoặc 'Không'"],
    "error_description": "Mô tả chi tiết lỗi dữ liệu nếu có"
  },
  "chain_of_thought_analysis": {
    "issue_category": "Phân loại sự cố",
    "policy_check": "Đánh giá theo chính sách cửa hàng"
  },
  "fallback_response_to_customer": "Câu trả lời mẫu gửi khách hàng (dùng trong trường hợp thiếu thông tin hoặc cần xác thực thêm)",
  "final_action_recommendation": "Hành động cụ thể nhân viên hoặc hệ thống cần thực hiện"
}

```

---

## Ví dụ Chạy thử (Test Run)

Khi đưa đoạn Input mẫu vào prompt trên, AI sẽ trả về kết quả cấu trúc như sau:

```json
{
  "validation_result": {
    "is_valid": false,
    "missing_or_invalid_fields": ["Tên khách hàng", "Số điện thoại"],
    "error_description": "Tên khách hàng không được cung cấp; số điện thoại '0901234xxx' bị che mờ, không đủ 10 chữ số hợp lệ."
  },
  "chain_of_thought_analysis": {
    "issue_category": "Lỗi sản phẩm (Rách đường chỉ tay áo)",
    "policy_check": "Thuộc diện đổi trả trong 7 ngày, nhưng chưa thể xác minh danh tính và đơn hàng do thiếu thông tin."
  },
  "fallback_response_to_customer": "Chào bạn, cửa hàng rất tiếc về sự cố rách đường chỉ ở tay áo sản phẩm trong đơn #DH9823. Để hỗ trợ bạn đổi mới nhanh chóng, bạn vui lòng cung cấp lại Họ tên đầy đủ và Số điện thoại chính xác giúp cửa hàng nhé ạ!",
  "final_action_recommendation": "Tạm dừng quy trình tự động đổi trả. Yêu cầu nhân viên chăm sóc khách hàng liên hệ qua kênh chat để xác thực thông tin số điện thoại."
}

```