# Kho dữ liệu Superstore

Hiện thực bốn bước: chốt thiết kế vật lý, tạo PostgreSQL, nạp dữ liệu và truy vấn OLAP. Các file báo cáo và notebook tiền xử lý được giữ nguyên.

## Thiết kế

Schema dw có năm bảng chiều và fact_sales. Tên bảng trong PostgreSQL dùng chữ thường.

| Bảng | Mức chi tiết / khóa chính |
|---|---|
| dim_date | Một ngày; PK full_date |
| dim_customer | Một khách hàng; PK customer_id |
| dim_product | Một tổ hợp sản phẩm; PK (product_id, product_name) |
| dim_geography | Một tổ hợp địa lý; PK (country, state, city, region, market) |
| dim_order | Một tổ hợp đơn; PK (order_id, order_date, customer_id) |
| fact_sales | Một dòng hàng; PK row_id, giữ từ Row.ID |

Fact_Sales có 6 FK: hai ngày cùng tham chiếu Dim_Date; khách hàng tham chiếu Dim_Customer; ba FK ghép tham chiếu sản phẩm, địa lý và đơn. Không sinh thêm cột khóa.

| Trường trong Fact_Sales | Kiểu | Ràng buộc |
|---|---|---|
| row_id | integer | PK, dương |
| order_date, ship_date | date | NOT NULL, FK đến ngày; ngày gửi không trước ngày đặt |
| product_id, product_name | varchar(30), varchar(255) | NOT NULL; FK ghép đến sản phẩm |
| country, state, city, region, market | varchar(100), varchar(100), varchar(100), varchar(50), varchar(20) | NOT NULL; FK ghép đến địa lý |
| customer_id | varchar(20) | NOT NULL; FK khách hàng và thành phần FK đơn |
| order_id | varchar(30) | NOT NULL; cùng ngày đặt và khách hàng tạo FK đơn |
| weeknum | smallint | NOT NULL, 1–53; giữ nguyên tuần nguồn |
| sales | numeric(18,5) | NOT NULL, không âm |
| profit | numeric(18,5) | NOT NULL; giữ lợi nhuận âm |
| quantity | smallint | NOT NULL, dương |
| discount | numeric(6,4) | NOT NULL, 0–1 |
| shipping_cost | numeric(18,5) | NOT NULL, không âm |
| profit_margin | numeric(18,8) | NULL khi sales = 0; khớp profit/sales trong sai số 1e-8 |

Chiều ngày chứa lịch liên tục bao phủ cả ngày đặt và ngày gửi. Năm–quý–tháng và năm–tuần ISO chỉ nằm ở chiều ngày. Kiểu dữ liệu và PK/FK theo sơ đồ; các ràng buộc NOT NULL và CHECK kiểm tra dữ liệu khi nạp.

Khi tổng hợp biên lợi nhuận dùng SUM(profit)/NULLIF(SUM(sales),0), không cộng hay trung bình tỷ lệ dòng. Đếm đơn theo đủ ba trường nhận diện, không đếm riêng order_id. Số đơn không cộng trực tiếp giữa các nhóm sản phẩm. Phân tích chiết khấu mô tả lịch sử, chưa xác định tác động nhân quả.

## Cách chạy

Chạy từ thư mục gốc bằng Python 3. Các chương trình ở đây chỉ dùng thư viện chuẩn; không cần cài thêm pandas hoặc driver.

    python src/warehouse/setup_local_postgres.py
    python src/warehouse/warehouse.py all

Lần đầu, chương trình tải PostgreSQL Windows x64 chính thức từ EDB vào .runtime, tạo cụm riêng và chạy trên 127.0.0.1:55432. Không cài dịch vụ Windows. Database: superstore_dw; user: superstore.

Mật khẩu tự sinh nằm trong .runtime/password.txt; các lệnh tự dùng .runtime/pgpass.conf. Không đưa .runtime lên Git.

Có thể chạy riêng từng bước:

    python src/warehouse/warehouse.py create
    python src/warehouse/warehouse.py load
    python src/warehouse/warehouse.py verify
    python src/warehouse/warehouse.py olap

