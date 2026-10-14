# Lab 25B — Report

**Họ tên / nhóm:** Nguyễn Thị Chinh (MSV: 2A202602876)  **Commit:** 21cd1657469962fb7b3eb96fce76aee70fd46fe1  **Điểm tự động (`make grade`):** 84.0 / 88

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | N/A | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | N/A | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0s | Đủ lớn hơn p99 lúc bình thường (~5.8s ở G1), nhưng đủ ngắn để tránh treo call khi gặp provider chết (ở G6 giúp failover kịp trước deadline 30s của user). |
| max_attempts_per_provider | 2 | Cho phép đúng 1 retry tại chỗ trên cùng provider đối với lỗi transient; nếu vẫn lỗi thì failover ngay, chặn đứng retry storm (ở G2 giảm amplification từ 9.904 xuống 1.041). |
| backoff_base_s / cap_s | 0.25s / 4.0s | Giảm tối đa hiện tượng thắt nút cổ chai (thundering herd) bằng full jitter; thời gian chờ tối đa 4s không làm lố deadline của lượt. |
| retry_budget_ratio | 0.1 | Đảm bảo lượng retry không vượt quá 10% tổng request, trực tiếp giúp vượt qua kiểm tra của băng G3 (`retry_ratio` = 0.034 <= 0.10). |
| breaker.window_size / min_calls | 20 / 10 | Đủ mẫu để loại trừ biến động ngẫu nhiên, phản ứng nhanh sau 10 calls khi provider gặp sự cố (như G2 và G4). |
| breaker.failure_rate_threshold | 0.5 | Trip OPEN khi lỗi >= 50% trong window để cắt tải kịp thời bảo vệ provider. |
| breaker.slow_call_s | 5.0s | Phát hiện vùng chậm ở G4 (khi latency median bình thường là 1.2s); đưa p95 từ 13.764s của naive xuống 5.562s. |
| breaker.open_duration_s | 10.0s | Phù hợp với `max_queue_wait_s` = 10s của provider, cho phép hàng đợi xả sạch trước khi thử lại ở HALF-OPEN. |
| breaker.half_open_max_probes | 2 | Giới hạn tối đa 2 request thăm dò khi chuyển sang HALF-OPEN, không gây nghẽn lại provider vừa hồi phục. |

Sơ đồ quyết định của `on_error` (ASCII hoặc bảng: loại lỗi → ghi breaker? → retry? → failover? → cooldown?):

