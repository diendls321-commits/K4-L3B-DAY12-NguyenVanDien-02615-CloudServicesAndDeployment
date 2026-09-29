# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Văn Điền  Mã học viên: 02615

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy ứng dụng lên môi trường Cloud (như Railway/Render/Kubernetes), nếu người triển khai quên thiết lập biến môi trường `AGENT_API_KEY`:
- Nếu đặt mặc định là `"changeme"`: Service vẫn khởi động bình thường, healthcheck xanh và tiếp nhận traffic từ Internet. Các bot quét mạng hoặc kẻ xấu có thể dùng khóa mặc định này để gọi API LLM liên tục, gây tiêu tốn ngân sách hoặc xâm nhập tài nguyên. Khi đó lỗi âm thầm diễn ra và ta chỉ nhận biết khi nhìn thấy hóa đơn tiền triệu.
- Nếu không có mặc định (Fail Fast): Pydantic ném `ValidationError` ngay lúc ứng dụng khởi động. Container dừng ngay lập tức tại bước deployment khiến quá trình rollout bị chặn đứng và báo lỗi rõ ràng trên log. Kỹ sư deploy nhìn thấy ngay lỗi thiếu khóa và sửa đổi trước khi bất kỳ request nào từ bên ngoài lọt vào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T10:05:00.000000+00:00", "user_id": "sv01", "tokens_in": 15, "tokens_out": 40, "cost_usd": 0.00015}`

Hai việc làm được với dòng log này mà `print("đã trả lời xong")` không làm được:
1. **Truy vấn, lọc và tổng hợp số liệu tự động (Log Aggregation & Metrics):** Các công cụ gom log tập trung (như Datadog, Grafana Loki, ELK, CloudWatch) có thể parse các trường JSON để chạy query thống kê chính xác, ví dụ: tính tổng `cost_usd` của từng `user_id` trong ngày, hay vẽ biểu đồ lượng token tiêu thụ theo thời gian thực mà không cần viết regex phức tạp.
2. **Cảnh báo tự động dựa trên ngưỡng (Automated Alerting):** Có thể thiết lập quy tắc giám sát tự động để bắn thông báo ngay khi `level == "error"` hoặc phát hiện một request có chi phí `cost_usd > 0.05`. Với `print()`, log xuống dòng không có cấu trúc máy đọc, dễ vỡ khi gom log và không kích hoạt được rule cảnh báo chính xác.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> *Câu trả lời của bạn*

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> *Câu trả lời của bạn*

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> *Câu trả lời của bạn*

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> *Câu trả lời của bạn*

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> *Câu trả lời của bạn*

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> *Câu trả lời của bạn*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> *Câu trả lời của bạn*

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *Câu trả lời của bạn*
