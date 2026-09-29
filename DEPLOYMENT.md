# Thông tin deploy — Checkpoint 5

## Thông tin học viên

| Mục | Nội dung |
| --- | --- |
| Họ và tên | Võ Huy Hoàng |
| Mã học viên | 2A202602548 |
| Repo | https://github.com/hoanghuy12351/K4-L3B-DAY12-VoHuyHoang-2A202602548-Cloud-Service-And-Deployment |

## Service

| Mục | Nội dung |
| --- | --- |
| Public URL | https://day12-agent-gr8d.onrender.com |
| Platform | Render, Blueprint `day12-vohuyhoang-cp5` |
| Ngày deploy | 2026-09-29 |
| Web service | `day12-agent`, gói Free, trạng thái `Live` khi kiểm tra |
| Key Value | `day12-redis`, được tạo từ cùng Blueprint |
| Commit deploy | `50565852d6e548bbf5c1e7396ff90778d8827ec6` |

## Biến môi trường trên Render

Chỉ ghi tên biến và nguồn cấp, không ghi giá trị secret.

| Biến | Nguồn cấp |
| --- | --- |
| `AGENT_API_KEY` | Nhập trực tiếp trên Render lúc tạo Blueprint; không nằm trong repo |
| `REDIS_URL` | Render lấy `connectionString` nội bộ của `day12-redis` qua `fromService` |
| `RATE_LIMIT_PER_MINUTE` | Giá trị `10` trong `render.yaml` |
| `MONTHLY_BUDGET_USD` | Giá trị `10.0` trong `render.yaml` |
| `LOG_LEVEL` | Giá trị `INFO` trong `render.yaml` |
| `PORT` | Dockerfile đọc biến `PORT` do môi trường chạy cung cấp; nếu vắng mặt dùng 8000 |

## Kiểm tra từ máy cá nhân qua Internet

Chạy trong PowerShell, tại thư mục gốc repo:

```powershell
$base = 'https://day12-agent-gr8d.onrender.com'
Invoke-WebRequest -Uri "$base/health" -TimeoutSec 90
Invoke-WebRequest -Uri "$base/ready" -TimeoutSec 90
try {
    Invoke-WebRequest -Uri "$base/ask" -Method Post -ContentType 'application/json' -Body '{"question":"Hello"}' -TimeoutSec 90
} catch {
    [int]$_.Exception.Response.StatusCode
}
```

Kết quả quan sát ngày 2026-09-29:

```text
GET  /health       200  {"status":"ok","service":"day12-agent","version":"1.0.0"}
GET  /ready        200  {"status":"ready","redis":true}
POST /ask (no key) 401  Unauthorized
```

Ba kết quả trên chứng minh service công khai đang sống, kết nối được Redis và từ chối request không có khóa. Chưa kiểm tra `/ask` có khóa hoặc rate limit trên cloud; không ghi nhận các kết quả đó như đã chạy.

## Ảnh minh chứng

- `screenshots/blueprint-sync.png`: ảnh Blueprint sync tạo Key Value và web service do học viên cung cấp.
- Ảnh dashboard `Live` và ảnh gọi `/health` cần được học viên chụp thêm trước khi nộp theo `SUBMISSION.md`.
