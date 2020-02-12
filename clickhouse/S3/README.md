# Работа с сервисом Managed Service for Clickhouse

Мы создадим кластер Clickhouse и попробуем развеять некоторые мифы о разнице погоды Москвы и Питера.
В этом нам поможет история метеонаблюдений за 10 лет, которую мы загрузим в СУБД.
Также в этом варианте будут использоваться только данные из публичного бакета S3.

## Подготовка кластера

* Зайдите в консоль Яндекс.Облака https://console.cloud.yandex.ru
* Cоздайте себе каталог, выбрав опцию создания сети по умолчанию
* Зайдите в созданный каталог
* В задании используется публичный бакет, так что данные уже загружены.

### Создание

* Перейдите на главную страницу сервиса Managed Service for Clickhouse
* Создайте кластер Clickhouse со следующими параметрами:
- Версия: 19.17 или более новая
- Класс хоста - `s2.micro`
- Хосты - добавьте один или два хоста в разных зонах доступности (чтобы суммарно было не менее двух хостов) и укажите необходимость публичного доступа (публичного IP адреса) для них
- Имя пользователя - `user1`
- База данных - `db1`
- Пароль - `CH1-test-123456`

Остальные параметры оставьте по умолчанию, либо измените по своему усмотрению.

* Нажмите кнопку Создать кластер и дождитесь окончания процесса создания (Статус кластера = `RUNNING`). Кластер создается от 5 до 10 минут
* Обратите внимание на калькулятор стоимости услуги и изменение стоимости при изменении параметров кластера.

### Вставка данных

Кликхаус умеет работать с S3 напрямую. Ничего загружать не потребуется!

### Запросы о погоде

Давайте попробуем разобраться как на самом деле обстоят дела с климатом в двух крупнейших мегаполисах России.
Запросы ниже удобнее всего будет делать через веб-консоль: для этого отройте стартовую страницу сервиса, перейдите в ваш кластер и нажмте на вкладку "SQL" (слева).

* Правда ли что в Питере и Москве разный климат? Посмотрим разницу среднегодовой температуры у этих городов.

```sql
SELECT
    Year,
    msk.t - spb.t
FROM
(
    SELECT
        toYear(LocalDate) AS Year,
        avg(TempC) AS t
    FROM s3(
        'https://storage.yandexcloud.net/arhipov/weather_data.tsv',
        'TSV',
        'LocalDateTime DateTime, LocalDate Date, Month Int8, Day Int8, TempC Float32,Pressure Float32, RelHumidity Int32, WindSpeed10MinAvg Int32, VisibilityKm Float32, City String')
    WHERE City = 'Moscow'
    GROUP BY Year
    ORDER BY Year ASC
) AS msk
INNER JOIN
(
    SELECT
        toYear(LocalDate) AS Year,
        avg(TempC) AS t
    FROM s3(
        'https://storage.yandexcloud.net/arhipov/weather_data.tsv',
        'TSV',
        'LocalDateTime DateTime, LocalDate Date, Month Int8, Day Int8, TempC Float32,Pressure Float32, RelHumidity Int32, WindSpeed10MinAvg Int32, VisibilityKm Float32, City String')
    WHERE City = 'Saint-Petersburg'
    GROUP BY Year
    ORDER BY Year ASC
) AS spb USING (Year)
```
Как насчет скорости ветра? Или влажности (Питер все-таки морской город)?
Попрубуйте изменить некоторые поля в запросе, чтобы узнать.

Ветер -- `WindSpeed10MinAvg` (среднее за 10 минут)

Относительная влажность -- `RelHumidity`

* Наверняка самая низкая температура была зарегистрирована в Питере. Давайте убедимся.
```sql
SELECT
    City,
    LocalDate,
    TempC
FROM s3(
        'https://storage.yandexcloud.net/arhipov/weather_data.tsv',
        'TSV',
        'LocalDateTime DateTime, LocalDate Date, Month Int8, Day Int8, TempC Float32,Pressure Float32, RelHumidity Int32, WindSpeed10MinAvg Int32, VisibilityKm Float32, City String')
ORDER BY TempC ASC
LIMIT 1
```
Можно посмотреть также и жару, изменив один параметр.

* Давайте попробуем что-то аналитическое. Например, где раньше начинается лето?
Будем считать, что лето -- это когда в течение 10 дней (`864000` секунд) было более 15 градусов хотя бы три раза.
```sql
SELECT
    City,
    toYear(md) AS year,
    min(md) AS month
FROM
(
    SELECT
        City,
        toYYYYMM(LocalDate) AS ym,
        min(LocalDate) AS md,
        windowFunnel(864000)(LocalDateTime, TempC >= 15, TempC >= 15, TempC >= 15) AS warmdays
    FROM s3(
        'https://storage.yandexcloud.net/arhipov/weather_data.tsv',
        'TSV',
        'LocalDateTime DateTime, LocalDate Date, Month Int8, Day Int8, TempC Float32,Pressure Float32, RelHumidity Int32, WindSpeed10MinAvg Int32, VisibilityKm Float32, City String')
    GROUP BY
        City,
        ym
)
WHERE warmdays = 3
GROUP BY
    year,
    City
ORDER BY
    year ASC,
    City ASC,
    month ASC
```
А когда начинается осень? Поменяйте агрегатную функцию во внешнем `SELECT`, чтобы узнать.

Кстати, при помощи похожего запроса можно [считать](https://clickhouse.yandex/docs/ru/query_language/agg_functions/parametric_functions/#windowfunnel-window-timestamp-cond1-cond2-cond3) конверсию покупок в онлайн-ретейле.


### Удалите кластер

* Удалите кластер выбрав соответствующее действие в UI консоли.
* Обратите внимание, что данные удаляемого кластера можно восстановить в течении 7 дней после удаления, т.к. в течении этого периода сохраняются его резервные копии.
