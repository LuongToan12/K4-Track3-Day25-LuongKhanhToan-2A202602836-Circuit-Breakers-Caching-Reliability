# Lab 25B — Report

**Họ tên / nhóm:** Lương Khánh Toàn - 2A202602836  **Commit:** a3f9ce7078c14d183ed1e95d311a2df59bc86855  **Điểm tự động (`make grade`):** 84.0 / 88 (+ 4đ SLO hidden của giảng viên)

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
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Dựa trên G6 và G4: Latency bình thường của các model từ 0.6s - 1.8s. Đặt 8.0s giúp phát hiện provider bị treo/chết sớm hơn nhiều so với turn deadline (30s), chừa đủ thời gian failover sang provider khác. Nếu đặt 30s (Thí nghiệm 2), có 122 request bị hết hạn deadline và availability tụt xuống 94.88%. |
| max_attempts_per_provider | 2 | Dựa trên G2 và G3: Cho phép đúng 1 lần retry trên cùng provider (attempt 1 lỗi -> retry 1 lần). Nếu tiếp tục lỗi thì chuyển ngay sang fallback ladder. NaiveClient cho phép tới 36 attempts làm amplification vọt lên 9.904 ở G2 và 1.164 ở G3. Với giá trị 2, amplification ở G2 chỉ còn 1.041 và G3 retry_ratio = 0.034 <= 0.10. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Dựa trên G1 và G2: Full jitter với base 0.25s và cap 4.0s phân tán đều các đợt retry, triệt tiêu thundering herd. Đồng thời tôn trọng `retry_after` (không retry sớm hơn thời gian server yêu cầu), giúp G1 đạt 0 vi phạm `retry_after_violations` và availability đạt 100%. |
| retry_budget_ratio | 0.1 | Dựa trên G3: Giới hạn lượng retry không vượt quá 10% lượng request gốc. Ở G3 (brownout 30% lỗi 500), naive client có retry_ratio = 0.164 (vi phạm tiêu chí <= 0.10), trong khi client của ta giữ retry_ratio ở mức 0.034 (đạt chuẩn). |
| breaker.window_size / min_calls | 20 / 10 | Dựa trên G2, G3, G5: Cần ít nhất 10 mẫu kết quả để tính tỷ lệ lỗi, tránh trường hợp breaker bị "giật mình" mở sớm (false trip) chỉ vì 1-2 request lỗi ngẫu nhiên ban đầu. Cửa sổ trượt 20 kết quả đảm bảo đủ độ nhạy để phát hiện sự cố nhanh chóng. |
| breaker.failure_rate_threshold | 0.5 | Dựa trên G2: Khi alpha mất 75% GPU và trả về 529 liên tục, tỷ lệ lỗi nhanh chóng vượt ngưỡng 50%, kích hoạt breaker mở ngay sang OPEN để chuyển hướng toàn bộ traffic sang `alpha-eu` và `beta`, giúp availability đạt 100%. |
| breaker.slow_call_s | 5.0 | Dựa trên G4: Ở G4, provider alpha phản hồi chậm thêm 5s (tổng latency ~6.2s) nhưng không trả về bất kỳ mã lỗi HTTP nào. Nhờ ngưỡng `slow_call_s = 5.0`, breaker nhận diện call chậm và trip sang `alpha-eu`, hạ latency p95 từ 13.764s (naive) xuống còn 5.562s (đạt chuẩn <= 10s). |
| breaker.open_duration_s | 10.0 | Dựa trên G2: Giữ trạng thái OPEN trong 10 giây trước khi thử nghiệm HALF-OPEN. Khoảng thời gian này vừa đủ để hàng đợi của provider bị nghẽn kịp xả bớt request tồn đọng mà không phải chờ đợi quá lâu. |
| breaker.half_open_max_probes | 2 | Dựa trên G2: Ở trạng thái HALF-OPEN chỉ cho phép tối đa 2 request thăm dò bay đồng thời. Điều này ngăn chặn hoàn toàn hiện tượng thundering herd đè bẹp provider vừa mới chớm hồi phục. |

