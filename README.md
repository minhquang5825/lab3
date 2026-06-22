# 🔍 AIoT Anomaly Detection System

<div align="center">

![Python](https://img.shields.io/badge/Python-3.11-blue?logo=python)
![FastAPI](https://img.shields.io/badge/FastAPI-0.100+-green?logo=fastapi)
![scikit-learn](https://img.shields.io/badge/scikit--learn-IsolationForest-orange?logo=scikit-learn)
![Jupyter](https://img.shields.io/badge/Jupyter-Notebook-orange?logo=jupyter)
![License](https://img.shields.io/badge/License-MIT-yellow)

**Phát hiện bất thường chuỗi thời gian IoT** sử dụng Isolation Forest & Neural Autoencoder  
Chuyển đổi dữ liệu telemetry thô thành sự kiện có cấu trúc với severity và quyết định tự động  
Triển khai thực tế qua REST API FastAPI

</div>

---

## 📌 Tổng quan

Hầu hết hệ thống giám sát IoT chỉ dùng ngưỡng cứng — dễ bỏ sót lỗi tinh tế hoặc tạo ra quá nhiều cảnh báo sai khiến người vận hành mất niềm tin. Dự án này giải quyết vấn đề đó bằng cách xây dựng một **Event Intelligence Pipeline** hoàn chỉnh trên dữ liệu cảm biến thực tế từ Numenta Anomaly Benchmark (NAB).

> **Câu hỏi Lab 3:** Hệ thống có đang bất thường không?  
> **Đầu ra:** anomaly_score → severity → decision → event log → API

```
Dữ liệu cảm biến sạch
  → Trích xuất đặc trưng (rolling stats, z-score, delta, stuck-flag)
  → Mô hình Isolation Forest / Autoencoder
  → anomaly_score + ngưỡng thích nghi
  → event_type + severity (LOW / MEDIUM / HIGH)
  → Quyết định (NO_ALERT / CREATE_WARNING / HUMAN_CHECK)
  → anomaly_event_log.csv
  → REST API /detect-anomaly
```

**Ứng dụng thực tế:** Giám sát nhiệt độ phòng máy chủ, lớp học thông minh, kho lạnh, hệ thống HVAC — bất kỳ môi trường nào mà việc bỏ sót lỗi cảm biến có thể gây hỏng thiết bị hoặc mất an toàn.

---

## 🏗️ Kiến trúc hệ thống

```
┌──────────────────────────────────────────────────────────────────┐
│                         EVENT PIPELINE                            │
│                                                                    │
│  Cảm biến   →   Trích xuất   →   Mô hình    →   Tầng    →  API  │
│  IoT (RAW)      đặc trưng        bất thường      sự kiện         │
│                 (24 điểm)       IForest /        severity         │
│                                 Autoencoder      decision         │
│                                                                    │
│  OUTPUT:    anomaly_score   →   severity   →   event_log / JSON  │
└──────────────────────────────────────────────────────────────────┘
```

| Tầng | Vai trò | Đầu ra |
|---|---|---|
| **Dữ liệu** | Nạp và kiểm tra chuỗi thời gian | DataFrame sạch |
| **Đặc trưng** | rolling_mean, rolling_std, delta, z-score, stuck_flag | Ma trận đặc trưng |
| **Mô hình** | Isolation Forest (chính) + MLP Autoencoder (demo) | anomaly_score |
| **Sự kiện** | Ngưỡng → severity → decision rule | anomaly_event_log.csv |
| **API** | FastAPI endpoint `/detect-anomaly` | JSON sự kiện có cấu trúc |

---

## 📊 Dataset

| Thông tin | Chi tiết |
|---|---|
| Nguồn | [Numenta Anomaly Benchmark (NAB)](https://github.com/numenta/NAB) |
| File | `ambient_temperature_system_failure.csv` |
| Loại | Chuỗi thời gian IoT thực tế (5 phút/mẫu) |
| Thời gian | 2013-07-04 → 2014-05-28 |
| Tổng số dòng | 7.267 |
| Cột | `timestamp`, `value`, `label` |
| Nhãn bất thường | Lấy từ `combined_windows.json` của NAB |

**Chia train / test theo thứ tự thời gian (không random shuffle):**

| Tập | Khoảng thời gian | Số dòng | Số anomaly |
|---|---|---|---|
| Train | 2013-07-04 → 2014-02-02 | 4.723 | 363 |
| Test | 2014-02-02 → 2014-05-28 | 2.544 | 363 |

> ⚠️ Dữ liệu chuỗi thời gian IoT **bắt buộc phải chia theo thứ tự thời gian** để tránh data leakage — dữ liệu tương lai không được xuất hiện trong tập train.

---

## 🤖 Mô hình & Kết quả thực nghiệm

### Isolation Forest — Mô hình chính

Isolation Forest học ranh giới của hành vi "bình thường" mà **không cần nhãn anomaly** trong lúc train (unsupervised). Mô hình gán `anomaly_score` cho từng điểm; điểm vượt ngưỡng thích nghi sẽ kích hoạt sự kiện.

| Metric | Giá trị | Ý nghĩa trong AIoT |
|---|---|---|
| **Precision** | 0.2401 | Trong 100 cảnh báo → 24 cái đúng thật |
| **Recall** | 0.5675 | Trong 100 anomaly thật → bắt được 57 cái |
| **F1-score** | 0.3374 | Cân bằng giữa Precision và Recall |
| **Ngưỡng** | 0.6091 | Lấy từ phân bố train_normal |
| TP | 206 | Anomaly thật, báo đúng ✅ |
| FP | 652 | Báo nhầm (false alarm) ⚠️ |
| FN | 157 | Bỏ sót anomaly thật ❌ |
| TN | 1.529 | Bình thường, báo đúng ✅ |

**Confusion Matrix:**
```
                    Dự đoán Bình thường   Dự đoán Bất thường
Thực tế Bình thường        1529                  652
Thực tế Bất thường          157                  206
```

> **Nhận xét:** Recall 0.57 tốt cho an toàn hệ thống (bắt được hơn nửa lỗi thật). Precision thấp được xử lý bởi tầng severity — cảnh báo LOW tự động bị lọc, không làm phiền người vận hành.

---

### Neural Autoencoder Demo — Mô hình phụ

MLP Autoencoder học tái tạo lại cửa sổ 24 điểm của hành vi cảm biến bình thường. **MSE tái tạo cao** → cửa sổ đó lệch khỏi pattern bình thường → có khả năng bất thường.

| Metric | Giá trị |
|---|---|
| **F1-score** | 0.1702 |
| **Ngưỡng MSE** | 0.0466 |
| MSE trung bình (test) | 0.0363 |
| MSE cao nhất (test) | 0.2838 |
| Kích thước cửa sổ | 24 điểm |

> F1 Autoencoder thấp hơn là bình thường — đây là mô hình **demo minh họa ý tưởng mạng neural** cho anomaly detection, không thay thế Isolation Forest.

---

## ⚡ Tầng Event Intelligence

`anomaly_score` thô chưa đủ để vận hành thực tế. Pipeline này chuyển điểm số thành **sự kiện có thể hành động**:

```
anomaly_score < threshold    →  severity: LOW    →  NO_ALERT
anomaly_score ≥ threshold    →  severity: MEDIUM →  CREATE_WARNING_EVENT  
anomaly_score ≥ 0.80         →  severity: HIGH   →  CREATE_ALERT_AND_REQUIRE_HUMAN_CHECK
```

**Ví dụ một dòng trong `anomaly_event_log.csv`:**

| timestamp | value | anomaly_score | is_anomaly | event_type | severity | decision |
|---|---|---|---|---|---|---|
| 2014-03-15 10:00 | 81.3 | 0.87 | True | TEMPERATURE_PATTERN_DEVIATION | HIGH | CREATE_ALERT_AND_REQUIRE_HUMAN_CHECK |

---

## 🚀 API — `/detect-anomaly`

Mô hình đã train được triển khai thành REST API sử dụng **FastAPI + Uvicorn**.

### Danh sách endpoint

| Phương thức | Endpoint | Mô tả |
|---|---|---|
| `GET` | `/health` | Kiểm tra trạng thái model |
| `GET` | `/model-info` | Thông tin phiên bản model |
| `POST` | `/detect-anomaly` | Phát hiện bất thường |

---

### Kết quả thực tế — `/health`

> Endpoint `/health` xác nhận model đã được nạp thành công và sẵn sàng nhận request.

![Kết quả /health](https://raw.githubusercontent.com/khanhly-dn/AIoT_Anomaly_Detection_System_Lab3/main/health.png)

```json
{
  "status": "ok",
  "model_loaded": true,
  "model_bundle_path": "models/anomaly_model_bundle_iforest_v2.joblib"
}
```

---

### Kết quả thực tế — Swagger UI `/docs`

> Giao diện Swagger UI hiển thị đầy đủ 3 endpoint: `/health`, `/model-info`, `/detect-anomaly`.

![Swagger UI](https://raw.githubusercontent.com/khanhly-dn/AIoT_Anomaly_Detection_System_Lab3/main/KQ.png)

---

### Phản hồi mẫu từ `/detect-anomaly`

```json
{
  "model_output": {
    "anomaly_score": 0.87,
    "threshold_used": 0.6091,
    "is_anomaly": true,
    "model_version": "iforest_v2"
  },
  "event": {
    "event_type": "TEMPERATURE_PATTERN_DEVIATION",
    "severity": "HIGH",
    "decision": "CREATE_ALERT_AND_REQUIRE_HUMAN_CHECK",
    "explanation": "Nhiệt độ lệch mạnh khỏi pattern gần đây",
    "safety_note": "Không tự động điều khiển thiết bị khi anomaly cao"
  }
}
```

---

## 📁 Cấu trúc project

```
AIoT_Anomaly_Detection_System/
├── data/                          # Dataset NAB hoặc sample fallback
├── notebooks/
│   └── 01_anomaly_detection_event_intelligence.ipynb  ← notebook chính
├── src/
│   ├── download_data.py           # Tải dataset NAB + nhãn label windows
│   ├── train_anomaly.py           # Train/test Isolation Forest + Autoencoder
│   ├── app.py                     # FastAPI service
│   ├── test_api.py                # Test API (cần server đang chạy)
│   ├── test_api_local.py          # Test logic API (không cần mở port)
│   ├── plot_results.py            # Vẽ biểu đồ kết quả
│   └── utils.py                   # Hàm dùng chung
├── models/                        # Model .joblib đã train
├── outputs/                       # Metrics, dự đoán, event log, kết quả API
├── figures/                       # Biểu đồ kết quả
├── diagrams/                      # Sơ đồ minh họa pipeline
├── requirements.txt
└── README.md
```

---

## ⚙️ Cài đặt môi trường

```bash
# 1. Clone repository
git clone https://github.com/khanhly-dn/AIoT_Anomaly_Detection_System_Lab3.git
cd AIoT_Anomaly_Detection_System_Lab3

# 2. Tạo môi trường ảo
python -m venv .venv

# 3. Kích hoạt (Windows)
.venv\Scripts\activate

# 3. Kích hoạt (macOS/Linux)
source .venv/bin/activate

# 4. Cài thư viện
pip install -r requirements.txt
```

---

## ▶️ Chạy toàn bộ pipeline

```bash
# Bước 1 — Tải dataset
python src/download_data.py

# Bước 2 — Train & đánh giá mô hình
python src/train_anomaly.py

# Bước 3 — Vẽ biểu đồ kết quả
python src/plot_results.py

# Bước 4 — Test logic API (không cần server)
python src/test_api_local.py

# Bước 5 — Chạy notebook từng bước
jupyter notebook notebooks/01_anomaly_detection_event_intelligence.ipynb
```

---

## 🌐 Deploy & Test API

```bash
# Terminal 1 — Khởi động server
cd src
uvicorn app:app --reload

# Terminal 2 — Test API
python src/test_api.py
```

Mở Swagger UI: [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)

---

## ✅ Checklist sản phẩm

| Sản phẩm | Đường dẫn | Trạng thái |
|---|---|---|
| Model bundle chính | `models/anomaly_model_bundle_iforest_v2.joblib` | ✅ |
| Isolation Forest model | `models/isolation_forest_iforest_v1.joblib` | ✅ |
| Autoencoder model | `models/mlp_autoencoder_demo.joblib` | ✅ |
| Metrics Isolation Forest | `outputs/iforest_metrics.json` | ✅ |
| Metrics Autoencoder | `outputs/autoencoder_metrics.json` | ✅ |
| Dự đoán tập test | `outputs/iforest_test_predictions.csv` | ✅ |
| Nhật ký sự kiện bất thường | `outputs/anomaly_event_log.csv` | ✅ |
| Kết quả test API | `outputs/api_test_result.json` | ✅ |
| Biểu đồ phát hiện anomaly | `figures/anomaly_detection_result.png` | ✅ |
| Biểu đồ anomaly_score | `figures/anomaly_score_over_time.png` | ✅ |
| Notebook đã chạy đầy đủ | `notebooks/01_anomaly_detection_event_intelligence.ipynb` | ✅ |
| API `/health` → model_loaded: true | Kiểm tra trực tiếp | ✅ |
| API `/detect-anomaly` → JSON đúng cấu trúc | Kiểm tra trực tiếp | ✅ |

---

## 🛠️ Công nghệ sử dụng

| Công nghệ | Phiên bản | Vai trò |
|---|---|---|
| Python | 3.11 | Ngôn ngữ chính |
| scikit-learn | ≥1.3 | Isolation Forest, MLP Autoencoder |
| pandas | ≥2.0 | Xử lý dữ liệu & đặc trưng |
| numpy | ≥1.24 | Tính toán số học |
| FastAPI | ≥0.100 | REST API framework |
| Uvicorn | ≥0.23 | ASGI server |
| matplotlib | ≥3.7 | Trực quan hóa kết quả |
| joblib | ≥1.3 | Lưu/tải mô hình |
| Jupyter Notebook | ≥7.0 | Khám phá dữ liệu tương tác |

---

## 📚 Tài liệu tham khảo

- [Numenta Anomaly Benchmark (NAB)](https://github.com/numenta/NAB)
- [NAB Dataset README](https://github.com/numenta/NAB/blob/master/data/README.md)
- [scikit-learn Isolation Forest](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.IsolationForest.html)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)

---

<div align="center">
  <sub>Lab 3 — Triển khai, phát triển ứng dụng AI và IoT</sub>
</div>
