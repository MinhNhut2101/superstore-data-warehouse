# Dashboard Superstore bằng Streamlit

Dashboard chạy nội bộ tại **http://127.0.0.1:8501**.

## Mở dashboard

Tại thư mục `D:\KhoDuLieu`, chạy PowerShell:

```powershell
& .\src\dashboard\run_dashboard.ps1
```

Script khởi động PostgreSQL hiện có và chạy Streamlit trên cổng 8501. Mở địa chỉ
trên bằng trình duyệt; giữ terminal chạy ứng dụng, nhấn Ctrl+C để dừng Streamlit.
Nếu chính sách PowerShell không cho chạy script, dùng các lệnh tương đương:

```powershell
& .runtime/forecast-venv/Scripts/python.exe src/warehouse/setup_local_postgres.py
& .runtime/forecast-venv/Scripts/python.exe -m streamlit run src/dashboard/app.py --server.address 127.0.0.1 --server.port 8501 --server.headless true
```

Ứng dụng đã được cài trong `.runtime/forecast-venv`. Với máy mới:

```powershell
python -m venv .runtime/forecast-venv
& .runtime/forecast-venv/Scripts/python.exe -m pip install -r src/dashboard/requirements.txt -r src/forecasting/requirements.txt
& .runtime/forecast-venv/Scripts/python.exe src/warehouse/setup_local_postgres.py
& .runtime/forecast-venv/Scripts/python.exe src/warehouse/warehouse.py all
& .runtime/forecast-venv/Scripts/python.exe src/forecasting/forecast.py
& .\src\dashboard\run_dashboard.ps1
```

## Nội dung các trang

| Trang | Nội dung |
|---|---|
| Tổng quan | Doanh thu, lợi nhuận, biên lợi nhuận, số đơn; diễn biến tháng và tăng trưởng |
| Sản phẩm & thị trường | Drill-down phân nhóm sản phẩm, quốc gia và các sản phẩm có tổng lợi nhuận âm |
| Khách hàng & đơn hàng | Phân khúc khách hàng, top khách hàng, vận chuyển, ưu tiên đơn và chiết khấu |
| Khám phá OLAP | Chọn xem đủ 12 truy vấn, gồm ROLLUP, CUBE và slice & dice; tải kết quả CSV |
| Dự báo doanh thu | Dự báo 01/2015, biểu đồ thực tế–dự báo, MAE/RMSE/WAPE và đối chứng |

Các trang OLAP dùng bộ lọc tháng, thị trường, nhóm sản phẩm ở thanh bên. Khoảng
tháng bao gồm đầy đủ cả tháng đầu và cuối. Để bỏ bộ lọc, chọn lại toàn bộ giá trị.
Bỏ hết lựa chọn sẽ hiển thị hướng dẫn chọn lại thay vì hiểu ngầm là tất cả.

Trang dự báo có bộ chọn nhóm sản phẩm riêng; dự báo được huấn luyện trên toàn bộ
thị trường nên không áp dụng bộ lọc thị trường hoặc thời gian của OLAP. Tháng
01/2015 là mốc ngay sau dataset, không phải tháng hiện tại.

Biểu đồ có tooltip, biểu đồ đường hỗ trợ kéo và phóng to. Bảng chi tiết nằm trong
các mục mở rộng và có nút tải CSV; dữ liệu xuất giữ nguyên tên trường để dễ đối chiếu.

## Nguồn dữ liệu và quy tắc tính

- OLAP thực thi **SQL trực tiếp trên PostgreSQL**, sử dụng lại 12 truy vấn trong
  `src/warehouse/sql/04_olap_queries.sql`. Các giá trị lọc được truyền bằng tham số
  psycopg, không ghép chuỗi từ dữ liệu người dùng. Kết nối bật chế độ chỉ đọc và
  giới hạn thời gian truy vấn 20 giây.
- Số đơn là số tổ hợp phân biệt `(order_id, order_date, customer_id)`.
- Biên lợi nhuận là `SUM(profit)/SUM(sales)`, không lấy trung bình biên từng dòng.
- Biểu đồ sản phẩm chỉ lấy mức phân nhóm, không cộng các dòng tổng của ROLLUP.
- Bảng ROLLUP/CUBE giữ nguyên dòng tổng và giải thích cách đọc để tránh cộng trùng.
- Tăng trưởng dùng các tháng liên tiếp trong phạm vi đã chọn; tháng không có giao
  dịch được thể hiện bằng 0. Thiếu kỳ trước hoặc mẫu số bằng 0 thì hiển thị dấu —.
  Các tháng trước phạm vi đã chọn không tham gia tính tăng trưởng.
