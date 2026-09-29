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
| 1 stage (bản đầu) | 1.02 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~750 MB) bao gồm:
1. **Base image đầy đủ so với bản slim:** Bản `python:3.11` gốc dựa trên hệ điều hành Debian đầy đủ, tích hợp sẵn các gói công cụ lập trình, trình biên dịch C/C++ (`build-essential`, `gcc`, `g++`, `make`), gói header nhân và thư viện phát triển (`python3-dev`), các tiện ích gỡ lỗi, man pages và tài liệu trợ giúp. Bản `python:3.11-slim` đã loại bỏ hoàn toàn các gói phụ trợ này.
2. **Loại bỏ build dependencies và pip cache:** Với multi-stage build, toàn bộ quá trình download wheel và cache của pip (`--no-cache-dir`) chỉ diễn ra ở stage `builder`. Stage runtime cuối cùng chỉ copy kết quả các package thuần túy sang thư mục `/usr/local` mà không mang theo bất kỳ file rác hay trình biên dịch nào.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Quan sát thực tế:
- Với cấu trúc hiện tại: Lệnh `COPY requirements.txt .` và `RUN pip install ...` nằm trước. Khi sửa `app/main.py`, file `requirements.txt` không thay đổi nên Docker tái sử dụng lại toàn bộ cache (CACHED) từ layer cài đặt dependency trở về trước. Docker chỉ thực thi lại từ layer `COPY app ./app`, `COPY utils ./utils` và đổi quyền user. Thời gian build lại chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Bất kỳ thay đổi nào dù chỉ 1 ký tự trong mã nguồn cũng làm thay đổi checksum của lệnh `COPY . .`. Do Docker vô hiệu hóa cache từ layer bị thay đổi trở đi, lệnh `RUN pip install` buộc phải chạy lại từ đầu, kéo theo việc tải và cài đặt lại toàn bộ thư viện mỗi lần sửa code, làm chậm quá trình CI/CD và tốn băng thông nghiêm trọng.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện tấn công (Container Breakout & Privilege Escalation):
1. **Khai thác lỗi ứng dụng:** Kẻ tấn công lợi dụng một lỗ hổng trong code Python (ví dụ: lỗi Remote Code Execution - RCE qua `pickle`, `yaml.unsafe_load`, command injection hoặc lỗ hổng bảo mật của một thư viện phụ thuộc) để thực thi mã tùy ý.
2. **Chiếm quyền root trong container:** Do container mặc định chạy dưới quyền root (UID 0), kẻ tấn công lập tức có toàn quyền root trong không gian người dùng của container.
3. **Thoát khỏi container chiếm máy host:** Vì tiến trình trong container chia sẻ chung nhân Linux (kernel) với máy host, nếu container mount chung thư mục máy host, mount Docker socket `/var/run/docker.sock`, hoặc nhân Linux tồn tại lỗ hổng leo thang đặc quyền (kernel vulnerability), tiến trình root UID 0 có thể ghi đè các file hệ thống nhạy cảm của host (như `/etc/shadow`, `/etc/crontab`) để chiếm quyền điều khiển root trên máy chủ vật lý.
4. **Vị trí cắt đứt của lệnh `USER`:** Lệnh `USER appuser` (UID 10001) hạ đặc quyền của tiến trình xuống user thông thường. Khi bị khai thác RCE, kẻ tấn công chỉ có quyền hạn tối thiểu: không thể ghi vào thư mục hệ thống của container, không thể tương tác với các socket đặc quyền và không thể tận dụng UID 0 để leo thang ra ngoài máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request trong 2 giây liên tiếp**.

Cách đạt được:
Với cơ chế đếm theo phút đồng hồ cố định (Fixed Window Counter), bộ đếm lượt gọi sẽ được reset về 0 vào giây thứ 00 của mỗi phút mới:
- Ở giây `10:00:59` (1 giây cuối của phút thứ nhất), người dùng gửi dồn dập 10 request. Hệ thống ghi nhận 10/10 request hợp lệ.
- Sang giây `10:01:00` (giây đầu tiên của phút thứ hai), đồng hồ bước sang phút mới nên bộ đếm tự động reset về 0. Người dùng gửi tiếp ngay 10 request nữa.
Tổng cộng trong 2 giây (từ `10:00:59` đến `10:01:01`), người dùng đã gửi thành công 20 request mà không bị chặn, lưu lượng tăng đột biến gấp đôi hạn mức quy định. Cửa sổ trượt (Sliding Window) giải quyết triệt để lỗ hổng này vì nó luôn tính tổng request trong đúng 60 giây tính lùi từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác biệt cốt lõi:
- **Rate Limit (Tần suất):** Giới hạn *số lượng request* trong một cửa sổ thời gian ngắn (ví dụ: 10 request/phút) nhằm chống nghẽn mạng, chống DoS và đảm bảo tính sẵn sàng của hạ tầng server. Cơ chế này không quan tâm request đó tốn bao nhiêu token hay bao nhiêu tiền.
- **Cost Guard (Tài chính):** Giới hạn *tổng chi phí tiền tệ* trong một chu kỳ dài (ví dụ: 10 USD/tháng) nhằm bảo vệ ngân sách của chủ dịch vụ trước chi phí API LLM. Cơ chế này tính toán dựa trên lượng token in/out thực tế quy đổi ra USD.

Hai tình huống cụ thể:
1. **Rate limit cho qua nhưng Cost guard chặn (HTTP 402):** Người dùng chỉ gửi 1 request trong phút (hoàn toàn hợp lệ theo rate limit 10 req/phút). Tuy nhiên, tổng chi tiêu trong tháng của người dùng này đã đạt 9.99 USD / 10.0 USD budget. Khi người dùng gửi thêm một prompt dài có chi phí ước tính làm tổng vượt quá 10.0 USD, Cost Guard sẽ lập tức chặn lại và trả về lỗi 402 Payment Required.
2. **Cost guard cho qua nhưng Rate limit chặn (HTTP 429):** Vào đầu tháng, tài khoản người dùng còn nguyên 10.0 USD (chưa tiêu đồng nào). Người dùng chạy script bắn liên tiếp 15 request ngắn (mỗi request chỉ tốn 0.0001 USD, tổng 15 request mới chỉ mất 0.0015 USD - ngân sách còn hơn 9.99 USD). Cost Guard hoàn toàn đồng ý, nhưng từ request thứ 11 trở đi trong vòng 60 giây, Rate Limiter sẽ lập tức chặn lại và trả về mã lỗi 429 Too Many Requests để tránh làm quá tải server.

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
