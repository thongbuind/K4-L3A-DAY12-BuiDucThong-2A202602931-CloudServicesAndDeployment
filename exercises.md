# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Bùi Đức Thông  Mã học viên: L3A202602931

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Giả sử tôi deploy lên Railway và quên set `AGENT_API_KEY`. Nếu mặc định là `"changeme"`, app vẫn khởi động, health check xanh, và ai đọc repo cũng gọi được `/ask` bằng khóa đó, tôi chỉ biết khi hóa đơn LLM tăng. Với trường bắt buộc, `Settings()` ném `ValidationError` ngay lúc khởi động, deploy fail ở bước health check và log báo rõ thiếu `agent_api_key`, nên tôi sửa trước khi có người dùng nào bị ảnh hưởng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một dòng log thu được khi gọi `/ask` (chạy local với Redis giả):

```
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T02:17:10.905152+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
```

Hai việc làm được mà `print` không làm được: (1) lọc/tổng hợp theo trường, ví dụ cộng `cost_usd` theo `user_id` để biết ai tốn tiền nhất; (2) đặt cảnh báo trên `level == "error"` hoặc đếm số `ask_completed` mỗi phút, vì máy parse được JSON còn câu tiếng Việt thì không.

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

Máy tôi không cài Docker nên tôi **chưa đo được** dung lượng thật của hai bản, không ghi số bịa vào bảng.

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu, `python:3.11`) | chưa đo (Docker chưa cài) |
| Multi-stage (`python:3.11-slim`) | chưa đo (Docker chưa cài) |

Về lý thuyết, phần chênh lệch là base image đầy đủ (kèm gcc, công cụ build, header, nhiều gói hệ thống), pip cache và toàn bộ build context (`.git`, tests, tài liệu) mà bản 1 stage `COPY . .` vào. Bản multi-stage chỉ mang venv đã cài xong và code `app/`, `utils/` sang image slim.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile của tôi, `COPY requirements.txt` và `RUN pip install` đứng trước `COPY app`, `COPY utils`. Sửa một ký tự trong `app/main.py` thì các layer base, venv, pip install đều dùng lại cache; chỉ `COPY app` và các layer sau nó chạy lại. Nếu đặt `COPY . .` lên trước `pip install`, layer đó bị vô hiệu hóa mỗi lần đổi code nên pip install phải cài lại toàn bộ thư viện, build chậm hơn nhiều. (Đây là suy luận từ cơ chế cache, tôi chưa chạy `docker build` để quan sát.)

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: một lỗ hổng (ví dụ command injection hoặc deserialize không an toàn) cho kẻ tấn công chạy lệnh trong process Python; nếu process là root trong container thì họ có quyền root trong container, có thể ghi đè file hệ thống, cài công cụ, và nếu container được mount volume nhạy cảm hoặc gặp lỗi container-escape thì root đó chuyển thành quyền cao trên host. Lệnh `USER appuser` cắt chuỗi ở bước đầu: process chạy bằng uid thường 10001, không ghi được vào hệ thống file, không cài được gói, và dù có RCE thì phạm vi thiệt hại nhỏ hơn nhiều, cũng khó tận dụng lỗi escape hơn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa 20 request trong 2 giây. Đếm theo phút đồng hồ thì cửa sổ 10:00 cho phép 10 request, cửa sổ 10:01 cho phép 10 request nữa. Gửi 10 request lúc 10:00:59 (hết hạn mức phút 10:00) rồi 10 request lúc 10:01:00-10:01:01 (bộ đếm vừa reset) đều được chấp nhận. Sliding window đếm 60 giây gần nhất nên ở giây 10:01:01 vẫn còn thấy 10 request cũ và chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn số lượng request theo thời gian (60 giây, trả 429), cost guard giới hạn tổng tiền theo tháng (trả 402). Rate limit cho qua nhưng cost guard chặn: user gửi 1 request/phút, đều đặn dưới hạn mức, nhưng mỗi request là prompt dài tốn nhiều token, sau vài ngày tổng chi phí vượt 10 USD, cost guard trả 402. Ngược lại: user gửi 30 câu hỏi cực ngắn trong 10 giây, tổng chi phí chỉ vài phần nghìn USD nên cost guard cho qua, nhưng rate limit chặn từ request thứ 11 bằng 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp và kiểm tra Redis trong một endpoint dùng làm health check, thì khi Redis mất kết nối 30 giây: (1) health check của cả 3 container bắt đầu trả lỗi; (2) sau vài lần fail liên tiếp orchestrator coi cả 3 container là không sống; (3) nó restart cả 3 cùng lúc; (4) container khởi động lại vẫn không nối được Redis nên tiếp tục fail và bị restart lặp; (5) toàn bộ dịch vụ down, kể cả sau khi Redis đã hồi phục vài giây, cho tới khi cả cụm ổn định lại. Tách ra thì `/health` vẫn 200 nên không ai bị restart, còn `/ready` trả 503 chỉ khiến load balancer tạm ngừng gửi traffic và tự nhận lại khi Redis về.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với lịch sử lưu trong Redis, `history_length` tăng đều 0, 2, 4, 6... bất kể request rơi vào instance nào, vì cả 3 instance đọc chung một list. Nếu dùng dict Python trong RAM, mỗi instance có dict riêng nên `history_length` nhảy lộn xộn (0, 0, 2, 0, 2, 4...) tùy nginx đưa vào instance nào, và reset về 0 khi container restart. (Tôi chưa chạy `--scale agent=3` vì máy không có Docker; đây là suy luận từ thiết kế, đã kiểm chứng phần lịch sử dùng chung Redis qua `tests/test_cp4.py`.)

---

### Câu 10 — Deploy thật (CP5)

Lỗi tôi gặp khi đưa lên cloud: `git push origin main` báo `Permission denied (publickey)` (và sau khi đổi sang HTTPS thì `could not read Username ... Device not configured`), nên Railway không lấy được code từ GitHub. Tôi tìm nguyên nhân bằng `ssh -T git@github.com` với từng key trong `~/.ssh` (đều bị từ chối) và nhận ra máy không có credential GitHub nào dùng được. Cách sửa: cài Railway CLI, đăng nhập bằng `railway login --browserless` (lần đầu mã hết hạn vì tiến trình chờ tự thoát, phải chạy lại nền), rồi `railway up` để deploy thẳng từ thư mục local, Redis thêm bằng `railway add --database redis` và `REDIS_URL=${{Redis.REDIS_URL}}`. Kết quả `/ready` trả `{"status":"ready","redis":true}`.
