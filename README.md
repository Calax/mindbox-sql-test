# Mindbox interview task 2: SQL
**Этот репозиторий будет удален сразу после получения обратной связи от Mindbox.**

## 1. Покажи результат. 
Ссылка на решение в sqlfiddle: [http://sqlfiddle.com/#!18/be483/5](http://sqlfiddle.com/#!18/be483/5)


## 2. Создадим простейшую структуру из двух таблиц
```sql
CREATE TABLE Clients (
    Id INT NOT NULL IDENTITY(1,1) PRIMARY KEY,  -- Просто ID.
    Name NVARCHAR(255) NOT NULL,                -- Должны же быть хоть какие-то опознавательные данные.
    RegistrationTime DATETIME2 NOT NULL         -- От этой даты будем отсчитывать.
)
GO

CREATE TABLE Orders (
    Id BIGINT NOT NULL IDENTITY(1,1) PRIMARY KEY,   -- Просто  ID.
    RegistrationTime DATETIME2 NOT NULL,            -- Дата совершения покупки.
    ClientId INT NOT NULL,                          -- ID клиента.
    CONSTRAINT FK_Clients_Id FOREIGN KEY (ClientId) REFERENCES Clients (Id) -- FK на клиентов. Все по-честному.
)
GO
```


## 3. Наполняем тестовыми данными.
Вставляем двух клиентов. Подразумеваем, что все заказы сделаны сегодня (через getdate).
Создаем двух клиентов: для первого условие 5 дней после регистрации не будет выполняться (FalseClient), а для второго будет (TrueClient).

```sql
DECLARE @FalseClientId INT = 1
DECLARE @TrueClientId INT = 2

--Вставляем клиентов.
SET IDENTITY_INSERT Clients ON --Чтобы при вставке заказов не гадать, у какого клиента какой id. Нам потом на них ссылаться.
INSERT INTO Clients(Id, Name, RegistrationTime)
VALUES
    (@FalseClientId, N'FalseClient', DATEADD(dd, -7, GETDATE())),
    (@TrueClient, N'TrueClient', DATEADD(dd, -4, GETDATE))
SET IDENTITY_INSERT Clients OFF

--Вставляем заказы.
INSERT INTO Orders(RegistrationTime, ClientId)
VALUES
    (getdate(), @FalseClientId),
    (getdate(), @FalseClientId),
    (getdate(), @TrueClientId),
    (getdate(), @TrueClientId)
```

## 4. Сам запрос.
Запрос написан строго в соответствии с требованиями к нему:
1. Смотрим дату не любого заказа, а именно первого.
1. Считаем количество.

```sql
DECLARE @daysAfterClientRegistration INT = 5
;WITH firstOrders AS (
    SELECT *
    FROM (
        SELECT 
            o.ClientId AS ClientId,
            o.RegistrationTime AS RegistrationTime,
            row_number() over(partition by ClientId ORDER BY RegistrationTime ASC) AS OrderRank
        FROM [Orders] AS o
    ) AS t
    WHERE OrderRank = 1
)
SELECT COUNT(1)
FROM [Clients] AS c
    INNER JOIN firstOrders AS fo
        ON c.id = fo.ClientId
WHERE fo.RegistrationTime < DATEADD(dd, @daysAfterClientRegistration, c.RegistrationTime)
```

## 5. Рассуждения на тему.
### 5.1. Нет смысла счтать разницу именно с первым заказом. Логично предположить, второй будет после него. Поэтому запрос можно упростить:
```sql
DECLARE @daysAfterClientRegistration INT = 5

SELECT COUNT(1)
FROM [Clients] AS c
WHERE EXISTS (
    SELECT 1/0
    FROM [Orders]
    WHERE ClientId = c.Id AND RegistrationTime < DATEADD(dd, @daysAfterClientRegistration, c.RegistrationTime)
)
```
Вкусовщина, но я предпочту именно вариант с EXISTS, а не join с последующим count(distinct id).

### 5.2. Дата регистрации в таблице клиентов.
Если регистрация одна и не изменяется, обычно не доставляет каких-либо трудностей. Если регистрация обрастает своими доп.полями и мешает (надо поднимать много столбцов и т.д., лишний код при разработке), стоит вынести отдельно.

### 5.3. Отсутствие аудита.
Для расследования инцидентов стоило бы завести отдельно аудит. Кто-то может удалить клиента и клиент не получит свою заслуженную булочку - будет недоволен. Бизнес потребует найти и наказать.

### 5.4. Read-блокировка и конфликты с пишущим трафиком.
Если есть трафик на обновление данных, то пишущий трафик может блокировать читающий. А может не блокировать - зависит от настроек изоляции на БД. Если при чтении устроит получение предыдущий версии (а с точки зрения данной задачи, кажется, что вполне устроит), то для избежания блокировок стоит включить snapshot_isolataion и read_comitted_snapshot (предполагается, что используется уровень изоляции read_committed).

### 5.5. Аналитические запросы на транзакцинной базе.
- Запрос выглядит аналитическим. Такие лучше запускать не по оперативной базе, а по специально созданной аналитической БД. В зависимости от цели, назначения, имеющихся ресурсов и пр. - от read-only реплики до аналитических БД.
- Стоит рассмотреть вариант преобразования данных для аналитических целей. В текущей реализации невозможно построить эффективный индекс из-за необходимости вычислять dateadd.