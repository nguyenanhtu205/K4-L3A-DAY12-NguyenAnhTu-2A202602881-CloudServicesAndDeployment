# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng giữ chỗ bên dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Anh Tú  Mã học viên: 2A202602881

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu quên cấu hình `AGENT_API_KEY` trên Render, app sẽ báo lỗi thiếu trường cấu hình ngay khi khởi động và deploy sẽ fail. Nếu dùng mặc định yếu như `changeme`, service vẫn chạy nhưng người khác có thể đoán khóa rồi gọi `/ask`, làm cạn hạn mức hoặc ngân sách trước khi mình phát hiện.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Khi gọi `/ask` cục bộ với mock LLM, mình nhận được log: `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T10:53:10.365311+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 34, "cost_usd": 2.085e-05}`. Từ JSON này có thể lọc request của một `user_id` và cộng `cost_usd` để tìm người dùng tốn nhiều chi phí; cũng có thể đếm sự kiện theo `event`/`level` để theo dõi hoạt động và lỗi. `print("đã trả lời xong")` không có các trường dữ liệu để lọc và tổng hợp như vậy.

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
| 1 stage (bản đầu) | 1.73 GB (`agent:single`) |
| Multi-stage | 271 MB (`day12-agent:cp2-test`) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Theo cùng lệnh `docker images --format`, image một stage là 1.73 GB còn multi-stage là 271 MB, chênh xấp xỉ 1.46 GB. Bản một stage dùng base `python:3.11` đầy đủ và giữ kết quả cài dependency cùng cache trong image. Bản multi-stage dùng `python:3.11-slim`; stage cuối chỉ nhận dependency đã cài và source cần chạy, không mang theo phần dư của builder.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Trong Dockerfile hiện tại, khi sửa `app/main.py`, các layer từ `COPY requirements.txt` và `pip install` ở stage builder vẫn được cache nếu requirements không đổi. Các layer ở runtime từ `COPY app ./app` trở đi phải chạy lại. Nếu `COPY . .` đứng trước `pip install`, thay đổi source cũng làm mất cache của layer sau đó, khiến pip phải cài lại dependency dù requirements không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Nếu app bị khai thác khi container chạy bằng root, tiến trình bị chiếm quyền có thể dùng quyền root trong container để đọc/sửa file và khai thác cấu hình container yếu để tìm đường sang host. `USER 10001` chạy app dưới tài khoản thường, nên cắt bước cấp quyền root bên trong container và giới hạn tác động. Nó giảm rủi ro nhưng không tự ngăn được mọi kiểu container escape.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Với giới hạn 10 request mỗi phút theo phút đồng hồ, có thể gửi tối đa 20 request trong khoảng hai giây quanh ranh giới phút: 10 request ngay trước khi bộ đếm reset, rồi 10 request ngay sau reset. Sliding window 60 giây sẽ tính cả hai nhóm nếu chúng nằm trong cùng cửa sổ và chặn phần vượt hạn mức.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn số request trong một khoảng thời gian; cost guard giới hạn tổng tiền theo user trong tháng. Rate limit có thể cho qua một request nếu user chưa gửi quá nhanh, nhưng cost guard chặn khi chi phí đã ghi nhận vượt ngân sách tháng. Ngược lại, một user còn nhiều ngân sách nhưng gửi quá 10 request trong một phút sẽ bị rate limit chặn dù tổng chi phí còn thấp.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Khi Redis mất kết nối, cả ba container đều không ping được Redis nên probe gộp trả lỗi. Orchestrator hiểu cả ba container không khỏe và bắt đầu restart chúng; các lần restart không sửa được Redis, nên năng lực phục vụ giảm trong lúc Redis vẫn lỗi. Khi Redis kết nối lại, các container còn đang khởi động phải qua health check rồi mới nhận traffic. Tách `/health` khỏi Redis tránh restart process chỉ vì dependency tạm thời gặp sự cố; `/ready` báo instance ngừng nhận traffic trong thời gian đó.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis dùng chung, request đầu thường trả `history_length: 0`; request kế tiếp trả `2` vì lịch sử đã có message user và assistant của lượt trước. Nếu dùng dict trong RAM, mỗi instance có bản history riêng: request đi vào instance mới có thể lại thấy `0`, còn request quay về instance cũ thì thấy số lớn hơn. Vì vậy con số có thể dao động tùy instance nhận request.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Trong lần kiểm tra sau deploy, lệnh `/ask` đầu tiên trả `422` với thông báo JSON decode error. Mình xem response và nhận ra lệnh gửi từ Command Prompt có body JSON/URL bị dán sai định dạng. Sau khi dùng URL thuần và escape dấu ngoặc kép cho đúng cú pháp `cmd.exe`, request không có API key trả `401` như mong đợi. Đây là lỗi của lệnh kiểm tra, không phải lỗi deploy của service.
