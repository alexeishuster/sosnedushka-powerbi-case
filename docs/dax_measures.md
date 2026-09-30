# Справочник DAX-мер — проект «Соседушка»

> Все меры написаны с учётом специфики датасета: таблица фактов fact_sales содержит строки чеков (один чек = несколько строк по товарам), поэтому для подсчёта числа чеков используется DISTINCTCOUNT, а не COUNTROWS.

---

## Базовые метрики

### revenue (Выручка)
    revenue = SUM(fact_sales[сумма_строки])

Бизнес-смысл: Сумма всех строк чеков за выбранный период. Поле сумма_строки уже учитывает количество и скидку, поэтому дополнительных умножений не требуется.

---

### quantity_checks (Количество чеков)
    quantity_checks = DISTINCTCOUNT(fact_sales[id_чека])

Почему DISTINCTCOUNT, а не COUNTROWS: В таблице фактов один чек представлен несколькими строками (по одной на каждый товар). COUNTROWS дал бы завышенное число «продаж», а не реальных чеков.

---

### avg_check (Средний чек)
    avg_check = DIVIDE([revenue], [quantity_checks])

Почему DIVIDE, а не оператор /: Функция DIVIDE автоматически обрабатывает деление на ноль (возвращает BLANK), что защищает от ошибок в визуализациях.

---

### quantity_clients (Количество уникальных клиентов)
    quantity_clients = DISTINCTCOUNT(fact_sales[id_клиента])

Считает только клиентов с картой — пустые id_клиента функция DISTINCTCOUNT игнорирует автоматически.

---

## Сегментация по карте лояльности

Ключевой инсайт проекта. Поле id_клиента заполнено только у держателей карты (~55% чеков), у остальных — BLANK.

### avg_check_card (Средний чек по карте)
    avg_check_card = CALCULATE([avg_check], NOT ISBLANK(fact_sales[id_клиента]))

### avg_check_no_card (Средний чек без карты)
    avg_check_no_card = CALCULATE([avg_check], ISBLANK(fact_sales[id_клиента]))

Важно: Используется ISBLANK, а не проверка на = 0 или = "", потому что в датасете пропуски именно в формате NULL/BLANK.

---

## Временные сравнения

### sales_prev_year (Выручка за аналогичный период прошлого года)
    sales_prev_year = CALCULATE([revenue], SAMEPERIODLASTYEAR(dim_calendar[Date]))

Функция SAMEPERIODLASTYEAR сдвигает контекст фильтра ровно на 1 год назад. Работает только при наличии полноценной таблицы дат (dim_calendar) с непрерывным диапазоном.

---

### YoY_percent (Рост выручки год к году, в долях)
    YoY_percent = DIVIDE([revenue] - [sales_prev_year], [sales_prev_year])

Формат вывода в Power BI: Процентный, 2 знака после запятой.
Пример: значение 0,12 отображается как 12,00% (рост), -0,05 как -5,00% (падение).

---

## Таблица дат (dim_calendar)

Создана через DAX как Calculated Table — это best practice для Power BI:

    dim_calendar = 
    ADDCOLUMNS(
        CALENDAR(MIN(fact_sales[дата_время]), MAX(fact_sales[дата_время])),
        "Год", YEAR([Date]),
        "Номер месяца", MONTH([Date]),
        "Месяц", FORMAT([Date], "MMMM"),
        "Год-месяц", FORMAT([Date], "YYYY-MM")
    )

Почему своя таблица дат, а не встроенная иерархия Power BI:
- Даёт полный контроль над форматами (русские названия месяцев через FORMAT).
- Позволяет использовать time-intelligence функции (SAMEPERIODLASTYEAR, TOTALYTD, DATESINPERIOD).
- Связь с fact_sales[дата_время] устанавливается один раз и работает во всех мерах.

---

## Best practices, применённые в проекте

| Практика | Где применена | Зачем |
|---|---|---|
| DIVIDE вместо / | avg_check, YoY_percent | Защита от деления на ноль |
| DISTINCTCOUNT вместо COUNTROWS | quantity_checks | Корректный подсчёт чеков |
| ISBLANK вместо = 0 | avg_check_card | Корректная работа с NULL |
| Отдельная таблица дат | dim_calendar | Работа time-intelligence функций |
| Меры ссылаются друг на друга | avg_check → revenue | Переиспользование и читаемость |

---

*Файл создан в рамках проекта «Соседушка». Инструменты: Power BI, DAX, Power Query.*
