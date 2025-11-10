## 1. Постановка задачи

- **Цель:** обнаружение аномалий в температуре морозильных камер (два `io_id`).
- **Подход:** методы без учителя (IsolationForest, One-Class SVM, DBSCAN, LOF, PCA,
  PCA-IF, PCA-OCSVM), скользящие окна по минутной сетке, кодирование времени.
- **Оценка:** по размеченному датасету (`annotated_dataset.csv`, поле `anomaly` ∈ {0,1}).
  Считаются ROC-AUC, F1, Accuracy, Precision, Recall; агрегирование: **mean** и **IQR**
  по 5 сдвигам на 100-дневном интервале.

## 2. Архитектура системы

Сервисы и их роли (планируемая архитектура):

- **Collector (FastAPI):** отдача батчи данных из CSV (сырая/аннотированная серия),
  нормализация колонок.
- **Storage (FastAPI + SQLite):** хранение «запусков» (`runs`) и метрик (`metrics`).
- **ML (FastAPI, sklearn):** - обучение и оценка моделей, агрегация метрик.
- **Web Master (API-gateway):** сценарии верхнего уровня, агрегация данных из ML/Storage.
- **Visualization (Dash):** веб-дашборд для просмотра лучшей конфигурации и метрик
  (mean ± IQR) по выбранному `io_id` и модели.

Коммуникации - REST. Конфигурации - YAML. Контейнеризация - Docker compose.

```
services/
  collector/       → /batch
  storage/         → /runs/*, /metrics/*
  ml/              → (train_evaluate, status)  ← в этом архиве main удалён
  webmaster/       → /scenarios/*
  visualization/   → дашборд http://localhost:8014
```

## 3. Данные

В корне проекта создана папка `./data` и лежат:

- `freezer_temperature.csv` — сырая минутная телеметрия (колонки:
  `event_timestamp, io_id, value, value_min, value_max`). `value_min/max` игнорируются.
- `annotated_dataset.csv` — размеченные точки (те же колонки + `anomaly`).

Collector будет нормализовать имена колонок и удалять `value_min/value_max`.

## 4. Методика обучения и оценки (ML)

ЧТо будет содержать main.py:
- **Ресэмплинг:** интерполяция/выравнивание по минутной сетке.
- **Скользящие окна:** 1, 5, 10, 15, 30, 60 минут; признаки окна: mean/std/min/max/median/var/trend/q25/q75.
- **Кодирование времени:** `one_hot` (час/минута), `sin_cos` (час, минута),
  `cyclic` (сезонное кодирование), `sin_cos_cyclic` (комбинация).
- **Сплиты:** `70_30`, `50_50`, `30_70` — в сутках, суммарно 100.
- **Сдвиги:** 5 равномерных сдвигов 100-дневного окна (циклически).
- **Модели:** IF, OCSVM, DBSCAN, LOF, PCA-reconstruction, IF-PCA, OCSVM-PCA.
- **Метрики:** ROC-AUC, F1, Accuracy, Precision, Recall; агрегирование **mean** и **IQR**
  по сдвигам; выбор гиперпараметров — по ROC-AUC (mean).

## 5. Развёртывание

У себя разворачиваю на Docker Desktop (Windows 10).

**Требования:** Docker Desktop (Windows/macOS) или Docker Engine + docker compose (Linux).

```bash
# Корень проекта (рядом с docker-compose.yml)
docker compose up --build
docker compose up -d --build
```

Сервисы по умолчанию - будут реализованы через localhost:

- Collector:     http://localhost:8010/docs  
- Storage:       http://localhost:8011/docs  
- ML:            http://localhost:8012/docs
- Web Master:    http://localhost:8013/docs  
- Visualization: http://localhost:8014

Получается собирать docker-образы, но пока не прикладываю ml/main.py, т. к. пока там ошибки. Одна из важнейших доработок в будущем. Вы у себя пока не сможете запустить этот код, за исключением сбора образов.


## 6. API (планируемый)

### Collector
- `GET /batch?io_id=&start_ts=&end_ts=&annotated=&limit=` - JSON-список

### Storage
- `POST /runs/create` - создает конфигурацию прогонов.
- `POST /metrics/create` - сохраняет агрегированные метрики лучшей гиперпары.
- `GET /runs/latest` - последний run.
- `GET /metrics/by_run/{run_id}` - все метрики по run.

### Web Master (API-gateway)
- `POST /scenarios/train_evaluate` - запуск обучения/оценки.
- `GET /scenarios/latest_metrics` - агрегированные метрики последнего run.

### Visualization
- Веб-дашборд: выбор `io_id` и модели, график метрик (mean ± IQR), подпись лучших
  гиперпараметров и конфигурации (size/window/encoding).

## 7. Логирование и диагностика

Логи пишутся в `./logs/*.log`. Полезные команды:

```bash
docker compose logs -f collector
docker compose logs -f ml
docker compose logs -f storage
docker compose logs -f webmaster
```

Collector логирует запросы `/batch`. Storage — создание runs/metrics.

## 8. Планы развития

1. **Починка ML:** `services/ml/main.py`:
   - починить импорты,
   - стабилизировать пайплайн.
2. **Добавить/улучшить методы:**
   - **DBSCAN** - он сильно отличается от других методов.
   - **ETS (Exponential Smoothing)** - в основном коде для наусной работы еще это не релизовано.
3. **Логи промежуточных шагов обучения:**
   - писать в `ml.log` события по каждому **shift** (индекс сдвига, интервалы train/test,
     размеры `Xtr/Xte`, промежуточные метрики по гиперпарам).
4. **Общий случай:** `total_days` определять по данным (минимальный/максимальный timestamp).
5. **Отчётность:** экспорт отчёта (HTML/PDF) из Visualization с таблицами лучших конфигураций.
