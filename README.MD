# Огромный и страшный код
Снизу описание для людей, которые работают с БД в docker - там есть забавный момент.

Для работы кода, описанного тут необходимо, чтобы в таблице (seats)
появилась дополнительная колонка extra_space типа данных bool и с дефолтным
значением false.

Сделать это можно чем-то вроде:

``` sql
alter table seats
    add extra_space boolean default false not null;
```

Далее запрос в бд становится тревиальным, правда содержит в себе небольшую
кучу inner join. Важные свойства, которые были установлены при работе:
* На одинаковых рейсах (по номеру) всегда одинаковые самолеты
* Цена на один и тот же рейс НЕ зависят от дня недели
* Сиденья "разной" стоимости содержаться только в эконом классе
Если дальше у вас не появилось желания посидеть над задачей, то ответ содержится в конце файла


## Если работаете с бд в docker
Так как postgres любит shared memory, а в docker строгие лимиты на него,
то в docker-compose.yaml файл надо добавить свойство, которое описано ниже.
Его же аналог есть и для БД, которая была просто запущена через docker.

```yaml
version: "3.9"

services:
  db:
    image: postgres
    volumes:
      - ./database:/var/lib/postgresql/data
      - ./demo.sql:/demo.sql
    environment:
      - POSTGRES_DB=root
      - POSTGRES_USER=root
      - POSTGRES_PASSWORD=root
    ports: ["5432:5432"]
    networks:
      - postgres
    restart: no
    shm_size: 1G # HERE
```

## SQL ответ
Возвращает стоимость полета на рейсе в зависимости от его номера, класса обслуживания и extra_space
(таблицей для всех полетов)

Запрос имеет выражение HAVING сделанное специально для подтверждения данных, которые получены в результате запроса.
Он работает согласно очень простого свойства множеств: если MIN и MAX элементов множества равны, то множество состоит из
одного или одинаковых элементов. 
Если вы хотите убедиться, что все работает корректно, то просто раскомментируйте выражение 
HAVING и убедитесь, что после этого вывод запроса пустой.

```postgresql
SELECT flight_no,
       s.fare_conditions,
       extra_space,
       (array_agg(DISTINCT amount))[1] as price
FROM ticket_flights
         INNER JOIN flights f on f.flight_id = ticket_flights.flight_id AND status = 'Arrived'
         INNER JOIN boarding_passes bp on ticket_flights.ticket_no = bp.ticket_no AND
                                          ticket_flights.flight_id = bp.flight_id
         INNER JOIN seats s on bp.seat_no = s.seat_no AND
                               f.aircraft_code = s.aircraft_code
GROUP BY flight_no,
         s.fare_conditions,
         extra_space
-- HAVING MIN(amount) != MAX(amount)
ORDER BY flight_no, fare_conditions, extra_space
```
