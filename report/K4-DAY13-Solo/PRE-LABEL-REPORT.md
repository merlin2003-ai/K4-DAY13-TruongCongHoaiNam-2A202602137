# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: **H210 — làm solo**
- Thành viên: **Trương Công Hoài Nam** (làm solo; không có teammate).
- Trạng thái thực tế: **`executed-by-solo`**. Đây là mô tả trung thực việc thực hiện; template gốc liệt kê `executed-by-group` / `executed-on-room-LC-machine` / `provided-results`, nhưng lượt này được tự chạy một mình.
- Người thực sự chạy: **Trương Công Hoài Nam**
- Ngày/giờ chạy: **03/10/2026, 09:45:09–09:46:13 (GMT+7)**
- Hệ máy/architecture: **Windows 11, amd64**
- Docker runtime của bundle: **Linux, amd64**
- Image source tag: **`day13-pointpillars:lc-20261001-amd64`**
- Image ID: **`sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`**
- Phiên bản repo/bundle: repo revision **`0831856d921609312d42c7582c366e5a311bb7b1`**; runner ghi `working_tree_dirty=true` trong provenance của bundle.
- PCD được cấp / frame_id: **`demo.pcd` / `demo`**, dataset **KITTI**
- Nơi được phép chạy: **Phòng H210**
- Fingerprint input PCD: **SHA256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`**
- Checkpoint: **`/opt/PointPillars/pretrained/epoch_160.pth`**
- Checkpoint SHA256: **`482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`**
- Phạm vi: **front-window**
- Score threshold: **0.3**
- Giới hạn container: **4 CPU, 4 GB RAM**
- Giả định kênh thứ tư/intensity: PCD Student không giữ reflectance KITTI gốc; pipeline dùng adapter/kênh hằng theo bundle, nên không coi RGB/độ sáng là intensity LiDAR thật.
- Nguồn `z_ground`: được pipeline ước lượng từ PCD; trong output lần này **`z_ground = 0.075 m`**.

`smoke.json` báo **`status: passed`**; cả A, B, C và bước tạo QC cases đều hoàn tất thành công.

## Ba lượt inference thật

A/B/C là ba lượt inference trên cùng PCD, cùng pretrained PointPillars, cùng front-window và score threshold 0.3. Runner chạy đủ ba lượt từ một lệnh. Số hộp và `mean_z` lấy trực tiếp từ `run-A/B/C/summary.csv`; `mean_z` chỉ mô tả cao độ trung bình tâm hộp, không phải điểm chất lượng.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | ---: | ---: | ---: | ---: | --- | --- |
| A | 0 m | 0.16 m | **1** | **0.330 m** | `run-A/boxes-demo-delta-0-voxel-0.16.json`; `run-A/side-demo-delta-0-voxel-0.16.png`; `run-A/summary.csv` | Chỉ có 1 `vehicles`, tâm khoảng x = 13.15 m. Ảnh Side chỉ hiện một cuboid quanh vùng x ≈ 13 m. |
| B | 1.73 m | 0.16 m | **13** | **1.034 m** | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`; `run-B/side-demo-delta-1.73-voxel-0.16.png`; `run-B/summary.csv` | 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`; hộp trải từ x ≈ 3.70 m đến x ≈ 55.58 m. |
| C | 1.73 m | 0.32 m | **6** | **1.091 m** | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`; `run-C/side-demo-delta-1.73-voxel-0.32.png`; `run-C/summary.csv` | Cả 6 prediction là `pedestrian`; chủ yếu ở x ≈ 9–19 m và một hộp tại x ≈ 33.53 m. |

### A/B — chỉ đổi delta

A và B cùng dùng pillar `0.16 m`; biến thay đổi là `delta`, từ `0 m` ở A lên `1.73 m` ở B.

