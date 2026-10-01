# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Nhóm Đuôi Đài Sen Vàng / Phòng Lab 13
- Thành viên: xem `TEAMMATES.md` (Đặng Thanh Tiến - 02099, Ngô Lê Đức Anh - 02106, Phạm Thị Oanh - 02055, Phạm Bội Thúy - 02129).
- Trạng thái: `executed-by-group` (Đã chạy thành công qua runner gói Student và trích xuất kết quả thực tế tại `Downloads/ket-qua-nhom-01`).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Đặng Thanh Tiến (Đại diện nhóm vận hành); 01/10/2026 15:23 UTC (16:59 UTC+7); Linux container trên Windows host / `amd64`.
- Image tag và image ID: `day13-pointpillars:lc-20261001-amd64` (`sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`); Phiên bản repo: `0831856d921609312d42c7582c366e5a311bb7b1`.
- PCD được cấp / frame_id: `data/demo.pcd` (mẫu KITTI 000008, 17.238 điểm finite points); fingerprint `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- Checkpoint: PointPillars KITTI pretrained `epoch_160.pth` (`/opt/PointPillars/pretrained/epoch_160.pth`, SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`).
- Phạm vi: front-window (`range: [0.0, -39.68, -3.0, 69.12, 39.68, 1.0]`); score threshold: `0.3`.
- Giả định kênh thứ tư/intensity và nguồn z_ground: PCD không có intensity thật (discarded), kênh reflectance dùng hằng số adapter (0.0 cho `vehicles`, 0.7 cho `pedestrian`/`two-wheels`), RGB=0 placeholder. Nguồn `z_ground` ước lượng từ histogram peak: `z_ground = 0.075 m`.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z  | File JSON/Side/CSV                            | Quan sát có bằng chứng                                                                                                                       |
| ---- | ----- | --------- | ------ | ------- | --------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| A    | 0     | 0.16      | 1      | 0.330 m | `run-A/boxes-demo-delta-0-voxel-0.16.json`    | Chỉ nhận diện đúng 1 hộp xe (`vehicles` ở x=8.09m); các phương tiện khác không đủ điều kiện kích hoạt anchor do điểm z bị dịch lên cao.      |
| B    | 1.73  | 0.16      | 13     | 1.034 m | `run-B/boxes-demo-delta-1.73-voxel-0.16.json` | Baseline chuẩn: nhận diện đủ 13 hộp gồm 10 `vehicles`, 1 `two-wheels` (x=10.32m) và 2 `pedestrian` (x=18.67m, x=34.03m).                     |
| C    | 1.73  | 0.32      | 6      | 1.091 m | `run-C/boxes-demo-delta-1.73-voxel-0.32.json` | Giảm xuống còn 6 hộp (chỉ còn `vehicles`); mất hoàn toàn 3 vật thể nhỏ (`two-wheels` và `pedestrian`) do pillar 0.32m làm thô đặc trưng BEV. |

- **A/B — chỉ đổi delta**: A có 1 hộp; B có 13 hộp. Ảnh `side-demo-delta-0-voxel-0.16.png` và file JSON ở vùng x ≈ 10–55 m khác nhau rõ rệt về số lượng hộp và cao độ $z$. Lượt A có $z_{model} = z_{source} - z_{ground} - 0$ làm dữ liệu đầu vào bị đẩy lên cao khoảng 1.73m so với phân bố weights của PointPillars KITTI pretrained, khiến model không tạo được anchor proposal ở xa. Đây là chạy lại model trên input khác, không chỉ dịch hộp cũ; điều em còn chưa chắc là độ nhạy của score threshold đối với các cụm điểm thưa phía sau.
- **B/C — chỉ đổi pillar**: B có 13 hộp; C có 6 hộp. Ảnh `side-demo-delta-1.73-voxel-0.32.json` và file JSON ở vùng x ≈ 10–35 m mất toàn bộ các vật thể mảnh nhỏ (`two-wheels` tại x=10.32m score 0.38, `pedestrian` tại x=18.67m score 0.34 và x=34.03m score 0.32). Số lượng/lớp/vị trí thay đổi: giảm 7 hộp do ô pillar $0.32\text{m} \times 0.32\text{m}$ (diện tích gấp 4) làm nhòe ranh giới feature map BEV. Không đủ bằng chứng để kết luận C tốt hơn B, ngược lại B (0.16m) vượt trội hơn về độ phân giải chi tiết.
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**: Ảnh Side là hình chiếu 2D x-z nén toàn bộ bề rộng y [-39.68m, 39.68m], dẫn đến các xe xếp ở y khác nhau bị trùng lấp trên ảnh Side. ROI front-window giới hạn tầm nhìn x ∈ [0, 69.12m]; các đối tượng ở góc sau xe ego không nằm trong cửa sổ xử lý nên không có dự đoán (loại trừ do ROI, không phải model miss).
- **JSON nào còn chưa đủ cơ sở để import?**: JSON lượt A (sai delta sensor height khiến sót 12 hộp) và JSON lượt C (pillar thô khiến mất vật thể nhỏ) không dùng được. JSON lượt B dù phát hiện 13 hộp chuẩn cũng chỉ là prediction gợi ý (pre-label), học viên cần đối chiếu kĩ ảnh camera và điểm PCD thực tế trước khi Save vào CVAT.