```
+------------------+---------------------+-------------------+-------------------+------------------------------+
| ErrorKind        | Breaker Action      | Retry Provider?   | Failover?         | Cooldown Provider?           |
+------------------+---------------------+-------------------+-------------------+------------------------------+
| CLIENT_ERROR     | release permit      | NO (GiveUp)       | NO (GiveUp)       | NO                           |
| QUOTA_EXHAUSTED  | release permit      | NO                | YES (lập tức)     | YES (now + quota_cooldown_s) |
| RATE_LIMITED     | release permit      | NO                | YES (lập tức)     | YES (now + retry_after)      |
| OVERLOADED /     | on_failure          | YES (nếu tries<2  | YES (nếu hết lượt | NO                           |
| SERVER_ERROR /   |                     | & budget cho phép & retry / hết thời  |                              |
| TIMEOUT          |                     | & kịp deadline)   | gian deadline)    |                              |
+------------------+---------------------+-------------------+-------------------+------------------------------+
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.** Khi vô hiệu hóa Circuit Breaker (`breaker.min_calls: 1000000000`) và tăng `max_attempts_per_provider: 5`:
   - Nếu **có** Retry Budget: ở G2 amplification được hãm ở mức 1.383, retry_ratio = 0.058, recovery = 0.0s, availability đạt 99.87%; ở G3 retry_ratio chỉ là 0.068.
   - Nếu **tắt** Retry Budget: ở G2 amplification tăng vọt lên 1.672, retry_ratio nhảy lên 0.491, recovery kéo dài đến 30.0s và availability sụt xuống 96.49%; ở G3 retry_ratio tăng hơn gấp đôi lên 0.162.
   - Khi breaker hoạt động đúng, budget vẫn tối quan trọng vì breaker chỉ bảo vệ cục bộ từng provider khi tỷ lệ lỗi vượt ngưỡng, trong khi retry budget bảo vệ toàn hệ thống khỏi việc tổng lượng retry của hàng nghìn client đồng thời tích tụ làm sập provider. Cả hai tạo nên cấu trúc phòng thủ chiều sâu (defense-in-depth).

2. **`attempt_timeout_s: 30`.**
   - Trên băng G6 (outage), availability bị tụt xuống **0.9488** (so với 1.0000 của cấu hình 8.0s), với **109** lượt bị GiveUp vì lý do `deadline` và 13 lượt vì `no_provider_available`. Đáng chú ý, `latency_p95_s` lại hiển thị con số giảm ảo là **5.306s** (so với 5.961s). Đây là biểu hiện kinh điển của **survivorship bias**: p95 chỉ tính trên các request may mắn thành công, còn các request bị treo 30s đều đã fail và bị loại khỏi mẫu tính.
   - Trên băng G4 (slow region), availability vẫn giữ 1.0000 và p95 là 5.683s vì breaker slow-call (ngưỡng 5s) đã sớm phát hiện và chuyển mạch sang `alpha-eu` trước khi chạm mốc timeout.
   - Timeout chuẩn phải dựa vào: `p99 bình thường + headroom cho failover`, tuyệt đối không đặt bằng user deadline vì sẽ làm mất cơ hội failover sang provider lành lặn.

3. **Bỏ lọc tool calling khỏi `ladder()`.**
   - Khi chạy G6 mà không lọc tool calling, hệ thống ghi nhận **258** vi phạm `incompatible_routes`, kéo availability tụt từ 100% xuống **89.18%**.
   - Dưới góc độ Product Manager: "258 khách hàng yêu cầu hoàn tiền tự động đã bị hệ thống đẩy sang một AI giá rẻ không có khả năng thao tác với cổng thanh toán. Hậu quả là khách hàng nhận về câu trả lời cụt lủn hoặc lỗi hệ thống, gây khiếu nại nghiêm trọng và làm đổ vỡ cam kết tự động hóa dịch vụ."

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1496700 | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Trả lời:
- Chỉ Idempotency Key là **chưa đủ** vì nó chỉ bảo vệ downstream service không thực hiện 2 lần cùng một tác vụ khi gọi lại, nhưng worker mới sau khi crash sẽ không biết workflow đã đi đến bước nào và buộc phải chạy lại toàn bộ từ đầu, gây lãng phí gọi LLM (`wasted_llm_per_crash`).
- Chỉ Checkpoint cũng **chưa đủ** vì nếu worker crash đúng lúc payment vừa thực thi thành công nhưng chưa kịp lưu checkpoint, worker mới sẽ gọi lại payment mà không có idempotency key để de-duplicate, dẫn đến hoàn tiền trùng (duplicate refund).
- Nếu ghi checkpoint **trước** khi gọi payment: khi worker crash giữa lúc vừa ghi checkpoint xong nhưng tiền chưa kịp chuyển, worker mới khởi động lại sẽ nhìn thấy checkpoint đã có và bỏ qua bước thanh toán $\rightarrow$ dẫn đến sự cố nghiêm trọng: **mất tiền hoàn của khách (`lost_refunds > 0`)**.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page (delay 1m) | page | 27m |
| provider_outage | page | page (delay 0m) | page | 4m |
| deploy_blip | none | page (FALSE PAGE) | none | N/A (0 false page) |
| slow_leak | ticket | none (BỎ SÓT) | ticket | 1337m |
| degraded_evening | page | page (delay 214m) | page | 291m |
| night_spike | none | page (FALSE PAGE) | none | N/A (0 false page) |
| small_bump | none | page (FALSE PAGE) | none | N/A (0 false page) |

Error budget 30 ngày: đã tiêu **42.95%**, còn **57.05%**. Có đóng băng release không? **Không** (`freeze_releases = False`).

Vì sao `small_bump` (20 phút ở 12% lỗi) **không** nên đánh thức ai lúc 11 giờ trưa?
Vì trong cửa sổ 30 ngày, một sự cố 20 phút ở 12% lỗi chỉ tiêu tốn:
$$\frac{20 \times 0.12}{43200 \times 0.005} \approx 1.11\% \text{ error budget}$$
Mức tiêu hao này thấp hơn nhiều so với ngưỡng kích hoạt page 2% (cửa sổ 1 giờ, burn-rate 14.4x) hoặc 5% (cửa sổ 6 giờ, burn-rate 6x). Nó tự giới hạn và không đe dọa cam kết SLO 99.5%, do đó đánh thức on-call sẽ gây ra hiện tượng mệt mỏi cảnh báo (alert fatigue).

## 6. Postmortem — Băng G2 với client naive (blameless)

*Dùng `make plot` (hình `reports/gameday/G2_overload_storm.png`) và CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt (3 câu):**
Vào giây ảo t=120, provider chính (`alpha`) bị mất 75% năng lực GPU trong 3 phút dẫn đến hàng đợi bị ứ nghẽn và phát sinh lỗi quá tải (529/timeout). Client naive v1 với cơ chế 3 tầng retry cố định, thiếu jitter và không có circuit breaker đã bơm tải tăng gấp 10 lần (lên đến 189 attempts/s) vào provider đang suy yếu. Hậu quả là tạo ra vòng lặp hỏng hóc giả định (metastable failure), khiến hệ thống mất thêm 155 giây sau khi provider đã hồi phục hoàn toàn mới có thể trở lại bình thường, làm sụt giảm availability cả đợt xuống còn 45.60%.

**Ảnh hưởng:**
Có **1,285** lượt người dùng thất bại (good turns chỉ đạt 1,077 / 2,362 lượt; availability = 45.60%). Toàn bộ sự cố kéo dài tổng cộng **335 giây** (từ t=120 đến t=455). Lượng lỗi này tiêu tốn vượt quá **100%** error budget của cả tháng (tạo ra hàng chục nghìn request lỗi 429 và timeout).

**Timeline** (giây ảo — lấy từ CSV):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 120 | alpha bắt đầu mất 75% năng lực GPU | `attempts_alpha` = 5 req/s, bắt đầu xuất hiện độ trễ tăng cao. |
| 136 | Bùng nổ retry storm từ 3 tầng retry | `attempts_alpha` tăng vọt từ 21 lên 53 req/s; t=138 đạt 122 req/s. |
| 154 | Đỉnh điểm quá tải, dịch vụ tê liệt hoàn toàn | `attempts_alpha` đạt đỉnh **189 req/s**, `calls_ok` = 0 (tất cả cuộc gọi đều bị nghẽn/timeout). |
| 300 | alpha hồi phục hoàn toàn năng lực phần cứng | `attempts_alpha` giảm dần về ~115 req/s, tuy nhiên `calls_ok` vẫn bằng **0**. |
| 300–452 | Trạng thái Metastable Failure kéo dài | Suốt 152 giây sau khi hạ tầng đã tốt, `calls_ok` vẫn là 0 do hàng đợi FIFO kẹt cứng request cũ đã quá hạn. |
| 453–455 | Hàng đợi xả sạch, dịch vụ bắt đầu hồi phục | Tại t=453 ghi nhận `calls_ok` = 4; tại t=455 `calls_ok` = 4 và t=456 `calls_ok` = 8 (recovery delay = 155s). |

**Trigger vs root cause:**
- **Trigger:** Provider `alpha` bị suy giảm đột ngột 75% số slot xử lý GPU trong 180 giây (t=120 đến t=300).
- **Root cause:** Kiến trúc client naive v1 thiếu kiểm soát lưu lượng: 3 tầng retry chồng nhau (app $\times$ gateway $\times$ SDK = tối đa 36 attempt/call), retry delay cố định, không có circuit breaker để ngắt mạch, không có fallback ladder sang `alpha-eu`/`beta`, và không có retry budget.
- **Cơ chế Metastable Failure:** Do provider xử lý hàng đợi FIFO và vẫn tiếp tục xử lý các request mà client đã timeout bỏ cuộc, lượng retry ồ ạt (>150 req/s) liên tục lấp đầy hàng đợi 10s. Khi provider hồi phục lúc t=300, toàn bộ năng lực máy chủ bị lãng phí để xử lý "rác" (các request mà client đã timeout), khiến các request mới tiếp tục bị quá hạn $\rightarrow$ vòng lặp tự duy trì cho đến khi áp lực traffic tự giãn ở t=453.

**Điều gì giúp / điều gì làm tệ hơn:**
- *Làm tệ hơn:* Client timeout nhưng server không hủy request; retry không có jitter khiến các đợt thử lại dồn cục; thiếu circuit breaker tiếp tục ném request vào server đang chết.
- *Giúp ích:* Simulator tự giới hạn hàng đợi tối đa 10s (`max_queue_wait_s`) và trả về 529 giúp hàng đợi không phình to vô hạn.

**Action items** (cụ thể, có owner, đo được):

| # | Hành động | Loại (prevent / detect / mitigate) | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay thế `NaiveClient` bằng `ResilientClient` có Circuit Breaker sliding-window và Full Jitter Backoff | Mitigate | Client Team | Chạy CI mô phỏng băng G2 đạt amplification $\le 1.2$ và recovery $\le 5$s |
| 2 | Áp dụng Retry Budget cấp độ ứng dụng tối đa 10% và Fallback Ladder tự động sang `alpha-eu` / `beta` | Prevent | Core Infra | Chạy kiểm thử G2 đạt availability $\ge 98\%$ khi alpha mất 75% GPU |
| 3 | Triển khai hệ thống cảnh báo đa cửa sổ Multi-window Multi-burn-rate | Detect | SRE Team | Phát hiện sự cố quá tải và gửi Page trong vòng $\le 5$ phút, 0 false page |

## 7. Một điểm yếu còn lại

- **Tình huống yếu:** Khi provider sơ cấp `alpha` gặp sự cố kéo dài, client chuyển phần lớn lưu lượng sang `beta`. Mặc dù hệ thống duy trì được availability 100%, chi phí (`cost_per_call`) của `beta` đắt hơn 40% (1.4 so với 1.0 của alpha), đồng thời rate limit của beta thấp hơn (16 rps so với 20 rps của alpha). Nếu đợt chuyển vùng kéo dài trong giờ cao điểm, beta có thể bị nghẽn rate limit dây chuyền hoặc làm ngân sách chi phí LLM tăng vọt ngoài tầm kiểm soát.
- **Cách khắc phục:** Cài đặt cơ chế **Cost-aware Shedding / Degradation**: khi phải fallback sang `beta`, chỉ các request quan trọng (ví dụ xử lý hoàn tiền VIP hoặc hội thoại phức tạp) mới đi `beta`; các tác vụ FAQ hoặc tác vụ thông thường chuyển sang mô hình nén context hoặc chuyển một phần sang `alpha-eu` (cùng giá 1.0). Đồng thời dùng probe có trọng số để chủ động tái hồi lưu lượng về `alpha` ngay khi `alpha` vừa xuất hiện tín hiệu khỏe mạnh.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9124 | 0.9645 | 0.652 | 157 | 0 | 3 | 0.915 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.90 | Ngưỡng 0.80 quá lỏng làm sinh ra 157 lỗi `other_question` (trả lời nhầm câu hỏi khác). Tăng lên 0.90 triệt tiêu hoàn toàn lỗi này (`wrong_answers = 0`). |
| ttl_s | 60s | Khi carrier âm thầm thay đổi bảng phí ship (không có webhook ở t=480), TTL 60s giới hạn số câu trả lời cũ chỉ ở mức 3 câu (`outdated_answers = 3 <= 15`). |
| max_stale_s | 300s | Cung cấp cửa sổ stale 300s (lớn hơn thời gian outage 120s từ t=240-360), cho phép phục vụ 157 câu stale trong lúc mọi provider sập, nâng `faq_availability` lên 96.45%. |

Thí nghiệm (mỗi cái 2–4 câu + số):

1. **Ngưỡng similarity:**
   - Ở threshold = 0.80: hit ratio = 0.666, nhưng có **157** `wrong_answers` (toàn bộ do lý do `other_question`: các câu hỏi có ngữ nghĩa tương tự nhưng bản chất khác nhau như "thời gian hoàn tiền" bị nhầm thành "chính sách hoàn tiền"), cost/turn = 0.965, avail = 0.8561.
   - Ở threshold = 0.90: hit ratio = 0.652, `wrong_answers` = **0**, cost/turn = 0.915, avail = 0.9124.
   - Ở threshold = 0.97: ngưỡng quá khắt khe khiến hit ratio tụt xuống 0.581, cost/turn tăng lên 0.964, `wrong_answers` = 0. Ngưỡng tối ưu tuyệt đối là 0.90.

2. **TTL:**
   - Khi chạy với `ttl_s = 30s`: `outdated_answers = 2`, hit ratio = 0.592.
   - Khi chạy với `ttl_s = 60s`: `outdated_answers = 3`, hit ratio = 0.652.
   - Khi chạy với `ttl_s = 300s`: `outdated_answers = 95` (vượt xa giới hạn cho phép $\le 15$), hit ratio = 0.731.
   - Do sự thay đổi bảng giá ship diễn ra âm thầm ở phía đối tác vận chuyển mà không gửi webhook, cache không thể nhận tín hiệu invalidation chủ động. Vì vậy, TTL chính là cơ chế phòng thủ duy nhất để dọn sạch dữ liệu cũ.

3. **Stale-if-error:**
   - Khi đặt `max_stale_s: 0` (tắt stale-if-error): trong thời gian toàn bộ provider sập (t=240–360), hệ thống không thể phục vụ câu trả lời cũ nào (`stale_served = 0`), khiến `faq_availability` sụt giảm mạnh từ **96.45%** xuống còn **87.33%**.
   - Cơ chế stale-if-error là bậc dự phòng quan trọng giúp người dùng vẫn nhận được thông tin chính sách FAQ thay vì nhận lỗi trắng trang.

Trả lời:

- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?**
  1. *Rủi ro bảo mật & rò rỉ dữ liệu (PII):* Nếu phân scope nhầm hoặc logic matching có sai số, thông tin nhạy cảm của khách hàng (địa chỉ, số điện thoại, số tiền) có thể bị lộ cho người khác.
  2. *Dữ liệu thay đổi liên tục theo thời gian thực:* Trạng thái đơn hàng biến động theo từng giây (đang giao, đã nhận). Phục vụ dữ liệu cache cũ sẽ dẫn đến việc khách hàng nhận thông tin sai lệch, gây ra trải nghiệm tồi tệ và khiếu nại dịch vụ.

- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**
  - Không được dựa vào chỉ số availability ở tầng ngoài cùng của người dùng (vì đã bị cache che mờ).
  - Cần thiết lập alert dựa trên:
    1. **Tốc độ phục vụ stale (`stale_served_rate > 0`)**: Báo động ngay lập tức khi phát hiện hệ thống phải dùng dữ liệu cũ để trả lời.
    2. **Tỷ lệ lỗi backend của provider (`upstream_error_rate` hoặc `upstream_5xx_rate`)**: Đo lường trực tiếp tỷ lệ lỗi từ các call đi ra ngoài provider LLM.
    3. **SLO Burn-rate đo riêng trên upstream LLM calls**: Kích hoạt Page khẩn cấp cho on-call khi upstream provider bị sập, bất kể tỷ lệ cache hit của tầng giao diện người dùng đang cao thế nào.
