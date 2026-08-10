# Todo list truoc khi lam LAB_GUIDE

File nay tom tat viec can lam theo thu tu checkpoint trong `LAB_GUIDE.md`. Muc tieu la giup ban biet minh can sua file nao, vi sao phai sua, va lenh nao dung de kiem tra truoc khi chuyen sang phan tiep theo.

## Nguyen tac lam bai

- [ ] Lam theo checkpoint, khong nhay lung tung.
  - Giai thich: moi block co test rieng. Khi CP1 chua xanh ma lam CP3, loi se chong len nhau va rat kho debug.

- [ ] Sau moi checkpoint nen commit mot lan.
  - Giai thich: lich su commit cho thay qua trinh tu lam, de quay lai neu sua sai, va phu hop yeu cau lab.

- [ ] Khong commit `.env`.
  - Giai thich: `.env` chua secret nhu `API_TOKEN`. Secret vao Git thi coi nhu da lo.

- [ ] Neu ket qua test do, doc loi cua test truoc khi sua.
  - Giai thich: cac test trong lab thuong noi ro sai o dau va vi sao yeu cau do quan trong.

## CP0 - Setup moi truong

- [ ] Kiem tra ten repo dung mau `K4-DAY12-<MaHocVien>-<HoVaTenKhongDau>`.
  - Giai thich: sai ten repo bi tru diem va kho de Lab Coach map bai nop voi hoc vien.

- [ ] Tao virtual environment va cai dependencies.
  - Lenh Windows PowerShell:
    ```powershell
    python -m venv .venv
    .\.venv\Scripts\Activate.ps1
    python -m pip install -r requirements.txt
    ```
  - Giai thich: app va pytest can dung dung package trong `requirements.txt`.

- [ ] Tao file `.env` tu `.env.example`.
  - Lenh:
    ```powershell
    copy .env.example .env
    ```
  - Giai thich: app doc config tu bien moi truong, khong hard-code trong source code.

- [ ] Sinh `API_TOKEN` rieng va dien vao `.env`.
  - Lenh:
    ```powershell
    python -c "import secrets; print(secrets.token_urlsafe(32))"
    ```
  - Giai thich: CP1/CP3 can token that de test cau hinh va Bearer authentication.

- [ ] Bat Redis neu Docker Desktop dang chay.
  - Lenh:
    ```powershell
    docker compose up -d redis
    docker compose ps
    ```
  - Giai thich: Redis dung cho chat history, rate limit va cost guard. Neu Docker chua dung duoc, tam thoi dat `REDIS_URL=fake://` trong `.env` de lam CP dau.

- [ ] Chay checkpoint setup.
  - Lenh:
    ```powershell
    pytest tests/ -v -m "not docker"
    ```
  - Giai thich: o buoc nay test rot la binh thuong, mien la pytest chay duoc va khong loi import/package.

## CP1 - 12-Factor Config, Health va Logging

- [ ] Sua `app/config.py`: khai bao day du cac truong trong `Settings`.
  - Can chu y: `api_token` khong co gia tri mac dinh.
  - Giai thich: secret thieu thi app phai chet som luc deploy, thay vi chay voi token gia nhu `changeme`.

- [ ] Sua `app/logging_utils.py`: ham `emit()` in moi event thanh mot dong JSON.
  - Can co cac truong nhu `event`, `severity`, `ts` va payload bo sung.
  - Giai thich: cloud logging gom log theo tung dong. JSON mot dong giup loc theo severity, client, cost va event.

- [ ] Sua `app/main.py`: them `GET /healthz`.
  - Ket qua mong doi: `200 {"status": "ok", "service": ..., "version": ...}`.
  - Giai thich: `/healthz` la liveness probe, chi tra loi process con song hay khong; khong nen kiem tra Redis/database tai endpoint nay.

- [ ] Chay checkpoint CP1.
  - Lenh:
    ```powershell
    pytest tests/test_cp1.py -v
    ```

## CP2 - Docker

- [ ] Sua `Dockerfile` thanh multi-stage build.
  - Giai thich: stage builder co the nang va co compiler; runtime chi copy ket qua can chay, giup image nho hon.

