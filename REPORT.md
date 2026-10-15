MSSV: 2A202602505

Name: Văn Quốc Dũng

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): **B** (thị trấn "mưa nhiều": `wet` = 0,30, `pedLook` = 0,85, `cutGap` trung vị 12 m).

> Ghi chú chạy: MSSV có chữ cái (`2A…`) nên regex dòng đầu của `run_all.py` (chỉ nhận chữ số) không đọc được; mọi bước được chạy bằng `LAB11_MSSV=2A202602505 python run_all.py <bước>`, đúng cơ chế grader dùng (seed = SHA-256 của chuỗi MSSV). Toàn bộ `evidence/` là bản chạy đầy đủ, không `--quick` (`_run.quick = false`).

## 1. Phát lại log có trung thực không? (bước A)

| `replay.json` | giá trị |
|---|---|
| `n` / `n_used` | 400 / 400 |
| `dropped_human_crash_logs` (người lái tự gây va chạm, bị bỏ) | 0 |
| `replay.crashes` (planner trên log phát lại) | **88**: `rear` 73, `side` 15, `front` 0 |
| `reactive.crashes` (planner trong thế giới phản ứng) | **0** |
| `crash_in_both` | 0 |

83 % va chạm của "phát lại" là `rear` (73/88), và **không** vụ nào trong số đó xảy ra lại trong thế giới phản ứng (`crash_in_both = 0`): chúng là va chạm **giả**. Khi phát lại, xe bám đuôi TG chạy đúng quỹ đạo đã ghi khi nó theo **người lái**. Planner thận trọng hơn người: giảm về `cautionV` = 7,5 m/s khi tới gần vạch qua đường bị xe buýt che, và phanh sớm hơn. Nhưng TG trong log không "biết" điều đó, nó vẫn giữ tốc độ như lúc theo người lái, nên tông vào đuôi xe ta. Trong thế giới phản ứng, TG là IDM, thấy xe ta chậm lại thì phanh theo. 15 vụ `side` nhiều khả năng cùng cơ chế: C2 nhập làn theo đúng thời điểm trong log, không chờ khe dù vị trí xe ta đã khác người lái.

Ví dụ `replay_only[4]` (`scenario` 15, `contact` = `rear`): đường ướt (`mu` = 0,5), TG bám rất sát (`tgT` = 0,73 s). Planner giảm tốc trước vạch sớm hơn người lái; TG phát lại không giảm theo, tông đuôi. Cùng kịch bản trong thế giới phản ứng, TG phanh kịp: `min_ttc_reactive` = 2,89 s, không va chạm.

## 2. Kịch bản bất khả thi (bước B)

Luật trong `is_feasible` (loại nếu **bất kỳ** luật nào sai):

| Luật | Căn cứ vật lý |
|---|---|
| gia tốc ngang của cú chuyển làn cosin 3,5 m, (π/`cutDur`)²·3,5/2 > μg | lốp chỉ tạo được lực ngang tối đa μ·m·g (vòng tròn ma sát); đường ướt μ = 0,5 thì trần chỉ còn 4,9 m/s², nên chuyển làn trong < 1,9 s là không thể |
| `cutBrake` > μg | phanh không thể giảm tốc mạnh hơn ma sát cho phép |
| `pedV` > 2,5 m/s | đó là chạy, không phải đi bộ (đi bộ thường 1,2–1,5 m/s); phân phối tự nhiên `pedV` ~ N(1,35; 0,20) |
| `cutV` ∉ (0, 22] m/s | xe chuyển làn phải đang chạy, và ≤ 79 km/h trên phố đô thị |

