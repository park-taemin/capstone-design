# 🌤️ 기상·발전량 수집 및 LSTM 예측 시스템

> **공공 기상 API + IoT 센서 데이터를 MySQL에 수집하고, 머신러닝으로 발전량을 예측하는 end-to-end 파이프라인**

---

## 📌 프로젝트 개요

| 항목 | 내용 |
|------|------|
| **기간** | 2026.06 |
| **목적** | 기상 데이터와 IoT 발전량 데이터를 연계하여 LSTM 기반 예측 모델 구축 |
| **데이터** | Open-Meteo 기상 API (30일 × 24시간), Arduino Nano RP2040 Modbus RTU |
| **결과** | Baseline MAE 0.1264 kW / R² 0.9873, LSTM MAE 0.2405 kW |

---

## 🏗️ 시스템 아키텍처

```
Open-Meteo API          →  weather_hourly   (시간별 기상, 720행)
Arduino RP2040 Modbus   →  power_realtime   →  power_hourly (뷰, 720행)
                                ↓
                          JOIN (696행)
                                ↓
              train_baseline.py   →  HistGradientBoosting (t → t+1)
              lstm_train.py       →  LSTM (과거 35시간 × 8 feature → power_kw)
```

---

## 🛠️ 기술 스택

### Infrastructure
- **Docker** — MySQL 8.0 컨테이너
- **MySQL** — 기상·발전량 데이터 저장, VIEW로 시간별 집계

### IoT / Hardware
- **Arduino Nano RP2040 Connect** — Modbus RTU 슬레이브 펌웨어
- **pymodbus 3.x** — Python Modbus RTU 마스터 (COM 시리얼 통신)

### Backend / Data Pipeline
- **Python 3.12** + **uv** 패키지 관리
- **requests** — Open-Meteo Historical Weather API 수집
- **pymysql** — MySQL 적재 (ON DUPLICATE KEY UPDATE)
- **python-dotenv** — 환경변수 관리

### Machine Learning
- **pandas / scikit-learn** — 데이터 전처리, 베이스라인 모델
- **TensorFlow 2.21 / Keras** — LSTM 시계열 예측
- **HistGradientBoostingRegressor** — 베이스라인 (결측치 내성)

---

## 📂 파일 구조

```
weather-lab/
├── .env                          # 환경변수 (DB, API 키, Modbus 설정)
├── db.py                         # MySQL 연결 및 공통 저장 함수
├── weather_openmeteo.py          # Open-Meteo API 수집 모듈
├── collect_weather.py            # 기상 API JSON 1일 확인
├── collect_weather_backfill.py   # 30일 기상 데이터 backfill
├── collect_rp2040_modbus.py      # Modbus RTU 실시간 수집
├── ml_shared.py                  # ML 공통 유틸 (데이터 로드, 전처리, 시퀀스 생성)
├── train_baseline.py             # 베이스라인 모델 학습
├── lstm_train.py                 # LSTM 모델 학습 및 비교
├── seed/
│   └── power_realtime_seed.sql   # 30일치 더미 발전량 데이터
└── metrics_baseline.json         # 베이스라인 성능 지표
```

---

## 🗄️ 데이터베이스 스키마

### `weather_hourly` — 시간별 기상 관측
| 컬럼 | 타입 | 설명 |
|------|------|------|
| obs_time | DATETIME | 관측 시각 |
| source | VARCHAR | openmeteo / public |
| temperature | DECIMAL | 기온 (℃) |
| humidity | DECIMAL | 습도 (%) |
| wind_speed | DECIMAL | 풍속 (m/s) |
| solar_radiation | DECIMAL | 일사 (W/m²) |
| precipitation | DECIMAL | 강수 (mm) |

### `power_realtime` — Modbus 실시간 발전량
| 컬럼 | 타입 | 설명 |
|------|------|------|
| measured_at | DATETIME | 측정 시각 |
| device_id | VARCHAR | 장치 ID |
| power_kw | DECIMAL | 발전량 (kW) |
| temperature | DECIMAL | 패널 온도 |
| humidity | DECIMAL | 패널 습도 |

