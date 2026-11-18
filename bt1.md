# BÀI TẬP: SỬA ĐỔI PROMPT THIẾU CẤU TRÚC RÀNG BUỘC ĐỊNH DẠNG

## BƯỚC 1: PHÂN TÍCH LỖI

Prompt cũ: 

"Hãy giải thích ngắn gọn các nội dung của Chain-of-Thought, Prompt Chaining, xử lý đứt gãy."

**Các lỗi trong câu trả lời của AI:**

1. **Lời chào đầu bài:** AI thường mở đầu bằng những câu như "Xin chào! Mình rất vui được hỗ trợ bạn."
2. **Lời dẫn dài dòng:** Có những câu giới thiệu không cần thiết trước khi đi vào nội dung chính.
3. **Lời kết thừa:** AI thường thêm câu như "Hy vọng phần giải thích trên sẽ giúp ích cho bạn" dù không cần thiết.

**Nguyên nhân:** Prompt cũ chưa quy định rõ cấu trúc trình bày, định dạng và những nội dung AI không được phép thêm vào.

## BƯỚC 2: VIẾT PROMPT TỐI ƯU

**Prompt sau khi chỉnh sửa:**

Bạn hãy giải thích 3 nội dung sau:

- Chain-of-Thought.
- Prompt Chaining.
- Xử lý đứt gãy trong Prompt Chaining.

**Yêu cầu bắt buộc:**

- Trình bày bằng gạch đầu dòng.
- Mỗi nội dung giải thích từ 1–2 câu.
- Sử dụng ngôn ngữ tiếng Việt đơn giản, dễ hiểu.
- Đi thẳng vào nội dung chính.
- Không viết lời chào, lời dẫn nhập hoặc lời kết.
- Không giải thích lan man hoặc thêm thông tin ngoài yêu cầu.
- Chỉ trả về kết quả theo đúng định dạng đã yêu cầu.

## BƯỚC 3: KẾT QUẢ PHẢN HỒI CỦA AI

- **Chain-of-Thought:** Là kỹ thuật giúp AI phân tích vấn đề qua các bước logic để đưa ra câu trả lời phù hợp.

- **Prompt Chaining:** Là kỹ thuật chia một nhiệm vụ lớn thành nhiều prompt nhỏ, trong đó kết quả của bước trước được sử dụng làm đầu vào cho bước tiếp theo.

- **Xử lý đứt gãy:** Khi một bước trong chuỗi Prompt Chaining gặp lỗi hoặc cho kết quả không đúng, cần kiểm tra lại đầu ra, điều chỉnh prompt và thực hiện lại bước bị lỗi để đảm bảo kết quả cuối cùng chính xác.