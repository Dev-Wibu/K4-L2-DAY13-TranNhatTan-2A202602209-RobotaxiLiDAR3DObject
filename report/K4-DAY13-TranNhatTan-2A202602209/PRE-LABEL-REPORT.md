# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Cá nhân (Làm một mình)
- Thành viên: xem `TEAMMATES.md` (Trần Nhật Tân - MSSV: 2A202602209).
- Trạng thái: `executed-by-group`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Trần Nhật Tân; 02/10/2026; Windows 11 x86_64 / amd64.
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-amd64` (ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`); repo git: `0831856d`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `demo.pcd` (KITTI 000008, 17.238 points); chạy local CPU container; SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- Checkpoint: PointPillars KITTI có sẵn trong image; ghi checkpoint ID/hash nếu LC cấp: `/opt/PointPillars/pretrained/epoch_160.pth` (SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`).
- Phạm vi: front-window; score threshold: 0.3.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Bỏ reflectance nguồn, đặt hằng số theo class adapter (0.0 cho vehicles, 0.7 cho ped/cyclist); z_ground ước lượng từ đám mây điểm cục bộ.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | boxes-demo-delta-0-voxel-0.16.json / side-*.png | Chỉ phát hiện 1 hộp duy nhất do không có phép bù chiều cao sensor z (delta=0), mây điểm bị lệch khỏi khoảng cao độ phân phối mà model được huấn luyện |
| B | 1.73 | 0.16 | 13 | 1.034 | boxes-demo-delta-1.73-voxel-0.16.json / side-*.png | Phát hiện 13 hộp (10 vehicles, 2 pedestrian, 1 two-wheels). Khi dịch z đúng chiều cao sensor (1.73m), mây điểm khớp vùng học của model |
| C | 1.73 | 0.32 | 6 | 1.091 | boxes-demo-delta-1.73-voxel-0.32.json / side-*.png | Phát hiện 6 hộp (toàn bộ là pedestrian). Khi tăng kích thước pillar gấp đôi (0.32m), độ phân giải x-y bị thô, mất thông tin chi tiết của xe lớn |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh Side cho thấy ở A mây điểm nằm ở cao độ không khớp, model bỏ sót gần như toàn bộ xe phía trước; ở B có 13 hộp bám sát các cụm điểm thực tế. Đây là chạy lại model trên input khác, không chỉ dịch hộp cũ; điều em còn chưa chắc là độ chính xác tuyệt đối của tâm z do mặt đất chỉ là ước lượng trung bình.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Khi đổi pillar XY từ 0.16m lên 0.32m, số lượng hộp giảm mạnh và class bị đảo lộn (toàn bộ thành pedestrian, mất xe ô tô). Không đủ bằng chứng để kết luận C tốt hơn; thực tế checkpoint pretrained được huấn luyện cho grid 0.16m nên thay đổi biểu diễn làm giảm chất lượng nhận diện rõ rệt.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Góc Side là hình chiếu 2D (x-z) nên các đối tượng có cùng khoảng cách x và z nhưng khác y sẽ bị chồng chèn lên nhau; không thể xác định hướng quay (yaw) hay phân biệt các xe song song chỉ bằng ảnh Side.
- JSON nào còn chưa đủ cơ sở để import? Cả A và C đều chưa đủ cơ sở để import. Lượt B cũng chỉ là pre-label gợi ý, cần kiểm tra đối chiếu qua 4 góc nhìn và ảnh camera trước khi đưa vào gán nhãn chính thức.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Kiểm tra từng hộp trước khi dùng | Giữ nguyên phép chuyển đổi tọa độ chuẩn từ lượt B; các hộp khớp với cụm điểm trên ảnh Side |
| case-batch-z | 13 / 13 | -1.805 m (-(z_ground + delta)) | Không đổi | DỪNG BATCH, kiểm tra pipeline | Toàn bộ 13/13 hộp đều bị chìm sâu xuống dưới lòng đất cùng một lượng cố định; lỗi nằm ở khâu thiếu phép cộng ngược z_source = z_model + delta + z_ground |
| case-one-box-z | 1 / 13 | Lệch ở 1 hộp duy nhất | Không đổi ở 12 hộp còn lại | KIỂM TỪNG HỘP | 12 hộp vẫn bám sát mặt đất, chỉ có đúng 1 hộp bị lệch cao độ; đây là lỗi dự đoán cá biệt của model, không phải lỗi pipeline hệ thống |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

Trần Nhật Tân (MSSV: 2A202602209) — Vai trò: Thực hiện toàn bộ quy trình thí nghiệm, phân tích dữ liệu và đánh giá chất lượng cá nhân.
- Quan sát A/B/C: Lượt B (delta=1.73m, voxel=0.16m) cho kết quả phong phú và bám điểm tốt nhất với 13 hộp. Khi đổi delta=0 (lượt A), model gần như "mù" vì sai lệch hệ tọa độ sensor. Khi tăng voxel lên 0.32m (lượt C), độ phân giải lưới quá thô khiến các xe ô tô bị mất và nhận diện nhầm thành pedestrian.
- Diễn giải phép z thuận/ngược: Trong pipeline, trước khi inference, tọa độ mây điểm được chuyển đổi qua công thức `z_model = z_source - z_ground - delta`. Sau khi model dự đoán ra hộp trong hệ tọa độ model, bắt buộc phải thực hiện phép biến đổi ngược `z_source = z_model + z_ground + delta` để đưa hộp về lại hệ quy chiếu ban đầu của xe.
- Quyết định lỗi batch: Khi gặp hiện tượng 100% hộp trong frame bị lệch cùng một độ cao z (như trong `case-batch-z`), tuyệt đối không sửa tay từng hộp mà phải dừng toàn bộ pipeline, báo cáo kỹ sư/LC để sửa lại công thức biến đổi z trong code inference. Khi chỉ có 1 hộp bị lệch (`case-one-box-z`), đây là lỗi ngẫu nhiên của mô hình, xử lý bằng cách chỉnh tay từng hộp trên CVAT.
- Điều chưa chắc: Do dữ liệu PCD demo đã lược bỏ cường độ phản xạ (intensity) thực và gán giá trị hằng số, khả năng nhận diện các vật thể nhỏ hoặc vật liệu phản xạ đặc thù vẫn còn phụ thuộc nhiều vào hình học cụ thể của từng cụm điểm.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca: Đạt (sử dụng đúng bộ kit student KITTI).
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung: Đã phân tích chi tiết dữ liệu thí nghiệm.
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT: Đạt.
- Nhận xét từng thành viên và quyết định dừng pipeline: Đạt, hiểu rõ bản chất lỗi batch vs lỗi đơn lẻ.
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do: Đồng ý chuyển sang hoàn thiện job nguồn và QC.
