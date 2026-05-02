# SD-WAN Lab: Документация по системе мониторинга и автоматического переключения каналов

## Содержание

1. [Обзор проекта](#обзор-проекта)
2. [Архитектура сети](#архитектура-сети)
3. [Технологии и инструменты](#технологии-и-инструменты)
4. [Настройка сетевых интерфейсов](#настройка-сетевых-интерфейсов)
5. [Имитация каналов связи (tc)](#имитация-каналов-связи-tc)
6. [Маршрутизация](#маршрутизация)
7. [Скрипт sdwan_monitor.sh](#скрипт-sdwan_monitorsh)
8. [Алгоритм работы](#алгоритм-работы)
9. [Тестирование и валидация](#тестирование-и-валидация)
10. [Устранение неполадок (ошибки, с которыми мы столкнулись и как их исправить)](#устранение-неполадок)

---

## Обзор проекта

### Цель проекта
Создание программно-определяемой сети (SD-WAN) с двумя каналами связи:
- **Основной канал (Fiber/Optics)**: Низкая задержка, высокая надежность
- **Резервный канал (Satellite)**: Высокая задержка, возможные потери пакетов

### Функциональность
- ✅ Непрерывный мониторинг качества каналов (ping)
- ✅ Автоматический расчет метрик качества
- ✅ Динамическое переключение трафика на лучший канал
- ✅ Защита от "мигания" (hysteresis)
- ✅ Логирование всех событий

---

## Архитектура сети

### Топология

```
                    VMware NAT Network (192.168.229.0/24)
    ┌─────────────────────────────────────────────────────────────┐
    │                                                             │
    │  debian1                         debian2                    │
    │  ┌─────────────┐               ┌─────────────┐             │
    │  │ 192.168.229 │               │ 192.168.229 │             │
    │  │    .146     │               │    .145     │             │
    │  │             │               │             │             │
    │  │ ens33: .146 │◄──────┬──────►│ ens33: .145 │             │
    │  │ ens37: .147 │──────┼───────│ ens37: .148 │             │
    │  └─────────────┘      │       └─────────────┘             │
    │                       │                                    │
    │         ┌─────────────┴─────────────┐                     │
    │         │                           │                     │
    │    🟢 Fiber (5ms)            🔴 Satellite (200ms)        │
    │         │                           │                     │
    │         └───────────────────────────┘                     │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘
```

<img width="783" height="641" alt="image" src="https://github.com/user-attachments/assets/57b622c3-50a0-4d51-886f-c4387cdb8133" />


### IP-адресация

| Хост | Интерфейс | IP-адрес | Назначение |
|------|-----------|----------|------------|
| **debian1** | ens33 | 192.168.229.146/24 | Оптический канал (eth0) |
| **debian1** | ens37 | 192.168.229.147/24 | Спутниковый канал (eth1) |
| **debian2** | ens33 | 192.168.229.145/24 | Оптический канал (eth0) |
| **debian2** | ens37 | 192.168.229.148/24 | Спутниковый канал (eth1) |

---

## Технологии и инструменты

### Используемые компоненты Linux

1. **tc (Traffic Control)** - утилита управления трафиком ядра Linux
   - Эмуляция задержек сети (netem)
   - Имитация потерь пакетов
   - Добавление джиттера

2. **ip route** - управление таблицей маршрутизации
   - Добавление статических маршрутов
   - Изменение метрик маршрутов
   - Приоритизация каналов

3. **ping** - утилита проверки доступности
   - Измерение RTT (Round Trip Time)
   - Подсчет потерь пакетов
   - Мониторинг качества каналов

4. **Bash scripting** - автоматизация
   - Циклический мониторинг
   - Парсинг вывода команд
   - Логика принятия решений
5. **Эмодзи** - для визуализации и понятного представления информации (https://emojicopy.org/ru/#symbols)
6. **Диаграмма** - для визуализации сети (https://app.diagrams.net/)

---

## Настройка сетевых интерфейсов

### Проверка текущей конфигурации

```bash
ip -br a
```

**Вывод на debian1:**
```
lo               UNKNOWN        127.0.0.1/8 ::1/128
ens33            UP             192.168.229.146/24 fe80::20c:29ff:fed0:e439/64
ens37            UP             192.168.229.147/24 fe80::250:56ff:fe35:178f/64
```

### Настройка IP-адресов (если не настроены DHCP)

```bash
# На debian1
sudo ip addr add 192.168.229.146/24 dev ens33
sudo ip addr add 192.168.229.147/24 dev ens37
sudo ip link set ens33 up
sudo ip link set ens37 up

# На debian2
sudo ip addr add 192.168.229.145/24 dev ens33
sudo ip addr add 192.168.229.148/24 dev ens37
sudo ip link set ens33 up
sudo ip link set ens37 up
```

---

## Имитация каналов связи (tc)

### Что такое tc netem?

**tc (Traffic Control)** - подсистема ядра Linux для управления сетевым трафиком.  
**netem (Network Emulator)** - модуль для эмуляции характеристик WAN-сетей.

### Настройка эмуляции

#### На debian1 и debian2:

```bash
# Очистка существующих правил (если есть)
sudo tc qdisc del dev ens33 root 2>/dev/null
sudo tc qdisc del dev ens37 root 2>/dev/null

# ens33 - "Оптический канал" (быстрый, надежный)
sudo tc qdisc add dev ens33 root netem delay 5ms 2ms

# ens37 - "Спутниковый канал" (медленный, с потерями)
sudo tc qdisc add dev ens37 root netem delay 200ms 50ms loss 2%
```

### Расшифровка параметров tc

| Параметр | ens33 (Fiber) | ens37 (Satellite) | Описание |
|----------|---------------|-------------------|----------|
| **delay** | 5ms | 200ms | Базовая задержка |
| **jitter** | 2ms | 50ms | Случайное отклонение задержки |
| **loss** | 0% | 2% | Вероятность потери пакета |

### Примеры изменения параметров

**Имитация деградации канала:**
```bash
# Ухудшаем оптический канал (для тестирования failover)
sudo tc qdisc change dev ens33 root netem delay 500ms 100ms loss 15%

# Возвращаем в нормальное состояние
sudo tc qdisc change dev ens33 root netem delay 5ms 2ms
```

**Проверка текущих настроек tc:**
```bash
tc qdisc show dev ens33
tc qdisc show dev ens37
```

**Пример вывода:**
```
qdisc netem 800d: root refcnt 2 limit 1000 delay 5ms 2ms
qdisc netem 800e: root refcnt 2 limit 1000 delay 200ms 50ms loss 2%
```

---

## Маршрутизация

### Концепция метрик маршрутов

В Linux каждый маршрут имеет **метрику (metric)** - числовой приоритет:
- **Меньшее значение** = более предпочтительный маршрут
- **Большее значение** = резервный маршрут

### Настройка маршрутизации

#### На debian1:

```bash
# Удаляем старые маршруты (если есть)
sudo ip route del 192.168.229.145 2>/dev/null || true
sudo ip route del 192.168.229.148 2>/dev/null || true

# Добавляем маршруты с метриками
sudo ip route add 192.168.229.145 dev ens33 metric 100  # Основной
sudo ip route add 192.168.229.148 dev ens37 metric 200  # Резервный
```

#### На debian2:

```bash
# Удаляем старые маршруты
sudo ip route del 192.168.229.146 2>/dev/null || true
sudo ip route del 192.168.229.147 2>/dev/null || true

# Добавляем маршруты
sudo ip route add 192.168.229.146 dev ens33 metric 100  # Основной
sudo ip route add 192.168.229.147 dev ens37 metric 200  # Резервный
```

### Проверка маршрутов

```bash
# Показать все маршруты
ip route show

# Проверить маршрут к конкретному IP
ip route get 192.168.229.145

# Трассировка
mtr -n 192.168.229.145
```

**Пример вывода `ip route show`:**
```
192.168.229.0/24 dev ens33 proto kernel scope link src 192.168.229.146 metric 100
192.168.229.0/24 dev ens37 proto kernel scope link src 192.168.229.147 metric 200
```

### Динамическое изменение маршрутов

Скрипт использует команду `ip route replace` для изменения метрик:

```bash
# Переключение на резервный канал (ens37)
sudo ip route replace 192.168.229.145 dev ens33 metric 200
sudo ip route replace 192.168.229.148 dev ens37 metric 100

# Возврат на основной канал (ens33)
sudo ip route replace 192.168.229.145 dev ens33 metric 100
sudo ip route replace 192.168.229.148 dev ens37 metric 200
```

---

## Скрипт sdwan_monitor.sh

### Полный код скрипта

```bash
#!/bin/bash
set -uo pipefail

# === НАСТРОЙКИ ===
PEER_IP1="192.168.229.145"  # debian2 через ens33 (оптика)
PEER_IP2="192.168.229.148"  # debian2 через ens37 (спутник)
DEV1="ens33"
DEV2="ens37"
HYSTERESIS=30
INTERVAL=5
LOG_FILE="/var/log/sdwan_monitor.log"
ACTIVE=1

log() { echo "$(date '+%Y-%m-%d %H:%M:%S') $1" | tee -a "$LOG_FILE"; }

get_metrics() {
    local ip=$1
    local out
    
    out=$(ping -c 3 -W 3 "$ip" 2>&1)
    local ping_exit=$?
    
    # Если ping вообще не прошел
    if [ $ping_exit -ne 0 ] || ! echo "$out" | grep -q "packet loss"; then
        echo "100 9999"
        return
    fi
    
    # Парсим packet loss (дробное число)
    local loss=$(echo "$out" | awk -F', ' '/packet loss/ {print $3}' | awk '{print $1}' | tr -d '%' | cut -d. -f1)
    
    # Парсим avg rtt (второе число после /)
    local rtt=$(echo "$out" | awk -F'/' '/rtt/ {print int($5)}')
    
    # Если не удалось распарсить
    loss=${loss:-100}
    rtt=${rtt:-9999}
    
    echo "$loss $rtt"
}

switch_routes() {
    local new_active=$1
    log "🔄 Попытка переключения на канал $new_active..."
    
    if [ "$new_active" = "1" ]; then
        if ip route replace "$PEER_IP1" dev "$DEV1" metric 100 2>&1 && \
           ip route replace "$PEER_IP2" dev "$DEV2" metric 200 2>&1; then
            log "✅ Переключение на канал 1 (ens33/Оптика) УСПЕШНО"
        else
            log "❌ Ошибка переключения на канал 1"
        fi
    else
        if ip route replace "$PEER_IP1" dev "$DEV1" metric 200 2>&1 && \
           ip route replace "$PEER_IP2" dev "$DEV2" metric 100 2>&1; then
            log "✅ Переключение на канал 2 (ens37/Спутник) УСПЕШНО"
        else
            log "❌ Ошибка переключения на канал 2"
        fi
    fi
}

calc_score() {
    local loss=$1 rtt=$2
    echo $(( loss * 1000 + rtt ))
}

log "SD-WAN запущен. Активный: $ACTIVE (ens33)"

while true; do
    read loss1 rtt1 < <(get_metrics "$PEER_IP1")
    read loss2 rtt2 < <(get_metrics "$PEER_IP2")

    score1=$(calc_score "$loss1" "$rtt1")
    score2=$(calc_score "$loss2" "$rtt2")

    log "ens33: loss=${loss1}% rtt=${rtt1}ms (score=$score1) | ens37: loss=${loss2}% rtt=${rtt2}ms (score=$score2)"

    if [ "$ACTIVE" = "1" ] && [ "$score2" -lt "$(( score1 - HYSTERESIS ))" ]; then
        ACTIVE=2
        switch_routes 2
    elif [ "$ACTIVE" = "2" ] && [ "$score1" -lt "$(( score2 - HYSTERESIS ))" ]; then
        ACTIVE=1
        switch_routes 1
    fi

    sleep "$INTERVAL"
done
```

### Подробное объяснение скрипта

#### 1. Заголовок и настройки

```bash
#!/bin/bash
set -uo pipefail
```

- `#!/bin/bash` - shebang, указывает интерпретатор
- `set -uo pipefail` - строгий режим:
  - `-u` - ошибка при использовании неопределенных переменных
  - `-o pipefail` - ошибка если любая команда в пайпе упала

```bash
# === НАСТРОЙКИ ===
PEER_IP1="192.168.229.145"  # Оптика
PEER_IP2="192.168.229.148"  # Спутник
DEV1="ens33"
DEV2="ens37"
HYSTERESIS=30
INTERVAL=5
LOG_FILE="/var/log/sdwan_monitor.log"
ACTIVE=1
```

**Параметры:**
- `PEER_IP1/2` - IP-адреса для ping через каждый интерфейс
- `DEV1/2` - имена сетевых интерфейсов
- `HYSTERESIS=30` - порог в миллисекундах для защиты от "мигания"
- `INTERVAL=5` - интервал проверки в секундах
- `LOG_FILE` - путь к файлу логов
- `ACTIVE=1` - начальный активный канал (1=ens33, 2=ens37)

#### 2. Функция логирования

```bash
log() { echo "$(date '+%Y-%m-%d %H:%M:%S') $1" | tee -a "$LOG_FILE"; }
```

**Что делает:**
- Добавляет временную метку к сообщению
- Выводит на экран (stdout)
- Дописывает в файл логов (`-a` = append)

**Пример:**
```
2026-05-02 09:38:05 SD-WAN запущен. Активный: 1 (ens33)
```

#### 3. Функция get_metrics()

```bash
get_metrics() {
    local ip=$1
    local out
    
    out=$(ping -c 3 -W 3 "$ip" 2>&1)
    local ping_exit=$?
```

**Параметры ping:**
- `-c 3` - отправить 3 пакета
- `-W 3` - таймаут ожидания ответа 3 секунды
- `2>&1` - перенаправить stderr в stdout
- `ping_exit=$?` - сохранить код возврата ping

```bash
    # Если ping вообще не прошел
    if [ $ping_exit -ne 0 ] || ! echo "$out" | grep -q "packet loss"; then
        echo "100 9999"
        return
    fi
```

**Обработка ошибки:**
- Если ping не удался (код ≠ 0) или нет строки "packet loss"
- Возвращаем `100 9999` (100% потерь, 9999ms задержка)
- Это максимально плохая метрика

```bash
    # Парсим packet loss (дробное число)
    local loss=$(echo "$out" | awk -F', ' '/packet loss/ {print $3}' | awk '{print $1}' | tr -d '%' | cut -d. -f1)
```

**Пошаговый парсинг loss:**

1. `awk -F', '` - разбиваем по запятым
   ```
   Пример вывода ping:
   "3 packets transmitted, 2 received, 33.3333% packet loss, time 2000ms"
   
   Поле $3 = "33.3333% packet loss"
   ```

2. `awk '{print $1}'` - берем первое слово
   ```
   Результат: "33.3333%"
   ```

3. `tr -d '%'` - удаляем символ %
   ```
   Результат: "33.3333"
   ```

4. `cut -d. -f1` - убираем дробную часть
   ```
   Результат: "33"
   ```

```bash
    # Парсим avg rtt (второе число после /)
    local rtt=$(echo "$out" | awk -F'/' '/rtt/ {print int($5)}')
```

**Пошаговый парсинг rtt:**

1. `awk -F'/'` - разбиваем по символу `/`
   ```
   Пример строки:
   "rtt min/avg/max/mdev = 469.961/473.063/476.165/3.102 ms"
   
   Поля после разбиения:
   $1 = "rtt min"
   $2 = "avg"
   $3 = "max"
   $4 = "mdev "
   $5 = "473.063"  ← это avg RTT!
   ```

2. `int($5)` - преобразуем в целое число
   ```
   Результат: 473
   ```

```bash
    # Если не удалось распарсить
    loss=${loss:-100}
    rtt=${rtt:-9999}
    
    echo "$loss $rtt"
}
```

**Значения по умолчанию:**
- `${loss:-100}` - если loss пустой, используем 100
- `${rtt:-9999}` - если rtt пустой, используем 9999

#### 4. Функция switch_routes()

```bash
switch_routes() {
    local new_active=$1
    log "🔄 Попытка переключения на канал $new_active..."
```

**Параметр:**
- `$1` - номер канала для активации (1 или 2)

```bash
    if [ "$new_active" = "1" ]; then
        if ip route replace "$PEER_IP1" dev "$DEV1" metric 100 2>&1 && \
           ip route replace "$PEER_IP2" dev "$DEV2" metric 200 2>&1; then
            log "✅ Переключение на канал 1 (ens33/Оптика) УСПЕШНО"
        else
            log "❌ Ошибка переключения на канал 1"
        fi
```

**Переключение на канал 1 (ens33):**
- ens33 получает metric 100 (основной)
- ens37 получает metric 200 (резервный)
- `2>&1` - выводим ошибки
- `&&` - выполняем вторую команду только если первая успешна

```bash
    else
        if ip route replace "$PEER_IP1" dev "$DEV1" metric 200 2>&1 && \
           ip route replace "$PEER_IP2" dev "$DEV2" metric 100 2>&1; then
            log "✅ Переключение на канал 2 (ens37/Спутник) УСПЕШНО"
        else
            log "❌ Ошибка переключения на канал 2"
        fi
    fi
}
```

**Переключение на канал 2 (ens37):**
- ens33 получает metric 200 (резервный)
- ens37 получает metric 100 (основной)

#### 5. Функция calc_score()

```bash
calc_score() {
    local loss=$1 rtt=$2
    echo $(( loss * 1000 + rtt ))
}
```

**Формула расчета:**
```
Score = (Loss% × 1000) + RTT(ms)
```

**Примеры:**

| Канал | Loss | RTT | Расчет | Score |
|-------|------|-----|--------|-------|
| ens33 (норма) | 0% | 11ms | 0×1000 + 11 | **11** |
| ens37 (норма) | 0% | 315ms | 0×1000 + 315 | **315** |
| ens33 (сбой) | 40% | 10ms | 40×1000 + 10 | **40010** |
| ens37 (сбой) | 20% | 9999ms | 20×1000 + 9999 | **29999** |

**Почему loss × 1000?**
- Потери пакетов критичнее задержки
- Даже 1% потерь (score +1000) перевешивает 999ms задержки
- Это соответствует принципам real-world SD-WAN

#### 6. Основной цикл

```bash
log "SD-WAN запущен. Активный: $ACTIVE (ens33)"

while true; do
    read loss1 rtt1 < <(get_metrics "$PEER_IP1")
    read loss2 rtt2 < <(get_metrics "$PEER_IP2")
```

**Бесконечный цикл:**
- `while true` - выполняем пока истина
- `read loss1 rtt1 < <(...)` - читаем вывод функции в переменные
- Получаем метрики для обоих каналов

```bash
    score1=$(calc_score "$loss1" "$rtt1")
    score2=$(calc_score "$loss2" "$rtt2")

    log "ens33: loss=${loss1}% rtt=${rtt1}ms (score=$score1) | ens37: loss=${loss2}% rtt=${rtt2}ms (score=$score2)"
```

**Расчет и логирование:**
- Считаем score для каждого канала
- Выводим в лог текущие метрики

```bash
    if [ "$ACTIVE" = "1" ] && [ "$score2" -lt "$(( score1 - HYSTERESIS ))" ]; then
        ACTIVE=2
        switch_routes 2
```

**Условие переключения с ens33 на ens37:**
- Текущий активный: канал 1
- score2 < (score1 - 30)
- То есть канал 2 должен быть **минимум на 30 единиц лучше**

**Пример:**
- ens33: score = 40010 (40% потерь)
- ens37: score = 315 (0% потерь, 315ms)
- Проверка: 315 < (40010 - 30) → 315 < 39980 → **TRUE**
- Переключаемся на ens37

```bash
    elif [ "$ACTIVE" = "2" ] && [ "$score1" -lt "$(( score2 - HYSTERESIS ))" ]; then
        ACTIVE=1
        switch_routes 1
    fi
```

**Условие переключения с ens37 на ens33:**
- Текущий активный: канал 2
- score1 < (score2 - 30)
- Канал 1 должен стать лучше с запасом

```bash
    sleep "$INTERVAL"
done
```

**Пауза:**
- Ждем 5 секунд до следующей проверки
- Предотвращаем перегрузку сети частыми ping

---

## Алгоритм работы

### Общая схема

```
┌─────────────────────────────────────────────────────────────┐
│                      START (boot)                           │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
        ┌────────────────────────────────┐
        │  Настройка tc (delay, loss)    │
        └───────────────────────────────┘
                         │
                         ▼
        ┌────────────────────────────────┐
        │  Настройка маршрутов           │
        │  ens33: metric 100 (active)    │
        │  ens37: metric 200 (backup)    │
        └────────────────┬───────────────┘
                         │
                         ▼
        ┌────────────────────────────────┐
        │     Запуск sdwan_monitor.sh    │
        └────────────────┬───────────────┘
                         │
            ┌────────────┴────────────┐
            │      WHILE TRUE LOOP    │
            └────────────┬────────────┘
                         │
            ┌────────────▼────────────┐
            │  Ping PEER_IP1 (ens33)  │
            │  Ping PEER_IP2 (ens37)  │
            └────────────┬────────────┘
                         │
            ┌────────────▼────────────┐
            │  Parse: loss, rtt       │
            └────────────┬────────────┘
                         │
            ┌────────────▼────────────┐
            │  Calc Score:            │
            │  score = loss×1000+rtt  │
            └────────────┬────────────┘
                         │
            ┌────────────▼────────────┐
            │  Log metrics            │
            └────────────┬────────────┘
                         │
            ┌────────────▼────────────┐
            │  Compare scores         │
            │  with hysteresis        │
            └────────────┬────────────┘
                         │
         ┌───────────────┴───────────────┐
         │                               │
         ▼                               ▼
┌─────────────────┐            ┌─────────────────┐
│ score2 < score1 │            │ score1 < score2 │
│ - HYSTERESIS    │            │ - HYSTERESIS    │
└────────────────┘            └────────┬────────┘
         │                              │
         ▼                              ▼
┌─────────────────┐            ┌─────────────────┐
│  SWITCH TO      │            │  SWITCH TO      │
│  ens37 (backup) │            │  ens33 (main)   │
└────────────────┘            └────────┬────────┘
         │                              │
         └──────────────┬───────────────┘
                        │
                        ▼
            ┌───────────────────────┐
            │  sleep INTERVAL (5s)  │
            └───────────┬───────────┘
                        │
                        └─────► back to WHILE
```

### Сценарий работы

#### Сценарий 1: Нормальная работа

```
Время: 09:26:00
ens33: loss=0% rtt=11ms (score=11)
ens37: loss=0% rtt=335ms (score=335)

Решение: ens33 лучше (11 < 335)
Действие: Оставить ens33 активным
```

#### Сценарий 2: Деградация основного канала

```
Время: 09:26:59
ens33: loss=40% rtt=10ms (score=40010)
ens37: loss=0% rtt=315ms (score=315)

Проверка: 315 < (40010 - 30)?
          315 < 39980? → TRUE

Действие: Переключение на ens37
Лог: ✅ Переключение на канал 2 (ens37/Спутник) УСПЕШНО
```

#### Сценарий 3: Восстановление основного канала

```
Время: 09:27:30
ens33: loss=0% rtt=9ms (score=9)
ens37: loss=0% rtt=328ms (score=328)

Проверка: 9 < (328 - 30)?
          9 < 298? → TRUE

Действие: Переключение на ens33
Лог: ✅ Переключение на канал 1 (ens33/Оптика) УСПЕШНО
```

#### Сценарий 4: Hysteresis предотвращает "мигание"

```
Время: 09:28:00
ens33: score=280 (небольшая деградация)
ens37: score=300 (текущий активный)

Проверка: 280 < (300 - 30)?
          280 < 270? → FALSE

Действие: НЕ переключаться
Причина: Разница слишком мала (всего 20 единиц)
```

---

## Тестирование и валидация

### 1. Базовая проверка

**Запуск скрипта:**
```bash
chmod +x sdwan_monitor.sh
sudo ./sdwan_monitor.sh
```

**Ожидаемый вывод:**
```
2026-05 09:38:05 SD-WAN запущен. Активный: 1 (ens33)
2026-05 09:38:10 ens33: loss=0% rtt=11ms (score=11) | ens37: loss=0% rtt=335ms (score=335)
2026-05 09:38:15 ens33: loss=0% rtt=9ms (score=9) | ens37: loss=0% rtt=327ms (score=327)
```

### 2. Проверка маршрутов

**До переключения:**
```bash
ip route get 192.168.229.145
```

**Вывод:**
```
192.168.229.145 dev ens33 src 192.168.229.146 uid 0
    cache
```

**После переключения:**
```bash
ip route get 192.168.229.145
```

**Вывод:**
```
192.168.229.145 dev ens37 src 192.168.229.147 uid 0
    cache
```

### 3. Имитация сбоя

**Терминал 1 (мониторинг):**
```bash
tail -f /var/log/sdwan_monitor.log
```

**Терминал 2 (имитация сбоя):**
```bash
# Ухудшаем ens33
sudo tc qdisc change dev ens33 root netem delay 500ms 100ms loss 15%
```

**Ожидаемый результат в логе:**
```
2026-05-02 09:38:45 ens33: loss=15% rtt=523ms (score=15523) | ens37: loss=0% rtt=328ms (score=328)
2026-05-02 09:38:50 🔄 Попытка переключения на канал 2...
2026-05-02 09:38:50 ✅ Переключение на канал 2 (ens37/Спутник) УСПЕШНО
```

### 4. Проверка ping в реальном времени

**Запустите ping в отдельном терминале:**
```bash
ping -c 10 192.168.229.145
```

**До переключения (ens33 активен):**
```
64 bytes from 192.168.229.145: icmp_seq=1 ttl=64 time=9.23 ms
64 bytes from 192.168.229.145: icmp_seq=2 ttl=64 time=8.91 ms
64 bytes from 192.168.229.145: icmp_seq=3 ttl=64 time=10.4 ms
```

**Во время переключения:**
```
64 bytes from 192.168.229.145: icmp_seq=4 ttl=64 time=523 ms  ← задержка растет
64 bytes from 192.168.229.145: icmp_seq=5 ttl=64 time=328 ms  ← переключение
64 bytes from 192.168.229.145: icmp_seq=6 ttl=64 time=315 ms  ← стабильно на ens37
```

### 5. Восстановление

**Верните ens33 в норму:**
```bash
sudo tc qdisc change dev ens33 root netem delay 5ms 2ms
```

**Ожидаемый результат:**
```
2026-05 09:39:15 ens33: loss=0% rtt=11ms (score=11) | ens37: loss=0% rtt=328ms (score=328)
2026-05 09:39:20 🔄 Попытка переключения на канал 1...
2026-05 09:39:20 ✅ Переключение на канал 1 (ens33/Оптика) УСПЕШНО
```

### 6. Использование mtr для визуализации

**Запустите mtr:**
```bash
mtr -n 192.168.229.145
```

**Вывод (до переключения):**
```
My traceroute  [v0.95]
debian1 (192.168.229.146) -> 192.168.229.145 (192.168.229.145)
Keys:  Help   Display mode   Restart statistics   Order of fields   quit
                              Packets               Pings
 Host                Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. 192.168.229.145   0.0%    50    9.2   10.1  8.1  15.2  2.1
```

**Вывод (после переключения на ens37):**
```
 Host                Loss%   Snt   Last   Avg   Best  Wrst  StDev
 1. 192.168.229.145   0.0%    50   315   320   298   380   25.3
```

---

## Устранение неполадок

### Проблема 1: Скрипт падает с ошибкой "синтаксическая ошибка"

**Пример:**
```
./sdwan_monitor.sh: строка 46: 0%: синтаксическая ошибка: ожидается операнд
```

**Причина:**
В переменной `loss` остался символ `%`

**Решение:**
Убедитесь, что в функции `get_metrics()` есть:
```bash
loss=$(... | tr -d '%' | cut -d. -f1)
```

### Проблема 2: Все значения = 9999

**Пример:**
```
ens33: loss=100% rtt=9999ms (score=109999)
ens37: loss=100% rtt=9999ms (score=109999)
```

**Причины:**
1. Ping не проходит до хоста
2. Неправильный парсинг вывода ping

**Диагностика:**
```bash
# Проверка связи
ping -c 3 192.168.229.145

# Проверка вывода
ping -c 3 -W 3 192.168.229.145 2>&1 | grep "packet loss"
```

**Решение:**
1. Проверьте, что интерфейсы в сети UP:
   ```bash
   ip link show ens33
   ip link show ens37
   ```

2. Проверьте маршруты:
   ```bash
   ip route show
   ```

3. Проверьте, что tc настроен:
   ```bash
   tc qdisc show dev ens33
   ```

### Проблема 3: Переключение не происходит

**Пример:**
- Score меняется, но переключения нет
- В логе нет сообщений "✅ Переключение..."

**Причины:**
1. Не выполняется условие с hysteresis
2. Ошибка в команде `ip route replace`

**Диагностика:**
```bash
# Проверьте текущие метрики маршрутов
ip route show | grep 192.168.229

# Попробуйте переключить вручную
sudo ip route replace 192.168.229.145 dev ens33 metric 200
sudo ip route replace 192.168.229.148 dev ens37 metric 100

# Проверьте результат
ip route get 192.168.229.145
```

**Решение:**
1. Уменьшите HYSTERESIS в скрипте:
   ```bash
   HYSTERESIS=10  # вместо 30
   ```

2. Добавьте отладку в скрипт:
   ```bash
   # Перед проверкой условия
   log "DEBUG: ACTIVE=$ACTIVE, score1=$score1, score2=$score2"
   log "DEBUG: Threshold for switch: $(( score1 - HYSTERESIS ))"
   ```

### Проблема 4: Частые переключения (flapping)

**Пример:**
```
09:38:10 ✅ Переключение на канал 2
09:38:15 ✅ Переключение на канал 1
09:38:20 ✅ Переключение на канал 2
09:38:25 ✅ Переключение на канал 1
```

**Причина:**
Метрики каналов близки, разница < HYSTERESIS

**Решение:**
1. Увеличьте HYSTERESIS:
   ```bash
   HYSTERESIS=100  # вместо 30
   ```

2. Увеличьте INTERVAL:
   ```bash
   INTERVAL=10  # вместо 5
   ```

3. Увеличьте количество ping пакетов:
   ```bash
   ping -c 10 -W 5  # вместо -c 3 -W 3
   ```

### Проблема 5: tc не применяется

**Пример:**
```bash
sudo tc qdisc add dev ens33 root netem delay 200ms
# Ошибка: RTNETLINK answers: File exists
```

**Решение:**
```bash
# Сначала удалите существующий qdisc
sudo tc qdisc del dev ens33 root

# Затем добавьте новый
sudo tc qdisc add dev ens33 root netem delay 200ms 50ms loss 2%
```

**Проверка:**
```bash
tc qdisc show dev ens33
ping -c 5 192.168.229.145  # задержка должна вырасти
```

### Проблема 6: Скрипт не запускается автоматически

**Решение: Создайте systemd сервис**

```bash
sudo nano /etc/systemd/system/sdwan-monitor.service
```

**Содержимое:**
```ini
[Unit]
Description=SD-WAN Monitor and Failover
After=network.target

[Service]
Type=simple
ExecStart=/root/sdwan_monitor.sh
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**Активация:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable sdwan-monitor
sudo systemctl start sdwan-monitor

# Проверка статуса
sudo systemctl status sdwan-monitor

# Просмотр логов
journalctl -u sdwan-monitor -f
```

---

## Дополнительные ресурсы

### Полезные команды

**Мониторинг сети:**
```bash
# Постоянный ping с timestamp
ping -D 192.168.229.145

# Трассировка с потерями
mtr -n 192.168.229.145

# Статистика интерфейсов
watch -n 1 'ip -s link show ens33'

# Проверка задержки
tc qdisc show dev ens33
```

**Отладка маршрутизации:**
```bash
# Таблица маршрутизации
ip route show table all

# Статистика маршрутов
ip -s route show

# Проверка конкретного маршрута
ip route get 192.168.229.145 from 192.168.229.146
```

**Анализ логов:**
```bash
# Последние 50 строк
tail -50 /var/log/sdwan_monitor.log

# Поиск переключений
grep "Переключение" /var/log/sdwan_monitor.log

# Подсчет переключений
grep -c "✅ Переключение" /var/log/sdwan_monitor.log

# Статистика по часам
grep -oP '\d{2}:\d{2}:\d{2}' /var/log/sdwan_monitor.log | cut -d: -f1 | sort | uniq -c
```

### Рекомендуемая литература

1. **Linux Traffic Control**
   - [Official TC documentation](https://man7.org/linux/man-pages/man8/tc.8.html)
   - [NETEM examples](https://wiki.linuxfoundation.org/networking/netem)

2. **SD-WAN Concepts**
   - [Dynamic Path Selection](https://www.cisco.com/c/en/us/solutions/enterprise-networks/sd-wan/index.html)

3. **Linux Networking**
   - [IP Route documentation](https://man7.org/linux/man-pages/man8/ip-route.8.html)
   - [Linux Advanced Routing](https://lartc.org/)

---

## Changelog

### Версия 1.0 (2026-05-02)
- ✅ Базовая реализация мониторинга
- ✅ Автоматическое переключение каналов
- ✅ Логирование событий
- ✅ Защита от flapping (hysteresis)
- ✅ Обработка ошибок ping
- ✅ systemd сервис для автозапуска

---

## Конфигурация

Лабораторная работа по SD-WAN технологиям  
Debian 12 + VMware Workstation  
Конфигурация: 2 ВМ, 2 сетевых адаптера, NAT сеть

---

## License

Этот проект создан в образовательных целях.  
Свободно распространяется и модифицируется.
