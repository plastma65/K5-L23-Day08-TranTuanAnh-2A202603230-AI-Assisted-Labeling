# Báo cáo Day 08 — 2A202603230 - Trần Tuấn Anh

> Một file duy nhất cho cả bài (đã gộp recipe, QC và ghi chú cá nhân). Mọi con số kèm **mẫu số** và nguồn (file nào). Viết bằng lời của bạn.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| person / bicycle / motorbike / car / bus / truck | YOLO26m (`yolo26m.pt`) | person 0.2, motorbike 0.35; bicycle / car / bus / truck mặc định 0.3 (`threshold_sweep`) | Detector danh mục cố định bắt tốt các lớp COCO (r01). Không dùng ngưỡng chung 0.5 vì mất vật đúng ở img_032 (r02). bicycle / bus / truck quá ít vật trên gold (4 / 1 / 0 < 5) nên giữ mặc định; car không ngưỡng nào đạt recall 90% (tối đa 83%, 6 vật) |
| three_wheeler | YOLOE-26s (`yoloe-26s-seg.pt`), prompt: `tuk-tuk`, `cyclo`, `cargo tricycle` (từ `taxonomy.json`) | 0.15 (recall 100%, precision 78% trên 7 vật gold) | YOLO26 không có lớp xe ba bánh, không gán xích lô / xe ba bánh nào (r01, r02). YOLOE gõ chữ tìm được (r03) nhưng gán nhầm xe bán tải / quầy hàng (img_002, img_011) và sót xích lô ở góc (img_007), nên khi duyệt phải soi kỹ |

- Lệnh `merge_prelabels.py` đã dùng (dán nguyên): `python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --thr "person=0.2,motorbike=0.35,three_wheeler=0.15" --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json`
- **Phiên bản SAM bạn chọn**: Part 2 (r04) dùng **SAM 2.1** (`sam2.1_b.pt`, ~10 giây/ảnh CPU) để thử box YOLO26 → mask; Part 5 dùng **SAM 3** (`sam3.pt`, bản Ultralytics, ~80–95 giây/vật trên CPU). Lý do chọn SAM 3: đã có sẵn `sam3.pt` (không cần xin quyền HuggingFace) và kỳ vọng mask chất lượng hơn (ở r04 mask SAM 2.1 chỉ ôm khoảng 80% xe); chấp nhận chạy chậm hơn trên CPU.
- Lỗi **không sửa được bằng đổi ngưỡng**: YOLO26 là closed-set COCO, không có lớp `three_wheeler`, nên tăng ngưỡng 0.25 → 0.5 vẫn không gán được xích lô / xe ba bánh nào (r01, r02); ở r04 xe 3 bánh bị gán `car`. Sửa bằng **đổi model**: dùng YOLOE (open-vocabulary, gõ chữ) cho riêng lớp `three_wheeler`, và khi gộp ưu tiên box YOLOE (11 box `car` của YOLO26 chồng lên box three_wheeler đã bị loại khi merge).
- Box **điểm thấp mà đúng**: img_032, bicycle ở gần (0.36) và person (0.31) là vật đúng nhưng bị mất khi lên ngưỡng 0.5 (r02) → điểm số không phải xác suất đúng.
- Giới hạn: ngưỡng được **chọn và chấm trên cùng 4 ảnh gold (49 vật)**, nên precision / recall của máy trên gold là lạc quan (in-sample). Lớp ít vật (bus 1, truck 0, bicycle 4) không đủ để chọn ngưỡng riêng; three_wheeler chỉ 7 vật nên ngưỡng 0.15 là ước lượng thô — ở batch ngưỡng này để lọt nhiều box sai (18/43 bị xoá).

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 540 | 94.2% | 573 box máy |
| edited | 7 | 1.2% | 573 box máy |
| deleted | 26 | 4.5% | 573 box máy |
| added | 11 | 2.0% | 558 đối tượng sau duyệt |