`hunt.json: infeasible`: `crosswalk_block` **7** (nhiều nhất), `cut_in` 3, `bus_bypass` 2, `rear_ended` 2 (Monte Carlo: `mc.json: infeasible` = 16/3000). Tôi kiểm tra từng điểm `infeasible` trong `cloud.json`: cả 14 điểm đều vi phạm luật **gia tốc ngang khi chuyển làn**. Lý do cuộc săn `crosswalk_block` bị kéo về đó: điểm nguy hiểm của nó là "khoảng phanh còn lại trước người đi bộ", nên CEM đẩy `wet` lên p = 0,92 (μ = 0,5 làm quãng phanh dài gấp 1,6 lần) và kéo `cutDur` ngắn lại (`proposal.mean[cutDur]` = −0,35). Đường ướt cộng chuyển làn nhanh là đúng vùng mà lốp không bám nổi. Vùng "nguy hiểm nhất" và vùng "bất khả thi" nằm sát nhau, vì thứ làm kịch bản nguy hiểm (ít ma sát, chuyển động gắt) cũng là thứ vượt giới hạn vật lý.

Nếu **đếm** chúng là va chạm, ước lượng bị thổi phồng bởi những tai nạn không thể xảy ra, và CEM còn được thưởng khi đi vào vùng đó, nên đề xuất q dồn mẫu vào vùng bất khả thi. Ước lượng IS khi đó đo một thế giới không có thật.

## 3. Monte Carlo cho bạn biết gì (bước C1)

`mc.json`: `crashes` k = **2**, `n` = **3000**, `p` = **6,67·10⁻⁴**, Wilson 95 % [`lo`, `hi`] = **[1,83·10⁻⁴; 2,43·10⁻³]**. Cả 2 vụ đều là `bus_bypass` (`modes`).

Muốn khoảng hẹp còn ±20 % của p: n ≈ 96/p = 96 / 6,67·10⁻⁴ ≈ **144 000** lần chạy, gấp 48 lần ngân sách MC đã chạy.

Kiểu **chưa hề xuất hiện** trong 3000 lần: `cut_in`, `crosswalk_block`, `rear_ended`, `other`. Không báo 0 cho các kiểu này: con số đúng là cận trên Wilson với k = 0, `mc.json: per_mode.<kiểu>.hi` = **1,28·10⁻³** (≈ 3,84/n). Nghĩa là "với 95 % tin cậy, mỗi kiểu < 1,3·10⁻³", không phải "không bao giờ".

## 4. Importance sampling, ESS và độ suy biến (bước C3)

`estimate.json` (n = 600 mẫu/kiểu/seed, q = 0,8·CEM + 0,2·tự nhiên):

| kiểu | seed | `p` | `hits` | `ess` | `top_share` |
|---|---|---|---|---|---|
| bus_bypass | 0 | 3,11·10⁻⁴ | 37 | 139 | 0,35 |
| bus_bypass | 1 | 4,69·10⁻³ | 29 | 133 | **0,91** |
| bus_bypass | 2 | 1,61·10⁻³ | 39 | 143 | **0,58** |
| crosswalk_block | 0 | 5,58·10⁻⁶ | 108 | 119 | **0,95** |
| crosswalk_block | 1 | 3,57·10⁻⁷ | 90 | 108 | 0,21 |
| crosswalk_block | 2 | 8,37·10⁻⁷ | 106 | 119 | 0,29 |
| cut_in | 0/1/2 | 0 → cận trên `upper_zero_hits` = **0,025** | 0 | 150/138/134 | 0 |
| rear_ended | 0/1/2 | 0 → cận trên `upper_zero_hits` = **0,025** | 0 | 108/118/128 | 0 |

Tổng `total` theo seed: 3,17·10⁻⁴ · 4,69·10⁻³ · 1,61·10⁻³ (trung vị 1,61·10⁻³). `other`: k = 0/3000 từ `mc.json`, nên p = 0 với cận trên `other.hi` = 1,28·10⁻³.

