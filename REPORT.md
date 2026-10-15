# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `Trần Công Thiện` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `02579` |
| GitHub | `trancongthienai-ship-it` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `7366 / [9545, 9546, 9547, 9548, 9549]` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `5` |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.262 | 19.3 | BLOCK |
| IDM | 1.549 | 0.1 | (baseline) |
| EgoMLP | 0.761 .. 0.778 (tb: 0.766 ± 0.006) | 14.7 .. 29.2 (tb: 18.6) | BLOCK |
| PercMLP | 0.622 .. 0.631 (tb: 0.627 ± 0.003) | 0.5 .. 1.4 (tb: 0.8) | BLOCK |
| PercMLP+ (sau khi sửa) | 0.826 ± 0.014 | 0.3 .. 0.3 (tb: 0.3) | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Trong Bước 0 (kiểm toán dữ liệu `evidence/data_audit.json`), ta phát hiện ba vấn đề nghiêm trọng:
1. **Cột `Acc_LV` không thể sử dụng:** Hệ số tương quan giữa cột `Acc_LV` (gia tốc xe dẫn thô) với đạo hàm vận tốc thực tế $d(v)/dt$ gần như bằng 0 (`acc_lead_column_corr_with_dv_dt` = 0.045), trái ngược hoàn toàn với xe sau có tương quan 0.929. Cột này bị nhiễu cảm biến hoặc lưu trữ sai lệch. Do đó, ta bắt buộc phải thay thế bằng đạo hàm của chuỗi vận tốc xe dẫn sau khi đã làm mượt (`a_lead` tính từ `v_lead_s`).
2. **Trùng lặp dữ liệu lớn:** Có đúng 383 quỹ đạo bị trùng lặp hoàn toàn (`exact_duplicates_removed` = 383) nằm rải rác ở các ID xa nhau, làm bộ dữ liệu giảm từ 9.993 quỹ đạo thô xuống còn 9.610 quỹ đạo sạch (tổng 992.679 dòng). Nếu không loại bỏ, việc phân chia ngẫu nhiên sẽ làm rò rỉ dữ liệu (*data leakage*) từ tập train sang test, dẫn đến đánh giá ảo.
3. **Phân chia theo quỹ đạo:** Ta bắt buộc phải chia theo từng quỹ đạo (trajectory) thay vì chia theo từng khung hình (frame), bởi vì các khung hình liên tiếp ở tần số 10 Hz có mức tương quan chuỗi thời gian cực kỳ cao. Nếu chia theo frame, mô hình chỉ cần nội suy giữa khung $t-1$ và $t+1$ là đạt điểm L2 hoàn hảo mà không hề học được quy luật lái xe tổng quát.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Theo kết quả tại `evidence/replay.json` trên 600 đoạn log:
1. **Bản refactor float32:** Có 600/600 đoạn không giống từng bit (`not_bit_identical` = 600), nhưng có **0/600 đoạn vượt dung sai** (`differ_beyond_tol` = 0) với sai khác tuyệt đối tối đa chỉ là $1.22 \times 10^{-4}$ m/s² (hoàn toàn nằm trong dung sai `atol = 1e-3, rtol = 1e-3`). Phép cộng số thực không có tính chất kết hợp (ví dụ: $(0.1+0.2)+0.3 = 0.6000000000000001 \neq 0.1+(0.2+0.3) = 0.6$), việc tính toán trên float32 và thay đổi thứ tự tích lũy chắc chắn sinh ra sai số làm tròn bit cuối (ULP), nhưng hành vi vật lý điều khiển là tương đương nên kiểm tra bằng dung sai là bắt buộc.
2. **Thay đổi AEB:** Làm lệch đúng 7/600 đoạn (`differ_beyond_tol` = 7, bao gồm các đoạn 60, 139, 144, 180, 195, 237, 466) với độ lệch lệnh phanh lên tới 1.84 m/s². Khi kiểm tra chi tiết, AEB chỉ kích hoạt và làm lệch ở những khung hình khẩn cấp nguy hiểm có thời gian va chạm $TTC \le 1.5$ s và xe đang tiến gần ($closing > 0$), còn ở các khung hình an toàn khác xe vẫn tuân theo IDM nên lệnh không đổi.
3. **Phát nhanh nhất vs. Thời gian thật:** Chế độ phát nhanh nhất (*fastest*) chạy tuần tự trong bộ nhớ nên che giấu toàn bộ vấn đề về trễ hạn deadline (0 late, 0 dropped frames). Ngược lại, phát thời gian thực (*real-time*) mô phỏng đúng hệ thống nhúng thực tế: khi gặp tải tính toán đỉnh (spike), hệ thống làm lộ ra 582 khung bị trễ (*late*) và 577 khung bị bỏ rơi (*dropped*) trên tổng số 59.822 khung hình, làm biến đổi hành vi ở 368/600 đoạn log với sai lệch gia tốc tối đa lên tới 3.08 m/s².

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Dựa trên số liệu từ `evidence/scenarios.json`:
1. **Thông số lưới:** Tốc độ cruise $v_{ego} = 14.0$ m/s, khoảng cách $gap \in [2.76, 59.31]$ m (60 mốc), tốc độ tiến gần $dv \in [0.0, 14.0]$ m/s (45 mốc), tạo thành lưới $60 \times 45 = 2.700$ ô kịch bản.
2. **Vùng bất khả kháng vật lý:** Khi không có độ trễ ($\Delta t = 0.0$ s), có **3.6%** số ô (98 ô) là hoàn toàn không thể tránh được va chạm dù phanh tối đa $b_{max} = 9$ m/s². Khi độ trễ phản ứng tăng lên $\Delta t = 0.6$ s, vùng bất khả kháng mở rộng lên **10.2%** số ô (276 ô), tức tăng thêm 6.6 điểm phần trăm. Tỷ lệ va chạm của IDM tăng từ đúng 3.6% lên 10.2%, nghĩa là IDM không đâm ở bất kỳ ô nào có thể tránh được (`attributable_area = 0.0%`).
3. **Mức độ bao phủ của log thật:** Người lái trong log phanh mạnh nhất (phân vị 99.9%) chỉ đạt 3.62 m/s² (`human_brake_limit_p99.9`), trong khi bộ kích hoạt phanh xe có thể đạt tới 9.0 m/s². Trên lưới kịch bản, có tới **12.7%** số ô đòi hỏi mức phanh gấp vượt quá khả năng phanh mạnh nhất của con người trong log (`share_beyond_human_brake_limit` = 0.127). Quá trình khai thác chỉ tìm được 614 ca phanh gấp (chiếm 6.4% tổng số 9.610 quỹ đạo), với trung vị gia tốc đỉnh là -2.53 m/s² và phân vị 99 là -4.21 m/s². Các tình huống trong log hoàn toàn không bao phủ được các ca cắt vào nguy hiểm ở cự ly gần và tốc độ tiến gần cao.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng kết quả từ `evidence/closedloop.json` và `evidence/openloop.json`:

| Bộ điều khiển | L2 vòng hở (m) | Hạng vòng hở | Va chạm vòng kín (%) | Hạng vòng kín |
|---|---|---|---|---|
| PercMLP | 0.627 | **1** | 0.8% | **2** |
| EgoMLP | 0.766 | **2** | 18.6% | **3** |
| const-vel | 1.262 | **3** | 19.3% | **4** |
| IDM | 1.549 | **4** | 0.1% | **1** |

1. **Các cặp đảo thứ tự:** Có tới 3 cặp bị đảo ngược hoàn toàn giữa hai cách chấm: `('const-vel', 'IDM')`, `('IDM', 'EgoMLP')`, và `('IDM', 'PercMLP')`.
2. **Vì sao IDM có L2 cao nhất (1.549 m) nhưng lại lái tốt nhất (0.1% va chạm)?** L2 vòng hở đo độ lệch so với vết lái của con người cụ thể trong log. Con người lái xe có thói quen, độ rơ và độ trễ riêng nên quy tắc toán học của IDM không khớp từng mét quỹ đạo. Tuy nhiên, khi chạy vòng kín, IDM là một bộ điều khiển phản hồi có độ ổn định tiệm cận: khoảng cách càng co lại thì lực phanh càng tăng phi tuyến tính, chủ động triệt tiêu sai số tích lũy để bảo toàn khoảng cách an toàn.
3. **Vì sao EgoMLP có L2 tốt hơn const-vel (0.766 m vs 1.262 m) mà vẫn đâm tới 18.6%?** EgoMLP chỉ nhận đầu vào là lịch sử tốc độ của chính nó ($v_{ego}$), hoàn toàn mù thông tin môi trường phía trước (không biết $gap$ và $v_{lead}$). Khi chạy vòng hở, nó học thuộc xu hướng mượt của dòng xe nên L2 rất đẹp. Nhưng trong vòng kín, khi xe dẫn phanh gấp, EgoMLP không hề biết để phanh, khiến sai số tích lũy bùng nổ (*compounding errors*) dẫn tới tai nạn hàng loạt (riêng trên lát cắt xe dẫn phanh, EgoMLP đâm tới 54.6%).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Dựa trên dữ liệu 5 seed tại `evidence/closedloop.json`:
1. **Mức độ dao động của EgoMLP:** Tỷ lệ va chạm vòng kín của EgoMLP dao động cực lớn giữa 5 seed huấn luyện: từ **14.7%** đến **29.2%** (chênh lệch tới 14.5 điểm phần trăm, tức seed xấu nhất đâm gấp đôi seed tốt nhất).
2. **Khoảng tin cậy Wilson (CI 95%):** 
   - Seed tốt nhất (14.7% va chạm, k=425/2883) có CI 95% là $[13.5\%, 16.1\%]$.
   - Seed xấu nhất (29.2% va chạm, k=842/2883) có CI 95% là $[27.6\%, 30.9\%]$.
   - Hai khoảng tin cậy này hoàn toàn tách rời nhau và không có bất kỳ điểm chồng lấn nào. Điều này chứng minh sự chênh lệch này mang ý nghĩa thống kê rõ rệt chứ không phải do sai số ngẫu nhiên của việc lấy mẫu kiểm thử.
