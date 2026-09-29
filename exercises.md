# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

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

> Tôi đã thử khởi tạo `Settings` mà không nạp file `.env` và không cung cấp
> biến môi trường `AGENT_API_KEY`. Kết quả quan sát được là Pydantic phát sinh
> `ValidationError`, báo trường `agent_api_key` là bắt buộc, nên ứng dụng dừng
> trước khi có thể phục vụ request. Điều này được gọi là fail fast vì lỗi cấu
> hình được phát hiện ngay khi khởi động, thay vì để service chạy trong trạng
> thái cấu hình sai rồi mới phát hiện trên production. Trong tình huống deploy
> mà quên đặt secret, việc dừng sớm giúp tôi sửa cấu hình trước khi public API.
> Nếu secret có giá trị mặc định như `"changeme"`, service vẫn có thể khởi động
> và người khác có thể đoán hoặc dùng khóa mặc định để truy cập `/ask`, gây lạm
> dụng API và phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Khi chạy Uvicorn và gửi request thật tới `/ask`, tôi nhận được một dòng log:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:35:23.230066+00:00", "user_id": "cp4-smoke-2835e264ad1c4a6e81a410f708daa3da", "tokens_in": 7, "tokens_out": 46, "cost_usd": 2.865e-05}`.
> So với `print("đã trả lời xong")`, tôi có thể (1) lọc/đếm số lượt
> `ask_completed` của từng `user_id` trong một khoảng thời gian dựa vào
> `timestamp`, và (2) cộng `cost_usd` hoặc tổng token theo user để phát hiện
> mức sử dụng bất thường. JSON nằm trên một dòng stdout để hệ thống thu thập
> log nhận đúng một sự kiện, thay vì tách một object thành nhiều bản ghi.

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

> Tôi build bản một stage từ `python:3.11` và đo được 1.7 GB; bản multi-stage
> dùng `python:3.11-slim` và đo được 271 MB. Chênh lệch khoảng 1.43 GB, tức
> image multi-stage nhỏ hơn khoảng 84%. Phần lớn chênh lệch đến từ base image
> Python đầy đủ lớn hơn đáng kể so với `slim`. Bản runtime multi-stage cũng chỉ
> nhận các package đã cài từ builder và mã `app/`, `utils/`, không mang theo
> toàn bộ thư mục build. Cả hai phép đo dùng cùng máy và cùng dependency trong
> `requirements.txt`; đây là kích thước Docker hiển thị sau khi build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi thêm tạm một ký tự vào docstring trong `app/main.py` rồi chạy lại
> `docker build --progress=plain -t agent:cache-one-char .`. Trong log, các
> layer `COPY requirements.txt`, `RUN pip install`, `COPY --from=builder`
> đều hiện `CACHED`, vì file dependency và builder không thay đổi. Layer
> `COPY app ./app` phải chạy lại do source đã đổi; các layer filesystem phía
> sau như `COPY utils ./utils` và tạo user cũng chạy lại theo thứ tự layer.
> Khi `COPY . .` đặt trước `RUN pip install`, thay đổi bất kỳ file source nào
> cũng làm layer `COPY . .` đổi; do đó layer `pip install` phía sau bị vô hiệu
> cache và cài lại toàn bộ dependency dù `requirements.txt` không đổi. Đặt
> riêng `COPY requirements.txt` và cài dependency trước giúp giữ cache khi chỉ
> sửa mã nguồn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu ứng dụng Python có lỗ hổng cho phép chạy lệnh tùy ý, kẻ tấn công có thể
> chiếm quyền của tiến trình bên trong container. Nếu tiến trình chạy bằng
> root, kẻ đó có quyền cao nhất trong container, có thể đọc hoặc sửa file,
> cài công cụ và khai thác cấu hình hay lỗi khác để tìm cách thoát container;
> nếu việc thoát thành công, rủi ro với máy host sẽ lớn hơn. Lệnh `USER appuser`
> chạy tiến trình bằng tài khoản thường, nên giảm quyền và giới hạn tác động
> ban đầu của lỗ hổng. Đây là giảm thiểu rủi ro, không phải bảo đảm container
> không thể bị thoát hoặc host không thể bị ảnh hưởng.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với cách đếm theo phút đồng hồ, người dùng có thể gửi 10 request ở giây
> 10:00:59 và 10 request nữa ở giây 10:01:00: tổng cộng 20 request trong
> khoảng 2 giây, nhưng mỗi phút lịch vẫn chỉ ghi nhận 10 request. Cửa sổ trượt
> của bài lưu timestamp của từng request trong Redis Sorted Set. Trước mỗi lần
> kiểm tra, nó xóa các timestamp đã quá 60 giây rồi đếm phần còn lại; vì vậy
> lượt thứ 11 trong cùng cửa sổ bị trả 429, dù đồng hồ vừa sang phút mới.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tần suất gọi trong 60 giây, còn cost guard giới hạn tổng
> số tiền đã ghi nhận cho từng user trong tháng. Ví dụ, với hạn mức 10 lượt/phút
> và ngân sách 10 USD, một user chỉ gọi 1 lượt trong phút nhưng đã tiêu 10.01
> USD trong tháng: rate limit cho qua, cost guard trả 402. Ngược lại, user mới
> tiêu 0.01 USD nhưng đã gửi đủ 10 lượt trong 60 giây: lượt thứ 11 bị rate
> limit trả 429 dù vẫn còn ngân sách. Hai giới hạn giải quyết hai loại lạm dụng
> khác nhau, nên `/ask` kiểm tra cả hai trước khi gọi mock LLM.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp `/health` và `/ready` rồi bắt cả hai kiểm tra Redis, thứ tự sẽ là:
> (1) Redis mất kết nối; (2) cả ba container cùng trả 503 cho probe, dù tiến
> trình FastAPI vẫn sống; (3) load balancer có thể ngừng gửi request vào cả ba;
> (4) nếu nền tảng dùng kết quả đó làm liveness và đạt ngưỡng thất bại, nó còn
> khởi động lại các container, làm gián đoạn thêm các request đang xử lý mà
> không sửa được Redis. Khi Redis trở lại, các probe mới có thể báo 200 và
> traffic tiếp tục. Với cấu hình Docker Compose hiện tại, healthcheck chạy
> mỗi 10 giây và cần 5 lần thất bại; 30 giây chưa chắc đủ để đánh dấu
> `unhealthy`, và Docker Compose không tự restart chỉ vì healthcheck thất bại.
> Tách `/health` (process sống, vẫn 200) khỏi `/ready` (Redis chết, trả 503)
> tránh đánh đồng lỗi dependency với lỗi cần restart process.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi chạy ba container `agent` trên cùng mạng Compose và cùng Redis, rồi gọi
> `/ask` lần lượt vào từng container với một `X-User-Id`. Kết quả quan sát được
> của `history_length` là `0 → 2 → 4`: mỗi request ghi hai message (user và
> assistant), và container kế tiếp đọc được lịch sử do container trước ghi.
> Tôi dùng các container không publish cổng host vì `docker-compose.yml` đang
> cố định `8000:8000`, nên lệnh `docker compose up --scale agent=3` sẽ xung
> đột cổng. Nếu thay Redis bằng một dict Python trong từng process, mỗi
> container chỉ thấy lịch sử riêng của nó; khi lần lượt gọi ba container,
> kết quả sẽ là `0 → 0 → 0`, còn qua load balancer thì số này có thể nhảy
> không đều hoặc giảm khi request chuyển sang container khác.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *Câu trả lời của bạn*