- **Ba seed có đồng ý không?** Không hẳn. `bus_bypass` chênh 15 lần (3,1·10⁻⁴ đến 4,7·10⁻³), `crosswalk_block` chênh 16 lần. Đúng cái bẫy 2 của README: đề xuất CEM hẹp, vài mẫu nặng ký quyết định con số. Với Monte Carlo: MC cho `bus_bypass` 6,67·10⁻⁴ [1,8·10⁻⁴; 2,4·10⁻³], cả ba seed đều nằm trong hoặc sát khoảng Wilson đó. `crosswalk_block` ~10⁻⁶–10⁻⁷ cũng không mâu thuẫn với MC 0/3000 (cận trên 1,3·10⁻³), nhưng MC quá thô để kiểm chứng.
- **`top_share` > 0,5**: `bus_bypass` seed 1 (0,91), seed 2 (0,58), và `crosswalk_block` seed 0 (0,95). Khi đó một mẫu duy nhất mang > 90 % ước lượng: p ≈ w_max/n, tức con số thực chất là trọng số p/q của **một** kịch bản nằm ở rìa q nhưng không hiếm dưới p. Seed khác không rút trúng mẫu đó thì ra nhỏ hơn hàng chục lần (seed 1 so với seed 2 của `crosswalk_block`). Khoảng ±1,96·se khi đó hẹp sai. Và `top_share` thấp (seed 1 `crosswalk_block` 0,21) cũng **không** chứng minh đúng: q có thể bỏ sót hẳn một vùng va chạm.
- **Ngân sách**: `budget` = **16 800** lần chạy cho cả 4 kiểu × 3 seed (mỗi kiểu 8×300 vòng săn + 3×600 mẫu IS = 4 200). Để MC đạt sai số tương đối như IS seed 0 của `bus_bypass` (se/p = 1,25·10⁻⁴ / 3,11·10⁻⁴ ≈ 0,40) cần n ≈ (1−p)/(p·0,40²) ≈ 20 000 lần chạy cho riêng kiểu đó. Với `crosswalk_block` (p ~ 10⁻⁶, se/p ≈ 0,34–0,95) MC cần cỡ 10⁶–10⁷ lần, còn IS chỉ dùng 4 200. MC không thấy nổi một vụ.
- **`other`** không được săn vì không có điểm nguy hiểm riêng để CEM leo dốc (nó là "mọi thứ còn lại"). Phần của nó lấy từ Monte Carlo C1, nên tổng = các kiểu + `other` (`estimate.json: other.source = "mc.json"`).

## 5. Các kiểu thất bại (bước C2 + 3D)

| Kiểu | Ảnh (chương 9, đúng lúc sự kiện) | Chuỗi sự kiện |
|---|---|---|
| `bus_bypass` (B-01, va chạm C2 lúc 12,6 s) | `evidence/shots/q5_q6_bus_bypass_B01_ghost_fix.jpg` | Xe buýt đỗ ở làn ngoài E2. Xe tím C2 (chạy sau buýt) không muốn chờ nên lách trái sang làn E1 của xe ta khi đang chậm. Khe C2 chấp nhận rất nhỏ (`cutGap` = 4,7 m), đường ướt. Planner gốc chỉ coi C2 là "xe phía trước" khi tim C2 vào trong `cutinLat` = 2,15 m, nên phản ứng muộn và tông vào C2 đang lách qua đuôi buýt. |
| `crosswalk_block` (C-01, chặn lối lúc 21,6 s) | `evidence/shots/q5_crosswalk_block_C01.jpg` | Người đi bộ P1 bước ra từ sau đầu xe buýt (không ngó, `pedLook` = 0), đường ướt. Xe buýt che nên cảm biến xe ta thấy P1 muộn. Xe ta phanh gấp và dừng được, nhưng thân xe đã nằm trên vạch qua đường; P1 phải đứng lại. Đó là thất bại `kind == "block"`. |

Kiểu **không** tìm thấy: `cut_in` và `rear_ended`. `hunt.json: found` của cuộc săn `cut_in` = {`bus_bypass`: 16}, của cuộc săn `rear_ended` = {`bus_bypass`: 69, `crosswalk_block`: 1}: cả hai đều **trôi về `bus_bypass`** và không thấy một vụ đúng kiểu nào. `trace` cho thấy mức elite dừng ở −3,47 (cut_in) và −0,98 (rear_ended), không lên tới 0. Tôi tin chúng hiếm thật (TTC tới C2 khi C2 chạy nhanh vẫn ≥ 3,4 s), nhưng chỉ ở mức "cận trên": `upper_zero_hits` = 0,025 mỗi seed, và MC < 1,28·10⁻³. Đây không phải chứng nhận chúng không tồn tại.

