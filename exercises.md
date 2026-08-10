# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời...` bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyen Huy Toan  Mã học viên: 2A202601697

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy ứng dụng lên môi trường Production mà cấu hình thiếu biến môi trường `AGENT_API_KEY`:
> - Nếu sử dụng giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động bình thường nhưng hoạt động với khóa công khai dễ bị đoán biết. Kẻ tấn công hoặc bot quét tự động có thể gọi API này và tiêu tốn toàn bộ ngân sách LLM của hệ thống.
> - Khi sử dụng cơ chế Fail-fast, ứng dụng ném lỗi `ValidationError` và crash ngay lập tức tại bước khởi chạy. Hệ thống CI/CD hoặc Container Orchestrator sẽ phát hiện lỗi và dừng deploy, giúp bảo vệ an toàn hệ thống 100% trước khi traffic chạm tới app.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T02:04:13.491045+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 24, "cost_usd": 0.0000162}`
> 
> Hai việc làm được:
> 1. **Truy vấn và Lọc dữ liệu tự động (Filtering & Querying)**: Dễ dàng cấu hình các bộ thu thập log (như Datadog, ELK, Grafana) để lọc chính xác các request theo một `user_id` cụ thể hoặc tìm các lỗi có `level: "error"` chỉ trong vài giây.
> 2. **Phân tích và Cảnh báo chi phí (Metrics & Alerts)**: Có thể tự động tính toán tổng số token tiêu thụ, tổng chi phí LLM theo thời gian thực và tự động phát cảnh báo (Alert) tới Slack/Discord khi chi phí vượt ngưỡng cho phép.

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
| 1 stage (bản đầu) | 1020 MB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~750MB) là do phiên bản base image đầy đủ chứa các công cụ build (compiler như gcc, dev libraries, headers) và pip cache, package files không cần thiết ở runtime. Bản multi-stage chỉ copy thư mục cài đặt package cuối cùng (`/install` sang `/usr/local`) trên nền image `python:3.11-slim` cực nhẹ, giúp tối ưu dung lượng lưu trữ và tăng tốc độ kéo/đẩy image (pull/push).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - Những layer được dùng lại từ cache: Các layer từ builder stage (`FROM python:3.11-slim AS builder`, `WORKDIR /build`, `COPY requirements.txt .`, `RUN pip install...`) và layer copy dependencies (`COPY --from=builder /install /usr/local`). Layer `COPY app/ app/` và các lệnh phía sau (`EXPOSE`, `HEALTHCHECK`, `CMD`) sẽ phải chạy lại.
> - Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi sửa đổi chỉ một ký tự trong file `main.py`, cache của layer `COPY . .` bị mất hiệu lực, kéo theo layer `RUN pip install` phía sau buộc phải tải và cài đặt lại toàn bộ thư viện từ đầu, làm chậm quá trình build đáng kể.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> - Chuỗi sự kiện:
>   1. Ứng dụng Python tồn tại lỗ hổng thực thi mã từ xa (RCE) hoặc ghi đè file.
>   2. Kẻ tấn công khai thác lỗ hổng và chiếm quyền thực thi mã với tư cách user đang chạy container (ở đây là `root`).
>   3. Với đặc quyền `root` trong container, kẻ tấn công thực hiện kỹ thuật thoát container (container escape) thông qua các lỗ hổng nhân (kernel vulnerabilities) hoặc mount socket Docker `/var/run/docker.sock`.
>   4. Khi thoát thành công, kẻ tấn công chiếm toàn quyền kiểm soát (`root`) trên máy chủ vật lý (host).
> - Lệnh `USER appuser` cắt đứt chuỗi ở bước 2: Kẻ tấn công sau khi xâm nhập chỉ có quyền hạn cực kỳ hạn chế của `appuser`, không thể chạy các lệnh đặc quyền hay truy cập tài nguyên hệ thống để thoát container ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> - Tối đa: **20 request** trong 2 giây liên tiếp.
> - Giải thích: Với cơ chế đếm theo phút (Fixed Window reset lúc giây 00), người dùng có thể gửi 10 request ở giây thứ 59 của phút thứ nhất (vẫn thỏa mãn hạn mức 10/phút). Ngay khi bước sang giây thứ 00 của phút thứ hai, bộ đếm reset về 0, người dùng gửi tiếp 10 request nữa. Tổng cộng họ đã gửi thành công 20 request trong vòng 2 giây (giây 59 và giây 00) mà không bị hệ thống chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - Sự khác biệt: Rate Limit giới hạn **tốc độ gửi request** (tính bằng số lượng request/giây/phút) để bảo vệ hạ tầng hệ thống khỏi quá tải. Cost Guard giới hạn **tổng chi tiêu tài chính** (tính bằng số tiền USD/tháng) để tránh thâm hụt ngân sách LLM.
> - Tình huống Rate Limit cho qua nhưng Cost Guard chặn: Người dùng gửi 1 request mỗi giờ (thỏa mãn hạn mức tốc độ), nhưng request đó tải lên một tài liệu dài hàng triệu từ, tiêu tốn 15 USD tiền API LLM (vượt quá ngân sách tháng `monthly_budget_usd = 10.0` nên bị Cost Guard chặn).
> - Tình huống Cost Guard cho qua nhưng Rate Limit chặn: Người dùng mới sử dụng hết 0.01 USD trong tháng, nhưng gửi liên tiếp 50 request trong 1 giây (vi phạm tốc độ 10 request/phút nên bị Rate Limit chặn).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis gặp sự cố mất kết nối trong 30 giây.
> 2. Cả 3 container nhận thấy Redis chết thông qua kiểm tra sức khỏe đã gộp và đồng loạt trả về lỗi 503.
> 3. Hệ thống điều phối (như Kubernetes/Railway) nhận thấy liveness probe (/health) của cả 3 container đều hỏng và coi là cả 3 container đã chết.
> 4. Hệ thống lập tức ra lệnh khởi động lại (restart) đồng thời cả 3 container này.
> 5. Trong thời gian các container đang khởi động lại, toàn bộ dịch vụ của ứng dụng bị gián đoạn hoàn toàn (sập hệ thống dây chuyền).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần with cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> - Thay đổi: Lịch sử hội thoại hiển thị `history_length` sẽ tăng giảm bất thường không đồng bộ (ví dụ: `0, 0, 1, 0, 1, 2...`).
> - Giải thích: Mỗi container sở hữu vùng nhớ RAM độc lập và lưu một bản copy riêng của dict Python. Do Nginx điều phối request xoay vòng (round-robin), mỗi request tiếp theo của user sẽ được chuyển ngẫu nhiên tới 1 trong 3 container. Mỗi container chỉ nhớ được phần hội thoại trực tiếp gửi đến nó, khiến ngữ cảnh hội thoại bị chia cắt và mất tính nhất quán.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - Lỗi: Ứng dụng bị Crash/Timeout khi khởi chạy trên Cloud (như Railway/Render) với thông báo lỗi `Application failed to start / Health check timeout`.
> - Nguyên nhân: Phát hiện qua log dashboard của platform thấy Uvicorn cố định lắng nghe cổng `8000`, trong khi platform gán một cổng ngẫu nhiên thông qua biến môi trường `PORT`.
> - Cách sửa: Sửa lệnh CMD trong Dockerfile và config app để đọc động biến môi trường `PORT` (ví dụ: `port: int = 8000` của Pydantic Settings tự động map từ `PORT` và uvicorn chạy cổng cấu hình này).