Để dùng PostgreSQL có sẵn, đặt PSQL_PATH và SUPERSTORE_DATABASE_URL, hoặc truyền --dsn. Database phải tồn tại và tài khoản có quyền tạo schema/bảng. Khi đó không cần chạy setup.

Đầu vào duy nhất là data/processed/superstore_processed.csv. Thư mục data/processed tiếp tục chỉ chứa CSV này.

## Nạp và kiểm tra

1. Đọc CSV vào bảng tạm bằng psql COPY phía client.
2. Lấy các tổ hợp phân biệt nạp chiều trước, rồi nạp fact.
3. ON CONFLICT cập nhật theo khóa, tránh nhân đôi khi chạy lại; toàn bộ lần nạp nằm trong một transaction.
4. Nối đủ các chiều để đối chiếu từng dòng và toàn bộ 26 thuộc tính với CSV. Các tổng tiền được đối chiếu bằng Decimal, cùng số dòng và cả 6 FK.

Đây là cơ chế nạp snapshot hiện tại, chưa xử lý xóa nguồn hoặc lưu lịch sử SCD Type 2. Tổ hợp mô tả thay đổi sẽ được nhận diện là tổ hợp mới.

## Kết quả

Thư mục results/warehouse chứa:

- verification.json: số dòng từng bảng, ràng buộc và đối soát toàn bộ cột.
- reconciliation.csv: bốn tổng kiểm soát trước–sau.
- olap/*.csv: kết quả 12 truy vấn thực thi trên PostgreSQL.
- olap_summary.json: số dòng kết quả và trạng thái kiểm tra OLAP.
- integration_checks.json: kiểm tra nạp lại, ràng buộc và rollback khi dữ liệu sai.

SQL nằm trong sql/01_create_tables.sql đến sql/04_olap_queries.sql. View dw.v_sales_analysis cung cấp dữ liệu nối cho OLAP và dashboard về sau.

Các truy vấn trong sql/04_olap_queries.sql và CSV tương ứng:

| CSV trong results/warehouse/olap | Câu hỏi / thao tác |
|---|---|
| 01_overview.csv | Tổng doanh thu, lợi nhuận, sản lượng và số nhóm đơn |
| 02_monthly_growth.csv | 48 tháng; tăng trưởng so tháng trước và cùng tháng năm trước |
| 03_time_rollup.csv | Roll-up tháng → quý → năm → toàn kỳ |
| 04_product_drilldown.csv | Ngành hàng → nhóm sản phẩm |
| 05_geography.csv | Hiệu quả theo nhóm thị trường, thị trường, vùng và quốc gia |
| 06_customer_segments.csv | Doanh thu/lợi nhuận của ba phân khúc khách hàng |
| 07_top_customers.csv | 20 khách hàng có tổng lợi nhuận cao nhất |
| 08_order_shipping.csv | Hiệu quả theo phương thức vận chuyển và ưu tiên |
| 09_market_category_cube.csv | CUBE thị trường × ngành hàng và các tổng phụ |
| 10_slice_dice.csv | Ví dụ lọc năm 2014, thị trường US/EU, ngành Technology |
| 11_loss_making_products.csv | Các tổ hợp sản phẩm có tổng lợi nhuận âm |
| 12_discount_analysis.csv | Kết quả lịch sử theo khoảng chiết khấu |

Các tỷ lệ và tăng trưởng lưu dưới dạng thập phân (0.1 tương ứng 10%). Ô trống đầu kỳ tăng trưởng là chưa có kỳ so sánh. grouping_level đánh dấu cấp tổng hợp của ROLLUP/CUBE; không cộng đồng thời dòng chi tiết và dòng tổng phụ.

Kiểm tra tích hợp trên kho đã nạp:

    python src/warehouse/check_pipeline.py

Các thử nghiệm dữ liệu sai được thực hiện trong transaction không commit. File đầu vào kiểm tra chỉ tồn tại tạm tại .runtime và được xóa sau khi kiểm tra.

Để dừng đúng instance riêng của dự án:

    python src/warehouse/setup_local_postgres.py --stop

Nguồn PostgreSQL:

- https://www.postgresql.org/download/windows/
- https://www.enterprisedb.com/download-postgresql-binaries