Nếu chỉ chạy **một** cuộc săn chung: CEM co về kiểu dễ leo nhất. Bằng chứng ngay trong `hunt.json`: hai cuộc săn nhắm `cut_in` và `rear_ended` đã tự trôi về `bus_bypass`. Một cuộc săn chung sẽ chỉ báo `bus_bypass` (và có thể bỏ sót cả `crosswalk_block`, vốn cần đẩy `wet`, `pedLook`, `pedV` theo hướng rất khác). Vì vậy mỗi kiểu cần điểm nguy hiểm và cuộc săn riêng.

## 6. Sửa planner (bước E + 3D)

`fix.json`: policy được chọn bằng `choose_policy` trên TRAIN là **`{"cutinLat": 3.0}`** (mở rộng hành lang coi xe đang lấn sang là "xe phía trước" từ 2,15 m lên 3,0 m).

| | `fail` | `fail_hi_seen` | `fail_hi` | `unseen` | `progress` [m] | `harsh` [khung] |
|---|---|---|---|---|---|---|
| TRAIN, policy gốc (`baseline_train`) | 1,36·10⁻³ | 3,31·10⁻³ | 3,31·10⁻³ | — | 215,0 | 7,54 |
| TRAIN, `cutinLat` 3,0 (`chosen`) | 1,19·10⁻⁵ | 3,34·10⁻⁵ | 3,34·10⁻⁵ | `bus_bypass` | 215,0 | 7,29 |
| **TEST, `cutinLat` 3,0 (`chosen_test`)** | **1,95·10⁻⁶** | **4,09·10⁻⁶** | **3,88·10⁻²** | `bus_bypass` | 214,1 | 7,98 |
| **TEST, policy gốc (`baseline_test`)** | **2,71·10⁻³** | 6,67·10⁻³ | **7,95·10⁻³** | — | 214,2 | 8,40 |

- TRAIN chỉ dùng để sàng; con số báo cáo là TEST. Trên TEST, cuộc săn **mới nhắm vào chính `cutinLat` 3,0** cho 0 va chạm `bus_bypass` qua cả 8 vòng (mức elite chỉ lên tới −0,00), trong khi cùng cuộc săn với policy gốc tìm 59 va chạm ở vòng cuối. `crosswalk_block` không đổi (1,95·10⁻⁶, núm này không liên quan người đi bộ). Va chạm khác trên tập tự nhiên 3000 lần: 0 (`other_crashes.k` = 0, cận 1,28·10⁻³).
- **Vì sao `fail_hi` của bản sửa (3,88·10⁻²) lớn hơn `fail` của policy gốc (2,71·10⁻³) dù bản sửa tốt hơn thật?** Vì `bus_bypass` là `unseen`: không thấy va chạm nào trong n = 400 mẫu, nên cận hợp lệ duy nhất là `upper_zero_hits` = 3/(0,2·400) = **0,0375**, cộng cận `other` 1,28·10⁻³. Với n = 400, không ngân sách nào chứng nhận được một kiểu đã biến mất dưới ~10⁻². Đó là giới hạn của dữ liệu, không phải bản sửa kém. `fail_hi_seen` = 4,09·10⁻⁶ là "cái dữ liệu cho thấy", nhưng không phải cận của policy.
- Quãng đường gần như không đổi (214,1 so với 214,2 m, mất < 0,1 %, trong giới hạn 5 %) và phanh gấp ít hơn (7,98 so với 8,40 khung): phản ứng sớm với C2 nghĩa là phanh nhẹ sớm thay vì phanh gấp muộn.
- Ảnh có **bóng mờ**: `evidence/shots/q5_q6_bus_bypass_B01_ghost_fix.jpg` (B-01, 12,6 s). Xe ta sáng màu là policy gốc, tông C2; bóng mờ phía sau là `cutinLat` 3,0, dừng sau C2 và không va chạm. Tôi chạy lại 4 kịch bản B-01…B-04 bằng `avsim`: policy gốc va chạm C2 lúc 12,50–12,65 s, `cutinLat` 3,0 không va chạm ở cả 4. (Lưu ý: nhãn trong trình xem ghi ngược, "chính sách của bạn" là quỹ đạo của policy gốc, vì `run_all.py export` cố ý phát lại các va chạm bằng `core.POLICY` và đặt bản sửa làm bóng mờ.)
- **Núm "nghe thì an toàn hơn" nhưng phản tác dụng**: trên TRAIN, ứng viên [6] `{"cutinLat": 2.6}` có `bus_bypass` p = 2,06·10⁻⁵ (6 hits). Thêm hai núm "thận trọng hơn", `cautionV` 7,5 → 7,0 và `pedBuffer` 2,5 → 3,5 (ứng viên [12]), thì `bus_bypass` **tăng 7 lần** lên 1,51·10⁻⁴ và `fail` lên 1,63·10⁻⁴. Cơ chế khả dĩ (tôi chưa tách riêng từng núm): thế giới phản ứng lại. Xe ta chạy chậm hơn và dừng nhường người đi bộ sớm hơn gần trạm buýt, nên khe phía trước xe ta trong làn E1 kéo dài và lâu hơn. C2 đang kẹt sau xe buýt chấp nhận khe (gap acceptance) và lách ra **đúng lúc xe ta đi ngang**, vào hành lang mà `cutinLat` 2,6 vẫn phát hiện muộn. Tương tự `cautionV` 5,0 (chậm nhất) giảm thất bại nhưng mất 20 % quãng đường (172,7 m) và phanh gấp gần gấp đôi (`harsh` 13,0 so với 7,54), nên bị loại.

