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

**Решение**

Результат выполнения скрипта:
  
  <img src="images/Task_2_4.png" alt="Task_2_4.png" width="500" height="auto">

Результат выполнения запроса с EXPLAIN FORMAT = tree
  
  <img src="images/Task_2_1.png" alt="Task_2_1.png" width="750" height="auto">

  Здесь **наиболее** тонкие места: 
   - *Nested loop inner join* - вложенные петли внутреннего объеденения таблиц, что возможно соответствует переборке таблиц ```from payment p, rental r, customer c, inventory i, film f``` для [агрегирующей оконной функции](https://habr.com/ru/articles/664000/) *SUM*
   - Хэширование *Inner hash join* внутреннее хэширование и поиск по индексу
   - Фильтрация *Filter: (cast(p.payment_date as date)* выполнение условия WHERE и сканирование всей таблицы *payment*
   - Поиск по *rental_date* в таблице *rental*

Только поиск в таблице *film* осуществляется по индексам (результат выполнения запроса с EXPLAIN).
  
  <img src="images/Task_2_3.png" alt="Task_2_3.png" width="750" height="auto">

 Однако выборка из таблицы film не вляет на конечный результат и упрощение функции до ```sum(p.amount) over (partition by c.customer_id)``` не меняет результата. Назовём это певым упрощением скрипта:

  <img src="images/Task_2_5_.png" alt="Task_2_5_.png" width="500" height="auto">

```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```

- результат
  
  <img src="images/Task_1_.png" alt="Task_1_.png" width="750" height="auto">

---

