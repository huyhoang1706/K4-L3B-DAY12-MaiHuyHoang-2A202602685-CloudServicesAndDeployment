# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Mai Huy Hoàng  Mã học viên: 2A202602685

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ, tôi deploy service nhưng quên set `AGENT_API_KEY` trên Render. Nếu khóa
> có mặc định là `"changeme"`, app vẫn báo khởi động thành công và người lạ có
> thể đoán khóa để gọi `/ask`, làm phát sinh chi phí. Khi trường này bắt buộc,
> app dừng ngay lúc đọc cấu hình. Tôi thấy lỗi trước khi service nhận traffic và
> có thể bổ sung secret trong dashboard rồi deploy lại.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi lấy từ stack đang chạy là:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:55:07.528756+00:00", "user_id": "exercise-scale-check", "tokens_in": 88, "tokens_out": 44, "cost_usd": 3.96e-05}
> ```
>
> Từ dòng này, tôi có thể lọc tất cả event `ask_completed` của một `user_id` để
> điều tra request, đồng thời cộng `tokens_in`, `tokens_out` và `cost_usd` để làm
> dashboard hoặc cảnh báo chi phí. Chuỗi `print("đã trả lời xong")` không có tên
> trường ổn định, timestamp hay số liệu để máy lọc và tổng hợp.

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
| 1 stage (bản đầu) | 1.17 GB |
| Multi-stage | 209 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build lại hai image và đọc kích thước bằng `docker images`: bản một-stage
> dùng `python:3.11` là 1.17 GB, còn bản multi-stage dùng `python:3.11-slim` là
> 209 MB, giảm khoảng 961 MB. Phần chênh lệch chủ yếu đến từ base image Python
> đầy đủ chứa nhiều gói hệ điều hành và công cụ không cần lúc chạy. Với
> multi-stage, môi trường cài dependency được tạo ở builder; image cuối chỉ nhận
> virtualenv, source cần thiết và base slim, không mang cả builder sang runtime.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer lấy base image, `COPY requirements.txt`
> và cài dependency trong builder vẫn được lấy từ cache vì requirements không
> đổi. Ở runtime, layer copy virtualenv vẫn được cache; `COPY app/` và các layer
> đứng sau nó phải tạo lại. Nếu đặt `COPY . .` trước `RUN pip install`, một thay
> đổi nhỏ trong source cũng làm layer copy đổi, kéo theo `pip install` chạy lại
> dù dependency không thay đổi. Build vì thế chậm hơn và tải/cài package thừa.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng thực thi lệnh, kẻ tấn công có thể chạy lệnh với
> quyền của process trong container. Khi process là root, họ có quyền cao nhất
> bên trong container, dễ đọc secret, sửa file hoặc lợi dụng volume/Docker socket
> được mount; nếu có thêm cấu hình nguy hiểm hay lỗ hổng kernel, từ đó có thể tác
> động tới host. `USER app` cắt chuỗi ở bước đầu: mã bị chiếm quyền chỉ có UID
> thường, không thể tự do sửa file hệ thống hay thực hiện thao tác cần root. Đây
> là giảm quyền, không thay thế việc vá lỗ hổng và giới hạn volume/capability.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong khoảng 2 giây: gửi 10 request ở
> cuối phút, ví dụ từ `10:00:59`, rồi gửi tiếp 10 request ngay sau khi bộ đếm
> reset ở `10:01:00`. Mỗi phút đồng hồ vẫn chỉ ghi nhận 10 request, nhưng thực tế
> 20 request đã dồn vào một khoảng rất ngắn. Sliding window 60 giây vẫn nhìn thấy
> cả hai nhóm nên chặn nhóm thứ hai.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ gọi trong một cửa sổ ngắn, còn cost guard giới hạn
> tổng số tiền theo user trong cả tháng. Một user gửi ít request nhưng mỗi prompt
> rất dài hoặc dùng nhiều token vẫn qua rate limit, trong khi cost guard phải
> chặn vì vượt ngân sách. Ngược lại, một bot gửi 11 câu hỏi rất ngắn trong một
> phút có thể chưa tốn gần hết ngân sách tháng nhưng request thứ 11 vẫn bị rate
> limit chặn để bảo vệ tải hệ thống.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối làm endpoint gộp trả 503 trên cả ba container. Orchestrator
> hiểu nhầm đây là lỗi liveness nên lần lượt restart cả ba process. Redis vẫn
> chưa phục hồi nên các container vừa khởi động lại tiếp tục trả 503 và có thể
> rơi vào vòng lặp restart, làm mất toàn bộ năng lực phục vụ dù process Python
> vẫn sống. Khi tách endpoint, `/health` vẫn trả 200 để tránh restart; `/ready`
> trả 503 để load balancer tạm ngừng gửi request. Redis phục hồi thì readiness
> tự trở lại 200 mà không cần khởi động lại cả cụm.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi chạy 3 replica phía sau Nginx và gọi `/ask` năm lần với cùng user. Giá trị
> `history_length` quan sát được là `0, 2, 4, 6, 8`; mỗi lượt thêm một message
> user và một message assistant, bất kể request rơi vào container nào. Nếu dùng
> dict Python, mỗi replica có một bản history riêng nên kết quả có thể thành
> `0, 0, 0, 2, 2` tùy cách Nginx phân phối, không tăng đều. Khi một replica
> restart, phần history nằm trong dict của replica đó còn trở về 0.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lần deploy thật lên Render của tôi hiện chạy thành công; lỗi triển khai có log
> rõ nhất tôi gặp khi kiểm thử mô hình nhiều instance là
> `Bind for 0.0.0.0:8000 failed: port is already allocated`. Tôi chạy
> `docker compose ps` và thấy một agent đã giữ mapping `8000:8000`, nên các
> replica sau không thể dùng lại cùng host port. Tôi sửa Compose để agent chỉ
> `expose` cổng 8000 trong mạng nội bộ, thêm Nginx làm load balancer duy nhất map
> `8000:80`, rồi chạy lại `docker compose up -d --scale agent=3`. Kết quả là cả
> ba agent đều healthy và gọi `/health` qua Nginx trả 200.
