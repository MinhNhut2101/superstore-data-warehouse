# Dự báo doanh thu tháng theo nhóm sản phẩm

Mô hình nhận chuỗi doanh thu tháng và dự báo **một tháng tiếp theo** cho Furniture,
Office Supplies và Technology. Tổng doanh thu dự báo bằng tổng của ba nhóm (bottom-up).
Dữ liệu đọc trực tiếp từ PostgreSQL, không sửa dữ liệu nguồn hay notebook tiền xử lý.

## Cách chạy

Chạy PowerShell tại `D:\KhoDuLieu`. Môi trường đã cài trong dự án:

```powershell
python src/warehouse/setup_local_postgres.py
& .runtime/forecast-venv/Scripts/python.exe -X utf8 src/forecasting/forecast.py
```

Nếu máy mới chưa có môi trường và thư viện:

```powershell
python -m venv .runtime/forecast-venv
& .runtime/forecast-venv/Scripts/python.exe -m pip install -r src/forecasting/requirements.txt
```

Trên máy mới cũng cần tạo và nạp kho bằng `src/warehouse/warehouse.py all` trước khi
dự báo. Thông tin kết nối mặc định là `127.0.0.1:55432`, database `superstore_dw`;
mật khẩu lấy từ `.runtime/pgpass.conf`. Có thể truyền `--dsn` hoặc đặt
`SUPERSTORE_DATABASE_URL`, tương tự mã kho hiện có. Không đưa mật khẩu vào Git.

Chạy các kiểm tra nhỏ về mô hình đối chứng, sai số và thứ tự thời gian:

```powershell
& .runtime/forecast-venv/Scripts/python.exe -X utf8 -m unittest discover -s src/forecasting -p test_forecast.py -v
```

## Dữ liệu và cách đánh giá

- `monthly_sales.sql` lấy `SUM(sales)` theo tháng của **order_date** và `category`
  từ `dw.v_sales_analysis`: 144 quan sát, 48 tháng/nhóm từ 01/2011 đến 12/2014.
- Đối chiếu toàn bộ 144 tổng doanh thu với `superstore_processed.csv`. Không tự
  điền tháng thiếu bằng 0; dừng để kiểm tra nếu chuỗi thiếu hoặc trùng tháng.
- Dùng 2011–2012 làm lịch sử ban đầu (đủ hai chu kỳ mùa vụ).
- Năm 2013 là tập validation để chọn mô hình theo WAPE, riêng cho từng nhóm.
- Năm 2014 là tập test. Giữ nguyên lựa chọn mô hình sau validation.
- Dự báo cuốn chiếu **một tháng**: khi dự báo tháng t chỉ dùng dữ liệu trước t;
  sau khi đã có số thực tế tháng t mới bổ sung nó vào lịch sử cho tháng tiếp theo.
  Ví dụ, dự báo 02/2014 được phép dùng số thực tế 01/2014. Đây không phải dự báo
  cả 12 tháng từ một mốc cố định vào cuối 2013.
- Sau đánh giá, huấn luyện lại mô hình đã chọn bằng đủ 48 tháng để dự báo 01/2015.
  Không dùng kết quả test để chọn lại cấu hình.

Ba ứng viên, chốt trước khi chạy đánh giá:

| Tên trong mã | Phương pháp |
|---|---|
| `seasonal_naive` | Doanh thu cùng tháng năm trước, mốc đối chứng |
| `holt_winters` | Holt–Winters, xu hướng cộng, mùa vụ cộng, chu kỳ 12 tháng |
| `holt_winters_damped` | Như trên, thêm giảm dần xu hướng |

Tham số làm trơn được statsmodels tối ưu trên lịch sử tại từng mốc. Chia doanh thu
cho trung bình **tập huấn luyện hiện tại** để ổn định tối ưu số học; nhân lại khi
xuất dự báo. Các ứng viên cùng áp dụng chặn dự báo âm về 0. Nếu tối ưu không hội tụ,
chương trình báo lỗi, không âm thầm thay mô hình.

Với sai số `e = actual - predicted`:

- MAE = trung bình `abs(e)`, cùng đơn vị doanh thu gốc.
- RMSE = căn bậc hai trung bình `e²`, nhạy hơn với sai số lớn.
- WAPE (%) = `100 × SUM(abs(e)) / SUM(abs(actual))`, càng nhỏ càng tốt.

WAPE không phải độ chính xác phân loại; không diễn giải `100% - WAPE` thành
“accuracy”. Không gán đơn vị USD khi chưa xác minh đơn vị tiền tệ của nguồn.

## Kết quả đã chạy