## Ca QC có kiểm soát — không import CVAT

| Ca             | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng                                                                                                                                       |
| -------------- | ------------------------ | ---------- | --------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| case-correct   | 0 / 13                   | 0 m        | Không                 | Kiểm từng hộp / Đạt                    | 100% (13/13) hộp giữ nguyên phép chuyển chuẩn `z_source = z_model + z_ground + delta`, tâm z bám sát cụm điểm.                                   |
| case-batch-z   | 13 / 13                  | ~-1.805 m  | Không                 | **Dừng batch**                         | Toàn bộ 13/13 hộp bị tụt cao độ z một lượng đúng bằng $z_{ground} + delta = 0.075 + 1.73 = 1.805\text{m}$. Lỗi do pipeline quên phép cộng ngược. |
| case-one-box-z | 1 / 13                   | ~-1.805 m  | Không                 | **Kiểm từng hộp**                      | Chỉ duy nhất 1 hộp (hộp ID 0 ở x=8.09m) bị chìm z 1.805m, 12 hộp còn lại giữ đúng cao độ. Lỗi địa hình/đối tượng lẻ.                             |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

### 1. Đặng Thanh Tiến — 02099

- **Vai trò đã làm**: Lượt A (Người vận hành lệnh), Lượt B (Người kiểm cấu hình/JSON), Lượt C (Người xem hình học).
- **Quan sát A/B/C**: Qua kết quả chạy thực tế tại `Downloads/ket-qua-nhom-01`, lượt A (`delta=0`) chỉ ra 1 hộp (`n_boxes=1`, `mean_z=0.330m`), trong khi lượt B (`delta=1.73`) ra 13 hộp (`mean_z=1.034m`). Đối chiếu file `boxes-demo-delta-0-voxel-0.16.json`, 12 phương tiện ở khoảng xa x > 10m đều bị sụt giảm anchor do tọa độ z đầu vào bị đẩy lên cao.
- **Diễn giải phép z**: Phép dịch `z_model = z_source - z_ground - delta` chuẩn hóa cao độ điểm về gốc cảm biến trước khi gom pillar. Phép ngược `z_source = z_model + z_ground + delta` khôi phục cuboid dự đoán về đúng tọa độ không gian thực nghiệm của xe.
- **Quyết định lỗi batch**: Trong `case-batch-z`, khi 13/13 hộp đều bị chìm z đúng 1.805 m ($1.73m + 0.075m$), đây chắc chắn là lỗi hệ thống pipeline quên cộng ngược $z_{ground}+delta$. Em quyết định **dừng nộp batch** và yêu cầu LC sửa lại script khôi phục.
- **Điều chưa chắc**: Mức độ ảnh hưởng của việc gán hằng số reflectance (0.0 cho vehicles, 0.7 cho pedestrian/two-wheels) lên confidence score đối với các dòng xe có vật liệu sơn hấp thụ sóng LiDAR.

### 2. Ngô Lê Đức Anh — 02106

