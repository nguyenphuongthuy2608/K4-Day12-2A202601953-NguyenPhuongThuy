# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Phương Thùy |
| Mã học viên | 2A202601953 |
| Repo | https://github.com/nguyenphuongthuy2608/K4-Day12-2A202601953-NguyenPhuongThuy |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-chat-t3li.onrender.com |
| Platform | Render |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `API_TOKEN` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Render Key Value (`day12-chat-redis`) |
| `BUCKET_CAPACITY` | ✅ | 10 |
| `REFILL_PER_MINUTE` | ✅ | 10 |
| `DAILY_BUDGET_USD` | ✅ | 1.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Các lệnh đã dùng với Public URL thật:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-chat-t3li.onrender.com/healthz

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-chat-t3li.onrender.com/readyz

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST https://day12-chat-t3li.onrender.com/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-chat-t3li.onrender.com/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $DEPLOY_API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-chat-t3li.onrender.com/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $DEPLOY_API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Output ghi nhận lúc 16:44 ngày 2026-08-10 (giờ Việt Nam):

```text
$ curl -i https://day12-chat-t3li.onrender.com/healthz
HTTP/2 200
content-type: application/json

{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

$ curl -i https://day12-chat-t3li.onrender.com/readyz
HTTP/2 200
content-type: application/json

{"status":"ready","redis":true}

$ curl -i -X POST https://day12-chat-t3li.onrender.com/chat \
    -H "Content-Type: application/json" \
    -d '{"message":"Hello"}'
HTTP/2 401
content-type: application/json
www-authenticate: Bearer

{"detail":"invalid or missing bearer token"}

$ curl -i -X POST https://day12-chat-t3li.onrender.com/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $DEPLOY_API_TOKEN" \
    -H "X-Client-Id: deployment-doc" \
    -d '{"message":"Deploy là gì?"}'
HTTP/2 200
content-type: application/json

{"reply":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","client_id":"deployment-doc","turns_before":0,"usd_cost":2.145e-05,"usage":{"prompt":3,"completion":35}}
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được. Bài này đã deploy thành công trên Render nên
   không sử dụng phương án dự phòng.
