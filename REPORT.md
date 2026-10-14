MSSV: 202602682

# Lab 11 · Scenario Forge — báo cáo

Họ tên: **Trần Ngọc Khuyến**. MSSV dùng cho pipeline: **202602682**; thị trấn **B** (`meta.json: _run.mssv, _run.town`). Chạy `python run_all.py all` với hazard đã có, ngân sách đầy đủ, không `--quick`. 31/31 test TODO và 10/10 test parity pass; không sửa `avsim/`, `run_all.py`, `tests/`, `policy_candidates.json` hay cách sinh seed. Nhóm người đi bộ trong phiên bản repo này tên `crosswalk_block`, gồm sự kiện block và tiếp xúc P1.

## 1. Phát lại log có trung thực không?

`replay.json: n = 400, dropped_human_crash_logs = 2, n_used = 398, crash_in_both = 0`.

| Cách chạy | Thất bại | front | rear | side |
|---|---:|---:|---:|---:|
| Log phát lại | 112 | 0 | 102 | 10 |
| Thế giới phản ứng | 0 | 0 | 0 | 0 |

Nguồn: `replay.json: replay, reactive`; loại vắng mặt trong contact được đếm là 0. Phần lớn là rear vì TG giữ quỹ đạo trong log người lái, không biết planner giảm tốc/phanh sớm hơn; đó là va chạm giả khi dùng log để đánh giá planner. Ở `replay_only[0]` (scenario 0, rear), thế giới phản ứng không va chạm, `min_ttc_reactive = 7.460629498692649`. Chạy lại R-01 trong viewer cho thấy TG đâm ego tại **17,65 s**; thời điểm đọc từ viewer, không phải trường thời gian của replay.json.

![R-01: TG đâm ego trên log phát lại tại 17,65 s](evidence/shots/replay-rear.png)

## 2. Kịch bản bất khả thi

`is_feasible` loại gia tốc ngang chuyển làn `(π/cutDur)²·3,5/2 > μg` (vượt độ bám ngang), `cutBrake > μg` (phanh vượt độ bám), `pedV > 2,5 m/s` (ngoài miền người đi bộ của lab), và `cutV` ngoài `(0,22] m/s` (đứng yên hoặc vượt tốc độ hợp lệ).

`hunt.json: bus_bypass.infeasible = 5, cut_in.infeasible = 4, crosswalk_block.infeasible = 22, rear_ended.infeasible = 7`: **crosswalk_block** sinh nhiều mẫu bất khả thi nhất. Đối chiếu các hàng `out = infeasible` trong bốn đoạn 2.400 hàng của cloud.json cho thấy tất cả vượt gia tốc ngang. CEM chỉ tối ưu điểm nguy hiểm, nên có thể bị hút về chuyển làn quá gấp, làm ego phanh muộn hoặc phanh gấp trong tương tác với xe/người khác. Đếm chúng là va chạm sẽ cộng tình huống không có thật vào xác suất và làm sai hướng đề xuất; pipeline đánh dấu, không mô phỏng chúng.

## 3. Monte Carlo cho biết gì?

`mc.json: crashes = 4, n = 3000, p = 0.0013333333333333333, lo = 0.0005186172629797156, hi = 0.0034235346800521676, infeasible = 11`. Khoảng Wilson 95% là **[5.1861726e-4, 3.4235347e-3]**, còn rộng vì mới thấy bốn vụ.

Để hướng tới ±20% của p, `n ≈ 1,96²/(0,2²·p) ≈ 96/p`: cần khoảng **72,000 lần chạy** (72.030 nếu giữ 1,96²). Bốn vụ đều là bus_bypass; cut_in, crosswalk_block, rear_ended và other chưa xuất hiện. Với từng kiểu đó, `mc.json: per_mode.<kiểu>.k = 0, hi = 0.0012788956648046956`: báo **p ≤ 1.2788957e-3** theo Wilson 95%, không kết luận xác suất bằng 0.

## 4. Importance sampling, ESS và độ suy biến

Nguồn: `estimate.json: seeds[s].per.<kiểu>.{p,hits,ess,top_share,upper_zero_hits}`; mỗi hàng `n = 600`. Với kiểu chưa thấy, cột xác suất dùng `upper_zero_hits = 3/(0,2·600) = 0,025`, không dùng khoảng [0,0] của công thức chuẩn, và không dùng 3/ESS.

