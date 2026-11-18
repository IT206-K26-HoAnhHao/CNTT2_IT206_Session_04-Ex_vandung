# Thiết kế giải pháp Prompting ứng dụng thực tế cho Kỹ thuật Prompting Nâng cao (Logic & Chuỗi)

## 1. Mục tiêu & Bối cảnh bài toán

* **Mục tiêu:** Ứng dụng kiến thức **Kỹ thuật Prompting Nâng cao (Chain-of-Thought, Prompt Chaining, Xử lý đứt gãy)** để thiết kế một giải pháp tự động hóa quy trình xử lý thông tin thực tế, rèn luyện tư duy lập luận chặt chẽ trước khi triển khai mã nguồn.
* **Bối cảnh thực tế:** Cửa hàng bán lẻ tiếp nhận lượng lớn tin nhắn đặt hàng và phản hồi qua mạng xã hội với nội dung viết tắt, thiếu thông tin hoặc sai định dạng. Quy trình cần tự động trích xuất dữ liệu, kiểm tra ràng buộc hợp lệ và xử lý đứt gãy khi dữ liệu bị trống hoặc sai lệch.

---

## 2. Bước 1: Thiết kế giải pháp (I/O)

### Input (Đầu vào)
Một đoạn văn bản thô (tin nhắn của khách hàng) bao gồm các thông tin giao dịch như:
* Tên người nhận
* Số điện thoại liên hệ
* Tên sản phẩm & phân loại
* Số lượng đặt mua
* Địa chỉ giao hàng

### Output (Đầu ra)
Dữ liệu chuẩn hóa dưới định dạng **JSON** gồm 4 trường cố định phục vụ đẩy trực tiếp vào hệ thống quản lý đơn hàng (CRM/POS):
* `suy_luan_tung_buoc`: Tóm tắt quá trình suy luận qua từng chuỗi xử lý.
* `phan_loai`: Phân loại ý định (`Đặt hàng` | `Khiếu nại` | `Khác`).
* `trang_thai`: Trạng thái kiểm duyệt dữ liệu (`HOP_LE` | `DUT_GAY_DU_LIEU`).
* `du_lieu`: Chi tiết các trường thông tin đơn hàng đã làm sạch.
* `hanh_dong_tiep_theo`: Lệnh đẩy đơn sang kho hoặc tin nhắn mẫu tự động yêu cầu khách bổ sung thông tin.

---

## 3. Bước 2: Thiết kế Prompt hoàn chỉnh

Khung Prompt tích hợp đầy đủ **Role-playing**, **Chain-of-Thought**, **Prompt Chaining** và **Error Handling (Bẫy dữ liệu)**:

```markdown
# VAI TRÒ (ROLE)
Bạn là Chuyên viên Tự động hóa Vận hành Đơn hàng của cửa hàng bán lẻ. Nhiệm vụ của bạn là phân tích tin nhắn khách hàng, kiểm tra lỗi dữ liệu và xuất kết quả chuẩn hóa.

# QUY TRÌNH XỬ LÝ CHUỖI & SUY LUẬN (PROMPT CHAINING & CHAIN-OF-THOUGHT)
Hãy thực hiện tuần tự qua 3 bước suy luận trước khi đưa ra kết quả cuối cùng:

* **Chuỗi 1 - Phân loại ý định:** Đọc tin nhắn và xác định mục đích chính của khách (Đặt hàng, Khiếu nại, hay Hỏi đáp).
* **Chuỗi 2 - Trích xuất thực thể:** Lọc ra các trường bắt buộc gồm: [Tên khách], [Số điện thoại], [Sản phẩm], [Số lượng], [Địa chỉ].
* **Chuỗi 3 - Kiểm tra ràng buộc & Xử lý đứt gãy (Error Handling):**
  * *Trường hợp dữ liệu trống:* Nếu thiếu bất kỳ trường bắt buộc nào, gán giá trị trường đó là `"MISSING"`. Tuyệt đối không tự ý suy đoán thông tin.
  * *Trường hợp dữ liệu sai lệch:* Số điện thoại phải đủ 10 chữ số và bắt đầu bằng số `0`. Số lượng phải là số nguyên dương lớn hơn `0`. Nếu sai, gán trạng thái `"INVALID"`.
  * *Xử lý đứt gãy:* Nếu có bất kỳ trường nào bị `"MISSING"` hoặc `"INVALID"`, dừng quy trình lên đơn và soạn một tin nhắn lịch sự yêu cầu khách bổ sung đúng phần còn thiếu.

# ĐỊNH DẠNG ĐẦU RA (OUTPUT FORMAT)
Chỉ trả về kết quả theo cấu trúc JSON dưới đây, không giải thích thêm:
{
  "suy_luan_tung_buoc": "Tóm tắt ngắn gọn quá trình kiểm tra 3 chuỗi",
  "phan_loai": "Đặt hàng | Khiếu nại | Khác",
  "trang_thai": "HOP_LE | DUT_GAY_DU_LIEU",
  "du_lieu": {
    "ten_khach": "...",
    "so_dien_thoai": "...",
    "san_pham": "...",
    "so_luong": 0,
    "dia_chi": "..."
  },
  "hanh_dong_tiep_theo": "Đẩy đơn sang kho | Tin nhắn phản hồi khách"
}

# DỮ LIỆU ĐẦU VÀO (INPUT)
"Shop ơi cho chị 2 áo thun trắng size M nhé, giao về 123 Nguyễn Trãi, Q1. SĐT chị là 0912345"
```

---

## 4. Bước 3: Kết quả chạy thử từ AI (Kiểm thử bẫy dữ liệu)

### Tình huống kiểm thử (Test Case)
* **Input thử nghiệm:** `"Shop ơi cho chị 2 áo thun trắng size M nhé, giao về 123 Nguyễn Trãi, Q1. SĐT chị là 0912345"`
* **Bẫy dữ liệu cài cắm:**
  1. Thiếu tên người nhận (`MISSING`).
  2. Số điện thoại chỉ có 7 chữ số, không đủ 10 số theo quy chuẩn (`INVALID`).

### Kết quả phản hồi thực tế của AI
```json
{
  "suy_luan_tung_buoc": "Chuỗi 1: Khách muốn đặt hàng. Chuỗi 2: Trích xuất được 2 áo thun trắng size M, địa chỉ 123 Nguyễn Trãi Q1, SĐT 0912345, thiếu tên. Chuỗi 3: Phát hiện thiếu tên khách (MISSING) và SĐT không đủ 10 số (INVALID) -> Kích hoạt luồng xử lý đứt gãy.",
  "phan_loai": "Đặt hàng",
  "trang_thai": "DUT_GAY_DU_LIEU",
  "du_lieu": {
    "ten_khach": "MISSING",
    "so_dien_thoai": "INVALID (0912345)",
    "san_pham": "Áo thun trắng size M",
    "so_luong": 2,
    "dia_chi": "123 Nguyễn Trãi, Q1"
  },
  "hanh_dong_tiep_theo": "Dạ shop đã nhận thông tin đặt 2 áo thun trắng size M ạ. Chị vui lòng bổ sung giúp shop Tên người nhận và kiểm tra lại Số điện thoại (hiện đang thiếu số) để shop lên đơn ngay nhé!"
}
```