### `power_hourly` — 시간별 집계 VIEW
```sql
SELECT DATE_FORMAT(measured_at, '%Y-%m-%d %H:00:00') AS hour_time,
       device_id,
       AVG(power_kw), AVG(temperature), AVG(humidity), COUNT(*)
FROM power_realtime
GROUP BY hour_time, device_id;
```

---

## 🤖 머신러닝 모델

### Feature 구성 (8개)
| # | Feature | 출처 |
|---|---------|------|
| 1 | temperature | weather_hourly |
| 2 | humidity | weather_hourly |
| 3 | wind_speed | weather_hourly |
| 4 | solar_radiation | weather_hourly |
| 5 | precipitation | weather_hourly |
| 6 | power_kw | power_hourly |
| 7 | panel_temp | power_hourly |
| 8 | panel_humidity | power_hourly |

### 모델 비교

| 모델 | 입력 | MAE (kW) | RMSE (kW) | R² |
|------|------|----------|-----------|-----|
| **Baseline** (HistGradientBoosting) | 시각 t의 8 feature → t+1 예측 | **0.1264** | 0.1893 | **0.9873** |
| **LSTM** | 과거 35시간 × 8 feature → 다음 예측 | 0.2405 | 0.3112 | — |

> 더미 시드 데이터 특성상 베이스라인이 우세. 실제 Modbus 데이터 누적 시 LSTM 성능 향상 기대.

---

## ⚙️ 실행 방법

### 1. 환경 설정
```powershell
# Docker MySQL 실행
docker run -d --name weather-mysql \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=weather \
  -e MYSQL_USER=weather \
  -e MYSQL_PASSWORD=weatherpass \
  -p 3306:3306 mysql:8.0

# Python 환경
uv venv --python 3.12
uv sync
uv add requests pymysql python-dotenv pymodbus tensorflow scikit-learn pandas
```

### 2. 기상 데이터 수집 (30일 backfill)
```powershell
uv run --python 3.12 python collect_weather_backfill.py 30
```

### 3. Modbus 실시간 수집
```powershell
uv run --python 3.12 python collect_rp2040_modbus.py
```

### 4. 모델 학습
```powershell
uv run --python 3.12 python train_baseline.py
uv run --python 3.12 python lstm_train.py
```

---

## 📊 주요 결과

```
[BASELINE] HistGradientBoosting
  test MAE  (kW): 0.1264
  test RMSE (kW): 0.1893
  test R²:        0.9873

[LSTM] 과거 35시간 × 8 feature → power_kw
  test MAE  (kW): 0.2405
  test RMSE (kW): 0.3112

[COMPARE] Baseline vs LSTM MAE (kW)
  Baseline: 0.1264
  LSTM:     0.2405
```

---

## 🔧 환경변수 `.env` 설정

```env
WEATHER_SOURCE=openmeteo
LAT=35.95
LON=126.70
LOCATION_KEY=gunsan

MYSQL_HOST=127.0.0.1
MYSQL_PORT=3306
MYSQL_USER=weather
MYSQL_PASSWORD=weatherpass
MYSQL_DATABASE=weather

MODBUS_MODE=rtu
MODBUS_PORT=COM4
MODBUS_BAUD=9600
MODBUS_SLAVE_ID=1
DEVICE_ID=RP2040-EMU-01
```

---

## 📝 개발 환경

| 항목 | 버전 |
|------|------|
| OS | Windows 10 |
| Python | 3.12.13 |
| MySQL | 8.0 (Docker) |
| TensorFlow | 2.21.0 |
| pymodbus | 3.13.0 |
| uv | 최신 |

---

## 💡 향후 개선 방향

- [ ] 실제 태양광 패널 센서 연동으로 실데이터 수집
- [ ] Modbus 데이터 장기 누적 후 LSTM 재학습
- [ ] 공공데이터포털 ASOS API 연동 (일사량 실측값 활용)
- [ ] 예측 결과 대시보드 시각화 (Grafana 등)
- [ ] Docker Compose로 전체 스택 통합