- [ ] Sap xep layer Docker dung cache tot.
  - Mau thu tu nen co:
    ```dockerfile
    COPY requirements.txt .
    RUN pip install ...
    COPY app ./app
    COPY utils ./utils
    ```
  - Giai thich: sua code khong nen lam Docker cai lai toan bo thu vien.

- [ ] Chay app trong container bang user thuong, khong chay root.
  - Giai thich: neu app co lo hong, attacker khong nen co quyen root trong container.

- [ ] Dung `PORT` tu bien moi truong va bind `0.0.0.0`.
  - Giai thich: cloud platform tu gan port; bind `127.0.0.1` se lam ben ngoai container khong truy cap duoc.

- [ ] Bo sung `HEALTHCHECK` trong Dockerfile.
  - Giai thich: container platform can biet service con song de restart khi can.

- [ ] Kiem tra `.dockerignore` co loai `.env`, `.git`, `.venv`, `__pycache__`.
  - Giai thich: tranh dua secret, virtualenv va lich su Git vao image.

- [ ] Sua `docker-compose.yml`: them service `chat`.
  - Can co: `build`, `ports`, `depends_on: redis`, healthcheck, `API_TOKEN`, `REDIS_URL=redis://redis:6379/0`.
  - Giai thich: trong container, hostname `redis` tro toi service Redis; `localhost` la chinh container chat.

- [ ] Chay checkpoint CP2.
  - Lenh nhanh:
    ```powershell
    pytest tests/test_cp2.py -v -m "not docker"
    ```
  - Lenh day du:
    ```powershell
    pytest tests/test_cp2.py -v
    ```

## CP3 - API Security

- [ ] Sua `app/auth.py`: kiem tra Bearer token theo RFC 6750.
  - Can lam: tach scheme/token, chap nhan `Bearer` khong phan biet hoa thuong, dung `secrets.compare_digest`, loi 401 co header `WWW-Authenticate: Bearer`.
  - Giai thich: authentication phai chuan va khong ro ri thong tin "sai o dau" cho nguoi do token.

- [ ] Sua `app/rate_limiter.py`: cai token bucket.
  - Can lam: client moi co day bucket, refill theo thoi gian, chan tai `capacity`, moi request tru 1 token, cap nhat ca `tokens` va `ts`.
  - Giai thich: token bucket cho phep burst hop ly nhung chan spam lien tuc.

- [ ] Sua `app/cost_guard.py`: gioi han chi phi theo ngay.
  - Can lam: `spent()`, `check()`, `record()` theo key `spend:<client>:<YYYY-MM-DD>`.
  - Giai thich: daily budget gioi han thiet hai trong ngay va tu reset ngay hom sau.

- [ ] Gheps logic vao `POST /chat` trong `app/main.py`.
  - Thu tu nen la: verify token -> rate limit -> cost check -> doc history -> generate reply -> luu history -> record cost -> emit log.
  - Giai thich: phai chan truoc khi goi LLM/mock LLM de khong ton chi phi cho request khong hop le.

- [ ] Chay checkpoint CP3.
  - Lenh:
    ```powershell
    pytest tests/test_cp3.py -v
    ```

## CP4 - Scaling va Reliability

- [ ] Chuyen state dung chung sang Redis, khong giu dict/list global trong process.
  - Giai thich: khi scale nhieu container, moi process co memory rieng. State trong memory se lam moi replica thay lich su khac nhau.

- [ ] Sua `app/store.py`: luu chat history vao Redis va cat bot lich su qua dai.
  - Can chu y: neu dung Redis list, `ltrim(key, -N, -1)` giu N phan tu moi nhat.
  - Giai thich: history khong nen tang vo han va phai dung chung giua cac replica.

- [ ] Them `GET /readyz`.
  - Giai thich: readiness probe kiem tra dependency nhu Redis. Neu Redis loi, app khong ready nhan traffic nhung process khong nhat thiet phai restart.

- [ ] Xu ly graceful shutdown/SIGTERM trong `app/lifecycle.py` va `app/main.py`.
  - Giai thich: khi deploy version moi, container cu can co thoi gian dung nhan request moi va hoan tat request dang xu ly.