## 7. Từ mô phỏng ra đường thật

| Giả định (`space.py`) | Dữ liệu thật thay nó, lấy từ đâu | Nếu sai, con số của tôi sai theo hướng nào |
|---|---|---|
| **Khe chấp nhận khi C2 lách qua xe buýt** `cutGap` ~ logN(trung vị 12 m, σ 0,5) | Quỹ đạo tự nhiên của xe lách qua xe đang đỗ hoặc dừng: bộ dữ liệu drone/quỹ đạo (highD, inD, exiD của RWTH/ika), hoặc tự quay camera cao tại trạm buýt nội đô Việt Nam và đo khe lúc xe lách ra | `bus_bypass` (≈ 100 % xác suất của tôi) rất nhạy với đuôi trái của phân phối này. Nếu tài xế thật chấp nhận khe nhỏ thường hơn (giao thông Việt Nam), p bị **đánh giá thấp**; nếu thận trọng hơn thì đánh giá cao |
| **Tỉ lệ người đi bộ ngó qua xe buýt** `pedLook` = 0,85 | Quan sát video tại vạch qua đường cạnh trạm buýt (đếm người dừng nhìn trước khi bước ra), hoặc nghiên cứu hành vi qua đường ở vùng có vật che | Cuộc săn `crosswalk_block` đẩy `pedLook` về 0,03. Nếu tỉ lệ ngó thật thấp hơn 0,85, `crosswalk_block` (~10⁻⁶) bị **đánh giá thấp** đáng kể (xác suất của nhánh "không ngó" tăng tuyến tính) |
| **Đường ướt** `wet` = 0,30 (thị trấn B) với μ = 0,5 cố định | Số ngày và giờ mưa (dữ liệu khí tượng địa phương, ví dụ trạm Tân Sơn Nhất hoặc Láng), cộng đo hệ số bám mặt đường ướt (thiết bị đo ma sát, dữ liệu ABS/ESC của đội xe) | Cả hai kiểu săn được đều dồn về đường ướt (`crosswalk_block` p_wet = 0,92). Nếu tỉ lệ mưa thấp hơn hoặc μ ướt thật > 0,5, p bị **đánh giá cao**; mùa mưa nhiệt đới hoặc mặt đường trơn (μ < 0,5) thì **đánh giá thấp** |

## 8. Giới hạn