| Seed | Kiểu | p̂ hoặc cận khi chưa thấy | hits | ESS | top_share |
|---:|---|---:|---:|---:|---:|
| 0 | bus_bypass | 7.3272502e-4 | 26 | 120.54 | 0.2159 |
| 0 | cut_in | ≤ 2.5000000e-2 (cận) | 0 | 118.23 | 0.0000 |
| 0 | crosswalk_block | 3.7294901e-5 | 97 | 113.10 | 0.8853 |
| 0 | rear_ended | ≤ 2.5000000e-2 (cận) | 0 | 116.84 | 0.0000 |
| 1 | bus_bypass | 1.4799982e-3 | 33 | 155.60 | 0.1641 |
| 1 | cut_in | ≤ 2.5000000e-2 (cận) | 0 | 126.95 | 0.0000 |
| 1 | crosswalk_block | 1.9968513e-6 | 103 | 115.30 | 0.3016 |
| 1 | rear_ended | ≤ 2.5000000e-2 (cận) | 0 | 107.01 | 0.0000 |
| 2 | bus_bypass | 1.8775171e-3 | 28 | 123.78 | 0.4034 |
| 2 | cut_in | ≤ 2.5000000e-2 (cận) | 0 | 107.96 | 0.0000 |
| 2 | crosswalk_block | 2.5288430e-6 | 110 | 119.24 | 0.1650 |
| 2 | rear_ended | ≤ 2.5000000e-2 (cận) | 0 | 154.11 | 0.0000 |

| Seed | total | lo | hi |
|---:|---:|---:|---:|
| 0 | 7.7001992e-4 | 3.0666092e-4 | 2.5122746e-3 |
| 1 | 1.4819950e-3 | 6.0942177e-4 | 3.6334639e-3 |
| 2 | 1.8800460e-3 | 1.1747923e-4 | 4.9215084e-3 |

Nguồn bảng tổng: `estimate.json: seeds[s].{total,lo,hi}`. Tổng lớn nhất/nhỏ nhất chênh **2.44 lần**; trung vị **1.4819950e-3**, khoảng **1.11 lần** điểm MC, và cả ba điểm tổng đều nằm trong Wilson MC. Tuy nhiên, crosswalk_block seed 0 có `top_share = 0.8852865264572882`: một mẫu gánh **88,53%** ước lượng kiểu này; p̂ của nó lớn hơn seed 1 khoảng **18.68 lần**. Vì vậy tổng có vẻ đồng ý không có nghĩa mọi kiểu đã ổn định. top_share thấp cũng không chứng minh q đã phủ hết vùng thất bại.

`estimate.json: budget = 16800`; cộng MC để lấy other là 19.800, so với khoảng 72.000 mẫu MC để hướng tới ±20%. Nhưng IS chưa đạt cùng độ chính xác đó: nửa độ rộng chuẩn `1,96·se_total/total` của ba seed khoảng **60.2%, 58.9%, 93.8%**, chưa kể cận other. Dùng p̂/se của từng seed và `n_MC ≈ p̂(1-p̂)/se²` để so MC **cùng SE** cho khoảng **13,768, 7,467, 2,321** mẫu; không thể chỉ dựa budget thấp hơn để tuyên bố IS chính xác hơn.

`estimate.json: other.k = 0, other.n = 3000, other.hi = 0.0012788956648046956, other.source = "mc.json"`. Other là phần còn lại ngoài các kiểu được săn, lấy từ MC để giữ tổng đầy đủ; không có điểm nguy hiểm riêng để săn nó. Các trường tổng lo/hi là xấp xỉ, không thay cận 0,025 cho từng kiểu IS chưa thấy.

## 5. Các kiểu thất bại

| Cuộc săn | bus_bypass tìm được | crosswalk_block tìm được | Đúng kiểu mục tiêu? |
|---|---:|---:|---|
| bus_bypass | 109 | 0 | Có |
| cut_in | 31 | 0 | Chưa |
| crosswalk_block | 3 | 78 | Có |
| rear_ended | 12 | 11 | Chưa |

Nguồn: `hunt.json: <cuộc săn>.found`. Chưa tìm được cut_in/rear_ended đúng kiểu; đây là giới hạn bằng chứng, không phải chứng minh chúng không xảy ra. Săn chung dễ tập trung vào bus_bypass và bỏ vùng chặn lối người đi bộ; ngay cuộc săn nhắm cut_in/rear_ended cũng bị hút về các kiểu khác.

**B-01, bus_bypass:** C2 chờ khe rồi lấn sang làn ego để lách xe buýt, đồng thời giảm tốc; ego nhận xe lấn làn muộn, phanh trên đường ướt không đủ tránh tiếp xúc. Ảnh chương 9 tại **12,75 s** cho thấy C2 chậm cạnh xe buýt.

