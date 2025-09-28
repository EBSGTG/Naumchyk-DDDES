# Лабораторна робота №1

### Проектування систем з розподіленими базами даних в енергетиці

**Тема:** Дослідження продуктивності Apache Kafka для енергетичних
IoT-потоків\
**Варіант:** 6 --- Теплові електростанції Східної України\
**Підваріант:** A (оптимізація для real-time критичних алертів)

------------------------------------------------------------------------

## 🎯 Мета роботи

Дослідити продуктивність Apache Kafka для потоків даних з теплових
електростанцій.\
Фокус дослідження:\
- критичні real-time алерти\
- аналіз 50th та 95th percentile latency\
- вплив параметра `linger.ms` (1,2,3,5 ms)\
- моніторинг consumer lag та network jitter

------------------------------------------------------------------------

## ⚙️ Тестове середовище

-   **OS:** Ubuntu 24.04\
-   **Java:** 11\
-   **Kafka:** 3.7.1 (single-broker), Zookeeper 3.8\
-   **Python:** 3.12\
-   **Hardware:** 4 vCores, 4 GB RAM, SSD

### Тестові дані

-   Тип: дані з теплових електростанцій Східної України\
-   Формат: JSON, 256 байт\
-   Кількість: \~1100 згенерованих записів\
-   Ключові параметри: `fuel_flow`, `steam_pressure`, `power_output`,
    `efficiency`, `temperature`, `voltage`, `current`, `status`,
    `fuel_type`

------------------------------------------------------------------------

## 📊 Основні результати

### Batch size та linger.ms

  Конфігурація   Records/sec   Avg Latency (ms)   Використання
  -------------- ------------- ------------------ -----------------
  8KB, 0ms       650           4.5                Ultra real-time
  16KB, 0ms      900           6.0                Real-time
  64KB, 10ms     1500          12                 Balanced
  256KB, 50ms    2200          65                 Batch

🔹 **Оптимум:** `64KB + 5ms → ~1500 rec/sec при latency ~12ms`

### Compression

  ---------------------------------------------------------------------------
  Алгоритм   Records/sec   Latency (ms) Compression Ratio Використання
  ---------- ------------- ------------ ----------------- -------------------
  none       2100          5            0%                real-time алерти

  snappy     1900          8            50%               load regulation

  lz4        1750          12           60%               fuel optimization

  zstd       1500          18           70%               архівація
  ---------------------------------------------------------------------------

🔹 **Баланс:** LZ4 (60% компресія, latency \<15ms)

### Масштабування (партиції)

  Партиції   Records/sec   Scaling Factor   Ефективність
  ---------- ------------- ---------------- --------------------
  4          1000          1.0x             Baseline
  8          1800          1.8x             Good scaling
  12         2500          2.5x             Near-linear growth

🔹 **Оптимум:** 8 партицій для стабільного real-time + архівації

------------------------------------------------------------------------

## 🛠 Рекомендації для використання

-   **Real-time алерти**

        batch.size=16KB, linger.ms=0, compression=none, partitions=4

    Latency \<5ms \| Throughput \>900 rec/sec

-   **Batch-аналітика**

        batch.size=64KB, linger.ms=10, compression=snappy, partitions=8

    Економія \~40% \| Throughput \>1800 rec/sec

-   **Архівування**

        batch.size=128KB, linger.ms=50, compression=zstd, partitions=12

    Економія \~70% \| мінімум дискового простору

------------------------------------------------------------------------

## 📌 Висновки

-   **Latency ↔ Throughput trade-off** підтверджено\
-   **Zstd** дає найкращу компресію (70%), **LZ4** --- оптимальний
    баланс\
-   **Scaling** з партиціями близький до лінійного

➡️ **Рекомендована конфігурація:**\
`64KB + 5ms + LZ4 + 8 партицій → ~1500 rec/sec при latency ~12ms`

Практичне значення:\
- SCADA інтеграція для моніторингу fuel_flow у реальному часі\
- Архівування даних для environmental reporting\
- Балансування навантаження між блоками ТЕС за fuel_type

------------------------------------------------------------------------

## 📂 Додатки

### A. Генератор даних (Python)

``` python
import random, json
N = 12

def generate_powerplant_data():
    return {
        "device_id": f"TESC_{random.randint(1, N):03d}",
        "fuel_flow": round(random.uniform(120.0, 250.0), 2),
        "steam_pressure": round(random.uniform(14.0, 23.0), 2),
        "temperature": round(random.uniform(480.0, 560.0), 1),
        "timestamp": "2025-09-27T12:00:00Z"
    }

for _ in range(3):
    print(json.dumps(generate_powerplant_data(), ensure_ascii=False))
```

### B. Тестові команди Kafka

``` bash
kafka-producer-perf-test.sh   --topic test_powerplant   --num-records 100000   --record-size 256   --throughput -1   --producer-props bootstrap.servers=localhost:9092 batch.size=16384 linger.ms=0 compression.type=none
```

(інші команди для snappy та zstd наведені в звіті)

------------------------------------------------------------------------

✍️ **Автор:** Наумчик Є.Л., гр. ТР-52мп\
👨‍🏫 **Викладач:** Волков О.В.\
📍 КПІ ім. Ігоря Сікорського, 2025
