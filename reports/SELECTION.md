# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, nếu chỉ có ngân sách rà
5 ảnh, tôi ưu tiên các frame sau:

| Thứ tự | Frame | Điểm | Thời điểm | Rank | Lý do |
|---:|---|---:|---:|---:|---|
| 1 | `frame_0133.jpg` | 0.8400 | 53.2s | 1 | Có điểm selection cao nhất trong 50 ứng viên; độ bất định U=0.8646 và có 9 box mơ hồ trên 16 box. |
| 2 | `frame_0233.jpg` | 0.7621 | 93.2s | 3 | Có 26 box và 13 box mơ hồ, đồng thời có D=0.24, cung cấp một cảnh khác so với frame ở khoảng 53s. |
| 3 | `frame_0300.jpg` | 0.7500 | 120.0s | 4 | Có điểm cao, U=0.8945 và 10/18 box mơ hồ, phù hợp để rà các dự đoán chưa chắc chắn. |
| 4 | `frame_0082.jpg` | 0.7459 | 32.8s | 5 | Có 17 box và 10 box mơ hồ; bổ sung một thời điểm khác trong video. |
| 5 | `frame_0118.jpg` | 0.7182 | 47.2s | 11 | Có U=0.9374 và 7/19 box mơ hồ; dù điểm tổng thấp hơn một số ứng viên khác nhưng uncertainty rất cao. |

Tôi ưu tiên không chỉ theo score tổng mà còn xét U, số box mơ hồ và độ khác
biệt về thời gian/cảnh để tránh dùng toàn bộ ngân sách cho các frame gần giống nhau.

## Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/contact sheet

### 1. `frame_0133.jpg`
- Rank: 1
- Score: 0.8400
- Thời điểm: 53.2s
- U: 0.8646
- A: 0.6923
- D: 1.00
- Số box: 16
- Box mơ hồ: 9
- `selected=True`
- Lý do: Đây là frame có score cao nhất trong 50 ứng viên và có nhiều box
  mơ hồ cần con người kiểm tra.

### 2. `frame_0233.jpg`
- Rank: 3
- Score: 0.7621
- Thời điểm: 93.2s
- U: 0.8281
- A: 1.0000
- D: 0.24
- Số box: 26
- Box mơ hồ: 13
- `selected=True`
- Lý do: Có số lượng box và box mơ hồ cao, trong đó A đạt 1.0, nên có nhiều
  dự đoán cần rà soát.

### 3. `frame_0300.jpg`
- Rank: 4
- Score: 0.7500
- Thời điểm: 120.0s
- U: 0.8945
- A: 0.7692
- D: 0.36
- Số box: 18
- Box mơ hồ: 10
- `selected=True`
- Lý do: Độ bất định cao và có nhiều box mơ hồ, đồng thời nằm ở thời điểm
  khác với hai frame trên.

## Một frame có điểm cao nhưng không chọn

`frame_0131.jpg` có rank 2 và score 0.7933, cao thứ hai trong 50 ứng viên,
nhưng không được chọn (`selected=False`).

Frame này có thời điểm 52.4s, chỉ cách `frame_0133.jpg` 0.8 giây. Vì vậy,
nếu hai frame có nội dung/cảnh gần giống nhau trên contact sheet, ưu tiên
`frame_0133.jpg` giúp tránh dùng ngân sách rà nhãn cho các frame quá gần nhau.

Đây là một quyết định có xét yếu tố gần trùng thay vì chỉ lấy 5 score cao nhất.

## Điều phép chọn này chưa chứng minh về chất lượng mô hình

Điểm selection chỉ phản ánh mức độ ưu tiên của frame dựa trên các tín hiệu
được tính trong quá trình chọn mẫu như uncertainty, ambiguity và diversity.
Nó không trực tiếp chứng minh rằng nhãn của frame là đúng hoặc mô hình có
độ chính xác cao.

Chất lượng mô hình cần được đánh giá riêng trên tập kiểm thử cố định bằng
các metric như AP50, Precision, Recall và F1.