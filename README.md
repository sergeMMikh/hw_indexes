# Домашнее задание к занятию «SQL. Индексы» - Михалёв Сергей

---

### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

**Решение**

```sql
SELECT SUM(index_length)/SUM(data_length)*100 AS percentage 
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_TYPE = 'BASE TABLE'
AND TABLE_SCHEMA = 'sakila';
```

- результат
  
  <img src="images/Task_1.png" alt="Task_1_.png" width="150" height="auto">

---

### Задание 2

Выполните explain analyze следующего запроса:
```sql
select distinct concat(c.last_name, ' ', c.first_name),
sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30'
and p.payment_date = r.rental_date
and r.customer_id = c.customer_id
and i.inventory_id = r.inventory_id;
```
- перечислите узкие места;
- оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

**Решение.**
**Анализ скрипта**

Результат выполнения скрипта:
  
  <img src="images/Task_2_4.png" alt="Task_2_4.png" width="500" height="auto">

Результат выполнения запроса с EXPLAIN FORMAT = tree
  
  <img src="images/Task_2_1.png" alt="Task_2_1.png" width="750" height="auto">

  Спускаясь по иерархии дерева через Здесь *Nested loop inner join* - вложенные петли внутреннего объеденения таблиц ``from payment p, rental r, customer c, inventory i, film f``` для [агрегирующей оконной функции](https://habr.com/ru/articles/664000/) *SUM* можно определить наиболее тонкие места: 
   - Хэширование *Inner hash join* внутреннее хэширование и поиск по индексу
   - Фильтрация *Filter: (cast(p.payment_date as date)* выполнение условия WHERE и сканирование всей таблицы *payment*

Только поиск в таблице *film* осуществляется по индексам (результат выполнения запроса с EXPLAIN).
  
  <img src="images/Task_2_3.png" alt="Task_2_3.png" width="750" height="auto">

**Оптимизация**

Выборка из таблицы film не вляет на конечный результат и упрощение функции до ```sum(p.amount) over (partition by c.customer_id)``` не меняет итоговой картины. Так же напрямую можно связать платежи с клиентом, уменьшая список таблиц.
  <img src="images/Task_2_5_2.png" alt="Task_2_5_2.png" width="500" height="auto">

Теперь в дереве работы анализа скрипта основной лимитирующий процесс- фильтрация данных по дате покупки: *Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1633 rows=16086)*

<img src="images/Task_2_6.png" alt="Task_2_6.png" width="750" height="auto">

Здесь для оптимизации процесса можно создать индекс: ```create index day_of_payment on payment(payment_date);```

<img src="images/Task_2_7.png" alt="Task_2_7.png" width="500" height="auto">

<img src="images/Task_2_8.png" alt="Task_2_8.png" width="500" height="auto">

Но эффекта это не возымело:

<img src="images/Task_2_9.png" alt="Task_2_9.png" width="750" height="auto">

Далее решил избавится от оконной функции, тем самым упростить сам запрос:

```
select distinct 
concat(c.last_name, ' ', c.first_name), 
sum(p.amount)
from payment p JOIN customer c on p.customer_id  = c.customer_id 
where date(p.payment_date) = '2005-07-30' 
group by p.customer_id;
```
Сам результат выборки не изменился (см. JOHNSON PATRICIA):

<img src="images/Task_2_10.png" alt="Task_2_10.png" width="500" height="auto">

Результаты анализа на этот раз показал значительное ускаорение процесса.

<img src="images/Task_2_11.png" alt="Task_2_11.png" width="750" height="auto">

---

