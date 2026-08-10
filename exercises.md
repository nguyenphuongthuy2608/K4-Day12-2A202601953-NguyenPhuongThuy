# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Phương Thuỳ  Mã học viên: 2A202601953

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Một tình huống cụ thể là khi tôi deploy lên Render nhưng quên khai báo
> `API_TOKEN` trong phần Environment. Vì `api_token` không có giá trị mặc định,
> ứng dụng dừng ngay lúc khởi động và log Render báo lỗi validation, nên tôi biết
> cấu hình còn thiếu trước khi service nhận traffic. Nếu dùng mặc định
> `"changeme"`, service vẫn có thể báo deploy thành công nhưng bất kỳ ai đoán được
> token này đều gọi `/chat`, làm phát sinh chi phí và dữ liệu rác. Như vậy, fail
> fast biến một lỗi cấu hình âm thầm thành lỗi rõ ràng tại thời điểm an toàn nhất.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON tôi thu được khi gọi `/chat` là:
>
> ```json
> {"event":"chat_completed","severity":"INFO","ts":"2026-08-10T10:02:07.469793+00:00","client_id":"exercises-log","prompt_tokens":6,"completion_tokens":40,"usd_cost":2.49e-05}
> ```
>
> Từ dòng này, tôi có thể lọc và đếm số sự kiện `chat_completed` theo thời gian
> hoặc theo `client_id` để theo dõi lưu lượng của từng client. Tôi cũng có thể
> cộng trường `usd_cost` và số token để lập cảnh báo chi phí hoặc tìm client tiêu
> nhiều nhất. Chuỗi `print("đã trả lời xong")` không có trường dữ liệu ổn định nên
> máy khó truy vấn, tổng hợp và cảnh báo tự động.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1.74 GB (khoảng 1,740 MB) |
| Multi-stage | 296 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build lại Dockerfile một-stage ban đầu với `python:3.11` và đo được 1.74
> GB; image multi-stage hiện tại là 296 MB. Phần chênh lệch chủ yếu là các lớp
> hệ điều hành và công cụ phát triển có trong image Python đầy đủ, cùng những thứ
> chỉ cần ở giai đoạn cài/biên dịch dependency. Bản multi-stage dùng
> `python:3.11-slim` cho runtime và chỉ chép kết quả đã cài từ builder, nên không
> mang toàn bộ môi trường build sang image chạy production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi tôi chỉ sửa `app/main.py`, các layer base image, `WORKDIR`,
> `COPY requirements.txt`, `RUN pip install` trong builder, tạo `appuser` và
> `COPY --from=builder` vẫn dùng cache vì requirements không đổi. Cache bị vô
> hiệu từ `COPY app ./app`; layer này và các layer đứng sau nó phải được tạo lại,
> nhưng không phải tải/cài lại thư viện. Nếu đặt `COPY . .` trước
> `RUN pip install`, chỉ một ký tự trong `main.py` cũng làm layer copy thay đổi,
> kéo theo `pip install` chạy lại dù `requirements.txt` hoàn toàn không đổi. Build
> vì vậy chậm hơn và tốn mạng không cần thiết.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Giả sử code Python có lỗ hổng thực thi lệnh từ xa, kẻ tấn công trước tiên lấy
> được shell trong container. Nếu process chạy root, shell đó cũng có UID 0 trong
> container, nên kẻ tấn công dễ sửa file hệ thống, đọc tài nguyên được mount hoặc
> lợi dụng cấu hình đặc quyền, Docker socket hay một lỗ hổng kernel/container
> runtime để thoát ra host. Khi đã thoát khỏi container với quyền cao, họ có thể
> chiếm máy host và các container khác. `USER appuser` cắt chuỗi này ngay sau bước
> chiếm process: shell chỉ có quyền của user thường, vì vậy attacker phải tìm thêm
> một lỗ hổng nâng quyền trước khi thực hiện các bước nguy hiểm tiếp theo. Nó
> không bảo đảm tuyệt đối nhưng giảm mạnh phạm vi thiệt hại.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` cho client biết response 401 yêu cầu cơ chế xác thực
> Bearer theo chuẩn HTTP/RFC 6750; client hoặc công cụ API nhờ đó biết cần gửi lại
> header `Authorization: Bearer <token>`. Tôi dùng cùng một thông báo
> `invalid or missing bearer token` cho trường hợp thiếu header, sai scheme và sai
> token để không tiết lộ bước kiểm tra nào đã đúng. Nếu server trả lời chi tiết
> “scheme đúng nhưng token sai”, người dò token sẽ có thêm tín hiệu để thu hẹp
> từng bước tấn công. Chi tiết chẩn đoán nên nằm trong log nội bộ, không nằm trong
> response công khai.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Với `capacity=10`, dù im lặng 10 phút thì xô chỉ được tích tối đa 10 token, nên
> client gửi liên tiếp được 10 request; request thứ 11 bị 429 nếu bỏ qua lượng nạp
> rất nhỏ trong lúc gửi. Tốc độ nạp là 10 token/phút, nên 10 phút tạo thêm 100
> token. Nếu bỏ `min(capacity, ...)` và trước lúc chờ xô đang đầy 10 token, số dư
> sẽ thành 10 + 100 = 110 token, tức client có thể bắn khoảng 110 request trước
> khi bị chặn. Nếu xô bắt đầu từ 0 thì con số là 100. Đây là lý do bắt buộc phải
> chặn trên ở `capacity`.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức 30 USD/tháng, nếu sự cố bắt đầu lúc 2 giờ sáng khi tháng chưa tiêu
> gì, một client có thể gây thiệt hại tới 30 USD trước khi bị chặn; sau đó service
> chỉ tự mở lại khi bước sang tháng mới. Với hạn mức 1 USD/ngày, thiệt hại của
> ngày xảy ra sự cố bị giới hạn ở 1 USD. Client bị chặn trong phần còn lại của
> ngày và tự được phục vụ lại khi key/ngân sách chuyển sang ngày mới. Vì vậy hạn
> mức ngày giới hạn “bán kính thiệt hại” của sự cố tốt hơn, dù tổng danh nghĩa
> vẫn tương đương khoảng 30 USD trong một tháng.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp liveness và readiness rồi cho endpoint chung kiểm tra Redis, trình tự
> xảy ra sẽ là: Redis mất kết nối; probe của cả ba container cùng trả 503;
> orchestrator hiểu nhầm rằng cả ba process đã chết và lần lượt restart chúng.
> Trong lúc Redis vẫn mất 30 giây, các container vừa khởi động lại tiếp tục probe
> thất bại và có thể bị restart thêm lần nữa. Load balancer không còn instance ổn
> định để nhận traffic, nên một sự cố Redis ngắn biến thành gián đoạn toàn cụm và
> còn tạo thêm tải khi ba container cùng khởi động. Tách `/healthz` giúp process
> vẫn được coi là sống, còn `/readyz` chỉ rút instance khỏi traffic mà không
> restart vô ích.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi tôi thực sự gặp khi kiểm tra bản Render là mở Public URL gốc và nhận
> `{"detail":"Not Found"}` (HTTP 404). Tôi kiểm tra `app/main.py` và tài liệu lab,
> thấy ứng dụng chỉ khai báo `GET /healthz`, `GET /readyz` và `POST /chat`, không
> có route `GET /`. Vì vậy nguyên nhân không phải deploy hỏng mà là tôi gọi sai
> endpoint. Tôi sửa cách kiểm tra bằng cách gọi `/healthz` và `/readyz`; cả hai
> trả 200. Với `/chat`, tôi dùng POST: không truyền token nhận 401 và truyền đúng
> Bearer token trên Render nhận 200. Tôi không thêm route `/` vì checkpoint không
> yêu cầu route đó.
