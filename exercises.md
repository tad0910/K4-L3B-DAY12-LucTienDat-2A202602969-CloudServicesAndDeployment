# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng câu trả lời mặc định bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lục Tiến Đạt  Mã học viên: 2A202602969

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy dịch vụ lên Cloud, nếu vô tình quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard mà ứng dụng lại để giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động bình thường. Kẻ tấn công hoặc người dùng bên ngoài có thể gọi API với key mặc định `"changeme"` để truy cập hệ thống trái phép và làm tiêu tốn ngân sách gọi LLM. Việc không đặt mặc định giúp ứng dụng "fail fast" (báo lỗi sụp đổ ngay khi khởi động), buộc người vận hành phải phát hiện và bổ sung ngay key bảo mật trước khi service nhận traffic thật.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

```json
{"timestamp": "2026-09-29T02:48:12.123456+00:00", "event": "ask_completed", "service": "day12-agent", "user_id": "sv-test", "tokens_in": 5, "tokens_out": 20, "cost_usd": 0.001}
```

1. **Truy vấn và lọc tự động (Structured Parsing):** Các hệ thống tập trung log (Datadog, Elasticsearch, Grafana Loki) có thể tự động bóc tách các trường `user_id`, `tokens_in`, `tokens_out`, `cost_usd` để lọc log theo từng user hoặc tính toán số liệu thống kê mà không cần viết regex phức tạp.
2. **Cảnh báo và giám sát chi phí (Automated Alerting & Metrics):** Có thể trích xuất giá trị trường `cost_usd` và số lượng token để dựng biểu đồ giám sát thời gian thực, đồng thời cài đặt cảnh báo tự động (alert) khi chi phí của một user vượt ngưỡng cho phép.

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
| 1 stage (bản đầu) | ~950 MB |
| Multi-stage | ~180 MB |

Giải thích: Phần dung lượng chênh lệch (~770 MB) là các bộ công cụ build, trình biên dịch (gcc, g++, make), header files, bộ nhớ đệm cài đặt (`~/.cache/pip`), và các thư viện hệ thống không cần thiết cho quá trình chạy ứng dụng ở môi trường production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Các layer từ đầu cho đến trước `COPY app/ ./app` (như `FROM`, `WORKDIR`, `COPY requirements.txt .`, `RUN pip install`) đều được Docker sử dụng lại từ **cache**. Chỉ có layer `COPY app/ ./app` và các bước sau đó mới phải chạy lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi sửa bất kỳ file mã nguồn nào, cache của layer `COPY . .` sẽ bị invalid, khiến Docker buộc phải chạy lại toàn bộ lệnh `RUN pip install` từ đầu, làm tốn nhiều thời gian tải lại tất cả thư viện dependencies.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- **Chuỗi sự kiện:** Kẻ tấn công khai thác lỗ hổng Remote Code Execution (RCE) trong ứng dụng Python ➔ Vì container chạy bằng `root` (UID 0), kẻ tấn công chiếm quyền `root` bên trong container ➔ Kẻ tấn công lợi dụng lỗ hổng container escape để tương tác với Kernel máy Host ➔ Do UID 0 trong container trùng với `root` trên máy Host, kẻ tấn công chiếm toàn quyền điều khiển máy Host.
- **Lệnh `USER` cắt đứt chuỗi:** Lệnh `USER appuser` chuyển tiến trình sang tài khoản thường (non-root). Dù kẻ tấn công chiếm được shell trong container, họ cũng không có quyền cài đặt package hệ thống, không sửa được file hệ thống và không thể thực hiện hành vi leo thang đặc quyền ra máy Host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- **Tối đa:** **20 request**.
- **Giải thích:** Với đếm theo phút đồng hồ (Fixed Window), người dùng có thể gửi 10 request vào 1 giây cuối cùng của phút thứ nhất (giây `00:59`). Ngay giây tiếp theo (giây `01:00`), bộ đếm reset về 0 cho phút mới, người dùng gửi tiếp 10 request nữa. Như vậy trong 2 giây liên tiếp (`00:59` - `01:00`), hệ thống đã cho qua tổng cộng 20 request.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Khác biệt:** Rate Limit quản lý **tần suất/số lượng request** trong khoảng thời gian ngắn để chống quá tải hạ tầng. Cost Guard quản lý **ngân sách/chi phí tiền tệ** (tính theo số token LLM tiêu thụ) trong chu kỳ dài (tháng) để bảo vệ tài chính.
- **Rate Limit cho qua nhưng Cost Guard chặn:** User gửi 1 request duy nhất trong phút (thỏa mãn Rate Limit < 10 req/phút), nhưng prompt rất dài kèm response tạo ra lượng token lớn khiến tổng chi phí tháng vượt mốc `$10.0`. Cost Guard sẽ chặn.
- **Cost Guard cho qua nhưng Rate Limit chặn:** User gửi 15 request liên tiếp trong 10 giây (mỗi request cực ngắn tốn rất ít token, tổng chi phí mới `$0.001` - chưa vượt ngân sách `$10.0`). Rate Limit sẽ chặn từ request thứ 11.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis mất kết nối trong 30s ➔ Endpoint gộp kiểm tra thấy Redis lỗi nên trả về 503.
2. Orchestrator (K8s/Docker Swarm/Render) nhận được 503 từ Liveness Probe nên cho rằng tiến trình container bị lỗi/treo.
3. Orchestrator lập tức **kill và restart cả 3 container**.
4. Các container mới khởi động lại tiếp tục kiểm tra Redis (lúc này vẫn chưa hồi phục) và lại báo fail 503.
5. Orchestrator tiếp tục restart lặp đi lặp lại (CrashLoopBackOff/Restart Storm), gây lãng phí tài nguyên CPU/RAM máy host và làm hệ thống sập hoàn toàn thay vì chỉ tạm ngừng nhận traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- **Nếu lưu trong dict Python (Stateful):** Mỗi container giữ 1 dict riêng trong RAM. Khi Load Balancer chia đều request tới 3 container A, B, C, `history_length` sẽ nhảy thất thường (ví dụ: 1 ➔ 1 ➔ 1 ➔ 2 ➔ 2 ➔ 3...), vì mỗi container chỉ biết lịch sử của các request do chính nó xử lý.
- **Khi dùng Redis (Stateless):** Cả 3 container đều đọc/ghi chung vào Redis, do đó `history_length` luôn tăng đều đặn (1 ➔ 2 ➔ 3 ➔ 4...) bất kể request rơi vào container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi:** Lỗi `HTTP 500 Internal Server Error` (hoặc `status: not ready, redis: false`) khi gọi endpoint `/ready` trên Render.
- **Nguyên nhân:** Render Key Value cấp connection URL theo dạng SSL `rediss://...`. Thư viện `redis-py` trong Python yêu cầu xác thực SSL khi gặp `rediss://`, đồng thời hàm khởi tạo `redis.from_url()` chưa bọc `try...except` khiến exception ném ra làm sập FastAPI dependency.
- **Cách sửa:** Cập nhật hàm `get_redis_client()` trong `app/store.py` để xử lý scheme `rediss://` với tùy chọn `ssl_cert_reqs=None`, đồng thời bổ sung khối `try...except` khi khởi tạo client để nếu Redis bận thì endpoint `/ready` trả về HTTP 503 thay vì bị crash 500.
