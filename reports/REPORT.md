# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Hoàng Tiến Dũng

Công cụ gán nhãn đã dùng: CVAT

## 1. Dữ liệu và cách chia tập

Tập pool và tập kiểm thử được chia theo trục thời gian và có vùng đệm ở giữa
để hạn chế việc các frame gần nhau xuất hiện ở cả hai tập. Với video giao thông
cố định, các frame liên tiếp thường có nội dung rất giống nhau, cùng góc nhìn,
cùng điều kiện ánh sáng và cùng các phương tiện.

Nếu chia ngẫu nhiên theo từng frame, các frame gần như giống nhau có thể bị
chia sang cả pool/train và test. Khi đó mô hình có thể gặp trong quá trình
chọn mẫu hoặc fine-tune những cảnh rất giống với cảnh trong test. Số đo trên
test vì vậy có nguy cơ bị cao hơn thực tế do tương quan giữa các frame, thay
vì phản ánh khả năng tổng quát sang một khoảng thời gian khác.

Vì vậy việc chia theo thời gian và có vùng đệm giúp giảm nguy cơ leakage theo
thời gian và làm phép so sánh giữa các vòng nhất quán hơn.

## 2. Mô hình khởi đầu lạnh (cold start)

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
|---:|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Ở cold start, mô hình đã phát hiện được nhiều xe nhưng vẫn bỏ sót một số
đối tượng trong cảnh ban đêm, đặc biệt là các xe nhỏ hoặc khó quan sát.

Recall theo kích thước cho thấy:

- xe nhỏ: R = 0.182;
- xe vừa: R = 0.547;
- xe lớn: R = 0.561.

Như vậy khả năng phát hiện xe nhỏ thấp hơn rõ rệt so với xe vừa và xe lớn.
Điều này phù hợp với việc xe ở xa thường có ít pixel hơn và dễ bị hòa vào
nền tối hoặc ánh sáng của cảnh.

Tuy nhiên, không nên kết luận ngay mọi trường hợp không có box là lỗi của
model. Một trường hợp cần người rà lại nhãn tham chiếu là xe rất nhỏ hoặc
quá tối, khi ranh giới xe không đủ rõ. Theo guideline, object quá nhỏ hoặc
mờ cần đưa vào review nếu không chắc chắn thay vì tự suy đoán. Nếu ảnh không
đủ bằng chứng thì cần dừng suy đoán và escalate.

## 3. Chiến lược chọn mẫu

Notebook sử dụng:

`score = W_U · U + W_A · A + W_D · D`

với:

- W_U = 0.5: trọng số cho độ bất định của model;
- W_A = 0.3: trọng số cho mức độ box mơ hồ;
- W_D = 0.2: trọng số cho độ đa dạng theo thời gian.

Vì vậy score cao hơn khi frame có dự đoán chưa chắc chắn, có nhiều box cần
kiểm tra hoặc bổ sung sự đa dạng cho batch.

`MIN_GAP_S = 2.0` giây. Đây là khoảng cách thời gian tối thiểu giữa hai frame
được chọn trong cùng một batch. Mục đích là tránh chọn nhiều frame gần như
trùng nhau trong video cố định, qua đó giảm công rà nhãn bị lãng phí cho các
ảnh có thông tin gần giống nhau.

Trong `reports/SELECTION.md`, ba frame thuộc lô model chọn được dùng để minh
họa các trường hợp điểm selection cao:

1. `frame_0133.jpg` — rank 1, score 0.8400, thời điểm 53.2 s. Frame có
   U = 0.8646 và 9 box mơ hồ trên 16 box.
2. `frame_0233.jpg` — rank 3, score 0.7621, thời điểm 93.2 s. Frame có
   26 box và 13 box mơ hồ, với A = 1.0.
3. `frame_0300.jpg` — rank 4, score 0.7500, thời điểm 120.0 s. Frame có
   U = 0.8945 và 10 box mơ hồ trên 18 box.

Một frame có điểm cao nhưng không chọn là `frame_0131.jpg`, rank 2,
score 0.7933, thời điểm 52.4 s. Frame này chỉ cách `frame_0133.jpg`
0.8 giây. Nếu contact sheet cho thấy hai frame gần như cùng cảnh, việc
không chọn `frame_0131.jpg` thể hiện rằng selection không chỉ lấy các score
cao nhất mà còn xét nguy cơ gần trùng.

Điểm bất định không chứng minh rằng frame đó chắc chắn sẽ cải thiện mô hình.
Nó chỉ là tín hiệu để ưu tiên con người kiểm tra. Việc model có thực sự cải
thiện hay không phải được xác nhận sau khi sửa nhãn, fine-tune và đánh giá
trên cùng tập test.

## 4. Các vòng học chủ động (active learning)

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
|---:|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 210 | 0.473 | -0.298 | 1.000 | 0.144 | 0.252 | 0.000 | 0.108 | 0.634 |
| 2 | yolov8n fine-tune vong 1..2 | 24 | 427 | 0.903 | +0.132 | 0.989 | 0.452 | 0.620 | 0.000 | 0.497 | 0.854 |

### Vòng 1 — kiểm/sửa nhãn

Sau khi rà 12 ảnh, số box được giữ nguyên, chỉnh sửa, xoá và thêm mới được
ghi trong `outputs/round1_diff.md`:

- giữ nguyên: **[ĐIỀN THEO round1_diff.md]**
- chỉnh sửa: **[ĐIỀN THEO round1_diff.md]**
- xoá: **[ĐIỀN THEO round1_diff.md]**
- thêm mới: **[ĐIỀN THEO round1_diff.md]**

Tổng số box train sau vòng 1 là 210.

