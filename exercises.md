# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay phần giữ chỗ dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Trọng Bình  Mã học viên: L3B202602855

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy, nếu quên tạo `AGENT_API_KEY` mà ứng dụng vẫn dùng khóa mặc định
> `"changeme"`, bất kỳ ai đoán được giá trị này đều có thể gọi `/ask` và làm phát
> sinh chi phí. Với trường bắt buộc, tiến trình dừng ngay lúc khởi động và Render
> báo lỗi cấu hình; tôi phát hiện thiếu secret trước khi service nhận traffic,
> thay vì chỉ nhận ra sau khi endpoint công khai đã bị sử dụng sai.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng tôi thu được là:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:46:07.271632+00:00", "user_id": "sv01", "tokens_in": 12, "tokens_out": 9, "cost_usd": 2.1e-05}`.
> Từ log JSON này, hệ thống log có thể (1) lọc hoặc đếm riêng event
> `ask_completed` theo `user_id`, và (2) cộng `cost_usd`, theo dõi token rồi đặt
> cảnh báo theo thời gian. Chuỗi `print("đã trả lời xong")` không có field ổn định
> để máy truy vấn hay tổng hợp hai thông tin đó.

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
| 1 stage (bản đầu) | 431.8 MB (431,757,570 byte) |
| Multi-stage | 63.9 MB (63,863,717 byte) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build lại cả hai image và lấy số byte bằng `docker image inspect`. Bản
> 1-stage dùng image `python:3.11` đầy đủ nên mang theo nhiều gói hệ điều hành và
> công cụ không cần cho lúc chạy. Bản multi-stage dùng `python:3.11-slim`; môi
> trường cài đặt chỉ nằm ở stage `builder`, còn stage `runtime` chỉ nhận các
> dependency đã cài cùng mã nguồn. Vì vậy image cuối nhỏ hơn khoảng 368 MB và
> không mang toàn bộ nội dung trung gian của builder.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ mã nguồn thay đổi, log build của tôi cho thấy các layer base image,
> `COPY requirements.txt`, `RUN pip install` và `COPY --from=builder /install`
> đều dùng `CACHED`. Các layer `COPY app`, `COPY utils` và các layer runtime phía
> sau điểm thay đổi phải được xét/chạy lại. Nếu đặt `COPY . .` trước
> `RUN pip install`, chỉ một ký tự trong `app/main.py` cũng làm layer copy đổi,
> khiến Docker mất cache của mọi layer sau nó và cài lại toàn bộ dependency dù
> `requirements.txt` không thay đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi rủi ro là: lỗ hổng Python cho phép thực thi lệnh trong container; tiến
> trình đang là root nên mã độc có toàn quyền root trong container; nếu container
> còn được cấp quyền quá rộng, mount Docker socket/thư mục nhạy cảm, hoặc có lỗ
> hổng thoát container ở kernel/runtime, kẻ tấn công có thể tác động tới host với
> quyền cao. `USER appuser` cắt chuỗi ngay sau bước chiếm quyền thực thi: mã độc
> chỉ chạy với UID 10001, không thể tự ý sửa file hệ thống hay làm tác vụ cần root
> trong container. Đây là giảm thiểu thiệt hại, không thay thế việc bỏ privileged
> mode và các mount nguy hiểm.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa **20 request trong 2 giây**: gửi 10 request ở
> giây 59 của phút trước, rồi gửi tiếp 10 request ở giây 00 của phút sau. Bộ đếm
> theo phút đã reset ở ranh giới đó nên mỗi phút riêng vẫn chỉ thấy 10 request.
> Sliding window nhìn lại đúng 60 giây nên sẽ thấy cả hai đợt và chặn đợt vượt
> hạn mức.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ/số request trong cửa sổ 60 giây, còn cost guard giới
> hạn tổng tiền của từng user trong cả tháng UTC. Ví dụ user đã tiêu hết 10 USD,
> sau một giờ không gọi gì họ gửi một request: rate limit cho qua nhưng cost guard
> trả 402. Ngược lại, user mới chỉ tiêu vài cent nhưng gửi request rẻ thứ 11 trong
> 60 giây: ngân sách vẫn còn nhưng rate limiter trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu endpoint chung kiểm tra Redis và được dùng làm liveness: (1) Redis mất kết
> nối; (2) probe của cả ba container cùng trả 503; (3) orchestrator đánh dấu cả ba
> là unhealthy và lần lượt restart chúng; (4) Redis vẫn đang lỗi nên các container
> vừa lên lại tiếp tục trả 503; (5) cụm rơi vào vòng lặp restart và request đang xử
> lý có thể bị gián đoạn. Khi tách probe, `/health` vẫn 200 nên process không bị
> restart oan, còn `/ready` trả 503 để load balancer tạm ngừng gửi traffic; Redis
> hồi phục thì instance tự sẵn sàng lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, cùng một `X-User-Id` luôn đọc được lịch sử do bất kỳ
> instance nào ghi nên `history_length` tăng theo 0, 2, 4, 6... sau mỗi lượt hỏi
> (mỗi lượt thêm một message user và một message assistant), tối đa 20 message.
> Nếu dùng dict Python, mỗi container có một bản lịch sử riêng. Khi load balancer
> phân phối request qua ba container, tôi sẽ thấy các dãy độc lập như 0, 0, 0,
> 2, 2, 2... hoặc số đang lớn lại giảm khi request chuyển instance; restart một
> instance còn làm phần lịch sử của instance đó trở về 0.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi mở trực tiếp domain Render, tôi gặp `{"detail":"Not Found"}`. Ban đầu tôi
> tưởng web service deploy lỗi, nhưng kiểm tra các route trong `app/main.py` và gọi
> lần lượt `/health`, `/ready` cho kết quả 200; POST `/ask` không có key cũng trả
> đúng 401. Nguyên nhân là ứng dụng không khai báo route `/`, không phải build hay
> Render bị lỗi. Tôi sửa cách kiểm tra bằng URL
> `https://day12-agent-dl0f.onrender.com/health` (và lưu ảnh endpoint này), thay vì
> dùng domain gốc; lab không yêu cầu phải thêm trang chủ.