- Số liệu trên đã **loại 21 đối tượng nằm quanh 12 bẫy**: bẫy là lỗi nhân tạo, nếu tính vào sẽ làm sai chất lượng thật của máy. 3 trường hợp trong deleted + added là đổi lớp (box đúng chỗ, sai lớp), nên số vật máy sót thật ≈ 11 − 3 = 8.
- Lớp máy tệ nhất là **`three_wheeler`**: trong 43 box máy chỉ giữ 20 (46.5%), sửa 5, xoá 18 (41.9%). Các lớp khác accepted cao (person 330/331, motorbike 125/128). Nguyên nhân tôi thấy:
  - **YOLOE gán nhầm vật khác** thành `three_wheeler`: đã thấy từ Part 2 (r03), img_002 xe bán tải và quầy hàng, img_011 phần xe bán tải ở góc ảnh.
  - **Box chồng nhau / box lỏng**: nhiều box `three_wheeler` chồng lên cùng một xe hoặc chỉ ôm một phần xe (ví dụ img_003 có hai box three_wheeler 0.90 và 0.80 chồng cùng vùng).
  - Ngưỡng `three_wheeler=0.15` chọn trên gold để đạt recall 100% nên chấp nhận thêm box rác ở batch.
- So với gold: precision / recall của **máy** = 45/81 = **55.6%** (KTC 95%: 45–66%) / 45/49 = **91.8%** (KTC 95%: 81–97%) (mẫu số recall: 49 đối tượng gold; precision: 81 box máy trên 4 ảnh gold). Máy bắt gần đủ vật nhưng gán thừa nhiều (36 FP). Số này **lạc quan** vì ngưỡng chọn trên chính 4 ảnh gold.

## 3. Thiên kiến tự động hóa và ảnh bẫy

- Bắt bẫy: **5/12 = 41.7%** (KTC 95%: 19–68%; ngưỡng đạt 80%). Theo loại: missing **0/3**, loose 1/3, extra 2/3, wrong_class 2/3. Bẫy lọt: img_005, img_008, img_014 (missing); img_026, img_027 (loose); img_024 (extra); img_022 (wrong_class).
- Vì sao lọt: tôi **chỉ ẩn nhãn máy và tự quét ảnh ở vài ảnh đầu**, sau đó bỏ bước này cho nhanh và **chủ yếu soi từng box máy đã vẽ**. Cách duyệt này chỉ bắt được lỗi *trên* box có sẵn (extra, wrong_class: 4/6), còn bẫy **missing** không có box nào để soi nên lọt cả 3/3. Box lỏng lọt 2/3 vì tôi không kéo kiểm từng cạnh box khi box "trông đúng".
- Accepted 94.2% nhưng bắt bẫy chỉ 41.7% cho thấy accepted cao **không** có nghĩa máy tốt: phần lớn là tôi giữ nguyên box vì **nhãn máy trông đúng**, và với ảnh dày (005, 024…) tôi **ngại soi từng vùng**. Đây là **thiên kiến tự động hóa**: nhãn máy hợp lý làm tôi ít nhìn lại ảnh. `added` thấp (2.0%) cũng không chứng minh máy ít sót, vì tôi đã không tìm vật sót một cách có hệ thống (missing 0/3).

## 4. Máy có đáng dùng không?