Một ví dụ thực tế trong quá trình rà nhãn là `frame_0000.jpg`: xe tối gần
mép trái được thêm box vì vẫn quan sát được thân xe và đủ ranh giới theo
guideline. Đây là ví dụ cho thấy pre-label của model chỉ là gợi ý và cần
người kiểm tra từng trường hợp.

### Thay đổi metric ở vòng 1

So với cold start:

- AP50: 0.771 → 0.473, giảm 0.298.
- Precision: 0.925 → 1.000.
- Recall: 0.489 → 0.144.
- F1: 0.640 → 0.252.
- R small: 0.182 → 0.000.
- R medium: 0.547 → 0.108.
- R large: 0.561 → 0.634.

Vòng 1 vì vậy chưa cho thấy cải thiện trên test. Precision tăng nhưng Recall
giảm mạnh, cho thấy mô hình trở nên bảo thủ và bỏ sót nhiều xe.

### Vòng 2

Sau khi bổ sung vòng 2, số ảnh train tăng lên 24 và số box train lên 427.

AP50 tăng:

- từ vòng 1: 0.473 → 0.903, tăng 0.430;
- so với cold start: 0.771 → 0.903, tăng 0.132.

Recall tăng từ 0.144 lên 0.452 và F1 tăng từ 0.252 lên 0.620.

Theo kích thước:

- R medium: 0.108 → 0.497, cải thiện rõ;
- R large: 0.634 → 0.854, cải thiện;
- R small: vẫn 0.000, chưa cải thiện.

Do đó cải thiện vòng 2 chủ yếu thể hiện ở nhóm xe vừa và lớn. Khả năng
phát hiện xe nhỏ vẫn là hạn chế rõ ràng.

### Một ca kết quả thay đổi sau fine-tune

Một dạng ca cần đối chiếu trên `compare_round*.jpg` là các xe ở xa và xe tối.
Nếu trước fine-tune model bỏ sót hoặc box chưa đầy đủ nhưng sau fine-tune
xuất hiện box khớp hơn, khả năng kiểm là model đã học thêm các đặc trưng từ
các xe tương tự trong 12 ảnh được sửa.

Ngược lại, nếu một box trước đó đúng nhưng sau fine-tune biến mất hoặc bị
lệch, cần kiểm tra lại ảnh train và nhãn tương ứng trước khi kết luận rằng
fine-tune gây ra lỗi. Có thể nguyên nhân là batch mới chưa đại diện đủ cho
trường hợp đó hoặc model học thiên về các mẫu được bổ sung.

Cần phân biệt ba nguồn bằng chứng:

- `BLIND_SCAN.md`: quan sát của người trước khi xem pre-label AI;
- `REVIEW_LOG.csv` và `round1_diff.md`: những lỗi pre-label đã được người
  kiểm tra và sửa;
- `compare_round*.jpg` và metrics: kết quả của model sau fine-tune trên
  tập test.

Một ca khó theo guideline là xe rất nhỏ hoặc rất mờ trong cảnh tối. Nếu
không đủ bằng chứng để xác định object hoặc ranh giới box, không nên tự
suy đoán mà đưa vào review/escalate.

## 5. Kết luận và giới hạn

Sau vòng 2, AP50 đạt 0.903, cao hơn cold start 0.771 một mức 0.132.
Recall và F1 cũng phục hồi mạnh so với vòng 1. R medium và R large tăng rõ,
trong khi R small vẫn bằng 0.

Nếu xét mục tiêu của lab là chứng minh quy trình học chủ động, kết quả vòng
2 cho thấy việc bổ sung dữ liệu sau vòng 1 có thể cải thiện kết quả trên cùng
tập test. Tuy nhiên, kết quả không chứng minh rằng mọi nhóm đối tượng đều
được cải thiện.

Có thể tiếp tục một vòng sau nếu muốn tập trung vào các trường hợp xe nhỏ.
Hai nhóm ca nên ưu tiên:

1. Xe nhỏ/xe ở xa: chi phí rà nhãn cao hơn vì ranh giới khó quan sát, nhưng
   đây là nhóm đang có R small = 0 và vì vậy cần kiểm tra thêm.
2. Xe tối hoặc bị che một phần: cần kiểm tra kỹ box và guideline; đồng thời
   phải tránh chọn nhiều frame gần nhau trong cùng đoạn video vì có thể làm
   tăng công rà nhãn mà ít bổ sung thông tin.

Các giới hạn chính của phép đánh giá là:

- tập test chỉ có 20 ảnh;
- 14 box cao dưới 16 px được bỏ qua;
- nhãn tham chiếu test được tạo bởi model và chưa được người rà từng box;
- vì vậy AP50 phản ánh mức khớp với bộ tham chiếu hiện tại, không phải một
  ground truth thủ công hoàn toàn độc lập;
- kết quả có thể thay đổi nếu có thêm ảnh test hoặc reference được audit lại.

Do đó không nên kết luận từ AP50 = 0.903 rằng model đã đạt chất lượng thực
địa tương ứng.

Nếu một vòng sau có AP50 giảm, trước khi train thêm cần kiểm tra theo thứ tự:

1. xác nhận tập test và reference không thay đổi;
2. kiểm tra các label mới được sửa, đặc biệt box bị thêm/xóa/chỉnh;
3. kiểm tra batch mới có quá ít trường hợp đại diện hoặc có nhiều frame gần
   trùng hay không;
4. xem `compare_round*.jpg` để xác định model đang bỏ sót hay tạo box sai;
5. kiểm tra lại Recall theo small/medium/large để biết nhóm nào gây giảm;
6. chỉ sau đó mới quyết định thay đổi chiến lược chọn mẫu hoặc training.

Kết quả hiện tại cho thấy vòng 2 đã vượt cold start về AP50, nhưng khả năng
phát hiện xe nhỏ vẫn là vấn đề chưa được giải quyết.