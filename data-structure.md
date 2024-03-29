## Введение
ПЛК отправляет POST запросы на сервер с передачей данных в формате JSON, получает на запрос ответ.
Числа в ответе указывают время последнего обновления регистров на сервере. Порядковый номер соответствует числам “Target”. 
```javascript
{
    "Success": "1706447747:1706447747:1706447747:1706447747",
    "TankStatus": 0
}
```

## 1. Транзакции выдачи нефтепродукта
Каждый из множества ПЛК отправляет информацию о выполненной транзакции сразу после её завершения.
#### Пример запроса:
```javascript
{
    "GetFuel": {
        "ID_TRK": 14, // уникальный номер ТРК
        "startTM": "0", // время старта налива
        "stopTM": "0", // время завершения налива
        "OID": 0, // ID оператора - уникальный номер RFID-карты
        "doseRef": 0, // заданная доза
        "doseIss": 0, // выданная доза
        "errCode": 2 // код ошибки / предупреждений
    }
}
```
#### Ответ:
```javascript
{
    "GetFuel": 0,
    "Success": "1706447747: 1706447747: 1706447747: 1706447747"
}
```

## 2. Реестр заправляемой техники / пользователей
Эти данные ПЛК использует в том числе для вывода на HMI. Пользователь перед заправкой должен 
иметь возможность сверить данные и остановить процесс, если что-то указано неверно.
#### Пример запроса:
```javascript
{
    "Target": 1, // порядковый номер раздела в БД
    "Begin": 18,
    "End": 18
} // запрос 18-ой структуры
```
#### Ответ: 
```javascript
{
    "Body": {
        "18": { // номер структуры данных
            "Action": 2, // уровень доступа: ОПЕРАТОР_ЗАПРАВКИ / ОПЕРАТОР_ПРИХОДА / АДМИН
            "Board_Plate": 666, // бортовой номер техники
            "Company_Name": "OOO \"Region Trejd\" ", // наименование организации
            "Day_Limit": 9999, // суточный лимит
            "ID_Company": 2, // номер организации
            "ID_Operator": 822527997 // ID оператора (RFID-карта)
        }
    },
    "Target": "Register_Of_Vehicle"    //
}
```

## 3. Регистр заблокированных организаций
При наступлении некоторых событий (превышение группового лимита, невыполнение оплаты и пр.) 
серверное ПО может сгенерировать запрет на выдачу топлива указанным организациям.
#### Пример запроса:
```javascript
{
    "Target": 4, // порядковый номер в БД
    "Begin": 273,
    "End": 279
} // запрос с 273 по 279 структуру
```
*Для чего запрашивается только часть данных? Почему нельзя запросить весь раздел БД, содержащий инфу о текущих организациях?*
#### Ответ:
```javascript
{
    "Body": {
        "273": {
            "ID_Company": 273, // ID организации
            "Permission": 0 // разрешение: 0 - нет, 1 - да
        },
        "274": {
            "ID_Company": 274,
            "Permission": 1
        },
        "275": {
            "ID_Company": 275,
            "Permission": 1
        },
        "276": {
            "ID_Company": 276,
            "Permission": 1
        },
        "277": {
            "ID_Company": 277,
            "Permission": 0
        },
        "278": {
            "ID_Company": 278,
            "Permission": 0
        },
        "279": {
            "ID_Company": 279,
            "Permission": 1
        }
    },
    "Target": "Register_Blocked_Company"
}
```

## 4. Конфигурация ТРК
#### Пример запроса: 
```javascript
{
    "Target": 3, //адрес раздела БД, с конфигами всех ТРК
    "Begin": 8, // структура, соответствующая конкретной ТРК
    "End": 8
}
```
#### Ответ: 
```javascript
{
    "Body": {
        "8": {
            "Acc_Unit": 1, // единица измерения: 0 - л, 1 - кг
            "Adj_K": 310, //  юстировочный коэффициент
            "Counter_Type": 7, //  тип счётчика
            "ID_TRK": 12, //    уникальный ID ТРК
            "Locked": 1, // заблокирована ТРК или нет
            "Max_Limit_Issue": 1700, // максимальная доза
            "Min_Level_Fuel": 3500, //  уровень топлива в резервуаре, ниже которого блокируется выдача
            "Type_TRK": 7 //    тип ТРК
        }
    },
    "Target": "Configuration_TRK"
}
```
<hr>

