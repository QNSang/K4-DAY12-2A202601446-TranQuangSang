# Thong Tin Deploy - Checkpoint 5

## Thong Tin Hoc Vien

| Muc | Noi dung |
|-----|----------|
| Ho va ten | Tran Quang Sang |
| Mã học viên | 2A202601446 |
| Repo | K4-Day12-Cloud-Services-And-Deployment-2A202601446 |

## Service

| Muc | Noi dung |
|-----|----------|
| Public URL | Local fallback: http://localhost:8000 |
| Platform | Local fallback bang Docker Compose; cloud target co the dung Railway hoac Render |
| Ngay deploy | 2026-08-10 |

## Bien Moi Truong Da Set

Chi liet ke ten bien, khong ghi gia tri secret.

| Bien | Da set | Ghi chu |
|------|--------|---------|
| `PORT` | yes | local 8000; platform cloud se tu gan |
| `API_TOKEN` | yes | dat trong `.env` local hoac dashboard cloud, khong nam trong repo |
| `REDIS_URL` | yes | local compose dung `redis://redis:6379/0`; app local co the dung `redis://localhost:6379/0` |
| `BUCKET_CAPACITY` | yes | cau hinh rate limit |
| `REFILL_PER_MINUTE` | yes | cau hinh rate limit |
| `DAILY_BUDGET_USD` | yes | cau hinh cost guard |
| `LOG_LEVEL` | yes | cau hinh logging |
| `LOCAL_FALLBACK` | yes | dat `true` de test CP5 kiem tra local stack |

## Ket Qua Chay That

Dang dung phuong an du phong vi chua deploy len cloud trong luot nay. Stack local da chay bang:

```powershell
docker compose up -d
docker compose ps
```

Ket qua chinh:

```text
chat   Up (healthy)   0.0.0.0:8000->8000/tcp
redis  Up (healthy)   0.0.0.0:6379->6379/tcp
```

Kiem tra API:

```text
GET  http://localhost:8000/healthz  -> 200 {"status":"ok","service":"day12-chat-service","version":"1.0.0"}
GET  http://localhost:8000/readyz   -> 200 {"status":"ready","redis":true}
POST http://localhost:8000/chat without token -> 401
POST http://localhost:8000/chat with Bearer token -> 200, co truong reply
```

## Anh Chung Minh

Anh bang chung local fallback nam trong thu muc `screenshots/`.

## Ly Do Dung Phuong An Du Phong

Trong phien lam nay chua cau hinh tai khoan cloud Railway/Render va public domain. Vi vay CP5 duoc hoan thanh theo local fallback: Docker Compose chay day du `chat` va `redis`, `/healthz` va `/readyz` deu 200, `/chat` bat buoc Bearer token.
