# RFM-analysis

### Что такое RFM-анализ?

***RFM*** (Recency, Frequency, Monetary) анализ — это метод сегментации клиентов, основанный на трех ключевых метриках:
-	***Recency*** (давность) — как давно клиент делал свою последнюю покупку;
-	***Frequency*** (частота) — как часто клиент делает покупки;
-	***Monetary*** (сумма покупок) — сколько денег клиент потратил за определенный период времени.
***RFM*** анализ помогает компаниям лучше понять поведение клиентов, выявить лучшие сегменты аудитории и нацелить маркетинговые кампании более эффективно.

## Сегментация клиентов

Клиенты делятся на группы по трем метрикам:

### По давности заказа (Recency):
1. **Давние клиенты**
2. **Относительно недавние клиенты**
3. **Недавние клиенты**

### По частоте покупок (Frequency):
1. **Покупает очень редко** (единичные заказы)
2. **Покупает нечасто**
3. **Покупает часто**

### По сумме покупок (Monetary):
1. **Маленькая сумма**
2. **Средняя сумма**
3. **Большая сумма**

### Примеры клиентов:
- **«111»**: делал покупку давно, один раз и на маленькую сумму.
- **«333»**: покупает часто, на большую сумму, и его последняя покупка была недавно. Это наши лучшие клиенты.

**Важно отметить**, что диапазоны для 1, 2 и 3 вы задаете сами, определяя, что значит маленькая, средняя и большая сумма продаж для вашего бизнеса.

## Построение запросов в SQL:
Сосредоточимся на самом важном: начнем наш анализ, пропуская этап загрузки данных в базу и ее настройку.

```sql
create table customer_segments as
with aggregated_data as (
    select CustomerId,
           max(InvoiceDate) as LastDate,
           datediff('2011-12-09', max(InvoiceDate)) as Recency,
           count(distinct Invoice) as Frequency,
           round(sum(UnitPrice * Quantity), 2) as TotalSum
    from Sales
    group by CustomerId
),
normalized_data as (
    select CustomerId,
           Recency,
           Frequency,
           TotalSum,
           (Recency - min(Recency) over()) / (max(Recency) over() - min(Recency) over()) as Recency_normalized,
           (Frequency - min(Frequency) over()) / (max(Frequency) over() - min(Frequency) over()) as Frequency_normalized,
           (TotalSum - min(TotalSum) over()) / (max(TotalSum) over() - min(TotalSum) over()) as Monetary_normalized
    from aggregated_data
)
select CustomerId,
       ntile(3) over (order by Recency_normalized desc) as Recency,
       ntile(3) over (order by Frequency_normalized asc) as Frequency,
       ntile(3) over (order by Monetary_normalized asc) as Monetary
from normalized_data;
```
В данной части кода создаем таблицу `customer_segments`, которая будет содержать сегменты клиентов на основе анализа RFM. На первом этапе мы агрегируем данные о каждом клиенте. Для этого мы извлекаем идентификатор клиента, дату последней покупки, вычисляем давность (Recency), частоту покупок (Frequency) и общую сумму покупок (TotalSum). После агрегации данных мы проводим нормализацию метрик, чтобы привести их к одному диапазону. Это важно, так как каждая из метрик может иметь разные масштабы, что может повлиять на результаты анализа.

## Что такое нормализация?

***Нормализация данных*** — это процесс преобразования данных так, чтобы все значения находились в одном диапазоне, обычно от 0 до 1. Это делает данные более сопоставимыми, особенно когда разные метрики имеют разные масштабы. Например, в контексте RFM-анализа метрики `Frequency` и `Monetary` могут значительно отличаться по своим диапазонам.

## Зачем нужна нормализация?

1. **Сравнимость данных**: Алгоритмы анализа могут быть предвзятыми, если метрики имеют разные диапазоны. Нормализация делает метрики сопоставимыми.
2. **Улучшение точности сегментации**: В RFM-анализе важно, чтобы каждая из метрик имела одинаковый вес для более точной сегментации клиентов.

Мы используем следующую формулу для нормализации:

![image](https://github.com/user-attachments/assets/89edf972-141c-4652-bc0d-440358822bc3)


## Использование функции NTILE

После нормализации данных мы применяем функцию `NTILE(3)`, чтобы разделить клиентов на три группы по каждой из метрик RFM.

Функция `NTILE` делит отсортированный набор данных на указанные группы (в данном случае на 3) и присваивает каждой записи номер группы. Это позволяет нам легко сегментировать клиентов в зависимости от их поведения и ценности для бизнеса.
## Сегментация клиентов на основе RFM-анализа

На основе ***RFM-анализа*** клиенты были распределены на следующие группы:

```sql
update Customer_Segments
set Segment = case 
    when Recency = 3 and Frequency = 3 and Monetary = 3 then 'VIP'
    when Recency = 3 and Frequency = 3 and Monetary = 2 then 'Постоянные со средним чеком'
    when Recency = 3 and Frequency = 3 and Monetary = 1 then 'Постоянные с маленьким чеком'
    when Recency = 3 and Frequency = 2 and Monetary = 3 then 'Постоянные с высоким чеком'
    when Recency = 3 and Frequency = 2 and Monetary = 2 then 'Постоянные со средним чеком'
    when Recency = 3 and Frequency = 2 and Monetary = 1 then 'Постоянные с маленьким чеком'
    when Recency = 3 and Frequency = 1 and Monetary = 3 then 'Новички с высоким чеком'
    when Recency = 3 and Frequency = 1 and Monetary = 2 then 'Новички со средним чеком'
    when Recency = 3 and Frequency = 1 and Monetary = 1 then 'Новички с маленьким чеком'
    when Recency = 2 and Frequency = 3 and Monetary = 3 then 'Спящие VIP'
    when Recency = 2 and Frequency = 3 and Monetary = 2 then 'Спящие постоянные со средним чеком'
    when Recency = 2 and Frequency = 3 and Monetary = 1 then 'Спящие постоянные с маленьким чеком'
    when Recency = 2 and Frequency = 2 and Monetary = 3 then 'Спящие редкие с высоким чеком'
    when Recency = 2 and Frequency = 2 and Monetary = 2 then 'Спящие редкие со средним чеком'
    when Recency = 2 and Frequency = 2 and Monetary = 1 then 'Спящие редкие с маленьким чеком'
    when Recency = 2 and Frequency = 1 and Monetary = 3 then 'Спящие разовые'
    when Recency = 2 and Frequency = 1 and Monetary = 2 then 'Спящие разовые'
    when Recency = 2 and Frequency = 1 and Monetary = 1 then 'Спящие разовые'
    when Recency = 1 and Frequency = 3 and Monetary = 3 then 'Уходящие VIP'
    when Recency = 1 and Frequency = 3 and Monetary = 2 then 'Уходящие постоянные'
    when Recency = 1 and Frequency = 3 and Monetary = 1 then 'Уходящие постоянные'
    when Recency = 1 and Frequency = 2 and Monetary = 3 then 'Уходящие редкие'
    when Recency = 1 and Frequency = 2 and Monetary = 2 then 'Уходящие редкие'
    when Recency = 1 and Frequency = 2 and Monetary = 1 then 'Уходящие редкие'
    when Recency = 1 and Frequency = 1 and Monetary = 3 then 'Одноразовые'
    when Recency = 1 and Frequency = 1 and Monetary = 2 then 'Одноразовые'
    when Recency = 1 and Frequency = 1 and Monetary = 1 then 'Потерянные экономные'
end;
```
Можно выделить группы клиентов с различным уровнем лояльности и ценности для бизнеса. Это позволяет разрабатывать более целевые маркетинговые стратегии для каждой группы и повышать их лояльность.

## Построение отчета в Power BI
Был построен отчет в ***Power BI***, содержащий две страницы: ***"Общие сведения"*** и ***"RFM"***.

На странице ***"Общие сведения"*** представлена основная информация по выручке, количеству заказов и проданных товаров. Также показаны графики динамики выручки и заказов по месяцам, тренды по количеству заказов в разрезе дней недели, а также топ продуктов по выручке.

На странице ***"RFM"*** представлена сегментация клиентов по их активности и ценности для бизнеса. В таблице отображены различные сегменты клиентов с указанием их вклада в выручку, количество заказов, проданных товаров и средний чек. Данные представлены таким образом, чтобы легко идентифицировать наиболее значимые группы клиентов и их долю в общем объеме продаж.

Была настроена кроссфильтрация для удобного анализа данных между различными элементами отчета. Кроме того, карточки ***KPI*** отображают актуальные данные по ключевым параметрам в зависимости от выбранного месяца. Эти карточки показывают процентное изменение показателей по сравнению с предыдущим месяцем, что позволяет быстро оценивать динамику изменений и принимать обоснованные решения на основе актуальных данных.
![Online_retail-1](https://github.com/user-attachments/assets/79c27fb3-7558-4768-a659-117e4872524d)
![Online_retail-2](https://github.com/user-attachments/assets/09609759-a5b4-423e-828d-1fb70e3c3fd2)


