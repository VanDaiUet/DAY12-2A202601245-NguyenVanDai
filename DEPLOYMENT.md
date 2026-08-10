# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Văn Đại |
| Mã học viên | 2A202601245 |
| Repo | https://github.com/VanDaiUet/DAY12-2A202601245-NguyenVanDai |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-2a202601245-nguyenvandai-production-6160.up.railway.app/ |
| Platform | Railway |
| Ngày deploy | 10/08/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của Railway |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```text
# 1. /health
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 10 Aug 2026 05:09:01 GMT
Server: railway-hikari
x-railway-request-id: qkSf8Bg6RmCunP4T2h0iww
Content-Length: 57
x-hikari-trace: sin1.d1nj
x-railway-edge: sin1
Connection: keep-alive

{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. /ready
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 10 Aug 2026 05:09:22 GMT
Server: railway-hikari
x-railway-request-id: bX_0E_meSridNc4-9o6EoQ
Content-Length: 31
x-hikari-trace: sin1.98a6
x-railway-edge: sin1
Connection: keep-alive

{"status":"ready","redis":true}

# 3. /ask không truyền API key
HTTP/1.1 401 Unauthorized
Content-Type: application/json
Date: Mon, 10 Aug 2026 05:09:37 GMT
Server: railway-hikari
x-railway-request-id: om4SZhPIR4SY-Bq70_TJvA
Content-Length: 39
x-hikari-trace: sin1.d1nj
x-railway-edge: sin1
Connection: keep-alive

{"detail":"invalid or missing API key"}
# 4. /ask truyền API key
Date: Mon, 10 Aug 2026 05:09:56 GMT
Server: railway-hikari
x-railway-request-id: YOSgl8fmRPSMvDEr9o6EoQ
Content-Length: 340
x-hikari-trace: sin1.tr00
x-railway-edge: sin1
vary: accept-encoding
Connection: keep-alive

{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud. (Mình đang nhớ 20 lượt trao đổi trước đó.)","user_id":"sv-test","history_length":20,"cost_usd":9.645e-05,"tokens":{"in":463,"out":45}}

# 5. Rate limit
200 200 200 200 200 200 200 200 200 429 429 429 429 429 429 
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

