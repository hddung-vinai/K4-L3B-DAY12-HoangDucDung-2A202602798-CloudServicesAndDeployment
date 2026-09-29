# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> Họ và tên: Hoàng Đức Dũng  Mã học viên: 2A202602798

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi tạo Blueprint trên Render, `AGENT_API_KEY` được khai báo `sync: false` nên
> Render phải hỏi giá trị lúc deploy. Giả sử mình bấm qua bước đó mà quên điền.
> Nếu có mặc định `"changeme"`, app vẫn lên **Live**, `/health` vẫn 200, nhìn
> dashboard thấy mọi thứ xanh. Nhưng thực tế ai đoán được chữ `changeme` (một
> trong những mật khẩu mặc định đầu tiên bot thử) là gọi `/ask` thoải mái, và
> mình chỉ biết khi hóa đơn LLM tăng hoặc ngân sách tháng bị đốt hết.
>
> Không có mặc định thì `Settings()` ném `ValidationError` ngay lúc khởi động,
> deploy báo **failed**, log ghi rõ thiếu trường `agent_api_key`. Mình biết lỗi
> trong vài giây, trước khi có bất kỳ request nào tới, và sửa bằng cách set biến
> trong dashboard. Lỗi "ồn ào" lúc deploy rẻ hơn rất nhiều so với lỗ hổng "im
> lặng" chạy trên production.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thu được từ `docker compose logs agent` khi chạy 3 instance:
>
> ```
> agent-1  | {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:45:20.484427+00:00", "user_id": "claude-check", "tokens_in": 134, "tokens_out": 44, "cost_usd": 4.65e-05}
> ```
>
> 1. **Lọc và tổng hợp theo trường.** Mình có thể lọc đúng `event = ask_completed`
>    của một `user_id` rồi cộng `cost_usd` để biết user đó tiêu bao nhiêu, hoặc
>    tính trung bình `tokens_in`. Với `print("đã trả lời xong")` thì không có user
>    nào, không có con số nào để cộng. Chính khi xem log này mình thấy `tokens_in`
>    tăng dần 1 → 43 → 88 → 134 → 179 qua các lượt, tức là lịch sử hội thoại đang
>    làm prompt dài ra — thứ mà một câu print không cho thấy.
> 2. **Đặt cảnh báo tự động và sắp xếp theo thời gian.** Vì mỗi dòng là một JSON
>    có `level` và `timestamp` chuẩn ISO (UTC), công cụ log của cloud có thể báo
>    động khi số dòng `level = error` vượt ngưỡng, hoặc ghép log của nhiều
>    container theo đúng thứ tự thời gian. Một chuỗi text tự do thì máy không biết
>    đâu là mức độ, đâu là thời điểm.

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
| 1 stage (bản đầu) | 1730 MB (1.73 GB) |
| Multi-stage |  271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh khoảng **1.46 GB**, image multi-stage chỉ bằng ~16% bản đầu. Phần chênh
> gồm:
>
> - **Base image đầy đủ `python:3.11`** (dựa trên Debian đầy đủ): có sẵn gcc,
>   make, header để biên dịch, git, curl và rất nhiều thư viện hệ thống. App của
>   mình chỉ cần Python để chạy, không cần compile gì lúc runtime. Bản `-slim` bỏ
>   hết những thứ đó — đây là phần lớn nhất của chênh lệch.
> - **Cache của pip**: bản đầu chạy `pip install` không có `--no-cache-dir` nên
>   file wheel tải về vẫn nằm lại trong image.
> - **Rác từ `COPY . .`**: bản đầu copy cả thư mục vào image, còn bản mới chỉ copy
>   `app/` và `utils/`, kết hợp `.dockerignore` loại `.venv`, `.git`, `tests/`...
>
> Multi-stage giúp mọi thứ phát sinh khi cài đặt nằm lại ở stage `builder` và bị
> bỏ đi; stage cuối chỉ nhận thư mục `/install` qua `COPY --from=builder`.
> Để so sánh, mình thử thêm bản 1 stage nhưng dùng `slim`: 287 MB — tức là riêng
> việc đổi sang slim đã giảm phần lớn, multi-stage giảm thêm phần còn lại.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Mình thêm một dòng comment vào `app/main.py` rồi build lại với
> `--progress=plain`:
>
> - **Dùng lại cache (`CACHED`)**: toàn bộ stage `builder` — `WORKDIR /build`,
>   `COPY requirements.txt .`, `RUN pip install ...` — và ở stage runtime là
>   `COPY --from=builder /install /usr/local`, `WORKDIR /app`.
> - **Phải chạy lại**: `COPY app ./app` (vì nội dung `app/` đổi) và mọi lệnh sau
>   nó: `COPY utils ./utils`, `RUN useradd ...`. Các bước này chỉ mất vài giây.
>
> Nguyên nhân: Docker so checksum của file được COPY; layer đầu tiên bị đổi sẽ làm
> hỏng cache của tất cả layer phía sau. `requirements.txt` không đổi nên bước
> `pip install` — bước chậm nhất — được giữ nguyên.
>
> Mình thử đặt `COPY . .` lên trước `pip install` rồi sửa `main.py` tương tự: lần
> này `COPY . .` bị đổi nên `RUN pip install` phải **chạy lại từ đầu, mất 66.3
> giây**, dù không đổi thư viện nào. Mỗi lần sửa một dấu phẩy là chờ hơn một phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện:
>
> 1. Code có lỗ hổng cho phép thực thi lệnh tùy ý (ví dụ một thư viện bị lỗi, hoặc
>    input của user bị đưa vào `eval`/`subprocess`).
> 2. Kẻ tấn công chạy được lệnh bên trong container **với danh tính của process
>    app**. Nếu container chạy root thì họ là root trong container: đọc mọi file,
>    sửa code, cài công cụ, đọc biến môi trường chứa `AGENT_API_KEY`.
> 3. Container không phải máy ảo — nó dùng chung kernel với host, và root trong
>    container mặc định là **cùng UID 0** với root trên host. Nếu có volume được
>    mount từ host, docker socket bị gắn vào, hoặc có một lỗ hổng kernel/runtime
>    để "thoát" khỏi container, họ bước ra ngoài với quyền root trên host.
>
> `USER appuser` cắt chuỗi ở **bước 2**: process app chạy bằng UID 10001 không có
> đặc quyền. Kẻ tấn công vẫn vào được app, nhưng chỉ có quyền của một user thường —
> không cài được gói, không sửa được file hệ thống, và nếu có thoát ra host thì
> cũng chỉ là một UID vô danh không có quyền gì. Mình kiểm tra bằng
> `docker compose exec agent whoami` → `appuser`.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> **20 request trong 2 giây.**
>
> Người dùng chờ tới cuối một phút, gửi 10 request lúc 10:00:59 — hết quota của
> phút 10:00 nhưng vẫn hợp lệ. Sang 10:01:00 bộ đếm reset về 0, họ gửi tiếp 10
> request lúc 10:01:00–10:01:01, cũng hợp lệ. Tổng cộng 20 request trong khoảng 2
> giây, gấp đôi hạn mức, mà không vi phạm luật nào của cách đếm theo phút.
>
> Với sliding window, lúc 10:01:01 hệ thống nhìn lại 60 giây gần nhất (từ
> 10:00:01) và thấy đã có 10 request, nên request thứ 11 bị 429 ngay. Khi test
> trên Render, 15 request liên tiếp cho đúng `200` × 10 rồi `429` × 5.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **tốc độ** (số request trong 60 giây gần nhất, trả 429, tự
> hồi phục sau một phút). Cost guard giới hạn **tổng tiền** trong tháng (cộng dồn
> `cost_usd` vào key `cost:<user>:<YYYY-MM>`, trả 402, chỉ reset khi sang tháng).
> Một cái chống spam, một cái chống cháy ngân sách.
>
> - **Rate limit cho qua, cost guard chặn:** một user gọi đều đặn 5 request/phút —
>   luôn dưới hạn mức 10 — nhưng mỗi request gửi câu hỏi rất dài kèm lịch sử 20
>   message, tốn nhiều token. Chạy liên tục nhiều ngày thì tổng chi phí vượt
>   `MONTHLY_BUDGET_USD=10.0`, cost guard trả 402 dù tốc độ hoàn toàn hợp lệ.
> - **Cost guard cho qua, rate limit chặn:** đầu tháng, một script lỗi gửi 50
>   request `"test"` trong vài giây. Mỗi request chỉ tốn khoảng 0.00002 USD nên
>   ngân sách gần như chưa suy suyển, nhưng từ request thứ 11 đã bị 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối. Cả 3 container vẫn chạy bình thường, chỉ là không nói
>    chuyện được với Redis.
> 2. Health check tiếp theo (mỗi 30 giây) gọi endpoint gộp → endpoint ping Redis
>    thất bại → cả 3 container cùng trả 503.
> 3. Sau vài lần thất bại liên tiếp (`retries: 3`), orchestrator đánh dấu **cả 3**
>    là unhealthy và kết luận process đã hỏng.
> 4. Orchestrator **restart cả 3 container cùng lúc**. Toàn bộ request đang xử lý
>    dở bị cắt, và trong lúc khởi động lại thì không còn instance nào phục vụ —
>    kể cả các request không cần Redis.
> 5. Redis quay lại sau 30 giây, nhưng hệ thống vẫn đang bận khởi động lại; nếu
>    các container lên trước khi Redis ổn định thì lại fail health check và rơi vào
>    vòng restart lặp.
>
> Kết quả: sự cố 30 giây của một dependency biến thành sập toàn bộ dịch vụ. Tách
> ra thì `/health` vẫn 200 (process sống, không restart), còn `/ready` trả 503
> để load balancer tạm ngừng gửi traffic; Redis quay lại là `/ready` về 200 và
> traffic chảy tiếp, không container nào bị restart.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Mình chạy 3 agent sau nginx và gọi 5 lần với cùng một user. Kết quả thực tế:
>
> | Lượt | Container xử lý | history_length |
> |------|-----------------|----------------|
> | 1 | agent-1 | 0 |
> | 2 | agent-2 | 2 |
> | 3 | agent-3 | 4 |
> | 4 | agent-1 | 6 |
> | 5 | agent-2 | 8 |
>
> Nginx chia round-robin nhưng con số vẫn tăng đều 0, 2, 4, 6, 8 vì cả 3 container
> đọc cùng một list `history:<user>` trong Redis.
>
> Nếu lưu trong dict Python, mỗi container có dict riêng và chỉ nhớ những lượt rơi
> vào chính nó, nên con số sẽ **nhảy lung tung**: lượt 1 → 0 (agent-1), lượt 2 → 0
> (agent-2 chưa thấy gì), lượt 3 → 0 (agent-3), lượt 4 → 2 (agent-1 chỉ nhớ lượt
> 1), lượt 5 → 2. Agent "mất trí nhớ" tùy theo request rơi vào đâu, và mọi thứ về
> 0 khi container bị restart hoặc deploy bản mới.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Sau khi deploy lên Render, mình gọi `/ask` có API key từ Windows PowerShell 5.1:
>
> ```
> Invoke-RestMethod ... -Body '{"question":"Deploy là gì?"}'
> Invoke-RestMethod : The remote server returned an error: (400) Bad Request.
> ```
>
> **Tìm nguyên nhân:** ban đầu mình nghĩ sai API key, nhưng key sai thì phải là
> 401 chứ không phải 400. Mình thử gửi cùng request với câu hỏi không dấu
> `"Deploy la gi?"` thì được 200 — vậy key đúng, service đúng, lỗi nằm ở body.
> Đọc body lỗi thì thấy `{"detail":"There was an error parsing the body"}`:
> FastAPI không parse được JSON. Nguyên nhân là PowerShell 5.1 không mã hóa chuỗi
> `-Body` theo UTF-8, nên các ký tự `à`, `ì` bị gửi thành byte không hợp lệ.
>
> **Cách sửa:** tự chuyển body sang byte UTF-8 trước khi gửi:
>
> ```powershell
> $body = [System.Text.Encoding]::UTF8.GetBytes('{"question":"Deploy là gì?"}')
> Invoke-RestMethod ... -ContentType "application/json; charset=utf-8" -Body $body
> ```
>
> Lần này trả 200. Phần `answer` hiển thị bị lỗi font (`CÃ¢u há»i hay`) vì
> PowerShell 5.1 cũng đọc response sai bảng mã; mình đọc byte gốc bằng
> `UTF8.GetString($r.RawContentStream.ToArray())` thì hiện đúng "Câu hỏi hay...",
> kèm `history_length: 2` chứng minh Redis trên cloud đang lưu lịch sử.
> Bài học: phân biệt mã lỗi (400 khác 401) giúp khoanh vùng nhanh, và thử lại với
> input tối giản để tách lỗi phía client khỏi lỗi phía server.
