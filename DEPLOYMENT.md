# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Đình Tuấn Anh |
| Mã học viên | 2A202602735 |
| Repo | https://github.com/tuananhdayne/K4-L3B-DAY12-NguyenDinhTuanAnh-2A202602735-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-i6jm.onrender.com |
| Platform | Render |
| Ngày deploy | 29/09/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Render Redis add-on (day12-redis connectionString) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Các lệnh kiểm tra đã điền sẵn Public URL (https://day12-agent-i6jm.onrender.com):

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-i6jm.onrender.com/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-i6jm.onrender.com/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-i6jm.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-i6jm.onrender.com/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-i6jm.onrender.com/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```text
# 1. Liveness (curl -i https://day12-agent-i6jm.onrender.com/health):
HTTP/2 200 
date: Tue, 29 Sep 2026 04:47:48 GMT
content-type: application/json
server: cloudflare
{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. Readiness (curl -i https://day12-agent-i6jm.onrender.com/ready):
HTTP/2 200 
date: Tue, 29 Sep 2026 04:48:52 GMT
content-type: application/json
server: cloudflare
{"status":"ready","redis":true}

# 3. Không có API key (curl -i -X POST .../ask -> 401):
HTTP/2 401 
date: Tue, 29 Sep 2026 04:49:02 GMT
content-type: application/json
server: cloudflare
{"detail":"invalid or missing API key"}

# 4. Có API key (curl -i -X POST .../ask có header X-API-Key -> 200):
HTTP/2 200
content-type: application/json
{
  "answer": "Deploy là quá trình đưa ứng dụng lên máy chủ cloud để phục vụ người dùng thực tế.",
  "user_id": "sv-test",
  "history_length": 0,
  "cost_usd": 0.0001,
  "tokens": {"in": 14, "out": 28}
}

# 5. Rate limit (gọi 15 lần liên tiếp qua cửa sổ 60s):
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
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
5. Ghi rõ lý do không deploy được (nếu dùng fallback).
