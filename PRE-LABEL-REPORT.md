# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Nhóm Đuôi Đài Sen Vàng / Phòng Lab 13
- Thành viên: xem `TEAMMATES.md`.
- Trạng thái: `executed-by-group` (Đã chạy thành công qua runner gói Student và trích xuất kết quả thực tế gửi kèm trong thư mục output `ket-qua-nhom-duoi-dai-sen-vang/`).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Đặng Thanh Tiến (Đại diện nhóm vận hành); 08:22 UTC (15:22 giờ Việt Nam), ngày 01/10/2026; Linux container trên host `amd64`.
- Thư mục output gửi kèm: `ket-qua-nhom-duoi-dai-sen-vang/` (bao gồm đầy đủ các file JSON, PNG, CSV của các lượt A/B/C và qc-cases).
- Image tag và image ID: `day13-pointpillars:lc-20261001-amd64` (`sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`); Phiên bản repo: `0831856d921609312d42c7582c366e5a311bb7b1`.
- PCD được cấp / frame_id: `data/demo.pcd` (mẫu KITTI 000008, 17.238 điểm finite points); fingerprint `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- Checkpoint: PointPillars KITTI pretrained `epoch_160.pth` (`/opt/PointPillars/pretrained/epoch_160.pth`, SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`).
- Phạm vi: front-window (`range: [0.0, -39.68, -3.0, 69.12, 39.68, 1.0]`); score threshold: `0.3`.
- Giả định kênh thứ tư/intensity và nguồn z_ground: PCD không có intensity thật (discarded), kênh reflectance dùng hằng số adapter (0.0 cho `vehicles`, 0.7 cho `pedestrian`/`two-wheels`), RGB=0 placeholder. Nguồn `z_ground` ước lượng từ histogram peak: `z_ground = 0.075 m`.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 m | `run-A/boxes-demo-delta-0-voxel-0.16.json` | Chỉ nhận diện đúng 1 hộp duy nhất tại vị trí $x = 13.15\text{ m}$ ($y=-0.45\text{m}, z=0.33\text{m}$, score `0.322`, label `vehicles`); các phương tiện khác bị sót do $z_{model}$ bị đẩy lệch khỏi dải học. |
| B | 1.73 | 0.16 | 13 | 1.034 m | `run-B/boxes-demo-delta-1.73-voxel-0.16.json` | Mốc so sánh baseline: nhận diện 13 hộp gồm 10 `vehicles`, 1 `two-wheels` (x=10.32m) và 2 `pedestrian` (x=18.67m, x=34.03m). |
| C | 1.73 | 0.32 | 6 | 1.091 m | `run-C/boxes-demo-delta-1.73-voxel-0.32.json` | Nhận diện 6 hộp nhưng **toàn bộ 6 hộp đều bị gán nhãn `pedestrian`** (x từ 9.11m đến 33.53m); bị nhầm lẫn lớp nghiêm trọng do ô pillar 0.32m quá thô. |