Cả ba nhóm chọn Holt–Winters xu hướng cộng, mùa vụ cộng theo validation 2013.

| Nhóm | WAPE đối chứng (test 2014) | WAPE mô hình chọn | MAE mô hình chọn | RMSE mô hình chọn |
|---|---:|---:|---:|---:|
| Furniture | 20,84% | 11,13% | 12.777,32 | 15.723,76 |
| Office Supplies | 22,59% | 9,15% | 9.953,57 | 15.610,73 |
| Technology | 26,37% | 15,76% | 21.231,49 | 27.721,75 |

Gộp sai số trên 36 cặp nhóm–tháng: WAPE giảm từ **23,45% xuống 12,27%**.
Nếu cộng ba nhóm thành doanh thu toàn hệ thống mỗi tháng rồi tính sai số trên
12 tháng, WAPE là **9,63%**, đối chứng **20,79%**. Hai chỉ số khác nhau vì sai số
các nhóm có thể bù trừ; không lấy trung bình ba WAPE để gọi là WAPE toàn hệ thống.

Dự báo điểm tháng **01/2015**, theo đơn vị gốc:

| Nhóm | Doanh thu dự báo |
|---|---:|
| Furniture | 112.812,22 |
| Office Supplies | 110.231,22 |
| Technology | 129.332,40 |
| Tổng | **352.375,83** |

Tổng tính từ giá trị chưa làm tròn. Tháng 01/2015 nằm ngoài dataset nên chưa có
số thực tế để chấm sai số. Đây là dự báo theo mốc lịch sử của bài tập, không phải
dự báo tháng hiện tại. Technology có sai số cao nhất, cần thận trọng hơn khi dùng
để lập mục tiêu. Chỉ có bốn năm dữ liệu và một năm kiểm tra, chưa đủ để khẳng định
hiệu quả ổn định trong mọi giai đoạn. Chưa ước lượng khoảng dự báo.

## Các tệp kết quả

Tất cả nằm trong `results/forecasting/`, giữ nguyên quy ước chỉ một tệp CSV tại
`data/processed/`.

| Tệp | Nội dung |
|---|---|
| `monthly_sales.csv` | 144 tổng doanh thu lấy từ kho |
| `validation_predictions.csv` | 108 dự báo validation của ba ứng viên |
| `validation_metrics.csv` | MAE, RMSE, WAPE của từng ứng viên theo nhóm |
| `selected_models.csv` | Lựa chọn dựa trên validation |
| `test_predictions.csv` | 72 dòng: 36 dự báo được chọn và 36 dự báo đối chứng; có mốc cuối huấn luyện |
| `test_metrics.csv` | Sai số test theo nhóm, gộp các cặp nhóm–tháng và tổng theo tháng |
| `test_total_monthly.csv` | Thực tế và dự báo tổng doanh thu theo tháng kiểm tra |
| `forecast_next_month.csv` | Dự báo 01/2015 theo nhóm, mô hình, tỷ lệ tăng so cùng tháng năm trước |
| `forecast_total.csv` | Tổng doanh thu dự báo 01/2015 |
| `forecast_overview.png` | Biểu đồ lịch sử, kiểm tra 2014 và dự báo mới |
| `model_*.pkl` | Ba mô hình cuối đã huấn luyện bằng dữ liệu đến 12/2014 |
| `evaluation.json` | Nguồn, phiên bản thư viện, thiết lập đánh giá, hash dữ liệu và kiểm tra |

Mô hình `.pkl` hoạt động trên doanh thu đã chia cho `training_scale` trong
`forecast_next_month.csv`; khi gọi `.forecast(1)` cần nhân lại thang đo này và
chặn về tối thiểu 0. Chỉ mở pickle do chính dự án tạo. Chạy lại `forecast.py` là
cách đơn giản nhất để cập nhật dữ liệu và tái tạo toàn bộ kết quả.

## Ý nghĩa cho DSS và phạm vi sử dụng

Có thể dùng dự báo để đặt mục tiêu doanh thu tháng tiếp theo và xác định nhóm hàng
cần theo dõi. Doanh thu không phải số lượng bán, nên không dùng trực tiếp kết quả
này làm số lượng nhập hàng. Mô hình dựa trên lịch sử doanh thu, không ước lượng
tác động nhân quả của khuyến mãi, giá bán hay các quyết định kinh doanh.

Nguồn phương pháp:

- [Statsmodels: ExponentialSmoothing](https://www.statsmodels.org/stable/generated/statsmodels.tsa.holtwinters.ExponentialSmoothing.html).
- [Forecasting: Principles and Practice — benchmark và đánh giá theo thời gian](https://otexts.com/fpp3/toolbox.html).
