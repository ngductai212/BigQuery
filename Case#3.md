# Urban Mobility Clustering – Full Case Study

## 🎯 MỤC TIÊU CASE STUDY
Phân tích dữ liệu di chuyển của taxi ở New York để:

- Phát hiện các **khu vực có nhu cầu di chuyển cao** (mobility hotspots).
- **Phân cụm** các vùng di chuyển tương tự nhau → hỗ trợ **quy hoạch** vị trí bãi đỗ, trạm sạc, điểm đón/trả taxi.

---

# 🧱 1️⃣ TẬP DỮ LIỆU CẦN CHUẨN BỊ

| Dataset | Nguồn | Vai trò | Ghi chú |
|--------|--------|---------|---------|
| 🟡 NYC TLC Taxi Trips (2021) | `bigquery-public-data.new_york_taxi_trips.tlc_yellow_trips_2021` | Dữ liệu gốc: các chuyến taxi | Có pickup/dropoff LocationID, timestamp, fare, passenger_count |
| 🗺️ Taxi Zone Lookup / Geometry | `bigquery-public-data.new_york_taxi_trips.taxi_zone_geom` | Bản đồ vùng (polygon) | Map LocationID ↔ tên vùng ↔ toạ độ |
| 🌦️ Weather (NOAA GSOD 2021) | `bigquery-public-data.noaa_gsod.gsod2021` | Điều kiện thời tiết ảnh hưởng nhu cầu | Lấy trạm JFK, LGA; đổi F→°C, in→mm |
| 📅 Event Calendar | dbt seed CSV | Danh sách ngày lễ, sự kiện | Giúp phát hiện nhu cầu bất thường |
| 📐 H3 Spatial Index | Thư viện H3 | Chia ô không gian | Chia map thành lưới hexagon |

---

# 🧰 2️⃣ CHUẨN BỊ MÔI TRƯỜNG

| Thành phần | Mục đích |
|------------|----------|
| Google BigQuery | Lưu & xử lý dữ liệu taxi, weather |
| dbt (Data Build Tool) | Làm sạch, transform, join dữ liệu |
| BigQuery GIS / H3 Functions | Phân tích không gian |
| BigQuery ML (K-Means) | Phân cụm vùng nhu cầu |
| Looker Studio / Deck.gl / Streamlit | Trực quan hoá bản đồ |
| Python | Pipeline & Feature Engineering |

---

# ⚙️ 3️⃣ KIẾN TRÚC PIPELINE (Luồng xử lý dữ liệu)

```
          +---------------------+
          |  Raw Data Sources    |
          |----------------------|
          |  TLC Taxi Trips      |
          |  Weather (NOAA GSOD) |
          |  Event Calendar (CSV)|
          +----------+-----------+
                     |
                     ▼
         +-----------+------------+
         |  BigQuery (Raw layer)  |
         |  Dữ liệu gốc năm 2021  |
         +-----------+------------+
                     |
                     ▼
         +-----------+------------+
         |  dbt Transform Layer   |
         |------------------------|
         | stg_taxi_trips.sql     |
         | stg_weather.sql        |
         | dim_datetime.sql       |
         | fct_hourly_demand.sql  |
         +-----------+------------+
                     |
                     ▼
         +-----------+------------+
         | Modeling & ML Layer    |
         |------------------------|
         | Feature: pickup_H3_id  |
         | Feature: demand_score  |
         | KMeans clustering (BQ) |
         +-----------+------------+
                     |
                     ▼
         +-----------+------------+
         | Visualization / API    |
         |------------------------|
         | Looker Studio Dashboard|
         | Streamlit map (H3)     |
         +------------------------+
```

---

# 🧮 4️⃣ CHI TIẾT CÁC BƯỚC TRIỂN KHAI

## 🔹 **Bước 1: Làm sạch dữ liệu taxi**

```sql
CREATE OR REPLACE TABLE myproject.raw.taxi_trips_2021_clean AS
SELECT
  SAFE_CAST(vendor_id AS INT64) AS vendor_id,
  SAFE_CAST(rate_code AS INT64) AS rate_code,
  pickup_datetime,
  dropoff_datetime,
  SAFE_CAST(passenger_count AS INT64) AS passenger_count,
  SAFE_CAST(trip_distance AS FLOAT64) AS trip_distance,
  SAFE_CAST(pickup_location_id AS INT64) AS pickup_location_id,
  SAFE_CAST(dropoff_location_id AS INT64) AS dropoff_location_id,
  total_amount
FROM `bigquery-public-data.new_york_taxi_trips.tlc_yellow_trips_2021`
WHERE trip_distance > 0 AND passenger_count > 0;
```

---

## 🔹 **Bước 2: Tính nhu cầu theo giờ và vị trí**

