# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lăng Nhật Minh  Mã học viên: MINHMAXXX

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống cứu bạn: Nếu bạn đưa ứng dụng lên production nhưng quên cung cấp giá trị cho biến `API_TOKEN`, việc thiếu giá trị mặc định sẽ làm ứng dụng lập tức văng lỗi và dừng hoạt động (crash) trong quá trình boot. Nếu có giá trị mặc định là `"changeme"`, ứng dụng vẫn chạy bình thường nhưng sử dụng API token mặc định mà mọi người đều biết, khiến ai cũng có thể truy cập hệ thống của bạn và bạn có thể bị lợi dụng tài nguyên.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được: `{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T10:30:00Z", "client_id": "c1", "prompt_tokens": 10, "completion_tokens": 20, "usd_cost": 0.001}`
> Hai việc làm được:
> 1. Đẩy log lên hệ thống giám sát (như Datadog hoặc Grafana) để tự động parse các trường và vẽ biểu đồ tổng chi phí (`usd_cost`) theo thời gian thực.
> 2. Tìm kiếm và filter siêu tốc: Dễ dàng viết câu truy vấn để lọc ra toàn bộ request thuộc về `client_id` cụ thể, hoặc những request có `usd_cost > 0.05`.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1000 MB |
| Multi-stage | ~150 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch là các thư viện compiler, package quản lý hệ thống, source code rác (như cache của pip, apt), và các toolchain dùng để build thư viện Python ở giai đoạn Builder. Nhờ Multi-stage, chúng không bị copy sang image cuối cùng (Production), giúp image gọn nhẹ và bảo mật hơn rất nhiều.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi sửa `app/main.py`: Các layer từ `FROM`, `WORKDIR`, `COPY requirements.txt .`, và `RUN pip install` sẽ được lấy hoàn toàn từ cache. Docker chỉ phải build lại layer từ đoạn `COPY app/ app/` trở về sau (tốn vài giây).
> Nếu đặt `COPY . .` trước `RUN pip install`: mỗi khi thay đổi `app/main.py`, layer `COPY . .` sẽ bị thay đổi mã hash và cache bị invalidate, kéo theo layer `RUN pip install` đứng sau cũng bị mất cache, ép Docker mất vài phút tải và cài lại toàn bộ các gói thư viện.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: 1. Có một lỗ hổng RCE trong code. 2. Kẻ tấn công lợi dụng RCE để chèn mã độc. 3. Vì container chạy quyền root, tiến trình mã độc cũng được thừa hưởng quyền root. 4. Kẻ tấn công lợi dụng quyền root để khai thác lỗi kernel, bẻ khoá cô lập của docker (container breakout) và chạy thoát ra máy host với quyền cao nhất.
> Lệnh `USER appuser` cắt đứt điều này ở bước 3, vì khi đó tiến trình ứng dụng chỉ chạy bằng quyền của `appuser` (bị giới hạn cực đoan). Dù chèn được mã, hacker cũng không có quyền thao tác trên hệ thống hay khai thác kernel.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> - Phải kèm header `WWW-Authenticate: Bearer` vì đó là tiêu chuẩn bắt buộc của HTTP (RFC 6750) để báo với client phương thức xác thực nào (ở đây là Bearer token) đang được máy chủ yêu cầu.
> - Trả cùng một thông báo lỗi vì: Việc chỉ ra chính xác điểm sai (sai scheme, thiếu header hay sai token) giúp kẻ tấn công thu hẹp phạm vi dò tìm (giảm entropy). Che giấu lỗi chi tiết làm cho quá trình đoán token/bruteforce khó khăn hơn.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Nó gửi được tối đa 10 request trước khi bị 429 vì xô chỉ chứa được cao nhất là 10 token.
> Nếu bỏ đoạn `min(capacity, ...)`: Con số đó sẽ thành 100 request (vì $10 \text{ token/phút} \times 10 \text{ phút} = 100 \text{ token}$). Khi đó, client im lặng tích luỹ lượng token vô hạn và có thể xả toàn bộ lượng request đó cùng một lúc, gây bạo tải (spike) làm sập hệ thống.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Hạn mức $30/tháng: Kẻ tấn công gọi liên tục từ 2h sáng có thể tiêu sạch toàn bộ $30 ngay lập tức, dịch vụ ngưng hoạt động cả tháng đó. Thiệt hại là $30.
> Hạn mức $1/ngày: Kẻ tấn công chỉ đốt tối đa $1 của ngày hôm đó và lập tức bị chặn lại (thiệt hại giảm 30 lần). Service tự động hồi phục vào 0:00 UTC của ngày hôm sau vì key ngân sách được tách biệt theo ngày, không cần sự can thiệp của con người.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis chết. Orchestrator (Docker/K8s) định kỳ gọi vào `/health` để check liveness và nhận về HTTP 503 (thay vì 200).
> 2. Trái với `/ready` (chỉ ngắt traffic), việc `/health` văng lỗi 503 làm orchestrator tưởng lầm bản thân container bị hỏng nghiêm trọng, và ra lệnh SIGKILL giết chết container ngay lập tức.
> 3. Toàn bộ 3 container liên tục bị kill và restart mà không có cơ hội được drain nhẹ nhàng, dẫn đến sụp đổ toàn bộ cluster chỉ vì 30s đứt kết nối mạng Redis tạm thời.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi gặp: Health check timeout trên render do binding sai port.
> Thông báo: Port binding failed or connection timed out after 300 seconds.
> Nguyên nhân: Trong Dockerfile, lệnh CMD gán hardcode `--port 8000` trong khi cloud sẽ ném port ngẫu nhiên vào biến `$PORT` nên app chạy ở port 8000 nhưng cloud route traffic vào port 12345 (ví dụ).
> Cách sửa: Đổi dòng `CMD` trong Dockerfile thành sử dụng shell mode để đọc biến `$PORT`: `CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`.