- A có **1 hộp**, class `vehicles`, tâm xấp xỉ `(x, y, z) = (13.154, -0.451, 0.330) m`, score ≈ 0.322.
- B có **13 hộp**, gồm **10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`**, phân bố từ x ≈ 3.70 m tới x ≈ 55.58 m.
- `mean_z` thay đổi từ **0.330 m** ở A lên **1.034 m** ở B.
- Trên ảnh Side, A chỉ có một cuboid quanh x ≈ 13 m; B có nhiều cuboid ở các vùng khoảng x ≈ 3–11 m, 13–25 m, 33–35 m, 41 m và 56 m.

Kết quả cho thấy khi thay `delta` trước inference, output của detector thay đổi không chỉ ở z mà còn ở số lượng, class và vị trí prediction. Đây là chạy lại model trên input đã được biến đổi, không phải lấy các hộp cũ rồi dịch chúng theo z.

Pipeline sử dụng:

`z_model = z_source - z_ground - delta`

và khi trả prediction về hệ nguồn:

`z_source = z_model + z_ground + delta`

Vì vậy không được cộng/trừ `1.73 m` thủ công cho từng cuboid sau inference.

B có nhiều prediction hơn A nhưng **không đủ bằng chứng để kết luận B chính xác hơn A**. `n_boxes` và `mean_z` không phải metric chất lượng, và thí nghiệm này không cung cấp ground truth để chọn cấu hình dựa trên số hộp.

Điều còn chưa chắc: từ Side view và prediction alone chưa thể xác nhận cuboid nào thật sự đúng hình học hoặc vùng nào chắc chắn là false negative; cần nhiều view/reference thích hợp để đánh giá từng đối tượng.

### B/C — chỉ đổi pillar

B và C giữ nguyên `delta = 1.73 m`; biến thay đổi là pillar XY:

- B: `0.16 m`
- C: `0.32 m`

Kết quả:

- B: **13 prediction**, `mean_z = 1.034 m`, gồm 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`.
- C: **6 prediction**, `mean_z = 1.091 m`, cả 6 đều là `pedestrian`.
- Số prediction giảm từ **13 xuống 6**.
- `mean_z` chỉ tăng nhẹ khoảng **0.057 m**.
- Ở B, prediction trải rộng tới x ≈ 55.58 m; ở C, prediction chủ yếu tập trung khoảng x ≈ 9–19 m và một hộp tại x ≈ 33.53 m.
- Các prediction xa quanh x ≈ 41 m và x ≈ 56 m xuất hiện ở B nhưng không còn ở C.

Pillar size quyết định cách các điểm LiDAR trên mặt phẳng x-y được gom thành representation trước khi đưa vào mạng. Khi tăng cạnh pillar từ 0.16 m lên 0.32 m, representation thay đổi đủ lớn để làm thay đổi cả số lượng và class prediction.

Tuy nhiên, **không có đủ bằng chứng để kết luận B tốt hơn C** chỉ vì B có nhiều hộp hơn. C cũng không thể bị coi là kém hơn chỉ vì còn 6 hộp. Lượt C vẫn dùng checkpoint pretrained ban đầu, không phải model được retrain riêng cho pillar 0.32 m.

### Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?

Các lượt inference dùng **front-window**, nên vật thể nằm ngoài ROI không thể được dùng làm bằng chứng detector bị miss trong phép so sánh này.

Ảnh Side là hình chiếu x-z; các vật thể có tọa độ y khác nhau có thể chồng lên nhau. Vì vậy Side view hữu ích để quan sát xu hướng cao độ z và các khác biệt lớn theo x, nhưng không đủ để chốt footprint, chiều rộng hoặc yaw của từng cuboid. Để đánh giá hình học đối tượng cần nhiều góc nhìn hơn và, ở phần Robotaxi, cần đối chiếu camera cùng frame.

### JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?

Cả ba JSON của A/B/C đều là prediction dùng cho thí nghiệm KITTI, **không phải ground truth** và không có đủ cơ sở để import vào job Robotaxi. Chúng thuộc frame/dataset khác với Robotaxi.

Trước khi sử dụng prediction cho annotation cần kiểm đúng frame, schema, coordinate transform, ROI và hình học từng cuboid. Các file trong `qc-cases` cũng tuyệt đối không được import vào CVAT vì chúng là ca `training_only` được biến đổi có chủ đích.

## Ca QC có kiểm soát — không import CVAT

Các ca QC được tạo từ prediction thật của lượt B, không phải ba lượt inference mới và không phải nhãn đúng.

Manifest của `qc-cases` ghi:

- `delta = 1.73 m`
- `z_ground = 0.075 m`
- `height_offset = delta + z_ground = 1.805 m`
- `box_count = 13`
- `training_only = true`

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | ---: | ---: | --- | --- | --- |
| `case-correct` | **0 / 13** | **0 m** | Không | **Mốc đối chiếu; không coi là ground truth** | Bản sao prediction B ở hệ tọa độ nguồn, không có biến đổi z có chủ đích. |
| `case-batch-z` | **13 / 13** | **-1.805 m mỗi hộp** | Không; chỉ z đổi | **Dừng batch, kiểm pipeline** | Cả 13 hộp đều bị hạ đúng 1.805 m. Ảnh Side cho thấy toàn bộ cuboid đồng loạt nằm thấp hơn cụm điểm. |
| `case-one-box-z` | **1 / 13** | **-1.805 m ở hộp đầu tiên** | Không; chỉ z của hộp đầu tiên đổi | **Kiểm riêng hộp** | Hộp đầu tiên quanh x ≈ 8.09 m bị hạ 1.805 m; 12 hộp còn lại giữ nguyên so với `case-correct`. |

### Phân tích `case-correct`

`case-correct` là bản sao prediction nguồn chưa bị sửa z. Ví dụ hộp đầu tiên:

- class: `vehicles`
- x ≈ 8.094 m
- y ≈ 1.208 m
- z ≈ 0.9215 m
- yaw ≈ 2.817 rad
- score ≈ 0.933