1. **Một con phố, một cấu hình**: một xe buýt đỗ ở đúng một trạm trước đúng một vạch qua đường, với 4 xe nền cố định. Xác suất báo cáo là "mỗi lần đi qua đoạn phố này", không phải mỗi km hay mỗi chuyến. Không có giao lộ, rẽ trái, xe máy hay mật độ xe thay đổi.
2. **Người lái là IDM + gap acceptance**: không có người vượt ẩu, không có sai sót nhận thức, không bấm còi hay giao tiếp bằng mắt. Phân phối hành vi (bảng ở câu 7) là giả định dạy học, nên các con số tuyệt đối chỉ có nghĩa so sánh giữa các policy.
3. **Cảm biến hoàn hảo trừ phần bị xe che**: không nhiễu, không trễ, không phát hiện sai, và người đi bộ chỉ bị che bởi xe buýt. Va chạm do nhận thức sai, điều thường gặp ngoài đời, nằm ngoài mô hình. Thêm nữa, ở ngân sách n = 400, `fail_hi` không chứng nhận được kiểu nào < 10⁻².

Thêm một tuần: (a) chạy TEST với n ≈ 15 000 trên vùng `bus_bypass` để chứng nhận `cutinLat` 3,0 xuống dưới 10⁻³; (b) săn tiếp `bus_bypass` với `cutinLat` 3,0 trong các thị trấn A, C, D (thị trấn C có `cutGap` trung vị 10 m, bạo hơn); (c) thêm nhiễu và trễ cảm biến (100–300 ms) xem `cutinLat` 3,0 còn đủ không; (d) đo tác dụng phụ của hành lang rộng: số lần phanh "ma" vì xe đạp hoặc xe máy chạy sát vạch làn bên.

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** một **xe đạp kẹt sau xe buýt đang đỗ ở trạm** (làn ngoài E2, đi lệch trái −0,8 m) **lạng trái sang làn của xe ta** để vượt qua xe buýt, không ngoái lại (`hazard.json`: `bike`, `lane` = E2, `x` = 60, `d` = −0,8, `v` = 4 m/s, `swerve` = 2,2 m, `swDur` = 2,4 s, `trig` = 8 m). Ngoài đời: xe đạp hoặc xe máy ở trạm buýt nội đô bật ra làn trong ngay trước mũi ô tô.
- **Căn cứ:** đọc `avsim/core.py`, planner chỉ coi một xe là "phía trước" khi tim xe nằm trong `cutinLat` = 2,15 m tính từ tim xe ta. Với người đi bộ thì có dự đoán (`pedBuffer`), nhưng với xe đạp ở làn bên thì không có dự đoán nào. Xe đạp chỉ được tính khi đã lấn gần hết, cùng lỗ hổng với `bus_bypass` nhưng là vật nhỏ, chậm, do người đạp không ngoái lại.
- **Giả định (`hazard.json: why`):** `space` = {`h0_x` [40, 95] (xuất phát trong 55 m ngay trước đuôi buýt, nên luôn phải dừng chờ sau buýt), `h0_d` [−1,0; −0,5] (nửa trái làn ngoài), `h0_v` [3, 7] m/s ≈ 11–25 km/h (tốc độ xe đạp đô thị), `h0_swerve` [1,4; 2,4] m (đủ để thân xe đạp 0,7 m thoát mép xe buýt rộng 2,55 m), `h0_swDur` [1,8; 3,0] s (cú lách dứt khoát, thêm giới hạn gia tốc ngang 3 m/s² của `avsim`), `h0_trig` [5, 20] m}. `exposure` = 0,1/chuyến là **giả định chưa đo**: ~5 trạm/chuyến × ~20 % có buýt đang đỗ ≈ 1 lần gặp, × ~10 % có xe hai bánh lách ra trong vòng 20 m trước mũi xe. Cần thay bằng đếm video tại trạm buýt.
- **Kết quả** (`mine.json`, giao thông thị trấn B; `accepted` = **0,962**, 1443/1500 phiên bản khả thi, `infeasible` = 57; `exact.planner` = [ego, H0, 13,8 s], đúng cảnh dựng ở chương 10):

  | | `crashes` | `avoidable` | `unavoidable` | `unattributed` | `low_speed` | `avoidable_foreknown` | P(lỗi planner \| rắc rối) [Wilson 95 %] | mỗi chuyến (`per_trip`, × 0,1) |
  |---|---|---|---|---|---|---|---|---|
  | policy gốc (`baseline`) | 421 | **402** | 18 | 1 | 84 | 402 | **0,279** [0,256; 0,302] | **2,79·10⁻²** (cận 3,02·10⁻²) |
  | `cutinLat` 3,0 (`chosen`) | 0 | **0** | 0 | 0 | 0 | 0 | 0 [0; 2,66·10⁻³] | 0 (cận **2,66·10⁻⁴**) |

  - 402/421 va chạm là `avoidable`: cùng planner, nếu phanh hết cỡ ngay khung đầu tiên xe đạp bắt đầu lạng **và** cảm biến thấy nó thì không va chạm. Đây là lỗi planner. 18 vụ `unavoidable` (xe đạp lạng ra khi xe ta đã ngay sát, phanh hết cỡ cũng không kịp) không nói gì về planner. 1 vụ `unattributed` là va chạm thường của thị trấn khi rắc rối chưa bắt đầu. 84 vụ `low_speed` (< 1 m/s): xe ta gần như đứng (đang chậm lại gần vạch, sau xe buýt) khi xe đạp lạng vào, tức "bị đâm khi đang chờ" hơn là "xe ta đâm vào".
  - `avoidable_foreknown` = `avoidable` = 402: xe đạp không bị che (nó ở ngay cạnh xe buýt, phía xe ta), nên tầm nhìn bị che không tốn gì. Lỗi hoàn toàn nằm ở **logic phản ứng**, không phải cảm biến.
  - **So với lách xe buýt**: với `exposure` 0,1, lỗi planner do rắc rối này là 2,79·10⁻²/chuyến, gấp ~17 lần `bus_bypass` (trung vị IS 1,61·10⁻³, MC 6,67·10⁻⁴ mỗi lần đi qua đoạn phố). Nhưng con số này phụ thuộc tuyến tính vào `exposure` giả định.
  - **"Tránh được" là chuẩn lạc quan theo hai hướng.** (1) Người phanh "oracle" biết rắc rối ngay khung nó bắt đầu và phanh tối đa ngay, không có thời gian nhận thức hay quyết định. Một planner thật không làm được vậy, nên `avoidable` **đếm nhiều hơn** số vụ một planner thật tránh được (cận trên của "lỗi planner"). (2) Oracle chỉ thử **phanh**, không thử đánh lái hay tránh sang trái, nên một số vụ `unavoidable` thật ra tránh được bằng đánh lái. Theo hướng này `avoidable` **đếm ít đi**.
