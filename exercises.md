# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
> Cách trả lời: thay các placeholder bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Đình Tuấn Anh  Mã học viên: 2A202602735

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy service lên môi trường cloud (như Render hoặc Railway), nếu người triển khai quên cấu hình biến `AGENT_API_KEY` trong dashboard:
- Nếu để giá trị mặc định `"changeme"`: Service vẫn khởi động bình thường và báo trạng thái healthy. Kẻ xấu hoặc bot trên Internet có thể dò ra giá trị mặc định phổ biến này và gọi API thoải mái, làm cạn kiệt ngân sách hoặc gây rò rỉ dữ liệu mà ta không hề hay biết cho đến khi nhận hóa đơn.
- Với cơ chế **fail fast** (không có mặc định): Ứng dụng sẽ ném ngoại lệ `ValidationError` và crash ngay lập tức ở thời điểm khởi động. Quá trình deploy sẽ báo lỗi đỏ (deployment failed) và container không bao giờ nhận traffic công khai. Điều này buộc lập trình viên phải thêm secret hợp lệ trước khi dịch vụ được đưa vào sử dụng thực tế.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được trong thực tế:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:49:02.123456+00:00", "user_id": "sv-test", "tokens_in": 14, "tokens_out": 28, "cost_usd": 0.0001}
```

Hai việc làm được với log JSON:
1. **Lọc và phân tích tự động theo trường dữ liệu (Machine-parseable / Queryable)**: Các hệ thống thu gom log tập trung (như Datadog, Elasticsearch, CloudWatch) có thể tự động bóc tách các trường JSON mà không cần viết regex phức tạp. Ta có thể lọc chính xác mọi request của một `user_id` cụ thể hoặc vẽ biểu đồ tổng hợp lượng token tiêu thụ theo thời gian.
2. **Tính toán và cảnh báo chi phí (Cost monitoring & Alerting)**: Ta có thể aggregate trường `cost_usd` để tính tổng chi phí theo ngày/tháng hoặc thiết lập ngưỡng cảnh báo (alert) tự động khi chi phí của một sự kiện tăng đột biến, điều mà một dòng text thô vô cấu trúc không thể làm được.

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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | ~271 MB (Disk usage: 271MB, Content: 63.9MB) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần chênh lệch (~750 MB) bao gồm:
1. Toàn bộ các công cụ biên dịch (compilers, build-essential, gcc/g++, make), header file C/C++ và các gói phát triển (dev tools/libraries) của Debian vốn chỉ cần trong lúc cài đặt/build package bánh xe (wheels), không cần thiết khi chạy code Python.
2. Cache của pip, apt cache, tài liệu, man pages và các tiện ích hệ điều hành phụ trợ không cần thiết trong base image đầy đủ. Trong multi-stage, stage runtime chỉ copy thư mục cài đặt sạch `/install` sang `/usr/local`, loại bỏ hoàn toàn các rác build context.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại:
  - Các layer trước `COPY . .` gồm `COPY requirements.txt .`, `RUN pip install ...`, và copy thư viện từ builder sang runtime đều được **dùng lại từ cache (CACHED)** vì file `requirements.txt` không hề thay đổi.
  - Chỉ có layer `COPY . .` và các layer kế tiếp trong runtime stage phải chạy lại. Nhờ đó lệnh build hoàn thành chỉ trong vài giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  Mỗi khi sửa dù chỉ một ký tự trong bất kỳ file code nào (như `app/main.py`), checksum của layer `COPY . .` bị thay đổi. Theo cơ chế của Docker layer cache, toàn bộ các layer phía sau nó (bao gồm cả `RUN pip install`) sẽ bị vô hiệu hóa cache (cache invalidated) và Docker sẽ phải tải và cài đặt lại toàn bộ thư viện từ đầu, làm tăng thời gian build từ vài giây lên vài phút mỗi lần thay đổi code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện khi chạy bằng root:
  1. Ứng dụng Python có lỗ hổng (ví dụ: Command Injection, RCE hoặc path traversal qua thư viện bên ngoài).
  2. Kẻ tấn công khai thác lỗ hổng để thực thi mã tùy ý bên trong container. Do tiến trình chạy bằng user `root` (UID 0), kẻ tấn công chiếm toàn quyền root trong container.
  3. Từ quyền root trong container, kẻ tấn công khai thác lỗ hổng kernel Linux, misconfiguration (như container chạy privileged, mount socket `/var/run/docker.sock`, hoặc bind mount thư mục nhạy cảm) để thoát khỏi container (container breakout) và truy cập trực tiếp vào máy host với quyền `root` của hệ điều hành chủ.
- Lệnh `USER` cắt đứt chuỗi:
  Lệnh `USER appuser` chuyển tiến trình sang chạy dưới quyền user thường (non-root, UID 10001). Ngay cả khi code Python bị khai thác RCE, kẻ tấn công chỉ có quyền hạn chế của `appuser`, không thể can thiệp vào file hệ thống, không có quyền thao tác trên docker socket và không thể khai thác hầu hết các kỹ thuật container escape vốn yêu cầu đặc quyền root (capabilities như `CAP_SYS_ADMIN`).

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Số request tối đa: **20 request** trong 2 giây liên tiếp.
- Cách đạt được:
  Với cơ chế fixed window đếm theo phút đồng hồ:
  1. Người dùng gửi 10 request vào giây `10:00:59` (giây cuối cùng của phút thứ 10). Các request này đều hợp lệ vì hạn mức trong phút thứ 10 là 10.
  2. Ngay sau đó 1 giây, khi đồng hồ nhảy sang `10:01:00`, bộ đếm được reset về 0. Người dùng lập tức gửi tiếp 10 request nữa vào giây `10:01:00`.
  Tổng cộng có 20 request được gửi trong khoảng thời gian chỉ 2 giây (từ 10:00:59 đến 10:01:00) nhưng vẫn không bị hệ thống chặn. Thuật toán sliding window giải quyết triệt để vấn đề này bằng cách luôn tính chính xác tổng số request trong 60 giây trôi qua tính từ thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Điểm khác nhau:
  - **Rate limit** kiểm soát **tần suất / số lượng** request trong một đơn vị thời gian ngắn (ví dụ: tối đa 10 request/phút) để chống nghẽn mạng, spam và từ chối dịch vụ (DoS).
  - **Cost guard** kiểm soát **tổng chi phí tài chính (ngân sách)** tích lũy trong một chu kỳ dài (ví dụ: tối đa $10.0/tháng) dựa trên lượng token thực tế mà LLM tiêu thụ.
- Tình huống Rate limit cho qua nhưng Cost guard chặn:
  Một người dùng chỉ gửi 1 request trong 10 phút (tần suất rất thấp, rate limit cho qua), nhưng người đó đã dùng hết $10 ngân sách tháng trước đó. Cost guard sẽ chặn request này và trả về 402 Payment Required.
- Tình huống Cost guard cho qua nhưng Rate limit chặn:
  Một người dùng mới chưa tiêu đồng nào (ngân sách còn nguyên $10.0), nhưng gửi dồn dập 15 câu hỏi ngắn liên tiếp chỉ trong 5 giây. Cost guard không có lý do để chặn vì tổng tiền còn rất thấp, nhưng Rate limit sẽ lập tức chặn từ request thứ 11 và trả về 429 Too Many Requests để bảo vệ hệ thống không bị quá tải.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Redis gặp sự cố mất kết nối hoặc quá tải trong 30 giây.
2. Endpoint gộp kiểm tra Redis thất bại và trả về mã lỗi 503 cho bộ kiểm tra liveness của Orchestrator (Docker/Kubernetes).
3. Do coi đây là lỗi liveness probe (cho rằng ứng dụng đã chết hoặc bị deadlock), Orchestrator lập tức kill và khởi động lại (restart) toàn bộ cả 3 container agent.
4. Khi các container vừa restart xong, Redis vẫn chưa phục hồi trong khoảng thời gian 30s đó, nên lần kiểm tra tiếp theo lại tiếp tục thất bại.
5. Cụm container rơi vào vòng lặp chết chóc (`CrashLoopBackOff`): liên tục restart, tiêu tốn CPU/RAM của hệ thống, không thể phục hồi và từ chối 100% người dùng kể cả các endpoint tĩnh không cần Redis.
Tách biệt `/health` (liveness - không kiểm tra Redis) và `/ready` (readiness - có kiểm tra Redis) giúp orchestrator hiểu đúng: chỉ cho load balancer tạm dừng điều hướng traffic đến container (`/ready` 503) mà không restart tiến trình (`/health` vẫn 200).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu trong Redis (Stateless):
  Mọi instance container đều đọc/ghi chung vào một nguồn dữ liệu tập trung (Redis List). Giá trị `history_length` sẽ tăng đều đặn và liên tục: 0, 2, 4, 6, ... bất kể request của người dùng được load balancer phân phối ngẫu nhiên tới container nào.
- Nếu lưu trong một dict Python trong bộ nhớ của tiến trình:
  Do mỗi container sở hữu một vùng nhớ RAM tách biệt, khi load balancer phân phối các request kế tiếp nhau lần lượt tới Instance 1, Instance 2, rồi Instance 3:
  - Người dùng sẽ thấy `history_length` nhảy hỗn loạn hoặc bị reset (ví dụ: request 1 vào Instance 1 -> length=0; request 2 vào Instance 2 -> length=0 vì Instance 2 chưa có dữ liệu; request 3 vào lại Instance 1 -> length=2).
  - Agent sẽ có hiện tượng "mất trí nhớ từng chặp", không nhớ được ngữ cảnh mà user vừa nói ở request trước đó.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải**: Lỗi xác thực `HTTP 401 Unauthorized` khi gọi endpoint `/ask` trên cloud Render bằng API key cục bộ.
- **Thông báo lỗi**: `{"detail": "invalid or missing API key"}` (HTTP status 401).
- **Cách tìm ra nguyên nhân**:
  1. Kiểm tra mã nguồn trong `app/auth.py`: Hàm `verify_api_key` so sánh giá trị header `X-API-Key` với biến môi trường `get_settings().agent_api_key`.
  2. Kiểm tra file `render.yaml`: Trường `AGENT_API_KEY` được cấu hình với `sync: false`, nghĩa là Render sẽ yêu cầu người dùng nhập giá trị bí mật trên giao diện dashboard lúc deploy chứ không đồng bộ từ repository.
  3. Nhận ra rằng giá trị `AGENT_API_KEY` được điền trên Render dashboard lúc tạo Blueprint khác với giá trị trong file `.env` cục bộ trên máy.
- **Cách sửa**:
  Truy cập vào Render Dashboard -> vào Web Service `day12-agent` -> mục **Environment** -> kiểm tra hoặc cập nhật lại giá trị biến `AGENT_API_KEY` cho khớp với key cần sử dụng (hoặc cập nhật biến môi trường phía client), sau đó service được deploy lại và xác thực thành công trả về HTTP 200.
