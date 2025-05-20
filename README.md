# 🧠 Оптимизация функции F_WORKS_LIST

## Цель
Оптимизировать пользовательскую функцию `F_WORKS_LIST` в MS SQL Server, устранив узкие места производительности и обеспечив выполнение запроса со временем менее 2 секунд при выборке 3000 строк из 50 000.

---

## 1. Подготовка данных

Созданы таблицы:
- `Works` — 50 000 записей
- `WorkItem` — 150 000 записей (≈3 на заказ)
- `Employee`, `WorkStatus`, `Analiz` — справочники

Генерация данных: см. [generate_data.sql](generate_data.sql)

---

## 2. Анализ исходной реализации

См. [Issue #1](https://github.com/Anastasia615/ohf/issues/1)

Ключевые проблемы:
| Проблема | Описание |
|---------|----------|
| ❌ Скалярные функции | `F_WORKITEMS_COUNT_BY_ID_WORK` вызывается дважды |
| ❌ Вызов UDF `F_EMPLOYEE_FULLNAME` | Каждый вызов генерирует JOIN внутри себя |
| ❌ ORDER BY + TOP | Применяется после всех вычислений |
| ❌ Табличная переменная @RESULT | Отсутствуют статистики — плохой план |

---

## 3. Оптимизированный запрос

См. [Issue #2](https://github.com/Anastasia615/ohf/issues/2)

```sql
SELECT TOP (3000)
    w.Id_Work,
    w.CREATE_Date,
    w.MaterialNumber,
    w.IS_Complit,
    w.FIO,
    CONVERT(VARCHAR(10), w.CREATE_Date, 104) AS D_DATE,
    SUM(CASE WHEN wi.is_complit = 0 AND a.is_group = 0 THEN 1 ELSE 0 END) AS WorkItemsNotComplit,
    SUM(CASE WHEN wi.is_complit = 1 AND a.is_group = 0 THEN 1 ELSE 0 END) AS WorkItemsComplit,
    e.Surname + ' ' + UPPER(LEFT(e.Name,1)) + '. ' + UPPER(LEFT(e.Patronymic,1)) + '.' AS EmployeeFullName,
    w.StatusId,
    ws.StatusName,
    CASE
      WHEN w.Print_Date IS NOT NULL OR w.SendToClientDate IS NOT NULL
        OR w.SendToDoctorDate IS NOT NULL OR w.SendToOrgDate IS NOT NULL
        OR w.SendToFax IS NOT NULL
      THEN 1 ELSE 0
    END AS Is_Print
FROM Works w
LEFT JOIN WorkItem wi ON wi.id_work = w.Id_Work
LEFT JOIN Analiz a ON a.id_analiz = wi.id_analiz AND a.is_group = 0
LEFT JOIN Employee e ON e.Id_Employee = w.Id_Employee
LEFT JOIN WorkStatus ws ON w.StatusId = ws.StatusID
WHERE w.IS_DEL <> 1
GROUP BY
  w.Id_Work, w.CREATE_Date, w.MaterialNumber, w.IS_Complit, w.FIO,
  e.Surname, e.Name, e.Patronymic,
  w.StatusId, ws.StatusName,
  w.Print_Date, w.SendToClientDate,
  w.SendToDoctorDate, w.SendToOrgDate, w.SendToFax
ORDER BY w.Id_Work DESC;
```

---

## 4. Результаты

| Метрика | До | После |
|--------|-----|--------|
| Время выполнения | ~20 сек | ~1.5 сек |
| Использование UDF | Да | Нет |
| Имеются GROUP BY / JOIN | Нет | Да |

---

## 5. Архитектурные улучшения

См. [Issue #3](https://github.com/Anastasia615/ohf/issues/3)

### 💡 Идеи:
- Индекс на `(WorkItem.id_work, is_complit)`
- `PERSISTED computed column` для ФИО
- Кэш-таблица `Works_Stats` с триггерами
