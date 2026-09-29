# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: mỗi câu hỏi có một câu trả lời bằng lời của chính bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Quốc Huy  Mã học viên: 2A202602929

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu khi deploy một môi trường mới nhưng quên đặt `AGENT_API_KEY`, app sẽ dừng ngay lúc khởi động và ta sẽ biết cấu hình đang thiếu trước khi service nhận request. Nếu dùng mặc định `"changeme"`, thì app có thể vẫn chạy nhưng người khác biết khóa mặc định đó và gọi được API. Như vậy, lỗi cấu hình bị che đi và có thể dẫn tới truy cập ngoài ý muốn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T09:20:38.367544+00:00", "user_id": "exercise-log-demo", "tokens_in": 2, "tokens_out": 36, "cost_usd": 2.19e-05}
```

Từ user_id tôi lọc được request của cùng một người; từ cost_usd và số token tôi có thể cộng chi phí hoặc tìm request dùng nhiều token. Một dòng print chung chung không có dữ liệu để làm các việc đó.

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
| 1 stage (bản đầu) | 1.73 GB disk usage; 447 MB content size |
| Multi-stage | 271 MB disk usage; 63.9 MB content size |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Bản đầu dùng `python:3.11` đầy đủ, còn bản mới dùng `python:3.11-slim` và tách builder khỏi runtime. Phần giảm lớn nhất trong phép đo này đến từ base image gọn hơn; image cuối cũng không mang theo toàn bộ môi trường builder. Dockerfile cũ không cài compiler riêng nên không quy toàn bộ chênh lệch cho compiler.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Tôi đã thêm một comment vào app/main.py trong bản build thử. Log cho thấy WORKDIR, COPY requirements.txt, pip install và COPY --from=builder /install được dùng lại từ cache. COPY app phải chạy lại; những layer phía sau như COPY utils và tạo user appuser cũng chạy lại vì layer trước đã đổi. Nếu đặt COPY . . trước RUN pip install, thay đổi trong code sẽ làm layer COPY mất cache, kéo theo bước cài dependencies phải chạy lại.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu lỗ hổng cho phép kẻ tấn công chạy lệnh trong app, các lệnh đó có quyền của process trong container. Nếu process chạy bằng root, kẻ tấn công có thể sửa file hệ thống trong container hoặc đọc các thư mục được mount. Với mount nhạy cảm, Docker socket hoặc lỗi thoát container và ảnh hưởng còn có thể lan tới host. USER appuser khiến app chạy bằng quyền thường, nên kẻ tấn công không được root ngay cả khi chiếm được process; đây là cách giảm thiệt hại nhưng không thay thế hoàn toàn việc cô lập container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Với bộ đếm reset theo phút, người dùng có thể gửi 10 request sát cuối phút này rồi thêm 10 request ngay sau khi phút mới bắt đầu. Như vậy có thể có 20 request trong khoảng hai giây quanh ranh giới phút. Sliding window giữ các request trong một khoảng 60 giây liên tục nên vẫn tính cả hai nhóm và chặn khi vượt hạn mức.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn số request trong một khoảng thời gian, còn cost guard giới hạn chi phí cộng dồn trong tháng. Nếu user chưa gửi quá 10 request trong phút này nhưng ngân sách tháng gần hết và request tiếp theo sẽ vượt mức, cost guard phải chặn. Ngược lại, user còn ngân sách nhưng gửi dồn hơn 10 request trong một phút thì rate limit chặn dù chi phí chưa cao.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Redis mất kết nối thì cả ba container đều không truy cập được Redis. Nếu /health cũng phụ thuộc Redis, các probe sẽ lần lượt nhận trạng thái lỗi dù process vẫn sống. Sau đủ số lần probe thất bại, orchestrator có thể đánh dấu cả ba container unhealthy và restart chúng. Redis vẫn chưa hoạt động nên các container mới khởi động lại lại fail probe, tạo thành vòng lặp. Nếu tách hai endpoint, /health chỉ kiểm tra process; /ready báo chưa sẵn sàng để load balancer ngừng gửi traffic tới instance.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Tôi khởi chạy ba replica với Redis dùng chung rồi gọi lần lượt từng replica bằng cùng X-User-Id. Sáu response đều HTTP 200, history_length lần lượt là 0, 2, 4, 6, 8, 10. Mỗi request thêm một tin nhắn user và một tin nhắn assistant, nên lịch sử tăng hai mục; replica sau đọc được dữ liệu replica trước ghi vào Redis. Nếu mỗi replica dùng dict riêng và request luân phiên qua từng replica, mình dự kiến thấy 0, 0, 0, 2, 2, 2.
---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Khi chuẩn bị deploy, tôi đã gặp lỗi ở bước tạo token vì hệ thống yêu cầu xác minh tài khoản. Mỗi lần bấm liên kết xác minh, trang lại báo “Page Not Found”, nên tôi không thể lấy token để tiếp tục. Tôi đã kiểm tra lại và xác định lỗi nằm ở bước xác minh tài khoản, trước cả khi ứng dụng được deploy. Để hoàn tất lab, tôi chuyển sang Render, kết nối repository và cấu hình các biến môi trường cần thiết. Sau đó deploy thành công, và hai endpoint `/health` với `/ready` đều trả HTTP 200.