- Thời gian (từ `time_log.csv`): gán tay 20 phút / 4 ảnh = **5 phút/ảnh** vs duyệt 6 phút / 24 ảnh = **0.25 phút/ảnh (~15 giây)**, nhanh hơn ~20 lần. Nhưng so sánh này thiên lệch: duyệt nhanh một phần vì tôi bỏ bước quét ảnh (bẫy chỉ bắt 41.7%), nên 15 giây/ảnh **không** phải thời gian của một lần duyệt đạt chuẩn; ngoài ra gán gold trước giúp tôi đã quen kiểu ảnh.
- Kết luận: **có, trừ `three_wheeler`**. Dựa trên: person / motorbike / car accepted cao (330/331, 125/128, 52/54) và recall máy trên gold 91.8%; còn `three_wheeler` chỉ giữ 20/43 box (xoá 18) nên phải duyệt rất kỹ hoặc vẽ tay. Với bất kỳ lớp nào, kết quả chỉ tin được khi duyệt đúng checklist (lần này bẫy lọt 7/12).
- **Ba quy tắc giảm thiên kiến của bạn** (mỗi quy tắc kiểm tra được bằng số hoặc việc làm):
  1. **Ẩn nhãn máy, tự đếm trước** ở **mọi** ảnh: ghi số vật mỗi lớp mình đếm, so với số box máy; lệch thì tìm cho ra. Kiểm tra: có bảng đếm đủ 24/24 ảnh; bắt bẫy missing ≥ 2/3.
  2. **Chia ảnh dày thành 4 ô** (005, 018, 021, 024, 028, 029) và quét từng ô tìm vật sót. Kiểm tra: mỗi ảnh dày có ghi nhận đã quét 4/4 ô; `added` > 0 hoặc ghi rõ "không sót" cho từng ảnh dày.
  3. **Soi kỹ mọi box `three_wheeler`**: đếm số bánh và so định nghĩa guideline (xe bán tải → `car`, ≥ 4 bánh có thùng → `truck`) trước khi giữ. Kiểm tra: 100% box `three_wheeler` (43/43) đã được chọn mở trong CVAT; ghi số box đổi lớp.

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| img_013 | SAM 3 (`sam3.pt`, Ultralytics) | 1 box + 1 click dương (497,353) | Có lem: mask hơi lấn vào **người đang ngồi trong xe** | Ôm đủ xe, sát biên; lem vào người ngồi trong xe | `three_wheeler_img013_sam3.jpg` |
| img_023 | SAM 3 | 1 box + 1 click dương (292,436) | Không thấy gồm người / hàng | Ôm đủ xe, sát biên; lem rất ít ở biên | `three_wheeler_img023_sam3.jpg` |
| img_017 | SAM 3 | 1 box + 1 click dương (676,478) | Không thấy gồm người / hàng | Ôm đủ xe, sát biên; mask **lem sang bánh xe của xe kế bên** | `three_wheeler_img017_sam3.jpg` |

- So với SAM 2.1 ở Part 2 (r04, mask xe chỉ ôm khoảng 80% xe): SAM 3 **tốt hơn rõ**, cả 3 mask ôm đủ xe và sát biên, đổi lại chạy chậm hơn nhiều trên CPU (78–96 giây/vật so với ~10 giây/ảnh).
- Đối chiếu guideline mục 4.2: lỗi còn lại đều là **lem sang vật khác**. img_013 mask lấn vào người ngồi trong xe, trái với nguyên tắc người là vật riêng (mục 5, ca xích lô); img_017 lem sang bánh xe của xe kế bên, trái với 4.2.4 (hai vật chạm nhau mỗi vật một polygon, không chồng). Guideline 4.2 nói mask gồm gì nhưng **chưa nói cách làm với SAM** khi vật chạm nhau.
- Quy tắc rút ra: **box đúng thì SAM 3 ôm đủ vật, nhưng box chứa cả vật khác thì mask dễ lem**; khi xe chở người hoặc đứng sát xe khác, thêm **click âm** lên người / xe bên cạnh rồi kiểm lại biên. Mask đẹp cũng không sửa được lớp: lớp vẫn do box quyết định (r04 SAM 2.1 vẽ mask cho xe 3 bánh bị gán `car`).
- *(Tuỳ chọn)* Interactor SAM trong CVAT: không làm (phần tuỳ chọn).

## 6. Calibration (Part 6)

