# Thông tin học viên

- Họ và tên: Trần Tuấn Anh
- MSHV: 2A202603230
- Lớp (L2 / L3 / khác): L2
- Tài khoản GitHub: plastma65
- Repo bài nộp (URL): https://github.com/plastma65/K5-L23-Day08-TranTuanAnh-2A202603230-AI-Assisted-Labeling
- Commit chốt bài (hash): 5b030c0
- Máy: Windows 11, không có GPU (chạy CPU)
- CVAT (Docker): v2.74.1

## Kết quả tóm tắt (điền cuối bài; số lấy từ `submission/qc/*`, kèm mẫu số)

- Máy: accepted / edited / deleted: 540/573 = 94.2% / 7/573 = 1.2% / 26/573 = 4.5% (mẫu số: 573 box máy); added: 11/558 = 2.0% (mẫu số: 558 đối tượng sau duyệt)
- So với gold: precision / recall của **máy** 45/81 = 55.6% / 45/49 = 91.8% (49 đối tượng gold, lạc quan vì ngưỡng chọn trên chính gold)
- Bắt bẫy: 5/12 = 41.7% (missing 0/3, loose 1/3, extra 2/3, wrong_class 2/3)
- Thời gian: gán tay 20/4 = 5 phút/ảnh; duyệt có pre-label 6/24 = 0.25 phút/ảnh (~15 giây, duyệt quá nhanh, bỏ bước quét ảnh)
- Lỗi QC: 40.0% theo ảnh (2/5, cận trên 95% = 77%), 3.7% theo đối tượng (4/108)
- **Quyết định:** Rework — lý do một dòng: ảnh lỗi 2/5 = 40% > 2%, lỗi lặp box `three_wheeler` chồng (thừa 3/4 lỗi), bắt bẫy 5/12 = 41.7% < 80%