- **Hình:** `evidence/shots/q9_bike_behind_bus_crash.jpg` (chương 10, 13,8 s: xe ta tông xe đạp H0 vừa lạng ra từ sau xe buýt).
- **Bạn tự lái thì sao:** tôi chưa thử chế độ "Bạn" (tự lái) cho cảnh này. Theo kết quả `avoidable_foreknown`, một người lái chú ý chỉ cần nhả ga hoặc phanh khi thấy xe đạp bắt đầu nghiêng ra (≈ 2 s trước va chạm, `fire_t` = 11,65 s so với va chạm 13,35 s trong `examples[0]`) là đủ.
- **Cách sửa:** núm **`cutinLat`** sửa được: bản sửa ở bước E (`cutinLat` 3,0) đưa va chạm từ 421 xuống **0** trên cùng 1443 phiên bản (cận trên 2,66·10⁻³), vì xe đạp ở nửa trái làn ngoài (cách tim xe ta ≈ 2,5–3,0 m) được tính là "phía trước" ngay khi bắt đầu lạng. Điều này xác nhận chẩn đoán: `bus_bypass` và rắc rối này chung **một** nguyên nhân là hành lang 2,15 m quá hẹp. Nhưng hành lang tĩnh rộng hơn là sửa chắp vá: nó sẽ phanh cả cho xe đạp chỉ chạy thẳng sát vạch. Logic tốt hơn: **dự đoán quỹ đạo ngang** của vật ở làn bên (vận tốc ngang, hướng đầu xe, vị trí cạnh vật cản phía trước như xe buýt đỗ) và giảm tốc khi thời gian tới khi vật lấn vào hành lang < thời gian tới khi xe ta tới ngang, giống `pedBuffer` nhưng cho xe đạp và xe máy, đặc biệt khi làn bên bị chặn bởi xe đỗ.
