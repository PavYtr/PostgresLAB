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

Сначала при помощи подзапроса `amount_for_aircrafts` посчитаем среднюю стоимость билета на каждый класс обслуживания для каждого самолета при помощи `AVG` для всех строк с одинаковыми `aircraft_code` и `fare_conditions`. Такое возможно если при помощи `JOIN` соединить таблицы `flights` и `ticket_flights` по общему столбцу `flight_id` и применить группировку `GROUP BY` по `aircraft_code` и `fare_conditions`.

Для нахождения стоимости топлива на место за километр пути используем второй подзапрос `fuel_per_seat`

Далее делаем основной запрос из `aircrafts_data` откуда берем модель самолета и его дальность полёта для подсчёта отношения денег за билет на километр. При помощи `JOIN` добываем среднюю стоимость каждого класса для каждого самолета и сортируем всё это по данному отношению в порядке возрастания. 

Таким образом, самый выгодный вариант для пассажира - билет **эконом класса в Боинге 777-300**, так как для пассажира не имеет значения стоимость топлива, то ему нужно знать лишь отношение цены билета на километр пути

А для авиакомпании имеет значение такой показатель, как (цена билета за километр - цена (топлива на километр / число пассажиров этого класса)). Возьмем среднее значение стоимости топлива на километр пути для этих самолетов равным **150**руб. Тогда получим, что самый выгодный полет для авиалиний будет **билет бизнес класса на Аэробусе А319-100**


```sql
WITH ticket_per_km AS (
	SELECT aircraft_code, fare_conditions,
		AVG(amount) as money_for_ticket 
			FROM flights
			JOIN ticket_flights ON flights.flight_id = ticket_flights.flight_id
			GROUP BY aircraft_code, fare_conditions
			ORDER BY aircraft_code
),
fuel_per_seat AS (
	SELECT seats.aircraft_code, fare_conditions, 
			(150 / (count(seats.seat_no))) as money_for_fuel
		FROM aircrafts_data
			JOIN seats ON aircrafts_data.aircraft_code = seats.aircraft_code	
			GROUP BY seats.fare_conditions, seats.aircraft_code
)
SELECT DISTINCT model, ticket_per_km.fare_conditions, 
				(money_for_ticket / aircrafts_data.range)::numeric(14, 2) as ticket_per_km_amount,
				(money_for_ticket / aircrafts_data.range - money_for_fuel)::numeric(14, 2) as money_for_airlines	
	FROM aircrafts_data
		JOIN ticket_per_km ON aircrafts_data.aircraft_code = ticket_per_km.aircraft_code
		JOIN fuel_per_seat ON aircrafts_data.aircraft_code = fuel_per_seat.aircraft_code
			AND ticket_per_km.fare_conditions = fuel_per_seat.fare_conditions
		ORDER BY ticket_per_km_amount;
```