![B-01: va chạm lách xe buýt](evidence/shots/bus-bypass.png)

**C-01, crosswalk_block:** P1 không ngó qua xe buýt, chấp nhận khe ngắn để qua đường; ego chỉ thấy người đang đi khi đã gần vạch, phanh trên đường ướt rồi dừng thân xe chắn lối. P1 đứng chờ; đây là block, không phải người đi bộ bước vào thân xe. Ảnh tại **19,15 s** dùng đầy đủ params của `viewer.json: runs[id="C-01"]`; thời điểm hai ảnh đọc từ viewer.

![C-01: ego dừng chắn lối P1](evidence/shots/crosswalk-block.png)

## 6. Sửa planner trên TRAIN, báo TEST

`fix.json: chosen.policy = {"cutinLat": 3.0}`, tăng từ 2,15 lên 3,0 m để nhận xe lấn làn sớm hơn. `choose_policy` chỉ đọc TRAIN, giữ progress ít nhất 95% baseline, xếp fail_hi thấp nhất rồi ưu tiên progress cao hơn; không thêm hay chọn lại ứng viên theo TEST.

| Tập / policy | fail | fail_hi_seen | fail_hi | progress (m) | Khung phanh gấp trung bình |
|---|---:|---:|---:|---:|---:|
| TRAIN gốc (baseline_train) | 1.8432218e-3 | 3.4215542e-3 | 3.4215542e-3 | 216.198 | 8.977 |
| TRAIN chọn (chosen) | 7.1970404e-6 | 1.5945270e-5 | 1.5945270e-5 | 216.125 | 7.930 |
| TEST gốc (baseline_test) | 6.9524538e-4 | 1.2479496e-3 | 2.5268453e-3 | 214.259 | 8.078 |
| TEST chọn (chosen_test) | 3.7131549e-7 | 7.1867020e-7 | 3.8779614e-2 | 214.066 | 7.844 |

Nguồn: `fix.json: baseline_train, chosen, baseline_test, chosen_test`, cùng các trường ở đầu cột. TEST dùng seed mới, săn lại theo từng policy và 3.000 mẫu tự nhiên riêng; progress tính đến lúc thất bại. Quãng đường TEST giảm **0.0902%**, dưới 5%; số khung phanh gấp cũng giảm.

TEST bản sửa có `bus_bypass.hits = 0, upper_zero_hits = 0,0375` và `other_crashes.k = 0, hi = 0.0012788956648046956`. Vì thế `fail_hi = 0,0375 + 7.1867020e-7 + 1.2788957e-3 = 0.03877961433500541`, cao hơn cả điểm fail của policy gốc dù điểm ước lượng đã giảm. `fail_hi_seen` bỏ phần chưa thấy, không dùng để chứng nhận toàn bộ policy; chưa thể kết luận đã xóa bus_bypass hoặc rủi ro thật giảm theo tỷ số hai điểm ước lượng.

Ảnh dưới chạy lại **đúng params B-01** từ viewer.json với policy chọn làm xe chính, policy gốc làm bóng mờ: tại **12,75 s**, bản sửa giữ khoảng cách, bóng mờ chạm C2. Chỉ đổi policy khi xem so sánh, không làm tròn params hay sửa dữ liệu pipeline.

![B-01: bản sửa tránh va chạm, bóng mờ là policy gốc](evidence/shots/fix-ghost.png)

Một núm nghe an toàn nhưng không đơn điệu: hạ `cautionV` từ 9,0 xuống 8,5 làm TRAIN fail tăng từ **0.0004579925544773799** lên **0.0010487836611174036** (`fix.json: table[3], table[4]`). Ego đến vùng xe buýt vào thời điểm khác, làm khe C2 nhận và thời điểm người đi bộ qua đường đổi; không thể giữ quỹ đạo của các tác nhân khác cố định khi suy luận “chậm hơn luôn an toàn hơn”. Đây là quan sát TRAIN có khoảng bất định, không phải chứng nhận tần suất ngoài đường.

Có phản ví dụ trực tiếp cho chính núm sửa: **GC-01** trong câu 9 va chạm khi cutinLat=3 nhưng policy gốc tránh được. Đối chiếu thêm hai quỹ đạo trên cùng `mine.json: chosen.examples[0].params` cho thấy phanh sớm hơn giữ khe rộng, cho C2 tiếp tục lấn ngang; dưới policy gốc, ego gần hơn khiến C2 tạm dừng chuyển làn và không tiếp xúc. Đây là cơ chế gap acceptance/clearance phản ứng lại planner, không phải một quỹ đạo xe khác phát lại cố định.