3. **Sự nguy hiểm khi chỉ chạy một seed:** Nếu kỹ sư vô tình chỉ chạy một lần trúng seed tốt nhất và báo cáo tỷ lệ va chạm 14.7%, hệ thống sẽ bị đánh giá sai lệch hoàn toàn về độ an toàn. Trong thực tế, quá trình khởi tạo trọng số ngẫu nhiên của mạng nơ-ron có thể rơi vào các cực tiểu cục bộ rất kém, khiến mô hình học ra chính sách điều khiển cực kỳ bất ổn định. Việc không báo cáo dải biến thiên qua các seed sẽ che giấu rủi ro đuôi nghiêm trọng trước khi đưa vào sản phẩm thương mại.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng kết quả từ `evidence/agents.json` trên lưới kịch bản cắt vào tại độ trễ $\Delta t = 0.6$ s (tỷ lệ % ô va chạm):

| Bộ điều khiển | Xe sau phát lại (Replay) | Xe sau phản ứng (Reactive) | Tỷ lệ va chạm phía sau (Replay / React) |
|---|---|---|---|
| const-vel | 80.6% | 80.6% | 0.0% / 0.0% |
| IDM | 80.9% | 10.2% | 70.6% / 0.0% |
| EgoMLP | 84.7% | 84.7% | 0.0% / 0.0% |
| PercMLP | 77.1% | 31.7% | 46.3% / 0.0% |
| PercMLP+ | 80.5% | 11.4% | 69.9% / 0.0% |

