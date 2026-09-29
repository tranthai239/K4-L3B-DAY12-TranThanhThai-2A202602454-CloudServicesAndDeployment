# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời trực tiếp bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên:  Trần Thanh Thái  Mã học viên: 2A202602454

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Render, nếu quên cấu hình AGENT_API_KEY, ứng dụng sẽ dừng ngay lúc khởi động và log báo thiếu biến môi trường. Nhờ đó phát hiện lỗi cấu hình trước khi service nhận request. Nếu dùng mặc định "changeme", ứng dụng vẫn chạy và có thể được công khai trên Internet với khóa dễ đoán; người khác có thể gọi/ask, tiêu tốn ngân sách và làm sai dữ liệu rate limit hoặc lịch sử hội thoại. Vì vậy, fail fast giúp biến lỗi bảo mật âm thầm thành lỗi triển khai để sửa .

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thu được khi gọi `/ask` trong Docker Compose:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T05:01:59.377053+00:00", "user_id": "test-user", "tokens_in": 1, "tokens_out": 35, "cost_usd": 2.115e-05}
> ```
>
> Hai việc làm được với log JSON mà `print("đã trả lời xong")` không làm được:
>
> 1. **Lọc và tổng hợp tự động**: công cụ như `jq`, Datadog hoặc Grafana Loki có thể query thẳng theo trường `event`, `user_id`, `cost_usd` — ví dụ tính tổng chi phí theo ngày hay đếm số request mỗi user — không cần parse chuỗi tự do.
>
> 2. **Gắn ngữ cảnh chính xác vào từng sự kiện**: mỗi dòng chứa `timestamp` chuẩn ISO 8601, `user_id` và số token, nên khi có sự cố có thể trace đúng request gây lỗi mà không phải đoán từ thứ tự dòng in ra.

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
| 1 stage (bản đầu) | 1.7 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch khoảng 1.4 GB. Image single-stage dùng `python:3.11` (base đầy đủ, chứa GCC, build tools, header files phục vụ biên dịch C extension), tất cả đều giữ lại trong image cuối. Multi-stage dùng `python:3.11-slim` làm runtime, chỉ copy đúng các package đã cài từ stage builder sang — bỏ lại toàn bộ compiler, header, pip cache và build tools. Phần chênh lệch chính là toolchain biên dịch và thư viện hệ thống không cần thiết lúc chạy.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sau khi sửa một ký tự trong `app/main.py` rồi build lại, kết quả cho thấy:
>
> - **CACHED**: `WORKDIR`, `COPY requirements.txt`, `RUN pip install`, `COPY --from=builder` — toàn bộ stage builder và bước copy dependency được dùng lại vì `requirements.txt` không đổi.
> - **Chạy lại**: `COPY app ./app`, `COPY utils ./utils`, `RUN useradd` — vì source code thay đổi làm invalidate từ layer `COPY app` trở đi.
>
> Build chỉ mất ~2.6 giây vì bước nặng nhất (`pip install`, ~30 giây) được cache.
>
> Nếu đặt `COPY . .` trước `RUN pip install`: bất kỳ thay đổi nào trong source code cũng invalidate layer `COPY . .`, kéo theo `pip install` phải chạy lại từ đầu — mỗi lần sửa 1 dòng code mất thêm ~30 giây chờ cài lại toàn bộ dependency, dù dependency không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) Code Python có lỗ hổng, ví dụ endpoint cho phép inject lệnh qua input không sanitize → (2) Kẻ tấn công gửi payload chạy lệnh shell bên trong container → (3) Nếu container chạy root, lệnh đó có quyền root: đọc/ghi mọi file trong container, cài thêm tool, và nếu container mount volume từ host hoặc có Linux capability nguy hiểm (như `CAP_SYS_ADMIN`), kẻ tấn công có thể thoát ra host với quyền root — đọc secret, truy cập container khác, chiếm máy.
>
> Lệnh `USER 10001` trong Dockerfile cắt đứt chuỗi ở bước (3): dù kẻ tấn công chạy được lệnh, process chỉ có quyền user thường (UID 10001) — không ghi được file hệ thống, không cài package, không mount, không thoát container qua privilege escalation. Thiệt hại bị giới hạn trong phạm vi thư mục `/app` của user đó.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> **20 request trong 2 giây.** Cách đạt được: gửi 10 request ở giây :59 (cuối phút cũ), bộ đếm reset lúc :00, rồi gửi thêm 10 request ở giây :00 (đầu phút mới). Tổng cộng 20 request trong 2 giây liên tiếp, gấp đôi hạn mức mong muốn.
>
> Sliding window 60 giây của mình (dùng Redis ZSET) không bị lỗi này vì nó luôn đếm số request trong 60 giây gần nhất tính từ thời điểm hiện tại — không có ranh giới phút để khai thác. 10 request ở giây :59 vẫn nằm trong cửa sổ đến tận giây :59 phút sau, nên gửi thêm ở giây :00 sẽ bị chặn ngay.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Khác nhau**: Rate limit kiểm soát *tần suất* — tối đa 10 request/phút bất kể mỗi request tốn bao nhiêu. Cost guard kiểm soát *chi phí tích lũy* — tổng tiền trong tháng không vượt ngân sách.
>
> **Rate limit cho qua, cost guard chặn**: User gửi 1 request/phút (tần suất thấp), nhưng mỗi câu hỏi rất dài khiến token nhiều. Sau vài ngày tổng `cost_usd` vượt `MONTHLY_BUDGET_USD` → cost guard trả 402 dù rate limit vẫn cho qua.
>
> **Cost guard cho qua, rate limit chặn**: User mới, chưa tốn đồng nào trong tháng, nhưng viết script gọi 15 request liên tục trong vài giây → rate limit trả 429 ở request thứ 11, dù ngân sách còn nguyên.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối → endpoint gộp gọi `ping()` thất bại → trả 503.
> 2. Orchestrator (Docker healthcheck hoặc Kubernetes liveness probe) thấy 503 → coi container "chết" → restart cả 3 container.
> 3. Container mới khởi động, Redis vẫn chưa về → endpoint lại trả 503 → orchestrator restart tiếp.
> 4. Lặp lại restart loop cho đến khi Redis trở lại (~30 giây) — trong thời gian đó toàn bộ service gián đoạn, mọi request đều bị mất.
>
> Với thiết kế hiện tại: `/health` không gọi Redis nên trả 200 → container không bị restart. `/ready` trả 503 → load balancer ngừng gửi traffic mới vào, nhưng container vẫn sống. Khi Redis về sau 30 giây, `/ready` trả 200 lại → traffic tự phục hồi, không mất container nào.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Khi thử `docker compose up --scale agent=3`, bị lỗi `Bind for 0.0.0.0:8000 failed: port is already allocated` vì `docker-compose.yml` map cố định host port `8000:8000` — chỉ agent-1 khởi động được. Tuy nhiên, với 1 instance chạy thật, gọi `/ask` hai lần liên tiếp cùng `X-User-Id` cho kết quả `history_length: 0` rồi `history_length: 2` — chứng tỏ lịch sử nằm trong Redis, instance nào đọc cũng thấy.
>
> Nếu lưu trong dict Python: khi có nhiều instance, mỗi instance giữ dict riêng trong RAM. Request đầu vào instance A lưu history ở A, request sau vào instance B thấy dict rỗng → `history_length` nhảy không theo thứ tự (0, 0, 2, 0...), agent "mất trí nhớ" tùy thuộc load balancer đẩy request vào đâu. Restart container cũng mất toàn bộ history.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi gặp phải**: Khi thử scale 3 container trên Docker Compose trước khi deploy cloud, gặp lỗi:
> ```
> Bind for 0.0.0.0:8000 failed: port is already allocated
> ```
>
> **Nguyên nhân**: `docker-compose.yml` map cố định `ports: "8000:8000"`, nên tất cả replica đều giành cùng port trên host — chỉ container đầu tiên bind được, các container sau bị crash.
>
> **Cách tìm ra**: Đọc log Docker Compose, thấy agent-2 và agent-3 báo lỗi port ngay khi khởi động trong khi agent-1 vẫn healthy.
>
> **Bài học cho cloud**: Trên Render, platform tự gán `$PORT` cho mỗi instance và đặt load balancer phía trước, nên không cần map port cố định. Service deploy thành công, `/health` trả 200, `/ready` trả 200 với Redis kết nối qua Render Key Value.
