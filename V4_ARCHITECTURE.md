# WEATHER_ANALYZER V4 - Новая архитектура

Дата: 2026-07-22

## 1. Гарантированная совместимость с V3

### Входы (БЕЗ ИЗМЕНЕНИЙ)
```
W3_EN          : Bool      (включение блока)
W3_LAT         : Float     (широта Open-Meteo)
W3_LON         : Float     (долгота Open-Meteo)
W3_TEMP_C      : Float     (температура BME280)
W3_HUM_PCT     : Float     (влажность BME280)
W3_PRESS_HPA   : Float     (давление BME280)
W3_RAIN_USE    : Bool      (использовать датчик дождя)
W3_RAIN_STATE  : Bool      (состояние датчика дождя)
```

### Параметр (БЕЗ ИЗМЕНЕНИЙ)
```
W3_REQ_MIN     : Int       (интервал запроса Open-Meteo в минутах)
```

### Выходы (БЕЗ ИЗМЕНЕНИЙ)
```
W3_OUT_HOUR    : Int       (циклический час: 1-6)
W3_OUT_PROB    : Int       (вероятность: 0-100)
W3_OM_STATUS   : Bool      (статус Open-Meteo)
W3_HOUR_1..6   : Int       (фиксированные часы)
W3_PROB_1..6   : Int       (фиксированные вероятности)
```

## 2. Новая система памяти (Memory Storage)

### Размер памяти

**Увеличено с 36 до 200 записей:**
- Старое: 36 × 10 мин = 6 часов
- **Новое: 200 × 10 мин = 33 часа (полтора суток)**

### Структура записи

Каждая запись хранит:
```cpp
struct w4_MemoryRecord {
    float temp;           // температура
    float humidity;       // влажность
    float pressure;       // давление
    float dTemp;          // ΔТемпература
    float dHumidity;      // ΔВлажность
    float dPressure;      // ΔДавление
    float dewDiff;        // разница от точки росы
    byte hour;            // час (0-23)
    byte month;           // месяц (1-12)
    byte season;          // сезон (1=зима, 2=весна, 3=лето, 4=осень)
    byte rainConfirm;     // 0=нет дождя, 1=датчик, 2=Open-Meteo, 3=оба
    float prob[6];        // вероятности для часов 1-6
    float weight;         // вес записи (0.0 - 1.0)
    float seasonCoeff[6]; // коэффициент сезона для каждого часа
};
```

### Циклический буфер

```cpp
// После заполнения переходит на начало
static w4_MemoryRecord w4_memory[200];
static byte w4_memPos = 0;    // текущая позиция записи (0-199)
static byte w4_memCount = 0;  // количество заполненных записей (до 200)
```

## 3. Сезонная система обучения

### Коэффициенты сезона

```cpp
struct w4_SeasonalCoefficients {
    float openMeteoWeight;    // вес Open-Meteo (0.0-1.0)
    float localWeight;        // вес локального анализа (0.0-1.0)
    float historyWeight;      // вес истории (0.0-1.0)
    
    float tempFactor;         // влияние температуры
    float pressureFactor;     // влияние давления
    float humidityFactor;     // влияние влажности
    float dewFactor;          // влияние точки росы
    
    float morningAdjust;      // утренняя коррекция
    float dayAdjust;          // дневная коррекция
    float eveningAdjust;      // вечерняя коррекция
    float nightAdjust;        // ночная коррекция
};

static w4_SeasonalCoefficients w4_seasonal[4] = {
    // Зима (Dec, Jan, Feb)
    {0.45, 0.35, 0.20, 1.2, 1.1, 0.9, 1.0, -0.05, 0.0, 0.05, -0.1},
    // Весна (Mar, Apr, May)
    {0.40, 0.40, 0.20, 1.0, 1.3, 1.0, 0.9, -0.1, 0.0, 0.05, 0.0},
    // Лето (Jun, Jul, Aug)
    {0.35, 0.45, 0.20, 0.9, 0.8, 1.2, 1.1, -0.15, 0.05, 0.1, -0.05},
    // Осень (Sep, Oct, Nov)
    {0.40, 0.40, 0.20, 1.0, 1.2, 1.1, 1.0, 0.0, 0.0, 0.1, 0.05}
};
```

### Переход между сезонами