1. **Hiệu chỉnh thời khoảng cách $T$:** Giá trị hiệu chỉnh tối ưu là $T = 1.4$ s, cho khoảng cách Wasserstein cấp 1 ($W_1$) nhỏ nhất là 0.35 s so với phân phối khoảng cách thực tế của người lái (con người trong log giữ trung vị thời khoảng 1.86 s). Tác nhân này chân thực vượt trội so với một xe câm không bao giờ phản ứng (có $W_1 = 0.88$ s).
2. **Cơ chế "va chạm giả" (*spurious collisions*):** Khi xe sau chỉ phát lại tốc độ log trong quá khứ mà không phản ứng với môi trường, nếu xe thử (IDM) phát hiện tình huống nguy hiểm phía trước và chủ động phanh gấp để tránh đâm xe dẫn, chiếc xe sau sẽ đâm thẳng vào đuôi xe thử. Do đó, IDM bị tính oan tới **70.6%** va chạm phía sau, đẩy tổng tỷ lệ va chạm lên 80.9% (trông tệ hơn cả const-vel 80.6%).
3. **Tác động đến kết luận:** Ngược lại, xe const-vel không bao giờ phanh nên xe sau không bao giờ đâm đuôi nó (0.0% va chạm sau). Nếu chỉ kiểm thử trong thế giới phát lại tĩnh, một bộ điều khiển an toàn biết phanh khẩn cấp sẽ bị phạt nặng nề và bị coi là nguy hiểm hơn một bộ điều khiển vô tri không biết phanh. Khi chuyển sang thế giới phản ứng, toàn bộ 70.6% va chạm giả của IDM biến mất hoàn toàn (về 0.0%), đưa va chạm về đúng 10.2% (bằng đúng biên bất khả kháng vật lý).

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

Theo `evidence/gate.json`, `evidence/fix.json` và `evidence/openloop.json`:
1. **Các quy tắc chặn PercMLP:** PercMLP bị cổng chặn (*BLOCK*) bởi 4 quy tắc:
   - `R2_closed_loop_logs`: Tỷ lệ va chạm log 1.4%, cận trên CI 95% là 1.9% > ngưỡng cho phép (0.1% baseline + 1.0% margin = 1.1%).
   - `R3_braking_slice`: Trên lát cắt xe dẫn phanh, cận trên CI 95% là 8.5% > ngưỡng cho phép (1.0% + 5.0% = 6.0%).
   - `R4_scenario_grid`: Diện tích va chạm tránh được trên lưới cắt vào là 21.5% > ngưỡng cho phép (0.0% + 2.0% = 2.0%).
   - `R6_reactive_world`: Tỷ lệ va chạm trong thế giới phản ứng là 31.7% > ngưỡng cho phép (10.2% + 2.0% = 12.2%).
2. **Hai nguyên nhân gốc và cách sửa:**
   - *Nguyên nhân 1 (Thiếu dữ liệu kịch bản):* Log lái xe thông thường không chứa các tình huống xe cắt vào đột ngột ở cự ly gần. Sửa bằng cách bổ sung 20.000 kịch bản cắt vào mô phỏng (tạo ra 197.380 cửa sổ huấn luyện) với nhãn gán từ luật IDM.
   - *Nguyên nhân 2 (Trôi dạt phân phối - Covariate shift):* Khi tự lái vòng kín, sai số tích lũy đưa xe vào các trạng thái lạ mà mô hình không biết cách phục hồi. Sửa bằng thuật toán DAgger qua 2 vòng: cho mô hình tự lái trên log, ghi lại chính các trạng thái nó ghé thăm và gán nhãn bằng dự báo 3s của chuyên gia IDM tại đúng các trạng thái đó (huấn luyện trên tổng cộng 897.330 cửa sổ).