- Số quyết định: **5** (`calibration_decisions.csv`: 2 `cau_hoi_mo`, 1 `can_review`, 1 `loi_may`, 1 `bay_lot`). Quan trọng nhất: **CAL-05** (một vật một box) vì box `three_wheeler` chồng lên cùng một xe là lỗi lặp lớn nhất ở QC (3/4 lỗi là `thua`, img_003, img_035), và **CAL-03** (xe bán tải → `car`, quầy hàng → không gán) vì `three_wheeler` là lớp máy tệ nhất (xoá 18/43 box).
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`: chốt câu hỏi mở 2 (hàng xe máy gần camera gán từng xe, CAL-01) và 6 (một vật một box, 4.1.8, CAL-05); 4.3 vật mờ không tách được → 1 box + `can_review` (CAL-02); "Không gồm" của `three_wheeler` thêm xe bán tải và quầy hàng (CAL-03); quy trình QC thêm bước ẩn nhãn máy, tự đếm vật (CAL-04). Câu hỏi mở 1, 3, 4, 5, 7 chưa chốt.
- Nguyên nhân gốc: **guideline mơ hồ** ở CAL-01, CAL-02, CAL-05 (v1.0 chưa nói hàng xe gần camera, vật mờ không tách được, box chồng); **máy sai** ở CAL-03 (YOLOE gán nhầm; v1.0 đã nói xe bán tải là `car` nhưng cột "Không gồm" của `three_wheeler` chưa ghi rõ); **tôi sai** ở CAL-04 (bỏ bước quét ảnh nên lọt 3/3 bẫy missing) — ca này không đổi định nghĩa lớp, chỉ thêm bước quy trình. Lưu ý: CAL-05 tôi chỉ phát hiện ở bước QC; lúc calibration đầu tôi nhận định nhầm img_003 là 3 xe khác nhau, khi soi lại ảnh QC mới thấy là 1 xe có 3 box.

## 7. QC và quyết định (Part 6)

- Mẫu: **5** ảnh (0 ảnh `can_review` — batch không có box nào tick `can_review` — + 5 ngẫu nhiên, seed 42): img_003, img_006, img_014, img_031, img_035; 108 đối tượng. Người kiểm: tôi (**hạn chế**: tôi vừa là người duyệt nên dễ bỏ qua đúng những lỗi mình đã bỏ qua).
- Lỗi theo **đối tượng**: **4/108 = 3.7%** · theo **ảnh**: **2/5 = 40.0%** (ngưỡng Release ≤ 2%).
- Top lỗi: **thừa 3/4** (img_003: 1 xe ba bánh có 3 box `three_wheeler` mái / giữa / thân, box 17 và 21 thừa; img_035: 2 box `three_wheeler` 543 và 544 chồng dọc, ảnh mờ không chắc box nào sai, ghi 544 thừa); **sai vị trí / kích thước 1/4** (img_003 box 18: không box nào ôm đủ xe). img_006, img_014, img_031 không lỗi (img_006 box #76 lúc đầu tôi tưởng là `three_wheeler` vì nhầm màu, thực ra là `car` và đúng). Lỗi `thua` lặp lại đều ở `three_wheeler` → lỗi hệ thống, cần sửa quy tắc (CAL-05), không chỉ sửa từng ảnh.
- **Giới hạn mẫu nhỏ**: 2 ảnh lỗi / 5 ảnh mẫu → cận trên 95% (Wilson) của tỉ lệ ảnh lỗi thật là **77%**. Với 5 ảnh, kể cả 0 lỗi cũng không chứng minh được "dưới 2%" (quy tắc 3: 0/5 vẫn có thể tới ~3/5 = 60%). Ở đây số lỗi đã vượt ngưỡng nên kết luận "không đạt" chắc chắn hơn nhiều so với kết luận "đạt".
- Script kiểm được: file hợp lệ, lớp đúng taxonomy, đủ 24 ảnh, accepted / edited / deleted / added, bẫy bắt được, tỉ lệ lỗi từ phiếu QC. **Không** kiểm được: box có ôm sát vật thật không (img_003 box 18), box chồng là 1 vật hay nhiều vật (img_003, img_035), vật mà cả tôi và máy cùng bỏ sót ngoài gold, và việc tôi có theo checklist thật (tôi bỏ bước quét ảnh sau vài ảnh đầu). Những việc đó cần mắt người.

**Quyết định: Rework**, ngày 2026-11-18. Lý do (đi theo Hold → Rework → Release): **Hold?** Không — các lỗi gặp ở QC đã có quy tắc trong v1.1 (CAL-05 một vật một box, CAL-03 xe bán tải); 5 câu hỏi mở còn lại (1, 3, 4, 5, 7) không gây ra lỗi nào trong mẫu QC, dữ liệu không hỏng. **Rework?** Có — ảnh lỗi 2/5 = 40% > 2%, có lỗi lặp (`thua` 3/4 lỗi, đều là box `three_wheeler` chồng), và bắt bẫy 5/12 = 41.7% < 80% nên lô duyệt chưa đáng tin. Không Release vì không đạt cả ba điều kiện.
Việc cần làm tiếp (Rework): (1) duyệt lại cả 24 ảnh theo checklist mới — ẩn nhãn máy, tự đếm vật ở mọi ảnh, chia ảnh dày thành 4 ô; (2) soi lại toàn bộ 27 box `three_wheeler`: gộp box chồng của cùng một xe thành một box ôm sát (CAL-05), đổi xe bán tải sang `car`, xoá quầy hàng (CAL-03); (3) export `submission/labels/qc_fixed.json`, chạy lại `build_release.py` và `qc_sample.py --force`, ghi vòng QC thứ hai vào báo cáo.

## 8. Ghi chú cá nhân

**Ca khó khi gán gold (Part 1, box tick `can_review`):**

| Ảnh | Đối tượng | Vì sao không chắc | Liên quan guideline |
|---|---|---|---|
| img_001 | person #16 | Ảnh quá mờ, không nhận ra là 1 người hay 2 người; box vẫn cao ≥ 20 px nên vẫn phải gán | 4.1.6 chỉ nói kích thước, chưa nói vật mờ / không tách được cá thể |
| img_001 | motorbike #9, #10 | 3 xe máy đỗ cùng một hàng, cùng một hướng; không rõ nên gán box như thế nào | Câu hỏi mở 2 (xe máy đỗ dày gần camera); 4.1.6 chỉ cho `ignore_region` khi ≥ 5 vật và < 20 px |


- **Dùng AI để làm bài:** Claude Code (Claude, chạy trên máy tôi). Việc AI làm: cài môi trường và tải model; chạy các script (`prelabel`, `threshold_sweep`, `merge_prelabels`, `inject_traps`, `review_stats`, `sam2_points` với SAM 3, `build_release`, `qc_sample`, `qc_report`); giải nén file export CVAT và chạy `validate_labels`; giải thích guideline / rubric / cách dùng CVAT; liệt kê điểm `conf` của box máy theo ảnh (không đánh dấu bẫy); **soạn câu chữ** cho `model_log.csv`, REPORT, `calibration_decisions.csv` và guideline v1.1 **từ nhận xét tôi gửi và các đáp án tôi tự chọn** trong các câu hỏi trắc nghiệm AI đưa ra. AI **không** nhìn ảnh để gán nhãn, không duyệt hay QC thay tôi; gán gold, duyệt batch, soi ảnh `viz/`, xem mask và QC là tôi tự làm. Cách tôi kiểm tra: đọc lại từng câu trong file nộp, đối chiếu số liệu với `review_stats.md`, `threshold_sweep.md`, `qc_report_auto.md`. Một lần AI làm hỏng `model_log.csv` (dấu phẩy chưa bọc nháy) và đã dựng lại từ `work/runs.csv`. Ở bước QC tôi chọn Release trước, AI nhắc rubric và số liệu không đạt, tôi đổi sang Rework.
- Gold có bị nhiễm không? **Không**: tôi gán tay 4 ảnh gold trước khi chạy bất kỳ model nào; AI không xem ảnh gold, chỉ trả lời câu hỏi về quy tắc guideline (vật cao 40 px có thuộc diện < 20 px không, người ngồi trong xe ba bánh có gán không). `traps_key.json` tôi chỉ xem sau khi export `batch_reviewed.json`.
- Máy / GPU / CVAT dùng; vướng mắc cài đặt: Windows 11, không GPU (torch CPU); CVAT v2.74.1 Docker (WSL). Lúc đầu container `traefik` bị tắt nên phải `docker compose up -d` lại. Lần đầu chạy SAM 3 (`sam3.pt`, Ultralytics) thì ultralytics tự cài thêm `timm` vào venv.
- Dùng cache `data/cache/` thay vì tự chạy? **Không**, tự chạy tất cả trên CPU.
- Lỗi / bug script đã gặp: không sửa script nào. `sam2_points.py` không ghi vào `work/runs.csv`, nên dòng r07 (SAM 3) trong `model_log.csv` được ghi tay từ thời gian chạy thật.
- Một điều bạn làm khác lần sau: ẩn nhãn máy và tự đếm vật ở **mọi** ảnh khi duyệt, không chỉ vài ảnh đầu (lần này bỏ bước này nên lọt 3/3 bẫy missing và để sót box `three_wheeler` chồng ở img_003).