**Плавный переход в течение 2 недель:**

```
31 мая (начало лета)    -> 100% весна, 0% лето
1-15 июня               -> постепенный переход
15 июня                 -> 50% весна, 50% лето
1-15 июля               -> продолжение перехода
1 августа               -> 20% весна, 80% лето
1 сентября (осень)      -> 100% лето, 0% осень
```

**Формула интерполяции:**
```cpp
float seasonMix = (dayOfMonth - 1) / 14.0;  // день 1-15 -> 0.0-1.0
// Использование: weight_old * (1 - seasonMix) + weight_new * seasonMix
```

## 4. Режимы работы

### Режим ONLINE (есть интернет)

```
Входные данные
    ↓
[Open-Meteo Engine] — запрос к API
    ↓
Получены данные + локальные датчики
    ↓
[Local Analysis] — анализ давления/влажности/температуры
    ↓
[Trend Analyzer] — анализ тренда
    ↓
[Dew Point] — расчет точки росы
    ↓
[Similarity Search] — поиск похожих записей в памяти
    ↓
[Seasonal Learning] — применение сезонных коэффициентов
    ↓
[Final Decision] — взвешенное слияние
    ↓
Вероятности P_FINAL[1-6]
    ↓
[Memory Manager] — сохранение в историю
```

### Режим OFFLINE (нет интернета)

```
Входные данные
    ↓
[Local Analysis] — анализ давления/влажности/температуры
    ↓
[Trend Analyzer] — анализ тренда
    ↓
[Dew Point] — расчет точки росы
    ↓
[Similarity Search] — поиск похожих записей в памяти
    ↓
[Seasonal Learning] — применение сезонных коэффициентов
    ↓
[Final Decision] — взвешенное слияние (Open-Meteo=0)
    ↓
Вероятности P_FINAL[1-6]
    ↓
[Memory Manager] — сохранение в историю
```

## 5. Модульная архитектура

```cpp
// ========== МОДУЛЬ 1: LOCAL FORECAST ENGINE ==========
float w4_localForecast(float temp, float humidity, float pressure,
                       float dTemp, float dHumidity, float dPressure,
                       float dewDiff, byte hour, byte month);

// ========== МОДУЛЬ 2: ONLINE FORECAST ENGINE ==========
byte w4_onlineRequest(float lat, float lon, byte* status);
void w4_parseOpenMeteo(const String& payload);

// ========== МОДУЛЬ 3: TREND ANALYZER ==========
float w4_analyzeTrend(float current, float previous, 
                      float delta, byte trendType);

// ========== МОДУЛЬ 4: DEW POINT ==========
float w4_calculateDewPoint(float temp, float humidity);
float w4_getDewDifference(float temp, float humidity);

// ========== МОДУЛЬ 5: RAIN LEARNING ==========
void w4_learnRainPattern(byte rainConfirm, float localAnalysis);

// ========== МОДУЛЬ 6: SEASONAL LEARNING ==========
float w4_getSeasonCoefficient(byte month, byte hour);
void w4_updateSeasonalCoefficients(byte month);

// ========== МОДУЛЬ 7: SIMILARITY SEARCH ==========
float w4_findSimilarRecords(float temp, float humidity, float pressure,
                            float dewDiff, byte hour, byte month);

// ========== МОДУЛЬ 8: MEMORY MANAGER ==========
void w4_saveRecord(float temp, float humidity, float pressure,
                   float dTemp, float dHumidity, float dPressure,
                   float dewDiff, byte hour, byte month,
                   byte rainConfirm, const float* probabilities);

// ========== МОДУЛЬ 9: FINAL DECISION ==========
byte w4_calculateFinalProbability(float openMeteo, float local,
                                  float history, float seasonAdj,
                                  float timeAdj);
```

## 6. Новые внутренние переменные (БЕЗ новых входов/выходов)