- [ ] Neu co scale bang compose, kiem tra request qua nhieu replica van thay chung history.
  - Giai thich: day la dau hieu app stateless dung cach.

- [ ] Chay checkpoint CP4.
  - Lenh:
    ```powershell
    pytest tests/test_cp4.py -v
    ```

## CP5 - Cloud Deployment

- [ ] Chon platform deploy: Railway hoac Render.
  - Giai thich: ca hai deu doc Dockerfile; Railway nhanh hon, Render co blueprint `render.yaml`.

- [ ] Cau hinh bien moi truong tren cloud.
  - Can co: `API_TOKEN`, `REDIS_URL`, `BUCKET_CAPACITY`, `REFILL_PER_MINUTE`, `DAILY_BUDGET_USD`, `LOG_LEVEL`.
  - Giai thich: secret va config production nam tren dashboard cloud, khong nam trong repo.

- [ ] Deploy image len cloud.
  - Giai thich: muc tieu CP5 la service co public URL that, goi duoc tu ben ngoai.

- [ ] Kiem tra public URL.
  - Lenh mau:
    ```powershell
    curl -i https://<domain-cua-ban>/healthz
    curl -i https://<domain-cua-ban>/readyz
    curl -i -X POST https://<domain-cua-ban>/chat -H "Content-Type: application/json" -d '{"message":"Hello"}'
    ```
  - Giai thich: `/healthz` va `/readyz` phai 200; `/chat` khong token phai 401.

- [ ] Dien `DEPLOYMENT.md`.
  - Can dien: platform, public URL, ten bien moi truong, output test/curl, anh chup dashboard trong `screenshots/`.
  - Giai thich: day la bang chung deploy that. Chi ghi ten bien, khong ghi gia tri `API_TOKEN`.

- [ ] Neu khong deploy duoc, dung local fallback.
  - Can lam: dat `LOCAL_FALLBACK=true`, chay `docker compose up -d`, chup anh, ghi ly do vao `DEPLOYMENT.md`.
  - Giai thich: fallback van co diem nhung CP5 toi da thap hon deploy cloud that.

- [ ] Chay checkpoint CP5.
  - Lenh:
    ```powershell
    pytest tests/test_cp5.py -v
    ```

## Exercises va nop bai

- [ ] Tra loi 10 cau trong `exercises.md` bang quan sat cua minh.
  - Giai thich: day la phan diem rieng, kiem tra ban hieu nhung gi minh da lam.

- [ ] Chay cham thu.
  - Lenh:
    ```powershell
    python grade.py
    ```

- [ ] Kiem tra `.env` khong bi Git track.
  - Lenh PowerShell:
    ```powershell
    git ls-files | Select-String "^\.env$"
    ```
  - Giai thich: neu lenh nay in ra `.env`, phai xu ly truoc khi nop.

- [ ] Commit va push bai.
  - Lenh:
    ```powershell
    git add -A
    git commit -m "Hoan thanh lab Day 12"
    git push origin main
    ```

## Bonus - CI/CD voi GitHub Actions

- [ ] Chi lam bonus sau khi CP1 den CP5 da on.
  - Giai thich: CI/CD se tu dong hoa test/build/deploy; neu code chua on thi workflow chi them mot lop loi.

- [ ] Tao `.github/workflows/ci.yml`.
  - Can co job `test`, `build`, va `deploy`.
  - Giai thich: `test` bat loi code, `build` bat loi Docker, `deploy` chi chay khi moi thu xanh.

- [ ] Cau hinh `deploy` dung `needs: [test, build]` va chi deploy khi push len `main`.
  - Giai thich: pull request khong nen tu deploy production.

- [ ] Dua token deploy vao GitHub Secrets, khong ghi vao YAML.
  - Giai thich: secret trong YAML se nam trong git history.

- [ ] Them smoke test sau deploy.
  - Giai thich: deploy command chay xong khong dong nghia service song; phai goi `/healthz`.

- [ ] Them badge CI vao dau `README.md`.
  - Giai thich: nguoi xem repo thay ngay workflow main dang passing hay failing.

- [ ] Chay checkpoint bonus.
  - Lenh:
    ```powershell
    pytest tests/test_bonus_cicd.py -v
    ```