- Truy vấn slice & dice cố định Technology, US/EU, 2014 như truy vấn gốc; bộ lọc
  được áp dụng thêm theo phép giao, vì vậy có thể trả bảng rỗng.
- Dự báo đọc các kết quả đã đánh giá ở `results/forecasting/`, không huấn luyện lại
  mỗi lần mở trang. Trước khi hiển thị, ứng dụng đối chiếu toàn bộ 144 tổng doanh
  thu tháng với kho và kiểm tra hash của dữ liệu mô hình. Nếu khác, yêu cầu chạy
  lại `src/forecasting/forecast.py` để cập nhật.
  Hash chấp nhận cả xuống dòng LF và CRLF để Git chuyển định dạng khi deploy
  không bị nhận nhầm là thay đổi dữ liệu; nội dung thay đổi vẫn bị chặn.
- WAPE tổng trên trang dự báo được tính sau khi cộng doanh thu ba nhóm theo tháng:
  **9,63%** trên năm 2014. WAPE gộp trên 36 cặp nhóm–tháng là chỉ số khác (**12,27%**).
- Đơn vị doanh thu giữ theo nguồn, chưa gán nhãn USD. Không suy diễn quan hệ
  chiết khấu–lợi nhuận thành tác động nhân quả.

Kết quả truy vấn được lưu tạm trong bộ nhớ tối đa 60 giây; nút **Làm mới dữ liệu**
xóa bộ nhớ này. Nút không nạp lại CSV vào kho và không huấn luyện lại mô hình.

## Cấu hình và xử lý sự cố

Mặc định kết nối `127.0.0.1:55432`, database `superstore_dw`, user `superstore`;
mật khẩu đọc từ `.runtime/pgpass.conf`. Có thể dùng `SUPERSTORE_DATABASE_URL` và
`PGPASSFILE` nếu thay đổi máy chủ. Không có mật khẩu được ghi trong mã hoặc giao diện.

Nếu báo lỗi kết nối, chạy lại script khởi động kho rồi bấm **Làm mới dữ liệu**.
Nếu cổng 8501 đang có dashboard chạy, mở URL hiện có; để đổi cổng, dùng
`--server.port 8502` trong lệnh chạy Streamlit.

## Kiểm tra đã thực hiện

```powershell
& .runtime/forecast-venv/Scripts/python.exe -X utf8 src/dashboard/check_dashboard.py
```

Bộ kiểm tra sử dụng PostgreSQL thật và Streamlit AppTest:

- Đối chiếu cả 12 truy vấn với kết quả OLAP đã xuất.
- Đối chiếu KPI sau lọc và khóa tổ hợp đếm đơn với CSV nguồn.
- Mở 5 trang, thử đủ 12 lựa chọn truy vấn và 3 nhóm dự báo.
- Kiểm tra số liệu tổng, sai số từng nhóm, bộ lọc giao diện và lựa chọn rỗng.
- Kiểm tra dự báo được ẩn khi dữ liệu kho thay đổi, và thông báo phục hồi khi mất kết nối.

Kết quả kiểm tra lưu tại `results/dashboard/checks.json`. Các CSV và mô hình nguồn,
notebook tiền xử lý và tài liệu LaTeX không bị thay đổi khi chạy dashboard.

Kiểm tra trình duyệt tùy chọn bằng `check_browser.py` dùng Microsoft Edge cài ở
đường dẫn Windows mặc định và thư viện trong `requirements-dev.txt`; chạy khi
dashboard đã mở. Script lưu ảnh tổng quan và dự báo ở `results/dashboard/`.

Tài liệu API tham khảo: [Streamlit charts](https://docs.streamlit.io/develop/api-reference/charts/st.altair_chart),
[Streamlit AppTest](https://docs.streamlit.io/develop/api-reference/app-testing/st.testing.v1.apptest),
[psycopg parameter binding](https://www.psycopg.org/psycopg3/docs/basic/params.html).