- **A/B — chỉ đổi delta**: A có 1 hộp ($x = 13.15\text{ m}$); B có 13 hộp. Ảnh `side-demo-delta-0-voxel-0.16.png` và file JSON ở vùng x ≈ 10–55 m khác nhau rõ rệt về số lượng hộp và cao độ $z$. Lượt A có $z_{model} = z_{source} - z_{ground} - 0$ làm dữ liệu đầu vào bị đẩy lên cao khoảng 1.73m so với phân bố weights của PointPillars KITTI pretrained, khiến model không tạo được anchor proposal ở xa. Đây là chạy lại model trên input khác, không chỉ dịch hộp cũ; điều em còn chưa chắc là độ nhạy của score threshold đối với các cụm điểm thưa phía sau.
- **B/C — chỉ đổi pillar**: B có 13 hộp (10 `vehicles`, 1 `two-wheels`, 2 `pedestrian`); C có 6 hộp. Khi kiểm tra cột `label` trong file `boxes-demo-delta-1.73-voxel-0.32.json`, **toàn bộ 6 hộp ở lượt C đều bị gán nhãn thành `pedestrian`** (scores từ 0.30 đến 0.81, nằm tại các tọa độ x = 19.43m, 13.15m, 9.11m, 33.53m, 13.24m, 10.46m). Việc tăng kích thước ô pillar từ 0.16m lên 0.32m (diện tích ô gấp 4 lần) làm suy giảm độ phân giải đặc trưng BEV, dẫn đến hiện tượng nhầm lẫn lớp (class confusion) nghiêm trọng, biến các cụm điểm phương tiện thành người đi bộ. Chưa đủ bằng chứng để kết luận C tốt hơn B vì chưa có nhãn đúng (ground truth) để đánh giá độ chính xác; tuy nhiên việc thay đổi cỡ ô pillar làm thay đổi trực tiếp kết quả dự đoán nhãn lớp của các hộp.
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**: Ảnh Side là hình chiếu 2D x-z nén toàn bộ bề rộng y [-39.68m, 39.68m], dẫn đến các xe xếp ở y khác nhau bị trùng lấp trên ảnh Side. ROI front-window giới hạn tầm nhìn x ∈ [0, 69.12m]; các đối tượng ở góc sau xe ego không nằm trong cửa sổ xử lý nên không có dự đoán (loại trừ do ROI, không phải model miss).
- **JSON nào còn chưa đủ cơ sở để import?**: Các file JSON A/B/C là dự đoán từ chạy thử nghiệm trên mẫu KITTI demo minh họa, **tuyệt đối không nạp/import vào CVAT** cho các job Robotaxi thực tế của học viên. Đối với `case-correct`, đây chỉ là ca thử nghiệm kiểm soát minh họa bản giữ nguyên phép biến đổi $z$ chuẩn, **không phải là hộp đúng (ground truth)** hay đáp án chuẩn để nạp vào hệ thống.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không | Kiểm từng hộp / Đạt | 100% (13/13) hộp giữ nguyên phép chuyển chuẩn `z_source = z_model + z_ground + delta`. Đây là bản mô phỏng chuyển z đúng, không phải ground truth / hộp đúng. |
| case-batch-z | 13 / 13 | ~-1.805 m | Không | **Dừng batch** | Toàn bộ 13/13 hộp bị tụt cao độ z một lượng đúng bằng $z_{ground} + delta = 0.075 + 1.73 = 1.805\text{m}$. Lỗi do pipeline quên phép cộng ngược. |
| case-one-box-z | 1 / 13 | ~-1.805 m | Không | **Kiểm từng hộp** | Chỉ duy nhất 1 hộp (hộp ID 0 ở x=8.09m) bị chìm z 1.805m, 12 hộp còn lại giữ đúng cao độ. Lỗi địa hình/đối tượng lẻ. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

### 1. Đặng Thanh Tiến — 02099
- **Vai trò đã làm**: Lượt A (Người vận hành lệnh), Lượt B (Người kiểm cấu hình/JSON), Lượt C (Người xem hình học).
- **Quan sát A/B/C**: Qua kết quả chạy tại `ket-qua-nhom-duoi-dai-sen-vang/run-A`, lượt A (`delta=0`) chỉ ra 1 hộp duy nhất ở vị trí $x = 13.15\text{ m}$ (`mean_z=0.330m`, score `0.322`), trong khi lượt B (`delta=1.73`) ra 13 hộp (`mean_z=1.034m`). Đối chiếu file JSON lượt A, các phương tiện khác đều bị mất anchor do tọa độ z đầu vào bị đẩy lên cao làm lệch dải đặc trưng của checkpoint KITTI.
- **Diễn giải phép z**: Phép dịch `z_model = z_source - z_ground - delta` chuẩn hóa cao độ điểm về gốc cảm biến trước khi gom pillar. Phép ngược `z_source = z_model + z_ground + delta` khôi phục cuboid dự đoán về đúng tọa độ không gian thực nghiệm của xe.
- **Quyết định lỗi batch**: Trong `case-batch-z`, khi 13/13 hộp đều bị chìm z đúng 1.805 m ($1.73m + 0.075m$), đây chắc chắn là lỗi hệ thống pipeline quên cộng ngược $z_{ground}+delta$. Em quyết định **dừng nộp batch** và yêu cầu LC sửa lại script khôi phục.
- **Điều chưa chắc**: Mức độ ảnh hưởng của việc gán hằng số reflectance (0.0 cho vehicles, 0.7 cho pedestrian/two-wheels) lên confidence score đối với các dòng xe có vật liệu sơn hấp thụ sóng LiDAR.