3. **Kết quả và đánh giá PercMLP+:**
   - Sau khi sửa, PercMLP+ được phê duyệt phát hành (*RELEASE*) khi vượt qua toàn bộ 7 quy tắc (va chạm trên log giảm còn 0.3%, lưới cắt vào giảm từ 21.5% xuống 1.2%, thế giới phản ứng giảm xuống 11.4%). Đáng chú ý, L2 vòng hở của PercMLP+ lại tăng từ 0.627 m lên 0.826 m, khẳng định điểm L2 vòng hở không đo lường được năng lực vòng kín.
   - **PercMLP+ không tốt hơn IDM:** Nó được đào tạo bằng cách bắt chước chuyên gia IDM nên năng lực tối đa của nó bị chặn bởi IDM. Khi kiểm thử tổng quát hóa ngoài phân phối ở tốc độ cruise 20.0 m/s (chưa từng xuất hiện khi train $\le 16$ m/s), PercMLP+ vẫn va chạm từ 0.5% – 3.2% trong khi IDM đạt 0.0%.
   - **Đề xuất:** Chưa nên phát hành thương mại ngay. Cần tiếp tục kiểm thử trên dải tốc độ mở rộng, kiểm thử khi có trễ truyền thông và nhiễu cảm biến lớn, và chạy thử nghiệm ở chế độ bóng (*shadow mode*) trên xe thực trước khi cấp quyền điều khiển thực tế.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Để đưa ra quyết định phát hành một hệ thống xe tự hành vào thế giới thực, bài lab này có ba giới hạn cốt tử:
1. **Giả định động học 1 chiều (1D Longitudinal Dynamics):** Bài lab chỉ xem xét chuyển động dọc trên một làn đường phẳng lý tưởng, bỏ qua hoàn toàn chuyển động ngang (đánh lái, chuyển làn tránh chướng ngại vật), ma sát mặt đường (đường ướt, trơn trượt) và giới hạn lốp xe. Trong thực tế, việc phanh khẩn cấp ở gia tốc lớn (ví dụ 9 m/s²) có thể làm xe bị trượt lốp, văng đuôi hoặc xoay ngang nguy hiểm.
2. **Bỏ qua chuỗi nhận thức cảm biến đầy đủ:** Mô phỏng cấp cho mô hình giá trị khoảng cách ($gap$) và vận tốc xe dẫn ($v_{lead}$) từ ground-truth kèm một ít nhiễu Gaussian nhân tạo. Trong thực tế, hệ thống phải xử lý qua camera, radar, LiDAR với các hiện tượng mất dấu vật thể (*occlusion*), nhận diện vật thể ảo (*ghost braking*), sai số đo lường phụ thuộc thời tiết và độ trễ đường truyền phần cứng CAN bus.
3. **Mô hình hóa tác nhân xung quanh còn đơn giản:** Xe dẫn trong log hoàn toàn không phản ứng với hành vi của xe thử (vẫn lái theo kịch bản có sẵn dù xe thử có tiến sát), và xe sau chỉ được mô phỏng bằng một luật IDM duy nhất. Thế giới giao thông thực tế là một hệ thống đa tác nhân phức tạp, khó đoán với hành vi cắt đầu bất ngờ, xe máy lách làn và người đi bộ băng qua đường.

**Các bước cần làm tiếp theo để thu hẹp khoảng cách đến thế giới thực:**
- Tích hợp mô hình vào phần mềm mô phỏng 3D hoàn chỉnh đa vật lý (như CARLA, AirSim) kết hợp mô hình giả lập cảm biến ảnh và đám mây điểm.
- Kiểm thử phần cứng trong vòng lặp (Hardware-in-the-Loop - HIL) để đo đạc chính xác độ trễ tính toán, nhiệt độ chip xử lý và giới hạn cơ điện tử của hệ thống phanh.
- Triển khai thử nghiệm ở chế độ chạy bóng (*Shadow Mode*) trên đội xe thử nghiệm thực tế: cho mô hình chạy ngầm và so sánh quyết định phanh của mạng với thao tác của tài xế an toàn trên hàng trăm nghìn kilomet trước khi chính thức cấp quyền điều khiển xe thật.
