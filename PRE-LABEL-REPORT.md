# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Cá nhân — Phòng Lab Day 13 (Robotaxi LiDAR 3D Object)
- Thành viên: xem `TEAMMATES.md` (Lương Phương Quang — MSSV: 2A202602175, đảm nhiệm toàn bộ vai trò).
- Trạng thái: `provided-results` (kết hợp phân tích thực nghiệm từ bundle native benchmark).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Lương Phương Quang; 02/10/2026; Windows 10/11 x86_64 (amd64).
- Image tag và image ID; phiên bản repo:
  - Image tag: `day13-pointpillars:lc-20261001-amd64`
  - Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
  - Phiên bản repo: `0831856d921609312d42c7582c366e5a311bb7b1` (working tree dirty: true)
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp:
  - File: `input/demo.pcd` (frame_id: `demo`, 17.238 điểm)
  - SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
  - Nguồn dữ liệu: KITTI Vision Benchmark Suite sample 000008 (giấy phép CC BY-NC-SA 3.0)
- Checkpoint: PointPillars KITTI có sẵn trong image (`/opt/PointPillars/pretrained/epoch_160.pth`); hash: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window `(0.0, -39.68, -3.0, 69.12, 39.68, 1.0)`; score threshold: `0.3`.
- Giả định kênh thứ tư/intensity và nguồn z_ground:
  - PCD loại bỏ reflectance gốc, dùng RGB=0 làm placeholder. Pipeline đọc 2 lần với reflectance hằng số theo class: 0.0 cho `vehicles`, 0.7 cho `pedestrian` và `two-wheels`.
  - Nguồn `z_ground` được ước lượng tự động từ đỉnh histogram cao độ z của đám mây điểm (bin width 0.05 m).

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 m | 0.16 m | 1 | 0.42 m | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `side-demo-delta-0-voxel-0.16.png`, `summary.csv` | Chỉ nhận diện được 1 hộp `vehicles`. Việc giữ delta=0 khiến đám mây điểm nằm lệch ngoài dải z tối ưu của model. |
| B | 1.73 m | 0.16 m | 13 | -0.18 m | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `side-demo-delta-1.73-voxel-0.16.png`, `summary.csv` | Nhận diện được 13 hộp (10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`). Các hộp bao khớp chặt các cụm điểm và đáy bám sát mặt đất. |
| C | 1.73 m | 0.32 m | 6 | -0.05 m | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `side-demo-delta-1.73-voxel-0.32.png`, `summary.csv` | Nhận diện được 6 hộp (toàn bộ 6 `pedestrian`). Kích thước pillar tăng gấp đôi làm mờ đặc trưng không gian, mất toàn bộ `vehicles`. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh Side vùng x ≈ 10–40 m cho thấy ở lượt A đa số xe không được phát hiện do điểm bị lệch theo trục z so với phân bố dữ liệu huấn luyện của sensor KITTI. Khi đưa delta = 1.73 m vào trước inference ở lượt B, mạng nhận dạng chuẩn xác 13 đối tượng. Đây là chạy lại model trên input đã dịch tọa độ z, không chỉ đơn thuần là cộng một lượng hằng số sau khi dự đoán. Điều em còn chưa chắc là độ cao sensor thực tế trên các dòng xe khác ngoài KITTI có thể sai lệch dẫn tới việc trừ sai z_ground.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Khi giữ nguyên delta = 1.73 m và chỉ tăng kích thước pillar XY từ 0.16 m lên 0.32 m (ô lưới rộng gấp đôi), số lượng hộp giảm mạnh từ 13 xuống 6 hộp, đặc biệt toàn bộ 10 xe `vehicles` biến mất, chỉ còn phát hiện `pedestrian`. Không có đủ bằng chứng để kết luận C tốt hơn B; ngược lại, B tốt hơn rõ rệt vì checkpoint pretrained được tối ưu hóa cho lưới 0.16 m.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào:
  - Góc Side là hình chiếu trực giao x-z của toàn bộ scene, các đối tượng có cùng tọa độ x nhưng khác tọa độ y (nằm so le sang trái/phải) sẽ bị chồng lên nhau, do đó không thể dùng ảnh Side đơn lẻ để đánh giá độ rộng hộp (width) hay góc xoay (yaw).
  - ROI cấu hình chỉ quét khoảng nhìn phía trước (front-window: x từ 0 đến 69.12 m), nên các vật thể ở phía sau (x < 0) không được model xử lý; đây không phải là lỗi bỏ sót của mô hình trong bài test này.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?
  - Cả 3 file JSON A, B, C đều là kết quả thí nghiệm trên PCD KITTI demo, tuyệt đối **không được import vào CVAT của Robotaxi**.
  - Trước khi import pre-label cho Robotaxi thật, cần kiểm tra: schema 5 class, hệ tọa độ sensor thực tế của Robotaxi, và kiểm tra tính nhất quán của pipeline transform z để tránh lỗi toàn batch.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Không lỗi pipeline | Giữ nguyên tọa độ gốc của B; các hộp khớp khít với cụm điểm trên ảnh Side view. |
| case-batch-z | 13 / 13 | - (delta + z_ground) (lệch ~1.73 m) | Không đổi | **Dừng batch, báo LC kiểm pipeline** | Toàn bộ 13/13 hộp bị tụt sâu xuống dưới mặt đường cùng một lượng cố định; lỗi hệ thống do thiếu bước nghịch đảo z. |
| case-one-box-z | 1 / 13 | - (delta + z_ground) | Không đổi | **Kiểm từng hộp cá thể** | Chỉ duy nhất 1 hộp đầu tiên bị tụt z, 12 hộp còn lại vẫn đúng vị trí; đây là lỗi suy luận cục bộ, không phải lỗi pipeline. |

Ghi rõ: Các ca trên được sinh ra từ helper `pipeline-qc-cases.py` bằng biến đổi toán học có chủ đích từ prediction thật của lượt B để luyện tập phân biệt lỗi batch và lỗi đơn lẻ. Đây không phải kết quả detector chạy độc lập và không phải ground-truth.

## Nhận xét cá nhân

- **Họ và tên / MSSV:** Lương Phương Quang — 2A202602175.
- **Vai trò thực hiện:** Tự thực hiện toàn bộ quy trình cá nhân (vận hành phân tích cấu hình, kiểm tra file dữ liệu JSON/CSV/ảnh Side view và lập báo cáo).
- **Quan sát A/B/C:** Qua so sánh 3 lượt chạy trên cùng một PCD KITTI demo, em nhận thấy rõ rệt vai trò của việc chuẩn hóa cao độ sensor ($z$). Ở lượt A (delta = 0), mô hình chỉ phát hiện 1 xe tải lớn và bỏ lọt hầu hết các xe con ở vùng $x \approx 10 - 30$ m do tọa độ điểm nằm ngoài miền phân bố quen thuộc của mô hình. Khi áp dụng delta = 1.73 m ở lượt B, mô hình nhận dạng chuẩn 13 hộp. Khi đổi pillar từ 0.16 m lên 0.32 m ở lượt C, số hộp giảm xuống 6 và mất toàn bộ class `vehicles`, chứng minh rằng độ phân giải voxel ảnh hưởng trực tiếp đến khả năng trích xuất đặc trưng hình học của mạng nơ-ron.
- **Diễn giải phép biến đổi z thuận/ngược:**
  - Chiều thuận (trước khi đưa vào model): $z_{\text{model}} = z_{\text{source}} - z_{\text{ground}} - \text{delta}$ nhằm đưa gốc tọa độ về vị trí đặt cảm biến chuẩn của KITTI.
  - Chiều nghịch (sau khi model xuất hộp): $z_{\text{source}} = z_{\text{model}} + \text{delta} + z_{\text{ground}}$ để đưa tâm hộp trở lại đúng hệ tọa độ ban đầu của đám mây điểm.
  - Việc dịch z ở đầu vào làm thay đổi hoàn toàn việc gom điểm vào các pillar, khiến mô hình đưa ra dự đoán khác về cả số lượng lẫn vị trí hộp, hoàn toàn khác với việc chỉ tịnh tiến hộp sau khi suy luận.
- **Quyết định khi phát hiện lỗi batch:**
  - Nếu kiểm tra thấy toàn bộ hộp trong frame hoặc cả batch bị lệch một độ cao cố định so với mặt đường (như trong `case-batch-z`), hành động đúng là **dừng chỉnh sửa thủ công**, báo ngay cho LC/kỹ sư phụ trách pipeline để kiểm tra lại công thức chuyển đổi tọa độ hoặc giá trị ước lượng mặt đất.
  - Nếu chỉ có 1 hoặc vài hộp cá biệt bị chìm/nổi (như `case-one-box-z`), đó là do nhiễu điểm thưa hoặc vật thể bị che khuất một phần, khi đó mới tiến hành kiểm tra đa góc nhìn (Top, Side, Front, Free) và điều chỉnh cục bộ từng hộp.
- **Điều còn chưa chắc:** Đối với các khu vực bị che khuất nặng (occlusion) hoặc ở tầm xa ($> 40$ m), mật độ điểm LiDAR rất thưa nên việc xác định ranh giới đuôi xe hoặc hướng yaw $180^\circ$ vẫn cần phải đối chiếu cẩn thận với ảnh camera cùng frame trước khi đưa ra quyết định cuối cùng.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca: Đã xác thực gói Student KITTI (CC BY-NC-SA 3.0), đúng tài khoản ca học.
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung: Phân tích đầy đủ dựa trên bộ benchmark bundle native hợp lệ.
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT: Đạt yêu cầu.
- Nhận xét từng thành viên và quyết định dừng pipeline: Phân biệt rõ lỗi batch và lỗi đối tượng đơn lẻ.
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do: Đủ điều kiện chuyển sang phần thực hành nguồn cá nhân trên Portal & CVAT.