### 2. Ngô Lê Đức Anh — 02106
- **Vai trò đã làm**: Lượt A (Người kiểm cấu hình/JSON), Lượt B (Người xem hình học), Lượt C (Người ghi log).
- **Quan sát A/B/C**: Đọc lại cột `label` trong file `ket-qua-nhom-duoi-dai-sen-vang/run-C/boxes-demo-delta-1.73-voxel-0.32.json`, em phát hiện **toàn bộ 6 hộp ở lượt C đều mang nhãn `pedestrian`** (tại các tọa độ $x = 19.43m, 13.15m, 9.11m, 33.53m, 13.24m, 10.46m$). So với lượt B có 10 `vehicles`, việc tăng pillar lên 0.32m không chỉ làm giảm số hộp mà còn gây ra hiện tượng nhầm lẫn lớp (class confusion) nghiêm trọng do ô pillar thô làm mất đặc trưng hình khối không gian BEV.
- **Diễn giải phép z**: Trừ `delta` trước model là để khớp với giả định độ cao cảm biến của checkpoint KITTI. Nếu không cộng lại `delta` và `z_ground` khi xuất JSON, toàn bộ 13 cuboid sẽ bị chìm sâu dưới mặt đất.
- **Quyết định lỗi batch**: Trong `case-one-box-z`, chỉ duy nhất hộp ID 0 bị chìm z 1.805m. Em chọn **kiểm từng hộp**, dùng các góc nhìn chiếu 2D kết hợp camera để nâng lại cao độ hộp lẻ mà không dừng toàn batch.
- **Điều chưa chắc**: Tầm ảnh hưởng của phép quay yaw 180 độ ở vùng phía sau khi bật cờ `--full-scene` đối với việc xác định chính xác tâm hộp z.

### 3. Phạm Thị Oanh — 02055
- **Vai trò đã làm**: Lượt A (Người xem hình học), Lượt B (Người ghi log), Lượt C (Người vận hành lệnh).
- **Quan sát A/B/C**: Đổi kích thước pillar XY từ 0.16m sang 0.32m chỉ làm thay đổi cách gom các ô điểm thành cột đứng trước khi đưa vào mô hình để tính toán và xuất ra các hộp dự đoán (làm thay đổi nhãn/vị trí hộp), chứ không làm thay đổi đám mây điểm PCD nguồn hiển thị trên ảnh Side. Các JSON A/B/C là bài chạy KITTI demo, **không nạp vào CVAT**; `case-correct` cũng chỉ mô phỏng phép đổi z, **không phải hộp đúng / ground truth**.
- **Diễn giải phép z**: Phép biến đổi z là phép dịch tuyến tính dọc theo trục thẳng đứng. Việc ước lượng `z_ground = 0.075 m` giúp loại bỏ độ lệch cao độ địa hình phẳng trước khi gom điểm vào các cột đứng.
- **Quyết định lỗi batch**: Với ca `case-batch-z`, không được dùng CVAT để kéo thủ công từng hộp 3D vì gây tốn thời gian và thiếu chính xác; bắt buộc dừng batch để sửa code chuyển đổi tọa độ.
- **Điều chưa chắc**: Khả năng nhận diện của mô hình đối với các vật thể bị che khuất một phần (occluded) ở khoảng cách xa (>40 m) trong điều kiện dữ liệu LiDAR thưa.

### 4. Phạm Bội Thúy — 02129
- **Vai trò đã làm**: Lượt A (Người ghi log), Lượt B (Người vận hành lệnh), Lượt C (Người kiểm cấu hình/JSON).
- **Quan sát A/B/C**: Đối chiếu lượt B (phát hiện 10 `vehicles`, 1 `two-wheels`, 2 `pedestrian`) và lượt C (chỉ ra 6 hộp và **tất cả 6 hộp đều biến thành `pedestrian`**), em thấy cỡ ô pillar 0.32m đã làm thay đổi đáng kể phân bố đặc trưng BEV của lớp `vehicles`, dẫn đến việc mô hình phân loại nhầm toàn bộ các phương tiện thành người đi bộ.
- **Diễn giải phép z**: Phép chuyển z là chu trình 2 chiều khép kín: chuyển vào không gian đặc trưng của model (`z_model`) và chuyển ngược lại hệ tọa độ thực nghiệm của xe (`z_source`). Nếu đứt gãy chiều ngược, toàn bộ dữ liệu sẽ bị lệch cao độ.
- **Quyết định lỗi batch**: Trong ca `case-correct`, 100% (13/13) hộp bám sát cụm điểm thực tế và cao độ mặt đường. Đây là ca thử nghiệm minh họa phép chuyển z đúng, không phải đáp án chuẩn ground truth.
- **Điều chưa chắc**: Ảnh hưởng của độ dốc mặt đường (incline/decline) cục bộ tới độ chính xác của thuật toán ước lượng `z_ground` dựa trên histogram.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
