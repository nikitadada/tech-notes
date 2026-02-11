# Shared vs Dedicated

Два фундаментальных подхода к размещению пользовательских инстансов БД в DBaaS.

## Shared (разделяемые ресурсы)

Несколько пользовательских инстансов работают на одном физическом/виртуальном хосте.

```
┌─────────── Host ───────────┐
│  ┌─────┐ ┌─────┐ ┌─────┐  │
│  │DB A │ │DB B │ │DB C │  │
│  └─────┘ └─────┘ └─────┘  │
│        Shared CPU/RAM       │
│        Shared Disk I/O      │
└─────────────────────────────┘
```

**Плюсы:**
- Низкая стоимость (ресурсы делятся)
- Быстрый provisioning (не нужно создавать VM)
- Эффективное использование железа

**Минусы:**
- Noisy neighbor — один инстанс может повлиять на другие
- Ограниченная производительность
- Меньше изоляции (security boundary)
- Сложнее гарантировать SLA

**Применение:** dev/staging окружения, маленькие проекты, free tier.

## Dedicated (выделенные ресурсы)

Каждый инстанс работает на отдельной VM или bare-metal сервере.

```
┌──── Host 1 ────┐  ┌──── Host 2 ────┐  ┌──── Host 3 ────┐
│    ┌─────┐     │  │    ┌─────┐     │  │    ┌─────┐     │
│    │DB A │     │  │    │DB B │     │  │    │DB C │     │
│    └─────┘     │  │    └─────┘     │  │    └─────┘     │
│  Dedicated CPU │  │  Dedicated CPU │  │  Dedicated CPU │
│  Dedicated RAM │  │  Dedicated RAM │  │  Dedicated RAM │
└────────────────┘  └────────────────┘  └────────────────┘
```

**Плюсы:**
- Предсказуемая производительность
- Полная изоляция (network, compute, storage)
- Проще соответствовать compliance (PCI DSS, HIPAA)
- Гарантированные ресурсы

**Минусы:**
- Дороже
- Медленнее provisioning (создание VM)
- Менее эффективное использование железа

**Применение:** production, высоконагруженные системы, compliance-sensitive данные.

## Гибридные модели

### Shared compute, dedicated storage

```
┌────────── Host ──────────┐
│  ┌─────┐       ┌─────┐  │
│  │DB A │       │DB B │  │
│  └──┬──┘       └──┬──┘  │
│     │             │      │  Shared CPU/RAM
└─────┼─────────────┼──────┘
      │             │
  ┌───┴───┐    ┌───┴───┐
  │Disk A │    │Disk B │     Dedicated storage
  └───────┘    └───────┘
```

### Resource quotas (cgroups)

Shared host, но каждый инстанс ограничен по CPU/RAM через cgroups или Kubernetes resource limits.

```yaml
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "2"
    memory: "4Gi"
```

## Сравнение

| Критерий | Shared | Dedicated |
|----------|--------|-----------|
| Стоимость | Низкая | Высокая |
| Производительность | Варьируется | Предсказуемая |
| Изоляция | Логическая | Физическая |
| Provisioning | Быстрый | Медленнее |
| Compliance | Ограничен | Полный |
| Noisy neighbor | Да | Нет |
| Масштабирование | Ограничено хостом | Свобода выбора |

## Выбор для пользователя

Многие DBaaS предлагают тарифные планы, соответствующие этим моделям:

| Tier | Модель | Пример |
|------|--------|--------|
| Free / Hobby | Shared | Neon Free, Supabase Free |
| Standard | Shared с quotas | RDS db.t3, Cloud SQL shared-core |
| Production | Dedicated | RDS db.m5, Cloud SQL standard |
| Enterprise | Dedicated + isolation | Dedicated hosts, private clusters |