## 7. Từ mô phỏng ra đường thật

Ba giả định quan trọng của thị trấn B (`avsim/space.py: VARIANTS`) và dữ liệu cần thay:

| Giả định | Dữ liệu / nơi thu thập | Hướng sai có thể xảy ra |
|---|---|---|
| cutGap log-normal, trung vị 12 m | Camera trên cao hoặc quỹ đạo radar/lidar tại đoạn nhập làn cạnh bến xe buýt; đo khe được nhận và bị từ chối | Nếu tài xế thật nhận khe nhỏ hơn, mô hình có thể đánh giá thấp bus_bypass/cut_in |
| pedLook Bernoulli 0,85 | Video người qua đường tại bến xe buýt, gắn nhãn ngó/chờ/đi và hình học che khuất | Nếu người thật ít ngó hơn, mô hình có thể đánh giá thấp chặn lối hoặc va chạm người đi bộ |
| wet Bernoulli 0,30; μ ướt=0,5, khô=0,8 | Nhật ký hành trình ghép thời tiết/độ ướt mặt đường, đo ma sát bằng thiết bị chuyên dụng trên đúng tuyến | Nếu đường ướt nhiều hơn hoặc độ bám thấp hơn, mô hình có thể đánh giá thấp rủi ro phanh muộn |

Các hướng trên là giả thuyết theo cơ chế; cần hiệu chỉnh cả phụ thuộc giữa mưa, tốc độ và hành vi, rồi kiểm tra độ nhạy vì thế giới phản ứng có thể làm kết quả không đơn điệu. Không suy diễn trực tiếp xác suất mô phỏng thành tần suất tai nạn thật.

## 8. Giới hạn

Xác suất chỉ áp dụng cho một con phố và cửa sổ 24 s, không đại diện mạng đường/hành trình dài; tài xế dùng IDM/gap acceptance và người đi bộ có luật đơn giản, thiếu hành vi mất tập trung; cảm biến/nhận biết được lý tưởng hóa, che khuất dùng hình học hộp, không mô phỏng nhiễu, độ trễ hay lỗi theo dõi thật. Ngân sách cũng không chứng nhận kiểu chưa thấy dưới 10⁻³.

Nếu có thêm một tuần, thu quỹ đạo tại bến xe buýt để hiệu chỉnh ba prior ở câu 7, đánh giá nhiều seed/đề xuất phủ rộng hơn cho IS và retarget policy chọn, rồi thử độ trễ cảm biến và cảnh xe vừa lấn làn vừa phanh; tiếp tục giữ TRAIN/TEST tách biệt.

## 9. Rắc rối mới: xe chuyển làn phanh bất ngờ

Cảnh tải thật từ chương 10: C2 nhận khe 4 m; thẻ brake có `decel = 6,4 m/s², dur = 3 s, t0 = 6 s, trig = 16 m`. Phanh bắt đầu trước lúc C2 tìm khe và kéo dài trong pha chuyển làn, tương tự xe giảm tốc vì giao thông phía trước trong lúc đang nhập làn. Planner dùng ngưỡng ngang để nhận leader, nên có thể bỏ lỡ pha xe ở làn bên đang mất tốc độ và sắp lấn sang. `mine.json: exact.planner = [["ego","C2",7.8]], exact.you = []`: planner gốc chạm C2 ở **7,8 s**, bản ghi chế độ Bạn không va chạm.