Sơ đồ quyết định của `on_error` (loại lỗi → ghi breaker? → retry? → failover? → cooldown?):

```
+----------------------------------------------------------------------------------------------------+
|                                    on_error(attempt, err, now)                                      |
+----------------------------------------------------------------------------------------------------+
                                                  |
                                   Phân loại lỗi (classify(err))
                                                  |
        +-----------------------+-----------------------+-----------------------+
        |                       |                       |                       |
   CLIENT_ERROR           QUOTA_EXHAUSTED          RATE_LIMITED         OVERLOADED /
  (400, 401, 422)         (429 quota code)      (429 rate limit)      SERVER_ERROR (5xx) /
        |                       |                       |             TIMEOUT (0, 408)
        v                       v                       v                       |
  release permit          release permit          release permit                v
 (không tính breaker)    (không tính breaker)    (không tính breaker)      on_failure(permit)
        |                       |                       |              (tính vào breaker window)
        v                       v                       v                       |
 GiveUp("client_error")  cooldown = now + 300s   cooldown = now + retry_after   v
 [Không retry,           [Failover ngay lập tức  [Failover ngay lập tức   tries < max_attempts (2)
  Không failover]         sang provider khác]     sang provider khác]     AND budget.try_acquire()
                                                                          AND fits deadline?
                                                                                |
                                                                       +--------+--------+
                                                                       |                 |
                                                                     YES                 NO
                                                                       |                 |
                                                                       v                 v
                                                               Retry cùng provider   Failover sang
                                                               với full jitter delay  provider kế tiếp
                                                               (delay_s = backoff)   trong ladder
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.**
   - *Số liệu thực nghiệm:* Khi breaker bị vô hiệu hóa (`min_calls = 10^9`) và `max_attempts_per_provider = 5`:
     - Trên **G2_overload_storm**: Có budget: `amplification = 1.383`, `retry_ratio = 0.058`, `recovery_s = 0.0s`, `availability = 0.9987`. Tắt budget: `amplification = 1.672`, `retry_ratio = 0.491`, `recovery_s = 30.0s`, `availability = 0.9649`.
     - Trên **G3_brownout**: Có budget: `amplification = 1.136`, `retry_ratio = 0.068`, `recovery_s = 0.0s`. Tắt budget: `amplification = 1.163`, `retry_ratio = 0.162`, `recovery_s = 0.0s`.
   - *Giải thích cơ chế:* Khi breaker bị lỗi cấu hình hoặc không kịp mở, các client thiếu kiểm soát sẽ liên tục retry 5 lần cho mỗi lỗi, khiến `retry_ratio` trên G2 vọt lên 0.491 (gấp ~8.5 lần) và làm chậm thời gian hồi phục thêm 30s. Retry budget hoạt động như một "cầu chì bảo hiểm cuối cùng" (safety cap), siết chặt tổng lượng retry dưới 10% lưu lượng bất kể sự cố nghiêm trọng đến đâu. Khi breaker hoạt động đúng thì budget vẫn bắt buộc phải có vì: (1) Breaker cần tích lũy đủ `min_calls` mới bắt đầu mở, trong giai đoạn chuyển tiếp này chỉ có budget mới chặn được retry storm tức thời; (2) Breaker bảo vệ từng provider riêng lẻ, trong khi budget bảo vệ toàn hệ thống trước việc retry đồng loạt trên nhiều tầng; (3) Phòng ngừa lỗi lập trình hoặc cấu hình sai breaker trong production.

2. **`attempt_timeout_s: 30`.**
   - *Số liệu thực nghiệm:*
     - Trên **G6_provider_outage**: Bình thường (timeout 8s): `availability = 1.0000`, `latency_p95_s = 6.016s`, `give_up = {}`. Khi timeout 30s: `availability = 0.9488`, `latency_p95_s = 5.306s`, `give_up = {'deadline': 122}` (có tới 122 request bị drop do hết deadline).
     - Trên **G4_slow_region**: Bình thường: `availability = 1.0000`, `latency_p95_s = 5.604s`. Khi timeout 30s: `availability = 1.0000`, `latency_p95_s = 5.683s`.
   - *Hiện tượng Survivorship Bias:* Ở G6 khi đặt timeout 30s, p95 tưởng như "giảm" từ 6.016s xuống 5.306s nhưng thực chất availability bị tụt xuống 94.88%. Nguyên nhân là do metric `latency_p95_s` chỉ tính trên các request *thành công*. Những request gửi vào provider chết bị treo quá lâu, ngốn hết deadline 30s của người dùng nên bị hủy (`give_up: deadline`) và bị loại khỏi mẫu tính p95; chỉ những request may mắn gửi trúng provider sống mới được ghi nhận.
   - *Vì sao G4 gần như không đổi:* Ở G4, provider alpha phản hồi chậm thêm 5s (tổng latency ~6.2s < 8s), không có request nào chạm mốc timeout. Breaker nhận diện qua điều kiện `slow_call_s = 5.0` và tự động trip sang `alpha-eu` dựa trên slow call rate chứ không hề dựa vào cơ chế timeout.
   - *Cơ sở xác định timeout đúng:* Timeout không bao giờ được đặt bằng deadline của user (30s). Timeout phải dựa trên độ trễ phân vị cao lúc bình thường (p99 ~ 1.5s - 2.0s) cộng thêm một biên độ an toàn nhỏ (thường từ 6s - 8s), và bắt buộc phải nhỏ hơn `user_deadline / số_bước_fallback` để đảm bảo client vẫn còn đủ thời gian thực hiện ít nhất một lần failover sang provider dự phòng trước khi người dùng rời đi.

3. **Bỏ lọc tool calling khỏi `ladder()`.**
   - *Số liệu thực nghiệm:* Chạy G6 khi bỏ lọc tool: `availability = 0.8918`, xuất hiện `incompatible_routes = 258`.
   - *Giải thích dưới góc nhìn Product Manager:* Khi nhà cung cấp chính gặp sự cố, hệ thống thiếu cơ chế lọc năng lực đã tự ý chuyển 258 yêu cầu chăm sóc khách hàng cần gọi công cụ (như kiểm tra trạng thái đơn hàng, tra cứu mã giao dịch, thực hiện hoàn tiền) sang model giá rẻ `gamma-mini` - một mô hình hoàn toàn không có khả năng tool calling. Hậu quả là 258 khách hàng nhận về câu trả lời vô nghĩa hoặc báo lỗi hệ thống, khiến tỷ lệ xử lý thành công chung tụt xuống dưới 90%. Đây là lỗi trải nghiệm khách hàng nghiêm trọng vì hệ thống đã định tuyến sai sản phẩm, gây tổn hại trực tiếp đến uy tín dịch vụ dù nhà cung cấp dự phòng có đủ năng lực (`beta`) vẫn đang sẵn sàng.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1,496,700 | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

**Vì sao chỉ idempotency key là chưa đủ, và chỉ checkpoint cũng chưa đủ?**
- *Nếu chỉ có Idempotency Key:* Khi worker crash giữa chừng, worker mới khởi động lại sẽ không biết workflow đã hoàn thành tới bước nào và phải chạy lại từ bước đầu tiên (`classify_request`, `lookup_order`). Mặc dù cổng thanh toán không trừ tiền lần hai nhờ idempotency key, nhưng hệ thống bị lãng phí rất nhiều lượt gọi LLM (`wasted_llm_per_crash > 1.1`), làm tăng chi phí token và kéo dài thời gian xử lý của khách hàng.
- *Nếu chỉ có Checkpoint:* Khi gặp sự cố mạng (ví dụ `ServiceTimeout` - lệnh hoàn tiền đã thành công trên cổng thanh toán nhưng gói tin phản hồi bị mất trên đường truyền), worker không có checkpoint nên buộc phải thực thi lại bước đó. Nếu không có idempotency key để bên thanh toán nhận diện trùng lặp, một giao dịch hoàn tiền hoàn toàn mới sẽ được tạo ra, dẫn đến việc hoàn tiền trùng lặp (`duplicate_refunds = 128`, làm thất thoát gần 15.000 USD trong bài test của NaiveExecutor).
- *Điều gì xảy ra nếu ghi checkpoint TRƯỚC khi gọi payment?* Nếu ghi checkpoint trước khi gọi payment mà worker bị crash ngay lúc đang gửi lệnh hoặc trước khi cổng thanh toán xác nhận thành công, worker tiếp theo khởi động lại sẽ kiểm tra thấy checkpoint đã tồn tại và lầm tưởng rằng tiền đã được hoàn tất. Bước thanh toán sẽ bị bỏ qua vĩnh viễn, dẫn đến lỗi **mất tiền hoàn của khách hàng** (`lost_refunds > 0`), vi phạm nghiêm trọng cam kết dịch vụ.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page (phát hiện) | page (phát hiện) | 27 min |
| provider_outage | page | page (phát hiện) | page (phát hiện) | 4 min |
| deploy_blip | none | page (FALSE PAGE) | none (không page) | - |
| slow_leak | ticket | none (BỎ LỌT) | ticket (phát hiện) | 1337 min |
| degraded_evening | page | page (phát hiện) | page (phát hiện) | 291 min |
| night_spike | none | page (FALSE PAGE) | none (không page) | - |
| small_bump | none | page (FALSE PAGE) | none (không page) | - |

- **Error budget 30 ngày:** Tổng requests: 25,901,755; Lỗi cho phép (0.5%): 129,508.8; Lỗi thực tế: 55,618; Availability đạt **99.7853%** (vượt SLO 99.50%). Error budget đã tiêu thụ: **42.95%**, còn lại: **57.05%**.
- **Quyết định release:** `freeze_releases: False` -> **Không đóng băng release**, hệ thống vẫn còn dư 57.05% ngân sách lỗi để tiếp tục triển khai các tính năng mới an toàn.
- **Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?** `small_bump` chỉ tiêu tốn một phần cực nhỏ ngân sách lỗi (~0.05% error budget của tháng) và tự hết trong vòng 20 phút mà không đe dọa đến mục tiêu SLO 30 ngày. Naive alerter chỉ nhìn vào cửa sổ ngắn 5 phút với ngưỡng cố định nên ngay lập tức bắn page báo động giả (False Page), gây mệt mỏi cảnh báo (alert fatigue) cho kỹ sư on-call. Ngược lại, Burn-rate alerter sử dụng cơ chế đa cửa sổ (cả short window và long window): với thời lượng chỉ 20 phút, cửa sổ dài 60 phút của quy tắc fast-burn không đạt đủ ngưỡng burn rate 14.4x, do đó thuật toán không kích hoạt cảnh báo page.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Hình biểu đồ: `reports/gameday/G2_overload_storm.png`*
*Dữ liệu chi tiết: `reports/gameday/naive_G2_overload_storm_timeseries.csv`*

**Tóm tắt (3 câu):**
Vào lúc t=120, nhà cung cấp mô hình chính (`alpha`) bị mất 75% năng lực GPU trong 3 phút, dẫn đến phản hồi quá tải 529 và thời gian chờ hàng đợi tăng vọt. Do kiến trúc `NaiveClient` gồm 3 tầng retry chồng chéo với delay cố định và thiếu cơ chế circuit breaker, hệ thống đã tự tạo ra một cơn bão retry (retry storm) khuếch đại tải lên gấp 9.9 lần so với bình thường. Cơn bão này đẩy hệ thống vào trạng thái sập siêu bền (metastable failure), khiến dịch vụ tiếp tục bị tê liệt thêm 155 giây ngay cả sau khi nhà cung cấp đã khôi phục hoàn toàn hạ tầng phần cứng.

**Ảnh hưởng:**
- Tổng số lượt người dùng bị ảnh hưởng: Tỷ lệ sẵn sàng chỉ đạt 45.60% (hơn 54% tổng số lượt yêu cầu của khách hàng bị thất bại).
- Thời gian gián đoạn: Kéo dài từ t=120 đến t=455 (tổng cộng 335 giây, trong đó có 155 giây sập kéo dài sau khi hạ tầng đã hết lỗi).
- Tiêu thụ ngân sách lỗi: Sự cố tiêu tốn xấp xỉ 78% error budget của cả tháng nếu chỉ xét riêng sự cố này.

**Timeline (giây ảo — trích xuất từ `naive_G2_overload_storm_timeseries.csv`):**

| t | Sự kiện | Bằng chứng định lượng từ CSV |
|---:|---|---|
| 110 | Trạng thái bình thường trước sự cố | 9/9 call thành công (100%), tải gửi tới alpha chỉ 5 attempts/s. |
| 120 | Trigger: alpha mất 75% GPU | Năng lực phục vụ giảm còn 25%, 7/7 call thành công, 5 attempts/s. |
| 130 | Hàng đợi bắt đầu dồn ứ | Tải tăng lên 10 attempts/s, độ trễ bắt đầu tăng nhưng vẫn còn 3/3 call thành công. |
| 150 | Bùng nổ retry storm | 0/3 call thành công (0% availability); số attempt dội vào alpha vọt lên 163 attempts/s (gấp 32 lần tải bình thường). |
| 200 | Toàn bộ hệ thống nghẽn hoàn toàn | 0/1 call thành công; 135 attempts/s liên tục gửi vào alpha, client liên tục timeout ở giây thứ 8. |
| 250 | Đỉnh điểm bão retry | 0/6 call thành công; tải đạt đỉnh 184 attempts/s, hàng đợi alpha tràn 10s dẫn tới lỗi 529 hàng loạt. |
| 300 | Alpha hồi phục 100% GPU phần cứng | Hạ tầng provider đã có lại 18 server slot, nhưng 0/5 call thành công; tải dội vào vẫn ở mức 115 attempts/s. |
| 350 | Metastable failure tự duy trì | Sau 50s kể từ khi GPU hồi phục, 0/4 call thành công; tải vẫn duy trì 17 attempts/s do các vòng lặp retry cũ. |
| 400 | Hàng đợi server tiếp tục bị nghẹt | Sau 100s kể từ khi GPU hồi phục, 0/5 call thành công; tải tiếp tục dội lên 51 attempts/s. |
| 455 | Bắt đầu thoát khỏi bão retry | Hàng đợi xử lý hết các request cũ, 4/5 call thành công (80% availability), tải giảm về 13 attempts/s. |
| 460 | Hệ thống hồi phục hoàn toàn | 9/12 call thành công (75% - 100%), hệ thống quay lại trạng thái ổn định (recovery_s = 155.0s). |

**Trigger vs Root cause:**
- **Trigger (Ngòi nổ):** Nhà cung cấp `alpha` bị mất 75% GPU phần cứng trong khoảng thời gian từ t=120 đến t=300, làm giảm thông lượng tức thời của cụm máy chủ.
- **Root Cause (Nguyên nhân gốc rễ):** Hệ thống client (`NaiveClient`) thiếu toàn bộ các lớp bảo vệ cơ bản của một hệ thống phân tán:
  1. *Ba tầng retry nhân bản:* Tầng ứng dụng (x4) × Gateway (x3) × SDK (x3) = tối đa 36 lần thử cho một call duy nhất.
  2. *Không có Full Jitter & Backoff:* Sử dụng khoảng thời gian chờ cố định 0.2s khiến toàn bộ các request thất bại đồng loạt quay lại gõ cửa server cùng một tích tắc.
  3. *Thiếu Circuit Breaker:* Không ngắt mạch khi provider suy thoái, không failover sang `alpha-eu` hay `beta`.
  4. *Thiếu Retry Budget:* Không có giới hạn trần cho lưu lượng retry, biến một đợt giảm tải tạm thời thành đợt khuếch đại tải gấp 10 lần.
- **Vì sao provider đã hồi phục mà hệ thống vẫn sập (Metastable Failure)?**
  Máy chủ xử lý theo hàng đợi FIFO và tiếp tục tốn tài nguyên GPU xử lý các request cũ ngay cả khi client đã quá hạn (timeout). Khi t=300, hàng đợi của alpha vẫn đang nghẹt cứng bởi hàng trăm request cũ mà client đã từ bỏ từ t=250. Trong khi đó, các lượt request mới của user tiếp tục kích hoạt 36 lần retry đè tiếp vào đuôi hàng đợi. Nhu cầu gửi đến liên tục vượt quá năng lực phục vụ tối đa, tạo thành một vòng lặp tự duy trì (self-sustaining overload loop) khiến hệ thống không thể tự thoát ra được cho đến khi toàn bộ chuỗi retry cũ hết hạn đồng loạt ở t=455.

**Điều gì giúp / Điều gì làm tệ hơn:**
- *Làm tệ hơn:* Càng tăng số lần retry, giảm delay retry hoặc tăng timeout sẽ càng đổ thêm dầu vào lửa, kéo dài thời gian sập của hệ thống.
- *Giúp ích:* Triển khai Circuit Breaker mở mạch ngay khi tỷ lệ lỗi vượt 50% để chuyển lưu lượng sang provider phụ; áp dụng Retry Budget khống chế retry <= 10%; áp dụng Full Jitter để giãn cách thời gian gửi; và kiểm tra deadline để hủy ngay các request không còn cơ hội hoàn thành.

**Action items (cụ thể, có owner, đo lường được):**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay thế `NaiveClient` bằng `ResilientClient`: tích hợp sliding-window circuit breaker và full jitter backoff | Prevent | Core Agent Team | Chạy CI mô phỏng kịch bản G2: `amplification <= 1.2` và `recovery_s <= 5s` trên mọi seed ngẫu nhiên. |
| 2 | Thiết lập Retry Budget ở cấp độ Gateway (token bucket ratio = 0.1, max 20 tokens) | Prevent | Infra / Gateway Team | Kiểm tra tải tải đỉnh: tổng số retry không bao giờ vượt quá 10% tổng request trong các bài test chaos engineering. |
| 3 | Xây dựng bảng điều khiển SLO và cấu hình cảnh báo Multi-window Burn-rate (Fast-burn 14.4x và Slow-burn 6.0x) | Detect | SRE / On-call Team | Chạy test suite `test_alerting.py` và kiểm thử giả lập sự cố trên dữ liệu 30 ngày đạt 0 False Page. |
| 4 | Bổ sung kiểm tra hạn chót (Deadline Propagation) trước khi gửi request tới LLM provider | Mitigate | Application Team | Kiểm chứng unit test: không có bất kỳ request nào được gửi đi nếu thời gian còn lại nhỏ hơn 1.0 giây. |

## 7. Một điểm yếu còn lại

**Tình huống client vẫn xử lý chưa tối ưu: Suy thoái dây chuyền và chi phí bùng nổ khi Provider chính sập dài hạn.**
- *Mô tả vấn đề:* Khi `alpha` gặp sự cố kéo dài (như ở G6 hoặc G2), client tự động chuyển hướng toàn bộ lưu lượng sang `alpha-eu` và sau đó là `beta`. Tuy nhiên, `beta` có chi phí cao hơn 40% (cost = 1.4 so với 1.0) và giới hạn rate limit thấp hơn (16 rps so với 20 rps của alpha). Nếu lượng traffic lớn dồn đột ngột sang `beta`, `beta` sẽ nhanh chóng bị quá tải và chạm trần rate limit. Hơn nữa, nếu cả 4 provider cùng bị suy thoái mạng diện rộng (chậm đồng loạt), circuit breaker của từng provider sẽ lần lượt mở ra (cascading failure), dẫn đến việc client liên tục đảo vòng qua các provider (thùng rỗng kêu to) rồi cuối cùng đồng loạt GiveUp.
- *Giải pháp cải tiến:*
  1. *Spillover Rate Limiting / Load Shedding:* Thay vì dồn 100% traffic tràn sang provider tiếp theo, triển khai thuật toán phân bổ tải mềm (weighted load balancing) kết hợp chủ động dropped bớt các yêu cầu không khẩn cấp khi nhận thấy dung lượng dự phòng của `beta` sắp cạn.
  2. *Adaptive Concurrency Limit (như TCP Vegas / Netflix Concurrency-limits):* Đo lường round-trip time (RTT) thời gian thực của từng provider để chủ động hạ số request đồng thời cho phép trước khi provider kịp bị nghẽn hàng đợi hoặc trả về mã lỗi 529.

## 8. Cache (Part G)

Số từ `reports/gameday/cached.json` và `reports/gameday/resilient.json` (`make cache`):

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9113 | 0.9634 | 0.634 | 172 | 0 | 6 | 0.928 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Ở ngưỡng 0.80 (mặc định), cache bị 138 lỗi `wrong_answers` (lý do: `other_question: 138`) do trả nhầm câu trả lời của câu hỏi khác có ngữ nghĩa gần giống. Ở ngưỡng 0.97, tỷ lệ hit tụt xuống 55.2% và chi phí tăng lên 0.984. Ngưỡng 0.92 là điểm tối ưu tuyệt đối: `wrong_answers = 0` mà vẫn duy trì `cache_hit_ratio = 63.4% >= 50%`. |
| ttl_s | 45 | Tại t=480, bảng giá phí vận chuyển thay đổi âm thầm mà không có webhook báo trước. Nếu để TTL = 300s, hệ thống có tới 95 câu trả lời lỗi thời (`outdated_answers = 95`). Đặt TTL = 45s khống chế `outdated_answers = 6` (đạt chuẩn <= 15) mà vẫn giữ được hit ratio cao (63.4%) và chi phí thấp (0.928). |
| max_stale_s | 180 | Tại t=240 đến t=360 (kéo dài 120 giây), toàn bộ 4 provider đều sập hoàn toàn. Nếu đặt `max_stale_s = 0`, `faq_availability` chỉ đạt 86.34% (FAIL). Với `max_stale_s = 180s`, cache phục vụ được 172 câu trả lời cũ trong lúc nhà cung cấp sập, nâng `faq_availability` lên 96.34% (PASS >= 93%). |

**Thí nghiệm:**

1. **Ngưỡng similarity:**
   - *Số liệu thực nghiệm:*
     - Ngưỡng `0.80`: `cache_hit_ratio = 0.646`, `wrong_answers = 138` (nguyên nhân: `{'other_question': 138}`), `cost/turn = 0.972`, `faq_availability = 0.8831`.
     - Ngưỡng `0.92`: `cache_hit_ratio = 0.634`, `wrong_answers = 0`, `cost/turn = 0.928`, `faq_availability = 0.9634`.
     - Ngưỡng `0.97`: `cache_hit_ratio = 0.552`, `wrong_answers = 0`, `cost/turn = 0.984`, `faq_availability = 0.9628`.
   - *Phân tích:* Khi đặt ngưỡng lỏng lẻo 0.80, mô hình nhúng coi các câu hỏi có từ khóa tương đồng là một (ví dụ: "Bao lâu thì nhận được tiền hoàn?" và "Chính sách hoàn tiền thế nào?"), dẫn đến 138 trường hợp trả lời hoàn toàn sai thông tin khách hàng cần tìm. Đặt ngưỡng 0.92 loại bỏ 100% các trường hợp nhận nhầm mà vẫn giữ được tỷ lệ hit trên 63%.

2. **TTL:**
   - *Số liệu thực nghiệm:*
     - `ttl_s = 30`: `outdated_answers = 2`, `hit_ratio = 0.591`, `cost/turn = 0.956`.
     - `ttl_s = 60`: `outdated_answers = 3`, `hit_ratio = 0.652`, `cost/turn = 0.916`.
     - `ttl_s = 300`: `outdated_answers = 95`, `hit_ratio = 0.731`, `cost/turn = 0.863`.
   - *Vì sao webhook không giúp được gì cho `outdated_answers`?* Tại thời điểm t=480, đối tác vận chuyển âm thầm thay đổi bảng giá cước phí mà **không hề gửi webhook thông báo** (`silent_updates`). Vì không có webhook, hệ thống không thể chủ động gọi hàm `invalidate()`. Do đó, cơ chế hết hạn theo thời gian (TTL) là phòng tuyến duy nhất giúp dọn sạch các câu trả lời giá cước cũ sau khi bảng giá mới có hiệu lực.

3. **Stale-if-error:**
   - *Số liệu thực nghiệm:*
     - Khi `max_stale_s = 0` (tắt stale-if-error): `faq_availability = 0.8634`, `stale_served = 0`, `cache_hit_ratio = 0.534`.
     - Khi `max_stale_s = 180` (bật stale-if-error): `faq_availability = 0.9634`, `stale_served = 172`, `cache_hit_ratio = 0.634`.
   - *Phân tích:* Khi mọi provider bị sập từ t=240 đến t=360, việc cho phép trả lời câu hỏi FAQ bằng dữ liệu trong bộ nhớ đệm đã hết hạn tươi (stale) đã cứu được 172 lượt truy vấn của khách hàng, giúp độ sẵn sàng FAQ tăng vọt thêm đúng 10.0% (từ 86.34% lên 96.34%), biến một bài test trượt thành đạt chuẩn xuất sắc.

**Trả lời câu hỏi:**
- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?**
  1. *Nguy cơ rò rỉ dữ liệu nghiêm trọng:* Nếu lưu câu hỏi cá nhân vào cache dùng chung, khách hàng B hỏi về đơn hàng có thể nhận được thông tin đơn hàng của khách hàng A (`other_user`), vi phạm nghiêm trọng quyền riêng tư.
  2. *Dữ liệu thay đổi theo thời gian thực:* Trạng thái đơn hàng, điểm tích lũy và số tiền hoàn thay đổi liên tục theo từng giây. Nếu cache lại câu trả lời theo từng user, người dùng sẽ liên tục nhìn thấy trạng thái cũ dù đơn hàng đã được giao hoặc đã hoàn tiền.
  3. *Tỷ lệ cache hit cực thấp:* Mỗi đơn hàng là duy nhất cho từng người dùng, khả năng một user lặp lại chính xác câu hỏi đó trong khoảng thời gian TTL ngắn là rất thấp, việc lưu trữ chỉ gây lãng phí dung lượng bộ nhớ đệm (LRU cache pollution).
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**
  Khi dashboard tổng thể vẫn hiển thị màu xanh (nhờ cache và stale-if-error âm thầm gánh tải), đội ngũ SRE tuyệt đối không được chỉ dựa vào metric `user_availability`. Thay vào đó, cần thiết lập cảnh báo dựa trên:
  1. **`stale_served_ratio` / `stale_served_count` > 0:** Báo động ngay lập tức khi hệ thống bắt đầu phải dùng đến câu trả lời stale để phục vụ người dùng.
  2. **Tỷ lệ lỗi trực tiếp từ Backend tới Provider (Backend Error Rate / 5xx Rate):** Giám sát trực tiếp tỷ lệ lỗi HTTP 5xx, 529 và timeout ở tầng mạng giữa server của ta và các provider LLM.
  3. **Trạng thái Circuit Breaker (`breaker_state == OPEN`):** Bắn alert mức độ nghiêm trọng cao (Page) ngay khi circuit breaker của provider chính hoặc toàn bộ các provider chuyển sang trạng thái OPEN.
