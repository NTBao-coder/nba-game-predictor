# NBA Game Predictor

Dự đoán xác suất thắng của từng trận NBA mỗi ngày bằng XGBoost, cập nhật theo phong độ gần đây của các đội. Dữ liệu lấy từ NBA Stats API qua thư viện [`nba_api`](https://github.com/swar/nba_api).

Trạng thái: đang lên kế hoạch. Chưa có code.

## Bài toán

Trước mỗi ngày thi đấu, với mỗi trận trong lịch: xác suất mỗi đội thắng là bao nhiêu?

Input là kết quả và thống kê của các trận đã đá, lịch thi đấu và thông tin nghỉ ngơi của từng đội. Output là bảng xác suất cho mọi trận trong ngày, kèm những yếu tố ảnh hưởng nhiều nhất tới từng dự đoán.

Mục tiêu chính là xác suất được hiệu chỉnh tốt: khi mô hình nói 70%, đội đó phải thắng khoảng 70% số lần. Đoán đúng đội thắng là mục tiêu phụ.

## Liên hệ với NBA Cup Predictor

[`nba-cup-predictor`](https://github.com/NTBao-coder/nba-cup-predictor) dùng logistic regression với 5 đặc trưng để mô phỏng một giải đấu. Dự án này tập trung riêng vào mô hình từng trận cho toàn mùa giải, với nhiều đặc trưng hơn và mô hình phi tuyến. Nếu mô hình ở đây tốt hơn, nó có thể thay vào phần mô hình của `nba-cup-predictor`, vì phần mô phỏng ở đó chỉ cần xác suất thắng của từng cặp đội.

## Phương pháp

```
kết quả + box score ─▶ đặc trưng trước trận ─▶ XGBoost ─▶ hiệu chỉnh xác suất ─▶ dự đoán hằng ngày
lịch thi đấu ─────────┘
```

### Đặc trưng

Mọi đặc trưng chỉ dùng thông tin có trước trận đấu, và được tính dưới dạng chênh lệch giữa đội nhà với đội khách.

| Nhóm | Đặc trưng |
|---|---|
| Phong độ | Offensive, defensive và net rating trung bình trượt 5, 10 và 20 trận |
| Four factors | eFG%, tỉ lệ mất bóng, tỉ lệ bắt bóng bật bảng tấn công, tỉ lệ ném phạt, cho cả tấn công và phòng thủ |
| Sức mạnh dài hạn | Elo, cập nhật sau mỗi trận và kéo về trung bình giữa các mùa |
| Nhịp độ | Pace trung bình |
| Lịch thi đấu | Số ngày nghỉ, trận thứ hai liên tiếp (back-to-back), số trận trong 7 ngày gần nhất |
| Di chuyển | Khoảng cách di chuyển từ trận trước, chênh lệch múi giờ |
| Địa điểm | Sân nhà, sân khách hoặc sân trung lập |

Đầu mùa, chỉ số trượt được co về chỉ số mùa trước theo số trận đã đá, giống cách làm trong `nba-cup-predictor`.

### Mô hình

| Mô hình | Vai trò |
|---|---|
| Elo thuần | Baseline đơn giản |
| Logistic regression | Baseline tuyến tính, cùng bộ đặc trưng |
| XGBoost | Mô hình chính, bắt tương tác như back-to-back khi đá sân khách |

Siêu tham số của XGBoost được chọn bằng kiểm định chéo theo thời gian. Xác suất đầu ra được hiệu chỉnh bằng isotonic regression hoặc Platt scaling trên một mùa riêng. SHAP giải thích yếu tố nào đẩy xác suất lên hoặc xuống trong từng trận.

## Dữ liệu

| Endpoint `nba_api` | Nội dung |
|---|---|
| `LeagueGameFinder` | Kết quả và box score cơ bản của mọi trận |
| `BoxScoreAdvancedV3` | Offensive/defensive rating, pace của từng trận |
| `BoxScoreFourFactorsV3` | Four factors của từng trận |
| `ScoreboardV2` | Lịch thi đấu trong ngày |

Phạm vi dự kiến là các mùa từ 2015-16 đến nay. Toạ độ các nhà thi đấu được lưu sẵn để tính khoảng cách di chuyển.

## Đánh giá

- Kiểm định walk-forward: huấn luyện trên dữ liệu tới một thời điểm, dự đoán các trận sau đó, rồi dịch cửa sổ lên. Cách này mô phỏng đúng việc dự đoán hằng ngày.
- Chỉ số chính: log loss và Brier score. Chỉ số phụ: accuracy.
- Biểu đồ reliability để kiểm tra độ hiệu chỉnh của xác suất.
- So với ba baseline: đội nhà luôn thắng, Elo thuần và logistic regression.

## Công nghệ

Python, Pandas, scikit-learn, XGBoost, SHAP, `nba_api`, Streamlit cho dashboard, GitHub Actions để chạy dự đoán hằng ngày.

## Cấu trúc dự kiến

```
gamepred/
  data.py       Lấy và lưu dữ liệu từ NBA Stats API
  elo.py        Tính Elo
  features.py   Đặc trưng trước trận
  model.py      Huấn luyện, hiệu chỉnh, giải thích bằng SHAP
  evaluate.py   Kiểm định walk-forward và biểu đồ reliability
app.py          Dashboard Streamlit: xác suất các trận trong ngày
predict.py      Dự đoán các trận của một ngày
train.py        Huấn luyện và đánh giá
```

## Lộ trình

| Giai đoạn | Nội dung | Trạng thái |
|---|---|---|
| 1 | Thu thập kết quả và box score nâng cao | Dự kiến |
| 2 | Elo và baseline logistic regression | Dự kiến |
| 3 | Đặc trưng phong độ, four factors, lịch thi đấu, di chuyển | Dự kiến |
| 4 | XGBoost, kiểm định walk-forward, hiệu chỉnh xác suất | Dự kiến |
| 5 | Giải thích dự đoán bằng SHAP | Dự kiến |
| 6 | Dashboard Streamlit các trận trong ngày | Dự kiến |
| 7 | GitHub Actions chạy dự đoán hằng ngày và lưu lịch sử | Dự kiến |
| 8 | Thêm dữ liệu chấn thương và đội hình ra sân | Dự kiến |

## Hạn chế dự kiến

- Chưa có dữ liệu chấn thương và cầu thủ nghỉ, vốn là yếu tố ảnh hưởng lớn nhất tới kết quả từng trận.
- Đặc trưng cấp đội không phản ánh được thay đổi đội hình giữa mùa, như trao đổi cầu thủ.
- stats.nba.com giới hạn số request, nên tải box score nâng cao cho nhiều mùa sẽ mất nhiều thời gian.
- Dự án phục vụ học tập và phân tích, không dùng cho cá cược.
