# TCP (Transmission Control Protocol)

Надёжный, упорядоченный, ориентированный на соединение протокол транспортного уровня (L4).

## Ключевые свойства

- **Надёжность** — гарантия доставки (ACK, retransmission)
- **Порядок** — данные приходят в порядке отправки (sequence numbers)
- **Flow control** — получатель управляет скоростью (window size)
- **Congestion control** — адаптация к загрузке сети

## TCP Handshake (установка соединения)

```
Client              Server
  |--- SYN ---------->|    1. Клиент инициирует (seq=x)
  |<-- SYN+ACK -------|    2. Сервер отвечает (seq=y, ack=x+1)
  |--- ACK ---------->|    3. Клиент подтверждает (ack=y+1)
```

3-way handshake: SYN → SYN-ACK → ACK. После этого соединение установлено.

## Закрытие соединения

```
Client              Server
  |--- FIN ---------->|    1. Клиент хочет закрыть
  |<-- ACK -----------|    2. Сервер подтверждает
  |<-- FIN -----------|    3. Сервер тоже закрывает
  |--- ACK ---------->|    4. Клиент подтверждает
```

4-way teardown. После этого сокет в состоянии `TIME_WAIT` (2 * MSL).

## Состояния TCP

```
CLOSED → LISTEN → SYN_RECEIVED → ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED
```

Основные:
- `LISTEN` — сервер ожидает соединения
- `ESTABLISHED` — соединение активно
- `TIME_WAIT` — ожидание после закрытия (предотвращает путаницу пакетов)
- `CLOSE_WAIT` — получен FIN, ожидание закрытия со своей стороны

## Flow Control

Sliding window: получатель сообщает сколько данных готов принять (`window size`).

```
Sender: [sent & acked] [sent, not acked] [can send] [cannot send]
                        <--- window size --->
```

Если window = 0, отправитель останавливается (Zero Window Probe для проверки).

## Congestion Control

### Slow Start
Начинает с малого `cwnd` (congestion window), удваивает каждый RTT до `ssthresh`.

### Congestion Avoidance
После `ssthresh` — линейный рост cwnd (+1 за RTT).

### Fast Retransmit
3 duplicate ACK → ретрансмиссия без ожидания таймера.

### Алгоритмы
- **Reno** — классический (slow start + congestion avoidance)
- **Cubic** — Linux default, агрессивнее при высоком bandwidth
- **BBR** — Google, оценивает bandwidth и RTT, лучше при потерях

## TCP vs UDP

| Свойство | TCP | UDP |
|----------|-----|-----|
| Соединение | Да (handshake) | Нет |
| Надёжность | Гарантированная | Нет |
| Порядок | Гарантированный | Нет |
| Overhead | Высокий (20 байт header) | Низкий (8 байт header) |
| Применение | HTTP, gRPC, DB | DNS, video, gaming, QUIC |

## Проблемы и решения

### Head-of-Line Blocking
Потерянный пакет блокирует все последующие. Решение: HTTP/3 (QUIC over UDP).

### TIME_WAIT Exhaustion
Много кратковременных соединений → исчерпание портов.

```bash
# Проверка
ss -s | grep TIME-WAIT

# Решения
# - Connection pooling
# - SO_REUSEADDR
# - Увеличить диапазон ephemeral портов
sysctl net.ipv4.ip_local_port_range
```

### Nagle's Algorithm
Собирает маленькие пакеты в один (экономит bandwidth, увеличивает latency).

```go
// Отключение (для low-latency приложений)
conn.(*net.TCPConn).SetNoDelay(true)
```

## TCP в Go

```go
// Сервер
listener, _ := net.Listen("tcp", ":8080")
for {
    conn, _ := listener.Accept()
    go handleConn(conn) // горутина на каждое соединение
}

// Клиент
conn, _ := net.DialTimeout("tcp", "server:8080", 5*time.Second)
defer conn.Close()
conn.SetDeadline(time.Now().Add(10 * time.Second))
```
