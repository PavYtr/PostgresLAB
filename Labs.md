## ***Lab 1***

### 1. Вар. 2

```sql
SELECT flight_id FROM bookings.flights
	WHERE arrival_airport = 'SVO' AND 
		actual_arrival BETWEEN '2017-07-22 16:00' AND '2017-07-22 19:00'
		ORDER BY flight_id; 
```

### 2. Вар. 2

Используем `UPDATE` для обновления значений таблицы 

```sql
UPDATE ticket_flights 
	SET amount = amount * 1.1
	WHERE amount < 10000 AND fare_conditions = 'Business';
```

### 3. Вар. 4
- `DISTINCT` - чтобы каждый день вывелся один раз
- `LEFT` - срез `timestamp` переведенного в `text` так чтобы остались только год, месяц и день
-  Применение функции `count(*)` ко всем строкам где обрезанный `scheduled departure` одинаковый через оконную функцию
- `ORDER BY total_daily` - вывод строк в порядке возрастания для красоты

```sql
SELECT DISTINCT
	LEFT(scheduled_departure::text, 10) AS date,
	count(*) OVER(
		PARTITION BY LEFT(scheduled_departure::text, 10)
	) as total_daily

	FROM flights
	WHERE scheduled_departure >= '2017-08-01' 
    	AND scheduled_departure < '2017-09-01'
		AND status = 'Delayed'
	ORDER BY total_daily;
```

## ***Lab 2***

При помощи подзапроса `dist` считается дальность каждого полета, скорость каждого самолета берется за константу - **900** км/ч. `EXTRACT EPOCH` преобразует полную дату в секунды, которые далее преобразуются в часы и умножаются на скорость. Кроме того исключаются те строки с `NULL` в `actual_arrival` и `actual_departure`.


Подзапрос `money_for_flight` при помощи группировки по `flight_id` и `fare_conditions` считает цену для каждого класса обслуживания для каждого рейса. 
В основном запросе считаем средний коэффициент цены к дальности для пассажира и для авиакомпании. При помощи `JOIN`-ов присоединяtм подзапросы и таблицу с самолетами. Группируем все по модели и классу обсуживания.


Для большей наглядности ограничиваем количество цифр после запятой при помощи приведения к типу `numeric`.


По итогу самый выгодный для пассажиров - _**Сессна 208 Караван**_, а для авиакомпаний - _**Боинг 777-300**_
 

```sql
WITH dist AS (
    SELECT flight_id, aircraft_code, 
        (EXTRACT (EPOCH FROM (actual_arrival - actual_departure)) / 3600) * 900 AS distance
        FROM bookings.flights
        WHERE actual_departure IS NOT NULL AND actual_arrival IS NOT NULL
),
money_for_flight AS (
    SELECT flight_id, fare_conditions, SUM(amount) as total_money
        FROM bookings.ticket_flights
        GROUP BY flight_id, fare_conditions
)
SELECT model, ticket_flights.fare_conditions,
    AVG(amount / distance)::numeric(7, 2) as passenger_cost,
    AVG(total_money / distance)::numeric(7, 2) as airline_cost
    FROM bookings.ticket_flights
	    JOIN dist ON ticket_flights.flight_id = dist.flight_id
	    JOIN aircrafts_data ON dist.aircraft_code = aircrafts_data.aircraft_code
	    JOIN money_for_flight ON ticket_flights.flight_id = money_for_flight.flight_id AND
	        ticket_flights.fare_conditions = money_for_flight.fare_conditions
    	GROUP BY model, ticket_flights.fare_conditions;

```
