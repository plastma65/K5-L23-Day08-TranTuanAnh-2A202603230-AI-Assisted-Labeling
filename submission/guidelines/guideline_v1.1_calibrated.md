# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202603230 - Trần Tuấn Anh

> Chỉ ghi phần thay đổi so với v1.0. Mỗi ca khó có ảnh ví dụ và khớp một dòng trong `qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 2 | Khối xe máy đỗ dày **gần camera**: từng xe hay `ignore_region`? | **Gán từng xe.** Xe nào còn thấy ≥ 20% và box cao ≥ 20 px thì có box `motorbike` riêng; box chồng nhau là bình thường. `ignore_region` chỉ dùng khi cụm ≥ 5 vật **và** vật cao < 20 px (giữ nguyên 4.1.6). | img_001 (3 xe máy đỗ cùng một hàng, cùng hướng) | CAL-01 |
| 6 | **Box máy chồng nhau** (nhiều box trên cùng một vật) | **Một vật chỉ một box.** Box máy chồng lên cùng một xe (ví dụ box mái, box giữa, box thân) thì giữ / sửa một box ôm sát cả xe và xoá các box còn lại → thêm quy tắc 4.1.8. Phát hiện ở bước QC, không phải lúc duyệt. | img_003 (1 xe ba bánh có 3 box), img_035 (2 box chồng dọc) | CAL-05 |

Câu hỏi mở **chưa chốt** ở v1.1: 1 (xe đạp điện), 3 (hàng nhô ra), 4 (ngưỡng 20% người trong xe có mui), 5 (kính xe trong mask), 7 (conf tối thiểu). Lý do: trong 4 ảnh gold và 24 ảnh batch tôi chưa gặp ca đủ rõ để chốt (riêng câu 4 có liên quan mask img_013 ở Part 5 nhưng chỉ 1 ví dụ).

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| img_001 | Người quá mờ, không nhận ra 1 hay 2 người, box cao ≥ 20 px (box can_review person #16) | (gold, không xem máy) | 1 box `person` bao phần nhìn thấy + tick `can_review` | **Mục 4.3** thêm: vật mờ không tách được cá thể nhưng cao ≥ 20 px → vẽ **một** box theo lớp tin nhất + `can_review`; không đoán tách, không dùng `ignore_region` (CAL-02) |
| img_002, img_011 (pilot); batch có box `three_wheeler` bị xoá: img_003, img_009, img_014 | Xe bán tải, quầy hàng bị gán `three_wheeler` (thấy rõ ở img_002, img_011) | YOLOE-26s, prompt `tuk-tuk` / `cyclo` / `cargo tricycle`, ngưỡng 0.15; batch xoá 18/43 box `three_wheeler` | Xe bán tải → `car`; quầy hàng → không gán | **Mục 2**, cột "Không gồm" của `three_wheeler`: thêm *xe bán tải 4 bánh có cabin (→ `car`), quầy hàng / xe đẩy bán hàng (không gán)*. Chỉ gán `three_wheeler` khi đếm được 3 bánh hoặc thấy rõ dáng tuk-tuk / xích lô / ba gác; không đếm được → `can_review` (CAL-03) |
| img_005, img_008, img_014 | Bẫy **missing** lọt 3/3 | (bẫy cài vào pre-label) | Lỗi của người duyệt, không đổi định nghĩa lớp | **Mục 7** thêm quy trình: mỗi ảnh phải **ẩn nhãn máy, tự đếm vật** trước khi xem box máy (CAL-04) |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Chốt câu hỏi mở 2 (hàng xe máy gần camera gán từng xe) và 6 (một vật một box, thêm 4.1.8); thêm 4.3 vật mờ không tách được → 1 box + can_review; thêm "Không gồm" cho `three_wheeler` (xe bán tải, quầy hàng); thêm bước ẩn nhãn / tự đếm vào quy trình QC | CAL-01 … CAL-05 |