```sql
CREATE OR REPLACE TABLE myproject.analytics.hourly_demand AS
SELECT
  pickup_location_id,
  TIMESTAMP_TRUNC(pickup_datetime, HOUR) AS hour_ts,
  COUNT(*) AS trip_count
FROM myproject.raw.taxi_trips_2021_clean
GROUP BY 1, 2;
```

---

## 🔹 **Bước 3: Gắn thông tin không gian (H3)**

```sql
CREATE OR REPLACE TABLE myproject.analytics.hourly_demand_h3 AS
SELECT
  H3_FROMGEOG(z.zone_geom, 8) AS h3_id,
  hour_ts,
  SUM(trip_count) AS demand
FROM myproject.analytics.hourly_demand d
JOIN `bigquery-public-data.new_york_taxi_trips.taxi_zone_geom` z
ON d.pickup_location_id = z.zone_id
GROUP BY 1, 2;
```

---

## 🔹 **Bước 4: Join thời tiết và sự kiện**

```sql
CREATE OR REPLACE TABLE myproject.analytics.hourly_features AS
SELECT
  d.h3_id,
  d.hour_ts,
  d.demand,
  w.temp_celsius,
  w.rain_mm,
  e.is_event_day
FROM myproject.analytics.hourly_demand_h3 d
LEFT JOIN myproject.analytics.weather_daily w
  ON DATE(d.hour_ts) = w.date
LEFT JOIN myproject.seeds.events_calendar e
  ON DATE(d.hour_ts) = e.date;
```

---

## 🔹 **Bước 5: Phân cụm vùng có nhu cầu cao (K-Means)**

```sql
CREATE OR REPLACE MODEL myproject.ml.zone_clusters
OPTIONS(
  model_type = 'kmeans',
  num_clusters = 6
) AS
SELECT
  h3_id,
  AVG(demand) AS avg_demand,
  AVG(temp_celsius) AS avg_temp,
  AVG(rain_mm) AS avg_rain
FROM myproject.analytics.hourly_features
GROUP BY h3_id;
```

### Sau đó gán nhãn cụm:
```sql
SELECT
  h3_id,
  centroid_id AS cluster,
  avg_demand
FROM ML.PREDICT(
  MODEL myproject.ml.zone_clusters,
  (SELECT DISTINCT h3_id, avg_demand, avg_temp, avg_rain 
   FROM myproject.analytics.hourly_features)
);
```

---

# 🌐 5️⃣ DASHBOARD / TRỰC QUAN HÓA

## 🎨 **Dashboard đề xuất**

- **Layer 1:** Bản đồ H3 (màu theo `avg_demand`)
- **Layer 2:** Màu cụm (`cluster_id`)  
- **Layer 3:** Bộ lọc thời gian / ngày / sự kiện

## 🛠️ Công cụ:
- **Looker Studio** (dữ liệu BigQuery)
- **Deck.gl / Kepler.gl** (map tương tác)
- **Streamlit** (interactive dashboard)

## 📊 Insight gợi ý:
- Cụm 1: Midtown / Downtown → nhu cầu cao cả ngày  
- Cụm 2: Suburb → nhu cầu cao buổi sáng/chiều  
- Cụm 3: Gần sân bay → phụ thuộc giờ bay + thời tiết  

---

# 🔍 6️⃣ CHỈ SỐ & KẾT QUẢ MONG ĐỢI

| Mục tiêu | Chỉ số |
|----------|--------|
| Xác định hotspot | So sánh với top 10 zone lịch sử (Manhattan, JFK, LGA) |
| Ổn định cụm | Silhouette score |
| Hiểu ảnh hưởng weather/event | Correlation rain/temp/event với demand |

---

# 🧠 7️⃣ MỞ RỘNG (ADVANCED)

| Hướng mở rộng | Mô tả |
|----------------|------|
| Realtime simulation | Stream taxi → Pub/Sub → BigQuery |
| Traffic correlation | Kết hợp dữ liệu Google Mobility |
| Carbon estimation | Tính phát thải CO₂ theo quãng đường |
| Smart charging planning | Quy hoạch trạm sạc dựa trên cluster EV |

---

# ✅ TÓM TẮT DỰ ÁN “URBAN MOBILITY CLUSTERING”

| Thành phần | Nội dung |
|------------|----------|
| **Mục tiêu** | Phân tích hành vi di chuyển → quy hoạch trạm taxi/sạc |
| **Nguồn dữ liệu** | Taxi trips, weather, events |
| **Công cụ** | BigQuery, dbt, BigQuery ML, Looker Studio |
| **Kỹ thuật** | Spatial join (H3), Aggregation, KMeans |
| **Kết quả** | Cụm vùng nhu cầu + bản đồ heatmap |
| **Ứng dụng** | Smart City, Mobility Analytics, Transportation Planning |