Tên `correct` chỉ có nghĩa là helper giữ nguyên phép chuyển đổi hệ tọa độ của prediction B. Nó **không chứng nhận cuboid là ground truth hoặc annotation đúng**.

### Phân tích `case-batch-z`

Ở `case-batch-z`, toàn bộ **13/13 hộp** bị dịch xuống:

`delta + z_ground = 1.73 + 0.075 = 1.805 m`

Ví dụ hộp đầu tiên:

- `case-correct`: z ≈ `0.9215 m`
- `case-batch-z`: z ≈ `-0.8835 m`
- chênh lệch: `-1.805 m`

Các trường class, x, y, length, width, height, yaw và score được giữ nguyên; chỉ z bị thay đổi.

Ảnh `side-batch-z.png` cho thấy các cuboid đồng loạt nằm thấp hơn cụm point cloud, nhiều hộp xuống dưới đường tham chiếu z = 0. Vì lỗi xuất hiện trên toàn batch với cùng một lượng dịch, đây là dấu hiệu của lỗi có tính hệ thống, phù hợp với tình huống bỏ sót inverse conversion/coordinate transform.

**Quyết định:** dừng sửa tay từng cuboid; kiểm lại phép chuyển frame/pipeline và tạo lại prediction từ pipeline đúng.

### Phân tích `case-one-box-z`

Ở `case-one-box-z`, chỉ **1/13 hộp** bị dịch z; 12 hộp còn lại giữ nguyên.

Hộp đầu tiên:

- x ≈ `8.094 m`
- `case-correct`: z ≈ `0.9215 m`
- `case-one-box-z`: z ≈ `-0.8835 m`
- chênh lệch: `-1.805 m`

Ảnh `side-one-box-z.png` cho thấy chỉ cuboid quanh x ≈ 8 m bị rơi xuống dưới cụm điểm; các hộp khác vẫn trùng với `case-correct`.

**Quyết định:** không kết luận toàn pipeline sai chỉ vì một hộp. Cần kiểm riêng đối tượng bằng nhiều view/camera; nếu các hộp khác vẫn đúng hệ tọa độ thì đây phù hợp hơn với lỗi cục bộ của một prediction.

### Kết luận phần QC

- Nếu nhiều/tất cả cuboid cùng lệch một lượng giống nhau trong khi các thuộc tính khác giữ nguyên, cần nghi ngờ **coordinate transform/pipeline** và dừng sửa tay.
- Nếu chỉ một cuboid lệch còn các hộp khác ổn định, cần **kiểm riêng đối tượng** bằng nhiều góc nhìn trước khi chỉnh.
- Nếu bằng chứng không đủ thì ghi **chưa chắc**, không tự suy đoán.
- `case-correct`, `case-batch-z` và `case-one-box-z` đều là dữ liệu **training-only**, không được import vào CVAT và không phải ground truth.

## Nhận xét cá nhân

### Trương Công Hoài Nam

**Vai trò đã làm:** do thực hiện solo, em trực tiếp đảm nhiệm vận hành runner, kiểm cấu hình/JSON, xem ảnh Side, đối chiếu số liệu A/B/C, đọc QC cases và ghi báo cáo.

**Quan sát A/B/C:** khi thay `delta` từ 0 lên 1.73 m trong khi giữ pillar 0.16 m, số prediction thay đổi từ **1 lên 13**. Khi giữ `delta = 1.73 m` và tăng pillar từ 0.16 m lên 0.32 m, số prediction giảm từ **13 xuống 6**, đồng thời class output thay đổi mạnh: B có `vehicles`, `pedestrian`, `two-wheels`, còn C chỉ có `pedestrian`.

**Diễn giải phép z thuận/ngược:** em hiểu rằng `delta` được áp dụng trong phép chuyển z trước inference và được cộng ngược khi trả prediction về hệ tọa độ nguồn. Vì detector được chạy lại trên input đã biến đổi nên thay `delta` có thể làm thay đổi toàn bộ tập prediction; nó không tương đương với việc dịch một tập box cố định sau inference.

**Quyết định khi gặp lỗi batch:** trong `case-batch-z`, cả 13 hộp cùng lệch xuống **1.805 m**, nên em sẽ **dừng sửa tay và kiểm lại pipeline/coordinate transform**, sau đó tạo lại prediction. Trong `case-one-box-z`, chỉ một hộp lệch nên em sẽ kiểm riêng đối tượng bằng nhiều view thay vì kết luận toàn pipeline sai.

**Điều còn chưa chắc:** A/B/C và QC cases không cung cấp ground truth để đánh giá độ chính xác thực sự của từng cuboid. Vì vậy em không kết luận cấu hình nào tốt nhất chỉ từ số hộp, `mean_z` hoặc score; để đánh giá chất lượng hình học cần thêm nhiều view/camera hoặc reference đã được duyệt.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