- **Vai trò đã làm**: Lượt A (Người kiểm cấu hình/JSON), Lượt B (Người xem hình học), Lượt C (Người ghi log).
- **Quan sát A/B/C**: Ở lượt C (`voxel_size=0.32m`), số lượng hộp sụt giảm còn 6 hộp so với 13 hộp ở lượt B (`voxel_size=0.16m`). Kiểm tra `boxes-demo-delta-1.73-voxel-0.32.json` cho thấy các đối tượng kích thước nhỏ như `two-wheels` ($x=10.32m$) và 2 `pedestrian` ($x=18.67m$, $x=34.03m$) bị loại bỏ hoàn toàn do ô pillar lớn gộp đặc trưng làm nhòe độ phân giải BEV.
- **Diễn giải phép z**: Trừ `delta` trước model là để khớp với giả định độ cao cảm biến của checkpoint KITTI. Nếu không cộng lại `delta` và `z_ground` khi xuất JSON, toàn bộ 13 cuboid sẽ bị chìm sâu dưới mặt đất.
- **Quyết định lỗi batch**: Trong `case-one-box-z`, chỉ duy nhất hộp ID 0 bị chìm z 1.805m. Em chọn **kiểm từng hộp**, dùng các góc nhìn chiếu 2D kết hợp camera để nâng lại cao độ hộp lẻ mà không dừng toàn batch.
- **Điều chưa chắc**: Tầm ảnh hưởng của phép quay yaw 180 độ ở vùng phía sau khi bật cờ `--full-scene` đối với việc xác định chính xác tâm hộp z.

### 3. Phạm Thị Oanh — 02055

- **Vai trò đã làm**: Lượt A (Người xem hình học), Lượt B (Người ghi log), Lượt C (Người vận hành lệnh).
- **Quan sát A/B/C**: Xem ảnh `side-demo-delta-1.73-voxel-0.32.png` của lượt C, các mảng điểm x-z bị mờ ranh giới rõ rệt so với lượt B. Cấu hình B (`voxel=0.16m`) duy trì ranh giới thể tích vật thể chi tiết, giúp thuật toán NMS lọc bớt các hộp trùng lặp tốt hơn.
- **Diễn giải phép z**: Phép biến đổi z là phép dịch tuyến tính dọc theo trục thẳng đứng. Việc ước lượng `z_ground = 0.075 m` giúp loại bỏ độ lệch cao độ địa hình phẳng trước khi gom điểm vào các cột đứng.
- **Quyết định lỗi batch**: Với ca `case-batch-z`, không được dùng CVAT để kéo thủ công từng hộp 3D vì gây tốn thời gian và thiếu chính xác; bắt buộc dừng batch để sửa code chuyển đổi tọa độ.
- **Điều chưa chắc**: Khả năng nhận diện của mô hình đối với các vật thể bị che khuất một phần (occluded) ở khoảng cách xa (>40 m) trong điều kiện dữ liệu LiDAR thưa.

### 4. Phạm Bội Thúy — 02129

- **Vai trò đã làm**: Lượt A (Người ghi log), Lượt B (Người vận hành lệnh), Lượt C (Người kiểm cấu hình/JSON).
- **Quan sát A/B/C**: Lượt B là cấu hình chuẩn nhất (`delta=1.73m`, `voxel=0.16m`), nhận diện chính xác 13 hộp thuộc cả 3 class (`vehicles`, `two-wheels`, `pedestrian`). File `summary.csv` cho `mean_z = 1.034 m` phản ánh trung bình cao độ tâm thực tế của các vật thể trên đường.
- **Diễn giải phép z**: Phép chuyển z là chu trình 2 chiều khép kín: chuyển vào không gian đặc trưng của model (`z_model`) và chuyển ngược lại hệ tọa độ thực nghiệm của xe (`z_source`). Nếu đứt gãy chiều ngược, toàn bộ dữ liệu sẽ bị lệch cao độ.
- **Quyết định lỗi batch**: Trong ca `case-correct`, 100% (13/13) hộp bám sát cụm điểm thực tế và cao độ mặt đường. Đủ điều kiện làm pre-label gợi ý ban đầu để học viên đối chiếu kiểm tra trước khi Save.
- **Điều chưa chắc**: Ảnh hưởng của độ dốc mặt đường (incline/decline) cục bộ tới độ chính xác của thuật toán ước lượng `z_ground` dựa trên histogram.

## LC ghi nhận riêng