```cpp
// Режим работы
static byte w4_mode = 0;  // 0=OFFLINE, 1=ONLINE

// Последний запрос
static unsigned long w4_lastRequestMs = 0;
static unsigned long w4_lastSampleMs = 0;

// Данные Open-Meteo (при наличии интернета)
static float w4_omTemp[6];
static float w4_omHumidity[6];
static float w4_omPressure[6];
static float w4_omPrecipProb[6];

// Локальный анализ
static float w4_localProb[6];
static float w4_trendProb[6];
static float w4_historyProb[6];

// Память
static w4_MemoryRecord w4_memory[200];
static byte w4_memPos = 0;
static byte w4_memCount = 0;

// Сезонные коэффициенты (адаптируются со временем)
static w4_SeasonalCoefficients w4_seasonal[4];

// Последние значения для дельт
static float w4_prevTemp = 0;
static float w4_prevHumidity = 0;
static float w4_prevPressure = 0;
```

## 7. Ключевые алгоритмы

### A. Local Forecast Engine

**Анализирует давление, влажность, температуру:**

```
Давление:
  ΔP <= -1.5 hPa  → +50% вероятности
  -1.5 > ΔP > -1  → +30% вероятности
  -1 > ΔP > -0.5  → +15% вероятности
  ΔP >= +0.8      → -20% вероятности
  ΔP >= +1.5      → -40% вероятности

Влажность:
  H >= 90% и растет → +30%
  H >= 80% и растет → +20%
  H >= 70% и растет → +10%
  H < 50%           → -15%
  H падает          → -10%

Температура:
  T падает на 2-5°C → +10%
  T падает на 5°C+  → +20%
  T растет, H падает → -15%
```

### B. Similarity Search

**Поиск похожих исторических условий:**

```cpp
// Вычисляется расстояние до каждой записи в памяти:
distance = sqrt(
    (temp - record.temp)² +
    (humidity - record.humidity)² +
    (dewDiff - record.dewDiff)² +
    (hour - record.hour)² / 576  // нормализация часов
);

// Записи с distance < 0.5 считаются похожими
// Средняя вероятность похожих записей становится historyProb
```

### C. Seasonal Learning

**Автоматическая адаптация коэффициентов:**

```cpp
// В начале сезона (1-го числа месяца)
if (newSeason) {
    w4_seasonal[season].openMeteoWeight = 0.40;
    w4_seasonal[season].localWeight = 0.40;
    w4_seasonal[season].historyWeight = 0.20;
}

// После каждой записи в историю
if (rainConfirm) {
    // Если был дождь, а Open-Meteo предсказал неправильно
    if (omPrecip < 30 && rainConfirm > 0) {
        w4_seasonal[season].localWeight += 0.01;  // +1% к локальному
        w4_seasonal[season].openMeteoWeight -= 0.01;
    }
}

// Нормализация
float sum = openMeteoWeight + localWeight + historyWeight;
openMeteoWeight /= sum;
localWeight /= sum;
historyWeight /= sum;
```

## 8. Переход между режимами

```cpp
// Проверка каждый цикл
if (WiFi.status() == WL_CONNECTED && 
    nowMs - w4_lastRequestMs >= reqIntervalMs) {
    
    w4_mode = 1;  // ONLINE
    w4_onlineRequest(...);
} else if (WiFi.status() != WL_CONNECTED) {
    
    w4_mode = 0;  // OFFLINE
    // Используем только локальный анализ + историю
}
```

## 9. Вывод данных (НЕИЗМЕНЕН)

**Пользователь видит ровно то же:**
- W3_OUT_HOUR (1-6)
- W3_OUT_PROB (0-100)
- W3_OM_STATUS (true/false при интернете)
- W3_HOUR_1..6 (всегда 1-6)
- W3_PROB_1..6 (вероятности для каждого часа)

**Качество прогноза улучшится за счет:**
- Полтора суток истории вместо 6 часов
- Сезонной адаптации коэффициентов
- Автоматического режима OFFLINE
- Более точного поиска похожих условий

## 10. Примерный размер памяти

```
w4_MemoryRecord = ~60 байт
w4_memory[200]  = ~12 KB

w4_SeasonalCoefficients = ~60 байт
w4_seasonal[4]  = ~240 байт

Итого: ~12.5 KB (легко помещается на ESP32)
```

## 11. Обновление документации

Раздел V4 в README.md добавится:
- Описание новой архитектуры
- Объяснение режимов ONLINE/OFFLINE
- Описание сезонного обучения
- Примеры использования

---

**Важно:** Все входы, выходы, параметры и WebUI **ПОЛНОСТЬЮ СОВМЕСТИМЫ** с V3.
Меняется только внутренний алгоритм.
