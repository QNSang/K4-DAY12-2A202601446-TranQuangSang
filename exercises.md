# Phiếu Phản Ánh - K4 Ngày 12

> Bài làm cá nhân. Các câu trả lời dưới đây dựa trên quá trình chạy test, Docker Compose, Railway và GitHub Actions của repo này.
>
> Họ và tên: Tran Quang Sang  
> Mã học viên: 2A202601446

---

### Câu 1 - Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Railway, nếu tôi quên set `API_TOKEN`, app sẽ lỗi ngay lúc startup thay vì âm thầm chạy với token mặc định như `"changeme"`. Như vậy tôi phát hiện lỗi trong lúc đang nhìn log deploy và sửa biến môi trường ngay. Nếu app vẫn chạy với token mặc định, một người ngoài có thể đoán được token, gọi `/chat`, dùng tài nguyên và làm phát sinh chi phí trước khi tôi biết có vấn đề.

---

### Câu 2 - Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi nêu hai việc bạn làm được với dòng log đó mà `print("đã trả lời xong")` không làm được.

> Ví dụ một dòng log:
>
> ```json
> {"event":"chat_completed","severity":"INFO","ts":"2026-08-10T08:00:00+00:00","client_id":"sv-test","prompt_tokens":4,"completion_tokens":42,"usd_cost":0.0000258}
> ```
>
> Với log JSON này tôi có thể lọc theo `event` hoặc `severity` để xem lỗi/sự kiện quan trọng, và có thể nhóm theo `client_id` hoặc cộng `usd_cost` để biết client nào dùng nhiều chi phí. Nếu chỉ `print("đã trả lời xong")` thì không có cấu trúc để máy lọc, thống kê hoặc cảnh báo tự động.

---

### Câu 3 - Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | khoảng 426.3 MB theo log build Railway cũ |
| Multi-stage | 270 MB theo `docker images` local |

> Phần chênh lệch chủ yếu là do bản multi-stage chỉ copy phần dependency đã cài và source cần chạy sang runtime image, không giữ lại toàn bộ context build không cần thiết. Ngoài ra runtime dùng `python:3.11-slim` thay vì image Python đầy đủ, nên bỏ bớt nhiều package hệ thống. `.dockerignore` cũng giúp không copy `.env`, `.git`, `.venv`, cache và file thừa vào image.

---

### Câu 4 - Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt `COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, các layer `COPY requirements.txt` và `RUN pip install --prefix=/install -r requirements.txt` được dùng lại từ cache nếu `requirements.txt` không đổi. Khi chỉ sửa `app/main.py`, Docker chỉ cần chạy lại các layer copy source như `COPY app ./app`, `COPY utils ./utils` và các layer sau đó ở runtime. Nếu đặt `COPY . .` trước `RUN pip install`, chỉ cần sửa một dòng code là layer copy context thay đổi, làm Docker phải chạy lại `pip install`, khiến build chậm hơn nhiều.

---

### Câu 5 - Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu app Python có lỗ hổng cho phép chạy lệnh trong container, attacker sẽ chạy lệnh với quyền của user đang chạy process. Nếu container chạy root, attacker có quyền root trong container; khi có thêm lỗi cấu hình volume, socket Docker hoặc kernel/container runtime, quyền này có thể làm thiệt hại lan sang host. Lệnh `USER appuser` cắt chuỗi đó bằng cách làm process uvicorn chạy bằng user thường, nên khi app bị khai thác thì attacker không có quyền root mặc định trong container.

---

### Câu 6 - Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả cùng một thông báo lỗi cho cả ba trường hợp thiếu header, sai scheme, sai token thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> Header `WWW-Authenticate: Bearer` là cách response 401 nói cho client biết endpoint này cần xác thực bằng Bearer token. Nếu thiếu header, client hoặc thư viện HTTP khó biết phải gửi token theo chuẩn nào. Tôi trả cùng một thông báo lỗi cho thiếu header, sai scheme và sai token để không tiết lộ thêm thông tin cho người đang dò API. Nếu nói rõ "scheme đúng nhưng token sai" hoặc "token thiếu", attacker có thêm tín hiệu để điều chỉnh cách tấn công.

---

### Câu 7 - Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn `min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Với `capacity=10`, client im lặng 10 phút vẫn chỉ gửi được 10 request liên tiếp trước khi bị 429, vì bucket tối đa chỉ chứa 10 token. Nếu bỏ `min(capacity, ...)`, sau 10 phút client có thể tích thêm `10 * 10 = 100` token, nên có thể gửi khoảng 100 request liên tiếp. Điều đó sai vì token bucket phải cho phép burst có giới hạn, không phải tích lũy vô hạn khi client im lặng lâu.

---

### Câu 8 - Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức $30/tháng, sự cố lúc 2h sáng có thể đốt hết tối đa $30 trước khi bị chặn, và client chỉ tự có lại ngân sách khi sang tháng mới hoặc khi mình can thiệp. Với hạn mức $1/ngày, thiệt hại tối đa trong ngày đó chỉ khoảng $1, và service tự hồi phục vào ngày UTC tiếp theo vì key chi phí theo dạng `spend:<client>:<YYYY-MM-DD>`. Hạn mức theo ngày giới hạn thiệt hại nhỏ hơn và hồi phục nhanh hơn.

---

### Câu 9 - `/healthz` khác `/readyz` (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm 3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/healthz` cũng kiểm tra Redis, khi Redis mất kết nối 30 giây thì cả 3 container đều bắt đầu trả health check lỗi. Orchestrator hiểu nhầm là cả 3 process bị hỏng, nên restart container. Trong lúc restart, request đang xử lý có thể bị cắt ngang và cụm mất thêm thời gian khởi động lại. Redis chỉ lỗi ngắn nhưng vì health check bị gộp sai vai trò, sự cố dependency nhỏ biến thành sự cố toàn cụm. Tách `/healthz` và `/readyz` giúp process vẫn sống, còn load balancer chỉ tạm ngừng gửi traffic khi app chưa ready.

---

### Câu 10 - Deploy thật (CP5)

Ghi lại một lỗi bạn gặp khi deploy lên cloud: thông báo lỗi là gì, bạn tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi tôi gặp trên Railway là:
>
> ```text
> Error: Invalid value for '--port': '$PORT' is not a valid integer.
> ```
>
> Tôi đọc log deploy và thấy `uvicorn` nhận literal string `$PORT` thay vì số port thật. Nguyên nhân là lệnh start trong `railway.toml` không được shell expand biến môi trường trước khi truyền cho uvicorn. Tôi sửa thành:
>
> ```toml
> startCommand = "sh -c 'uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}'"
> ```
>
> Sau đó commit, push và redeploy để Railway đọc lại lệnh start mới.
