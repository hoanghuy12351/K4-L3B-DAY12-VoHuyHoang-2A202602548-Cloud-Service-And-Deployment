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

> Log tôi quan sát được khi gọi `log_event` trực tiếp là:
> `{"event": "cp1_test", "level": "warning", "timestamp": "2026-09-29T03:41:18.805427+00:00", "user_id": "sv01", "message": "Kiem tra structured log"}`.
> Dòng log gồm các trường `event`, `level`, `timestamp`, `user_id` và `message`.
> So với `print("đã trả lời xong")`, JSON log cho phép tôi lọc theo mức
> `warning` và nhóm hoặc đếm sự kiện theo `event` hay `user_id`; nó cũng cho
> phép hệ thống giám sát đọc chính xác thời gian xảy ra sự kiện. Yêu cầu một
> dòng quan trọng vì nền tảng cloud thường coi mỗi dòng stdout là một bản ghi;
> nếu một JSON bị tách thành nhiều dòng thì bộ thu thập log có thể coi chúng là
> nhiều sự kiện hỏng. Từ dòng log này, tôi có thể tạo truy vấn đếm số cảnh báo
> theo `event` trong 5 phút gần nhất, hoặc cảnh báo khi một `user_id` tạo quá
> nhiều sự kiện `warning`. Ở CP1 tôi chưa gọi được `/ask` vì endpoint đó còn
> phụ thuộc các TODO của CP3/CP4; sau khi hoàn thành các checkpoint đó, tôi sẽ
> thay dòng thử nghiệm này bằng log `ask_completed` thu được từ request thật.

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

> *Câu trả lời của bạn*

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> *Câu trả lời của bạn*

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> *Câu trả lời của bạn*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> *Câu trả lời của bạn*

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *Câu trả lời của bạn*
