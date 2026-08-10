# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu deploy thiếu `AGENT_API_KEY`, ứng dụng fail ngay khi khởi động thay vì chạy
với khóa `changeme`. Nhờ vậy health check báo lỗi trong lúc deploy và mình sửa
được cấu hình trước khi người lạ gọi API hoặc phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Ví dụ: `{"event":"ask_completed","level":"info","timestamp":"2026-08-10T09:30:00+00:00","user_id":"sv01","cost_usd":0.0001}`.
Từ đó có thể lọc theo user để tìm người tiêu nhiều nhất và tính tỷ lệ lỗi/chi
phí theo khoảng thời gian; một câu `print` không có schema để máy lọc và cộng.

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

Sau khi build thực tế, bản multi-stage thường nhỏ hơn vì runtime chỉ chứa
Python slim, thư viện runtime và source; các file cache pip, công cụ build và
dependency trung gian chỉ tồn tại ở stage `builder`. Số MB cụ thể phụ thuộc
Docker cache và phiên bản package, nên cần ghi lại output `docker images` của
máy mình thay vì đoán.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi chỉ sửa `app/main.py`, các layer copy `requirements.txt` và `pip install`
được dùng lại; layer copy source và các layer sau đó phải chạy lại. Nếu
`COPY . .` đứng trước `pip install`, mọi thay đổi source làm layer đó đổi và
Docker phải cài lại toàn bộ dependency, khiến build chậm hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Lỗ hổng có thể cho phép tiến trình trong container thực thi lệnh. Nếu tiến
trình là root, kẻ tấn công có quyền cao trong container và có thể khai thác
thêm cấu hình/runtime để tác động tới host. `USER appuser` chạy app với UID
không đặc quyền, nên giảm quyền của bước đầu tiên và giới hạn thiệt hại.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Với hạn mức 10/phút, cách đếm theo phút đồng hồ cho phép 20 request trong 2
giây: gửi 10 request ở 10:00:59 rồi 10 request nữa ở 10:01:00 hoặc 10:01:01.
Hai nhóm thuộc hai phút khác nhau dù thời gian thực chỉ khoảng hai giây;
sliding window loại được khe hở này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn số lần gọi trong 60 giây, còn cost guard cộng số USD theo
tháng cho từng user. Một request ít lần nhưng prompt/response rất lớn có thể
qua rate limit nhưng bị cost guard chặn. Ngược lại, nhiều request nhỏ có thể
chạm rate limit dù tổng chi phí tháng vẫn còn thấp; khi đó rate limiter chặn
trước và cost guard chưa cần chặn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu `/health` cũng ping Redis, Redis mất kết nối thì cả ba container đều báo
unhealthy; load balancer ngừng gửi traffic, rồi orchestrator có thể restart
cả ba dù process vẫn sống. Khi Redis hồi phục, chúng phải khởi động lại/đợi
health check. Tách `/health` (liveness, không dependency) và `/ready`
(readiness, có ping Redis) giúp chỉ ngừng nhận traffic khi dependency chưa sẵn
sàng, không biến lỗi Redis ngắn thành restart dây chuyền.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis, history tăng đúng theo các lượt trước dù request đi qua instance
nào: request đầu có `history_length` 0, request kế tiếp 2, rồi 4. Nếu dùng
dict Python, mỗi instance có bản sao riêng; khi load balancer đổi instance,
history có thể quay về 0 hoặc chỉ phản ánh các request từng vào instance đó.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Một lỗi thường gặp là readiness timeout vì đặt `REDIS_URL=redis://localhost:6379`
trên cloud. Mình đối chiếu log với cấu hình và thấy `localhost` trỏ vào web
service chứ không phải Redis. Sửa bằng connection string của Redis add-on
(hoặc `fromService` trên Render), rồi kiểm tra lại `/ready` đến khi trả 200.