### *Разделы 5, 6 - новые. На данный момент никак не реализованы.*
<hr>

## 5. Старт налива:
ПЛК с заданной периодичностью опрашивает сервер на предмет необходимости запуска налива.
При подтверждении команды на запуск, ТРК переходит из состояния В_ОЖИДАНИИ в ПОДГОТОВКА.
При этом ПЛК принимает в ответ на запрос данные:
- ID оператора, отдавшего команду;
- заданную дозу;
- наименование нефтепродукта / № резервуара;

Переход из состояния "ПОДГОТОВКА" осуществляется ПЛК после нажатия "полевым" оператором аппаратной кнопки ПУСК. Усложнение в целях повышения безопасности.

Во время налива, ПЛК с заданной периодичностью передаёт на сервер инфу о состоянии ТРК и объём выданного нефтепродукта на данный момент.

 ## 6. Состояние ТРК:
Статусы:
- В_ОЖИДАНИИ;
- ПОДГОТОВКА (идентификация пользователя, ввод дозы, запуск налива в случае запуска с полевого терминала (HMI));
- НАЛИВ;
- ЗАВЕРШЕНИЕ_НАЛИВА (ТРК выводит сообщение о количестве выданного нефтепродукта, выдерживает паузу/ждёт нажатия кнопки или сброса с ПО верхнего уровня);
- ОШИБКА;
<hr>

### *Разделы ниже НЕактуальны для пилотного проекта - "Модернизация нефтебазы СтрежТрансСервис"*
<hr>


## 7. Состояние резервуара
Каждый ПЛК в режиме ожидания отправляет запросы с заданой периодичностью (например, каждые 30 секунд).
#### Пример запроса: 
```javascript
{
    "TankStatus": {
        "ID_TRK": 100, // уникальный номер ТРК
        "liqLvl": 0, // уровень жидкости
        "fDens": 30010, // плотность нефтепродукта
        "liqTemp": 25, // температура жидкости
        "fVlm": 116660, // объём нефтепродукта
        "wVlm": 59545, // объём подтоварной воды
        "ttlCap": 58725 // общий объём жидкости
    }
}
```
#### Ответ: 
```javascript
{
    "Success": "1706447747: 1706447747: 1706447747: 1706447747",
    "TankStatus": 0
}
```

## 8. Транзакции прихода нефтепродукта
Каждый из множества ПЛК отправляет информацию о выполненной транзакции сразу после её завершения. 
Во время пополнения резервуара (прихода топлива) выдача топлива не производятся, т.е. одновременно 
с одного ПЛК могут генерироваться либо транзакции выдачи/налива, либо прихода/пополнения.
#### Пример запроса: 
```javascript
{
    "SetFuel": {
        "ID_TRK": 65501,    // 
        "startTM": "1706457815",    // время старта прихода нефтепродукта
        "stopTM": "1706457815",    // время завершения
        "oID": 39306,    // ID оператора прихода
        "vlmTTN": 0,    // объём по товарно-транспортной накладной (ТТН)
        "vlmLvl": 50000,    // объём по уровнемеру
        "fId": 0    // вид топлива (выбирает оператор из списка)
    }
}
```
#### Ответ: 
```javascript
{
    "SetFuel": 0,
    "Success": "1706447747: 1706447747: 1706447747: 1706447747"
}
```

## 9. Реестр видов нефтепродукта
#### Пример запроса: 
```javascript
{
    "Target": 2,
    "Begin": 3,
    "End": 5
} // запрос с 3 по 5 структуры
```
#### Ответ: 
```javascript
{
    "Body": {
        "3": {
            "Fuel_Name": "DT (zimnee)",
            "ID_Fuel": 3,
            "PTF": -32 // предельная температура фильтрации - ПЛК сможет сравнивать текущую температуру топлива с ПТФ и выдавать предупреждения
        },
        "4": {
            "Fuel_Name": "DT (arkticheskoe)",
            "ID_Fuel": 4,
            "PTF": -48
        },
        "5": {
            "Fuel_Name": "AI-95",
            "ID_Fuel": 5,
            "PTF": -60
        }
    },
    "Target": "Register_Type_Of_Fuel"
}
```

