# Compliance

Соответствие нормативным требованиям для хранения и обработки данных.

## Основные стандарты

| Стандарт | Область | Требования (ключевые) |
|----------|---------|----------------------|
| **SOC 2** | Security, Availability | Access controls, monitoring, incident response |
| **PCI DSS** | Платёжные данные | Encryption, network segmentation, audit logging |
| **HIPAA** | Медицинские данные (US) | Encryption, access controls, BAA |
| **GDPR** | Персональные данные (EU) | Data residency, right to erasure, consent |
| **ISO 27001** | Информационная безопасность | ISMS, risk management, controls |
| **FZ-152** | Персональные данные (РФ) | Хранение на территории РФ, уведомление РКН |

## Что требуется от DBaaS

### Encryption

```
✅ Encryption at rest (AES-256)
✅ Encryption in transit (TLS 1.2+)
✅ Backup encryption
✅ Key management (KMS integration)
✅ CMEK support (customer-managed keys)
```

### Access Control

```
✅ RBAC с принципом least privilege
✅ MFA для доступа к Control Plane
✅ IP allowlisting
✅ Audit logging всех действий
✅ Separation of duties (admin ≠ DBA ≠ developer)
```

### Data Residency

```
✅ Выбор региона хранения данных
✅ Гарантия что данные не покидают регион
✅ Cross-region replication только по явному запросу
✅ Бэкапы в том же регионе (или по выбору)
```

### Audit Trail

```
✅ Логирование всех API-вызовов (кто, что, когда)
✅ Логирование доступа к данным (pgAudit)
✅ Immutable logs (нельзя удалить/изменить)
✅ Retention: минимум 1 год (зависит от стандарта)
✅ Export логов для внешнего анализа
```

### Incident Response

```
✅ Процедура обнаружения и реагирования на инциденты
✅ Уведомление клиентов о breach (72 часа для GDPR)
✅ Post-mortem и corrective actions
✅ Regular security testing (penetration testing)
```

## GDPR специфика

### Right to Erasure (право на удаление)

```
User request → DELETE all personal data

Implementation:
1. Delete from active DB
2. Delete from backups (или crypto shredding)
3. Delete from logs (или anonymization)
4. Confirm deletion to user
```

**Crypto shredding:** удаление ключа шифрования → данные становятся нечитаемыми, даже если физически остались в бэкапе.

### Data Processing Agreement (DPA)

DBaaS-провайдер — data processor. Нужен DPA с описанием:
- Какие данные обрабатываются
- Цели обработки
- Меры безопасности
- Процедура удаления
- Sub-processors (третьи стороны)

## PCI DSS специфика

```
Scope: системы, хранящие/обрабатывающие/передающие cardholder data

Requirements for DBaaS:
- Network segmentation (separate PCI environment)
- Strong encryption (TLS 1.2+, AES-256)
- Access logging and monitoring
- Regular vulnerability scanning
- Penetration testing
- Quarterly ASV scans
```

## Compliance features в DBaaS

| Feature | Описание |
|---------|----------|
| Region lock | Данные не покидают выбранный регион |
| Dedicated instances | Физическая изоляция для compliance |
| CMEK | Пользователь контролирует ключи |
| Audit export | Экспорт логов в SIEM |
| Deletion certificate | Подтверждение уничтожения данных |
| Compliance reports | SOC 2 report, penetration test results |
| Data classification | Метки чувствительности данных |
