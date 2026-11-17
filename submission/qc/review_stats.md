# Số đo review (det)

- Ảnh tính: 24 (batch,gold); box máy: 573; đối tượng sau duyệt: 558; ngưỡng accepted: IoU >= 0.9
- Đã **loại 21 đối tượng nằm quanh các bẫy** khỏi bảng này: lỗi nhân tạo không phản ánh chất lượng máy.

## 1. Bốn trạng thái (AI error profile)
| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted (giữ nguyên) | 540 | 540/573 = 94.2% | 573 box máy |
| edited (sửa) | 7 | 7/573 = 1.2% | 573 box máy |
| deleted (xoá) | 26 | 26/573 = 4.5% | 573 box máy |
| added (vẽ thêm) | 11 | 11/558 = 2.0% | 558 đối tượng sau duyệt |

Trong số deleted + added, **3 trường hợp là đổi lớp** (máy gán sai lớp, box đúng chỗ): đó không phải máy bỏ sót. Số vật máy **sót thật** ≈ added − đổi lớp = 8.

### Theo lớp
| Lớp | accepted | edited | deleted | added |
|---|---|---|---|---|
| person | 330 | 0 | 1 | 2 |
| bicycle | 5 | 0 | 3 | 1 |
| motorbike | 125 | 1 | 2 | 3 |
| car | 52 | 1 | 1 | 1 |
| bus | 3 | 0 | 0 | 0 |
| truck | 5 | 0 | 1 | 3 |
| three_wheeler | 20 | 5 | 18 | 1 |

## 2. So với gold (nhãn gán tay từ đầu)
Ảnh gold: 4; đối tượng gold: 49 (mẫu số của recall)
| | TP | FP | FN | Precision | Recall |
|---|---|---|---|---|---|
| Máy (pre-label) | 45 | 36 | 4 | 55.6% (KTC 95%: 45–66%) | 91.8% (KTC 95%: 81–97%) |

*Lạc quan: ngưỡng được chọn trên chính các ảnh gold này (in-sample), và chỉ có 4 ảnh. Đọc khoảng tin cậy, đừng đọc con số đơn.*


(Ảnh gold không nằm trong nhãn sau duyệt: không tính residual / new error. Đó là chủ ý: duyệt lại ảnh bạn vừa tự gán tay sẽ đo trí nhớ, không đo thiên kiến.)

## 3. Ảnh bẫy
- Bắt được 5/12 = 41.7% (KTC 95%: 19–68%) (ngưỡng đạt: 80%). Chỉ 12 bẫy nên khoảng tin cậy rất rộng: đọc cùng danh sách bẫy lọt, đừng kết luận chỉ từ %.
  - missing: 0/3
  - loose: 1/3
  - extra: 2/3
  - wrong_class: 2/3
  - lọt: img_008.jpg (missing), img_014.jpg (missing), img_005.jpg (missing), img_024.jpg (extra), img_026.jpg (loose), img_022.jpg (wrong_class), img_027.jpg (loose)

## Gợi ý đọc số
- `deleted` cao → ngưỡng thấp hoặc ánh xạ nhãn sai; `added` cao → máy bỏ sót (xem lớp nào);
- `accepted` thấp (< 50%) → sửa mỗi box lâu hơn vẽ mới: đổi model/ngưỡng hoặc vẽ tay;
- bắt bẫy < 80% → duyệt lại lô, siết checklist (quét ảnh TRƯỚC khi nhìn nhãn máy).