`hazard.json: space`: decel [3,1;6,4], dur [1;4], t0 [4;10], trig [12;32], rút đều để kiểm tra độ nhạy. why lấy căn cứ từ [Hu và cộng sự (2015), mục 4.1](https://www.sciencedirect.com/science/article/pii/S2046043016301290): decel trên ngưỡng 10 ft/s² = 3,048 m/s² và tới gần 21 ft/s² = 6,4008; ở v=12 m/s, v/decel khoảng 1,875–3,871 s và v²/(2·decel) khoảng 11,25–23,23 m, làm căn cứ cho dải thời gian/khoảng kích hoạt. t0 nằm trong cửa sổ xe tìm khe của repo để quét phanh trước/trong/sau nhập làn. Mọi mẫu vẫn qua kiểm tra ma sát/khả thi; các dải không được khẳng định là phân phối thực nghiệm.

Nghiên cứu báo trung bình **0,17 lần phanh gấp/phút**; `exposure = 0,17·24/60 = 0.068` lần/chuyến 24 s. Đây là **prior chuyển tần suất một tài xế sang một xe C2 liên quan**, chưa đo tại thị trấn/Việt Nam; tính đại diện của xe đó và phân phối đều cũng là giả định. Kết quả mỗi chuyến chỉ có nghĩa dưới các giả định ghi rõ trong `hazard.json: why`, không phải tần suất tai nạn thật.

`mine.json: world = "town", conditional_on = "your town's traffic", accepted = 0.8573333333333333, infeasible = 214`: giữ **1286/1500 mẫu (85.733%)**, vượt 50%.

| Policy | n | crashes | avoidable | unavoidable | unattributed | low_speed | avoidable_foreknown |
|---|---:|---:|---:|---:|---:|---:|---:|
| Gốc (baseline) | 1286 | 4 | 4 | 0 | 0 | 0 | 4 |
| Chọn (chosen) | 1286 | 1 | 1 | 0 | 0 | 0 | 1 |

Nguồn: `mine.json: baseline, chosen`, cùng các trường ở đầu cột. avoidable là vụ quy được cho hazard và phanh tối đa ngay khi hazard đã bắt đầu **và nhìn thấy** cứu được; unavoidable thì phép thử đó vẫn va chạm; unattributed là va chạm không quy được cho hazard, gồm đâm tác nhân khác hoặc hazard chưa bắt đầu. low_speed là cờ chồng lên các lớp, không cộng thêm vào tổng crashes.

| Policy | P(lỗi planner \| hazard, khả thi) | Wilson lo | Wilson hi | Tần suất mỗi chuyến | Cận trên mỗi chuyến |
|---|---:|---:|---:|---:|---:|
| Gốc | 3.1104199e-3 | 1.2102065e-3 | 7.9704557e-3 | 2.1150855e-4 | 5.4199099e-4 |
| Chọn | 7.7760498e-4 | 1.3727603e-4 | 4.3916522e-3 | 5.2877138e-5 | 2.9863235e-4 |

Nguồn: `mine.json: baseline/chosen.{p,lo,hi,per_trip,per_trip_hi}`. Điểm tần suất hazard gốc khoảng **0.143 lần** trung vị bus_bypass gốc **1.4799982e-3** (`estimate.json: seeds[*].per.bus_bypass.p`); so sánh còn phụ thuộc sai số IS và prior exposure, không cộng hai số vì tình huống có thể chồng lấn. Điểm ước lượng giảm từ 4/1286 xuống 1/1286 nhưng các khoảng còn rộng; bản sửa vẫn có một lỗi avoidable, không tuyên bố đã xử lý hết rắc rối.

![Cảnh hazard: planner gốc chạm C2 ở 7,8 s](evidence/shots/hazard-collision.png)

Ở **GC-01**, `mine.json: chosen.examples[0]` cho `t = 12.45, fire_t = 9.35, brake_t = 9.35`: bản sửa chạm C2 nhưng phanh tối đa từ lúc thấy rắc rối tránh được. Viewer chạy lại cùng params cho thấy policy gốc **không va chạm** ở chính cảnh này; bản sửa đã tạo một vùng thất bại khác qua phản ứng của C2.

![GC-01: bản sửa va chạm, bóng mờ policy gốc không va chạm](evidence/shots/hazard-chosen.png)

Thử chế độ Bạn bằng điều khiển tự động của trình duyệt: giữ tốc độ đến giây 4 rồi phanh −6 m/s²; bàn đạp được viewer ghi trong `hazard.json: egoAcc`, `mine.json: exact.you` xác nhận không va chạm ở cảnh cụ thể. Thử này biết trước cảnh, không chứng minh người lái bình thường sẽ tránh được.

![Chế độ Bạn tránh được; bóng mờ planner gốc chạm C2](evidence/shots/hazard-drive.png)

Chuẩn avoidable lạc quan vì oracle không có độ trễ phản ứng và phanh tối đa ngay khi có thể biết rắc rối, nên tính nhiều vụ hơn một planner thực tế cứu được; ngược lại, chỉ thử phanh, không đánh lái, bỏ sót cách tránh khác. Trong mẫu này `avoidable_foreknown` bằng avoidable ở cả hai policy, chưa phát hiện chênh lệch do che khuất.

Tăng cutinLat cứu nhiều mẫu nhưng chưa đủ cho GC-01. Planner cần dự đoán vùng quét ngang **cùng khả năng giảm tốc** của xe đang nhập làn, giữ khoảng hở theo μ và dự đoán phản ứng của C2 khi ego nhường/phanh; không chỉ chờ xe đó thành leader. Mine chỉ so hai policy, chưa chứng minh mọi núm còn lại đều vô ích.
