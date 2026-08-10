# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `>Câu trả lời của bạn` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Văn Đại  Mã học viên: 2A202601245

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để mặc định là "changeme", khi đưa app lên production mà quên cài đặt biến môi trường, ứng dụng vẫn khởi động thành công. Hacker có thể dễ dàng dò ra và gọi API thoải mái bằng khóa "changeme", dẫn đến rò rỉ dữ liệu hoặc cạn kiệt tiền API mà đội ngũ dev không hề hay biết. Việc Fail-fast (chết ngay) buộc dev phải khai báo đúng key mới chạy được, ngăn chặn thảm họa bảo mật này từ trứng nước.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> {"level": "INFO", "timestamp": "2026-08-10T05:01:09Z", "message": "Q&A request", "user_id": "sv-test", "cost_usd": 3.315e-05, "tokens_in": 41, "tokens_out": 45}
> Hai việc làm được:
> 1. Dùng các hệ thống (ELK, Datadog) để tính tổng (sum) lượng tiền API (cost_usd) hoặc số token tiêu thụ trong tháng dễ dàng vì JSON đã định dạng sẵn thành cấu trúc số học (không cần parse chữ phiền phức).
> 2. Có thể dễ dàng truy vấn/filter chính xác toàn bộ cuộc trò chuyện của riêng một user bất kỳ bằng bộ lọc `user_id == "sv-test"` nhờ các trường dữ liệu tách biệt.

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
| 1 stage (bản đầu) | ~ 1 GB |
| Multi-stage | ~ 164 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch dung lượng bị loại bỏ gồm: Cache tải thư viện của Pip, các trình biên dịch mã nguồn C/C++ (như gcc, build-essential), và các file source code thừa sinh ra trong quá trình build các thư viện Python. Stage 2 chỉ việc copy thẳng bộ runtime đã build sẵn sang nên cực kỳ nhỏ gọn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi sửa `app/main.py`, các layer từ lệnh `FROM`, `WORKDIR`, `COPY requirements.txt` và đặc biệt là lệnh `RUN pip install` vẫn được tái sử dụng (CACHE). Chỉ có layer `COPY app/ app/` trở đi bị chạy lại.
> Nếu đặt `COPY . .` lên trước `RUN pip install`, chỉ một thay đổi nhỏ ở code (như sửa main.py) cũng làm layer COPY bị đánh dấu thay đổi. Docker sẽ hủy bỏ cache của toàn bộ các layer phía dưới nó, dẫn đến việc `RUN pip install` phải tải lại toàn bộ các thư viện Python rất tốn thời gian.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Kẻ tấn công lợi dụng lỗi (như RCE qua Pickle) để thực thi mã độc -> Cấu hình mặc định làm đoạn mã độc này chạy dưới quyền root trong container -> Lợi dụng việc container dùng chung kernel với máy Host, chúng khai thác lỗi leo thang đặc quyền để thoát ra (Container Escape) và chiếm quyền cao nhất trên Host.
> Lệnh `USER appuser` giới hạn quyền: Kẻ tấn công chỉ chiếm được quyền user thường (appuser). Chúng không thể xem /etc/shadow, không thể chỉnh sửa file hệ thống hay dùng các lệnh đặc quyền để thoát ra ngoài Host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Một người dùng có thể lách luật gửi được 20 request trong 2 giây.
> Giải thích: Với Fixed window (reset lúc giây 00), kẻ tấn công gửi 10 request vào lúc 23:59:59 (dùng hết hạn mức phút cũ). Ngay 1 giây sau, lúc 00:00:00 (bước sang phút mới, counter bị reset về 0), chúng lập tức gửi tiếp 10 request nữa. Kết quả là hệ thống phải gánh 20 request dồn dập chỉ trong 2 giây.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - Rate limit (chặn Spam/DDoS): Giới hạn theo tốc độ (req/phút).
> - Cost guard (chặn Cạn kiệt tài chính): Giới hạn theo tổng ngân sách ($/tháng).
> 
> Tình huống Rate limit cho qua, Cost guard chặn: Bạn hỏi tà tà mỗi phút 1 câu suốt tháng. Rate limit (10 câu/phút) luôn pass, nhưng đến cuối tháng tổng tiền token đã vượt $10, lúc này Cost guard sẽ chặn.
> Tình huống Rate limit chặn, Cost guard pass: Mới đầu tháng (tiêu 0$), bạn nện 20 request vào API trong 10 giây. Cost guard thấy bạn mới xài 0.001$ nên vẫn cho pass, nhưng Rate limit sẽ chặn 10 request cuối vì đã vượt 10 req/phút.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối.
> 2. Orchestrator (K8s/Docker) ping /health thấy kết quả trả về 503 thay vì 200.
> 3. K8s kết luận Container đã "bị lỗi hỏng" -> Quyết định Kill (tiêu diệt) container đó.
> 4. K8s khởi tạo container mới bù vào. Container mới lại gọi Redis -> Lại nhận 503 -> Lại bị Kill.
> 5. Vòng lặp CrashLoopBackOff diễn ra vô ích. Thay vì chỉ tạm rút container ra khỏi Load Balancer (Readiness check) và chờ Redis lên lại, nó giết luôn container hoàn toàn khỏe mạnh.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> `history_length` sẽ không tăng dần đều (2, 4, 6...) mà "nhảy cóc" và loạn xạ (ví dụ 2, 2, 2, 4, 4...). 
> Vì Load Balancer Nginx phân phát 5 request liên tiếp lần lượt rải vào 3 container khác nhau theo cơ chế vòng tròn (Round-Robin). Nếu lưu ở dict Python, mỗi container có bộ nhớ RAM độc lập và không chia sẻ cho nhau, container C sẽ không biết người dùng vừa chat gì với container A ban nãy. Agent sẽ bị "mất trí nhớ ngẫu nhiên".

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Thông báo lỗi: Container bị crash liên tục với dòng log báo `Invalid value for '--port': '$PORT' is not a valid integer`.
> Nguyên nhân: Khi khai báo cấu hình deploy trên `railway.toml` (hoặc Dockerfile), lệnh start được chạy thẳng bằng lệnh `exec` thay vì thông qua `shell`. Do đó, biến môi trường `$PORT` truyền thẳng chuỗi nguyên văn "$PORT" vào uvicorn thay vì thay bằng số thưc tế (vd: 8000).
> Cách sửa: Sửa chuỗi start command trong file toml để bọc lệnh khởi động vào một shell ảo: `startCommand = "sh -c 'uvicorn app.main:app --host 0.0.0.0 --port $PORT'"`. Lệnh `sh -c` sẽ nội suy (expand) biến môi trường $PORT thành giá trị thực.
