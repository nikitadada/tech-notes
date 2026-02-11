# Graceful Shutdown

Корректное завершение работы приложения: прекращение приёма новых запросов, завершение текущих, освобождение ресурсов.

## HTTP-сервер

```go
func main() {
    srv := &http.Server{
        Addr:    ":8080",
        Handler: setupRouter(),
    }

    // Запуск сервера в горутине
    go func() {
        log.Printf("server starting on %s", srv.Addr)
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            log.Fatalf("listen: %v", err)
        }
    }()

    // Ожидание сигнала завершения
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("shutting down...")

    // Даём время на завершение текущих запросов
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("forced shutdown: %v", err)
    }

    log.Println("server stopped")
}
```

`srv.Shutdown(ctx)`:
- Закрывает listener (перестаёт принимать новые соединения)
- Ждёт завершения активных запросов
- Если timeout — принудительное завершение

## Полный пример с несколькими компонентами

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    // Инициализация зависимостей
    db, err := initDB(ctx)
    if err != nil {
        log.Fatal(err)
    }

    cache := initCache()
    consumer := initKafkaConsumer()

    srv := &http.Server{
        Addr:    ":8080",
        Handler: newRouter(db, cache),
    }

    // Запуск компонентов
    g, gCtx := errgroup.WithContext(ctx)

    g.Go(func() error {
        log.Println("starting http server")
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            return err
        }
        return nil
    })

    g.Go(func() error {
        log.Println("starting kafka consumer")
        return consumer.Run(gCtx)
    })

    // Graceful shutdown при отмене контекста
    g.Go(func() error {
        <-gCtx.Done()
        log.Println("shutting down http server")

        shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer cancel()

        return srv.Shutdown(shutdownCtx)
    })

    g.Go(func() error {
        <-gCtx.Done()
        log.Println("closing resources")
        consumer.Close()
        cache.Close()
        return db.Close()
    })

    if err := g.Wait(); err != nil {
        log.Printf("exit with error: %v", err)
    }

    log.Println("shutdown complete")
}
```

## Kubernetes: preStop hook

В Kubernetes Pod получает SIGTERM, затем через `terminationGracePeriodSeconds` (default 30s) — SIGKILL.

**Проблема:** endpoints обновляются асинхронно. Pod может получить трафик после SIGTERM.

**Решение:** preStop hook с задержкой.

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]  # ждём обновления endpoints
```

Или в Go:

```go
// После получения SIGTERM — перестать проходить readiness probe
healthy := &atomic.Bool{}
healthy.Store(true)

// Readiness handler
mux.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
    if !healthy.Load() {
        w.WriteHeader(http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
})

// При shutdown
<-quit
healthy.Store(false)           // перестать быть ready
time.Sleep(5 * time.Second)    // подождать пока endpoints обновятся
srv.Shutdown(ctx)              // завершить текущие запросы
```

## Checklist

- [ ] Обработка SIGINT и SIGTERM
- [ ] Timeout на graceful shutdown
- [ ] Завершение HTTP-сервера через `Shutdown()`
- [ ] Закрытие соединений БД
- [ ] Остановка консьюмеров (Kafka, RabbitMQ)
- [ ] Flush логов и метрик
- [ ] preStop hook в Kubernetes
