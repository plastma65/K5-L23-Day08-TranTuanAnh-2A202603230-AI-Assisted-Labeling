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

- Số quyết định: <<>> (`calibration_decisions.csv`); quan trọng nhất (dẫn `CAL-xx`): <<>>
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`: <<>>
- Nguyên nhân gốc: <<guideline mơ hồ / chưa nắm quy tắc / máy>>

## 7. QC và quyết định (Part 6)

- Mẫu: <<n>> ảnh (<<m>> ảnh `can_review` + <<k>> ngẫu nhiên, seed 42); người kiểm: bạn (**hạn chế**: bạn vừa là người duyệt).
- Lỗi theo **đối tượng**: <<.../... = ...%>> · theo **ảnh**: <<.../... = ...%>> (ngưỡng Release ≤ 2%)
- Top lỗi (trong: bỏ sót, thừa, sai lớp, sai vị trí/kích thước, sai thuộc tính, sai mask) + một ảnh ví dụ mỗi loại: <<>>
- **Giới hạn mẫu nhỏ** (cận trên 95% trong `qc_report_auto.md`): <<>>
- Script kiểm được / không kiểm được (ví dụ ôm sát, cùng bỏ sót với máy, theo checklist thật): <<>>

**Quyết định: <<Release / Hold / Rework>>**, ngày <<YYYY-MM-DD>>. Lý do (đi theo Hold → Rework → Release): <<>>
Nếu Rework / Hold, việc cần làm tiếp: <<>>

## 8. Ghi chú cá nhân

**Ca khó khi gán gold (Part 1, box tick `can_review`):**

| Ảnh | Đối tượng | Vì sao không chắc | Liên quan guideline |
|---|---|---|---|
| img_001 | person #16 | Ảnh quá mờ, không nhận ra là 1 người hay 2 người; box vẫn cao ≥ 20 px nên vẫn phải gán | 4.1.6 chỉ nói kích thước, chưa nói vật mờ / không tách được cá thể |
| img_001 | motorbike #9, #10 | 3 xe máy đỗ cùng một hàng, cùng một hướng; không rõ nên gán box như thế nào | Câu hỏi mở 2 (xe máy đỗ dày gần camera); 4.1.6 chỉ cho `ignore_region` khi ≥ 5 vật và < 20 px |


- **Dùng AI để làm bài** (công cụ nào, việc gì, kiểm tra thế nào; "không dùng" nếu không): <<>>
- Gold có bị nhiễm không (bạn có nhìn pre-label hay hỏi AI trước khi gán xong)? <<không / có, như sau: ...>>
- Máy / GPU / CVAT dùng; vướng mắc cài đặt: Windows 11, không GPU (torch CPU); CVAT v2.74.1 Docker (WSL). Lúc đầu container `traefik` bị tắt nên phải `docker compose up -d` lại. Lần đầu chạy SAM 3 (`sam3.pt`, Ultralytics) thì ultralytics tự cài thêm `timm` vào venv.
- Dùng cache `data/cache/` thay vì tự chạy? <<không / có, vì ...>>
- Lỗi / bug script đã gặp (nếu có): <<>>
- Một điều bạn làm khác lần sau: <<>>
