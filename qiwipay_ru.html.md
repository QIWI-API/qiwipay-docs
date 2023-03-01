---
title: QIWI PAY Reference

metatitle: QIWI PAY Reference

metadescription: Интерфейс QIWI PAY предназначен для обслуживания операций по банковским картам. Сервис позволяет ТСП принимать безопасные платежи по картам от клиентов.

category: acquiring

language_tabs:
  - json: JSON
  - php: PHP
  - java: Java
  - html: HTML
  - shell

toc_footers:
  - <a href='/'>На главную</a>
  - <a href='mailto:bss@qiwi.com'>Обратная связь</a>

includes:

search: true
---

 *[WPF]: Web Payment Form — платежная форма для ввода реквизитов карты при оплате продукта/услуги.
 *[3DS]: 3-D Secure — протокол защиты, используемый для аутентификации держателя банковской карты во время совершения платежной операции посредством сети Интернет.
 *[OCT]: Original Credit Transaction — тип транзакции, когда карточный счет кредитуется со счета мерчанта.
 *[API]: Application Programming Interface —  набор готовых методов, предоставляемых приложением системой для использования во внешних программных продуктах.
 *[HMAC]: Hash-based message authentication code — код проверки подлинности сообщений, использующий хеш-функции.
 *[ТСП]: Торгово-сервисное предприятие
 *[PCI DSS]: Payment Card Industry Data Security Standard —  стандарт безопасности данных индустрии платёжных карт, учреждённым международными платёжными системами Visa, MasterCard, American Express, JCB и Discover.
 *[REST]: Representational State Transfer —  архитектурный стиль взаимодействия компонентов распределённого приложения в сети.
 *[JSON]: JavaScript Object Notation — текстовый формат обмена данными, основанный на JavaScript.
 *[Luhn]: Алгоритм Луна — алгоритм вычисления контрольной цифры номера пластиковой карты, предназначен для выявления ошибок, вызванных непреднамеренным искажением данных.
 *[МПС]: Международные платежные системы: Visa, MasterCard, МИР, AMEX, China UnionPay и т.д.
 *[PKCS#10]: Стандарт запроса на сертификат согласно RFC2986.
 *[HTTPS]: Расширение протокола **HTTP** для поддержки шифрования в целях повышения безопасности. Данные в протоколе HTTPS передаются поверх криптографических протоколов SSL или TLS. В отличие от HTTP с TCP-портом 80, для HTTPS по умолчанию используется TCP-порт 443.

# Введение {#start}

###### Последнее обновление: 2022-11-02 | [Редактировать на GitHub](https://github.com/QIWI-API/qiwipay-docs/blob/master/qiwipay_ru.html.md)

Сервис QIWI PAY предназначен для обслуживания операций по банковским картам. Сервис позволяет ТСП принимать безопасные платежи по картам от клиентов.

Сервис поддерживает операции выполнения платежа, подтверждения платежей, выполненных по двухшаговому сценарию, отмены платежа, возврата денежных средств, а также получения статуса операции.

# Способы взаимодействия {#ways}

Существует два способа работы с QIWI PAY:

* QIWI PAY WPF — платежная форма на стороне QIWI. Не требует сложной реализации, но ограничена по функциональности (поддерживает только операции платежа).

* QIWI PAY API — полнофункциональное API для платежных операций. API использует REST-архитектуру. Параметры передаются методом POST в теле запроса в формате JSON, ответы на запросы также в формате JSON. При этом платежная форма для ввода реквизитов карты реализуется на стороне ТСП.

<aside class="success">
Выполнять запросы через API, в которых передаются полные номера карт, допускается только в том случае, если ТСП имеет PCI DSS сертификат, т.к. в этом случае принимает и обрабатывает на своей стороне карточные данные.
</aside>

## URL сервисов оплаты {#urls}

* Для работы с QIWI PAY WPF необходимо перенаправить покупателя на URL:

`https://pay.qiwi.com/paypage/initial`.

* Для работы с QIWI PAY API необходимо отправлять HTTP POST запросы на URL:  

`https://acquiring.qiwi.com/merchant/direct`.

# Возможные операции {#operations}

В каждом запросе ТСП к API или при загрузке платежной формы WPF в параметре `opcode` должен указываться код операции. Операция определяет, какое именно действие должно быть выполнено.

Список доступных операций для каждого способа взаимодействия с QIWI PAY приведен в таблице.

Код операции | QIWI PAY WPF | QIWI PAY API | Операция | Финансовая | Описание
------------ | ----------- | ----------- | -------- | ---------- | --------
1 | + | + | sale | Да | Одношаговый сценарий оплаты
2 | - | + | finish_3ds | Зависит от сценария | Возврат в систему после 3DS аутентификации
3 | + | + | auth | Нет | Авторизационный запрос (холдирование средств) в случае двухшагового сценария оплаты
5 | - | + | capture | Да | Подтверждение авторизации в случае двухшагового сценария оплаты
6 | - | + | reversal | Нет | Отмена платежа (средства расхолдируются практически сразу)
7 | - | + | refund | Да | Возврат платежа (средства возвращаются в течение 30 дней)
20 | - | + | payout | Да | Операция выплаты (OCT)
30 | - | + | status | Нет | Запрос статуса операции
40 | - | + | get_cards_by_token | нет | Получение списка привязанных карт

<aside class="notice">
Финансовая операция означает, что по результатам данной операции будет произведено движение средств по счетам.
</aside>

# Типы операций покупки {#sale_types}

## Методы проведения оплаты {#methods}

Возможны два сценария платежа:

* [Одношаговый](#onestep) — операция `sale`
* [Двухшаговый](#twostep) — операция `auth` -> операция `capture`

Как правило, двухшаговый сценарий используется в том случае, когда ТСП проводит проверку возможности оказания услуги после факта оплаты. Т.к. после операции `auth` и до совершения операции `capture` можно сделать операцию `reversal`, которая не является финансовой.

Для операции `sale` также можно делать операцию `reversal`, но только до конца дня и не для всех банков-эквайеров. Подробности надо уточнять у вашего сопровождающего менеджера.

Чтобы точно понимать, какой тип операции можно выполнять для транзакции, необходимо запросить ее статус и действовать в соответствии с [таблицей статусов](#txn_status).

<aside class="warning">
Максимально возможный период между операциями <code>auth</code> и <code>capture</code> составляет 72 часа. Таким образом, если после операции <code>auth</code> ТСП не присылает операцию <code>capture</code> в течение 72 часов, она выполнится автоматически.
</aside>

С точки зрения наличия полей в запросе, все операции идентичны. Отличаются лишь [коды операций](#operations).

## Технология 3DS {#threeds}

3DS (3-D Secure) — общее название программ Verified By Visa и MasterCard Secure Code от платежных систем Visa и MasterCard соответственно. Суть программы в проверке подлинности держателя (то есть защита от несанкционированного использования карты) эмитентом перед оплатой. На практике это выглядит так: держатель указывает реквизиты карты, далее открывается сайт эмитента, где держателю предлагается ввести пароль или секретный код (как правило, код отправляется в СМС сообщении). Если код указан правильно, оплата будет проведена. Если нет — отклонена.

Операция покупки может быть проведена через QIWI PAY с использованием технологии 3DS, если по карте необходима 3DS-аутентификация.

Диаграмма функционирования 3DS на примере операции с использованием способа QIWI PAY WPF.

<div class="mermaid">
sequenceDiagram
participant Customer
participant Issuer
participant ips as Visa/MC
participant qp as QIWI PAY
participant qb as QIWI Bank
Note over Customer,Issuer: Issuer Domain
Note over ips: Interaction Domain
Note over qp,qb: Acquirer Domain
Customer->>qp:Отправка данных карты<br>с платежной формы
activate Customer
activate qp
qp->>ips:Отправка VEReq<br>для проверки карты на участие в<br>программе 3-D Secure
activate ips
ips->>Issuer:Запрос в ACS<br>для проверки карты
activate Issuer
Issuer-->>ips:Возврат результата проверки (VERes)
deactivate Issuer
ips-->>qp:Возврат результата<br>проверки (VERes)
deactivate ips
qp->>qp:Генерация PAReq<br>на основании VERes
qp-->>Customer:Перенаправление плательщика с PAReq на ACS URL<br>из VERes для ввода пароля 3-D Secure
deactivate qp
deactivate Customer
Customer->>Issuer:Переход плательщика на ACS URL
activate Customer
activate Issuer
Issuer-->>Customer:Отправка одноразового<br>пароля в SMS
Customer->>Issuer:Ввод одноразового пароля
Issuer->>ips:Отправка данных о результате<br>аутентификации в ACS
Issuer-->>Customer:Перенаправление плательщика на<br>страницу QIWI PAY с PARes
deactivate Customer
deactivate Issuer
Customer->>qp:Переход плательщика на QIWI PAY page
activate Customer
activate qp
qp->>qp:Проверка валидности PARes
qp->>qb:Отправка<br>авторизационного запроса<br>с данными из PARes
deactivate qp
deactivate Customer
</div>

# QIWI PAY WPF {#wpf}

Клиенту отображается платежная форма для ввода реквизитов карты на стороне QIWI PAY.

ТСП достаточно просто перенаправить клиента на платежную форму.

Схема одношагового сценария платежа с использованием платежной формы (WPF) на стороне QIWI PAY.

<aside class="notice">
При использовании QIWI PAY WPF для двухшагового сценария выполняется только первая операция (<code>auth</code>).
</aside>

<div class="mermaid">
sequenceDiagram 
participant Customer
participant Merchant
participant QiwiPay as QIWI PAY
participant QiwiBank as QIWI Bank
Customer->>Merchant:Оформление заказа,<br>предложение оплатить<br>с помощью пластиковой карты
activate Merchant
Merchant-->>Customer:Перенаправление на<br>платежную форму
deactivate Merchant
Customer->>QiwiPay:Запрос на показ<br>платежной формы
activate QiwiPay
QiwiPay-->>Customer:Отображение платежной формы
deactivate QiwiPay
Customer->>QiwiPay:Ввод данных карты и отправка формы
activate QiwiPay
QiwiPay->>QiwiPay:Первичная валидация<br>входных параметров,<br>создание транзакции
QiwiPay->>QiwiPay:Модуль борьбы с мошенничеством
rect rgb(255, 238, 223)
Note over Customer, QiwiPay:3-D Secure
QiwiPay->>QiwiPay:Проверяем карту<br>на участие в 3-D Secure
QiwiPay-->>Customer:Перенаправление плательщика на ACS URL
deactivate QiwiPay
Customer->>QiwiPay:Возврат с ACS с результатом верификации
activate QiwiPay
end
QiwiPay->>QiwiBank:Авторизационный запрос
activate QiwiBank
QiwiBank-->>QiwiPay:Успешная авторизация
deactivate QiwiBank
rect rgb(255, 238, 223)
Note over Customer, QiwiPay:Немедленное уведомление об операции
QiwiPay->>Merchant:Уведомление о результате<br>операции (callback)
activate Merchant
Merchant-->>QiwiPay:Подтверждение получения<br>уведомления
deactivate Merchant
end
QiwiPay->>Customer:Страница статуса<br>с результатом операции
QiwiPay->>Customer:E-mail уведомление о результате операции
deactivate QiwiPay
Customer->>Merchant:Возврат в магазин<br>со страницы статуса
</div>

## Перенаправление на платежную форму {#redirect}

~~~ html
<form method="post" action="https://pay.qiwi.com/paypage/initial">
  <input name="opcode" type="hidden" value="1">
  <input name="merchant_site" type="hidden" value="99">
  <input name="currency" type="hidden" value="643">
  <input name="sign" type="hidden" value="bb5c48ea540035e6b7c03c8184f74f09d26e9286a9b8f34b236b1bf2587e4268">
  <input name="amount" type="text">
  <input type="submit" value="Оплатить">
</form>
~~~

Все операции платежа (`sale`, `auth`) используют один и тот же набор параметров.

Параметр|Условие|Тип данных|Описание
--------|-------|----------|--------
opcode|Обязательно|integer|Код операции (только `1`, `3`)
merchant_site|Обязательно|integer|Идентификатор сайта ТСП
currency|Обязательно|integer|Валюта суммы операции в цифровом формате согласно ISO 4217
sign|Обязательно|string(64)|[Контрольная сумма переданных параметров](#sign)
amount|Опционально|decimal|Сумма операции
order_id|Опционально|string(256)|Уникальный номер заказа в системе ТСП
email|Опционально|string(64)|E-mail Покупателя
country|Опционально|string(3)|Страна Покупателя в формате 3-х буквенных кодов согласно ISO 3166-1
city|Опционально|string(64)|Город местонахождения Покупателя
region|Опционально|string(6)|Регион страны формате геокодов согласно ISO 3166-2
address|Опционально|string(64)|Адрес местонахождения Покупателя
phone|Опционально|string(15)|Контактный телефон Покупателя
cf1|Опционально|string(256)|Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf2|Опционально|string(256)|Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf3|Опционально|string(256)|Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf4|Опционально|string(256)|Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf5|Опционально|string(256)|Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
product_name|Опционально|string(25)|Описание услуги, которую получает Плательщик.
merchant_cheque|Опционально|string(4096)|[Данные для кассового чека по 54-ФЗ](#cheque)
merchant_uid|Опционально|string(64)|Уникальный идентификатор Покупателя в системе ТСП.
card_token |Опционально|string(40)|Токен карты
order_expire|Опционально|YYYY-MM-DDThh:mm:ss±hh:mm|Время истечения заказа в формате ISO8601 с временной зоной
callback_url|Опционально|string(256)|URL отправки [callback](#callback)
success_url|Опционально|string(256)|URL для перенаправления Покупателя в случае успешной оплаты
decline_url|Опционально|string(256)|URL для перенаправления Покупателя в случае не успешной оплаты

# QIWI PAY API {#api}

## Авторизация

Для передачи запросов в QIWI PAY API требуется авторизация. Авторизация выполняется методом валидации клиентского сертификата, который выдается ТСП и должен использоваться при каждом запросе к API.

<aside class="success">
Для получения сертификата необходимо обратиться в службу технической поддержки QIWI, предоставив запрос на сертификат (.csr) в формате PKCS#10.
</aside>

## Реализация сценариев платежа {#scenario}

### Схема одношагового сценария платежа {#onestep}

<div class="mermaid">
sequenceDiagram
participant Customer
participant Merchant
participant QiwiPay as QIWI PAY
participant QiwiBank as QIWI Bank
Customer->>Merchant:Отправка данных карты<br>с платежной формы
activate Merchant
Merchant->>QiwiPay:Запрос<br>на операцию "sale"
activate QiwiPay
QiwiPay->>QiwiPay:Модуль борьбы<br>с мошенничеством
rect rgb(255, 238, 223)
Note over Customer, QiwiPay:3-D Secure
QiwiPay->>QiwiPay:Проверка карты<br>на участие в 3-D Secure
QiwiPay-->>Merchant:Параметры для перенаправления<br>плательщика на ACS банка-эмитента
deactivate QiwiPay
Merchant-->>Customer:Перенаправление<br>плательщика на ACS URL
deactivate Merchant
Customer->>Merchant:Возврат с ACS<br>с результатом верификации 3-D Secure
activate Merchant
Merchant->>QiwiPay:Запрос<br>на операцию "finish_3ds"
activate QiwiPay
end
QiwiPay->>QiwiBank:Авторизационный запрос
activate QiwiBank
QiwiBank-->>QiwiPay:Успешная авторизация
QiwiPay->>QiwiBank:Финансовая операция<br>по результату авторизации
QiwiBank-->>QiwiPay:Подтверждение<br>финансовой операции
deactivate QiwiBank
rect rgb(255, 238, 223)
Note over Customer, QiwiPay:Немедленное уведомление об операции
QiwiPay->>Merchant:Уведомление о результате<br>операции (callback)
activate Merchant
Merchant-->>QiwiPay:Подтверждение<br>получения уведомления
deactivate Merchant
end
QiwiPay->>Merchant:Результат<br>выполнения операции
QiwiPay->>Customer:E-mail уведомление<br>о результате операции
deactivate QiwiPay
Merchant->>Customer:Страница статуса<br>с результатом операции
deactivate Merchant
</div>

### Схема двухшагового сценария платежа {#twostep}

<div class="mermaid">
sequenceDiagram
participant Customer
participant Merchant
participant QiwiPay as  QIWI PAY
participant QiwiBank as QIWI Bank
Customer->>Merchant:Отправка данных карты<br>с платежной формы
activate Merchant
Merchant->>QiwiPay:Запрос<br>на операцию "auth"
activate QiwiPay
QiwiPay->>QiwiPay:Модуль борьбы<br>с мошенничеством
rect rgb(255, 238, 223)
Note over Customer, QiwiPay:3-D Secure
QiwiPay->>QiwiPay:Проверка карты<br>на участие в 3-D Secure
QiwiPay-->>Merchant:Параметры для перенаправления<br>плательщика на ACS банка-эмитента
deactivate QiwiPay
Merchant-->>Customer:Перенаправление<br>плательщика на ACS URL
deactivate Merchant
Customer->>Merchant:Возврат с ACS<br>с результатом верификации 3-D Secure
activate Merchant
Merchant->>QiwiPay:Запрос<br>на операцию "finish_3ds"
activate QiwiPay
end
QiwiPay->>QiwiBank:Авторизационный запрос
activate QiwiBank
QiwiBank-->>QiwiPay:Успешная авторизация
deactivate QiwiBank
rect rgb(255, 238, 223)
Note over Customer, QiwiPay:Немедленное уведомление об операции
QiwiPay->>Merchant:Уведомление о результате<br>операции (callback)
activate Merchant
Merchant-->>QiwiPay:Подтверждение<br>получения уведомления
deactivate Merchant
end
QiwiPay->>Merchant:Результат<br>выполнения операции
QiwiPay->>Customer:E-mail уведомление<br>о результате операции
deactivate QiwiPay
Merchant->>Customer:Страница статуса<br>с результатом операции
deactivate Merchant
Note over Merchant:Ожидание запроса на подтверждение авторизации
Merchant-->>QiwiPay:Запрос<br>на операцию "capture"
activate Merchant
activate QiwiPay
QiwiPay->>QiwiBank:Финансовая операция<br>по результату авторизации
activate QiwiBank
QiwiBank-->>QiwiPay:Подтверждение<br>финансовой операции
deactivate QiwiBank
QiwiPay->>Merchant:Уведомление о результате<br>операции (callback)
deactivate QiwiPay
deactivate Merchant
</div>

## Операция покупки {#sale}

### Запрос {#sale-request}

~~~ json
{
  "opcode": 1,
  "merchant_uid": "10001",
  "merchant_site": 555,
  "pan": "4111111111111111",
  "expiry": "1219",
  "cvv2": "123",
  "amount": "4678.50",
  "currency": 643,
  "card_name": "cardholder name",
  "order_id": "order1231231",
  "ip": "127.0.0.1",
  "email": "merchant@qiwi.com",
  "country": "RUS",
  "city": "Moscow",
  "region": "606008",
  "address": "South Park",
  "phone": "79166554321",
  "user_device_id": "12312321Un43243",
  "user_timedate": "2017-03-15T09:57:21+03:00",
  "user_screen_res": "1080x1794",
  "user_agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 10_2_1 like Mac OS X) AppleWebKit/602.4.6 (KHTML, like Gecko)...",
  "cf1": "cf1",
  "cf2": "cf2",
  "cf3": "cf3",
  "cf4": "cf4",
  "cf5": "cf5",
  "callback_url": "http://domain.tld/callback_service",
  "success_url": "http://domain.tld/success",
  "decline_url": "http://domain.tld/decline",
  "product_name": "Название услуги",
  "receiver_name": "Ivanov, Ivan",
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e..........................."
}
~~~

Все операции платежа (`sale`, `auth`) используют один и тот же набор параметров.

Параметр|Условие|Тип данных|Описание
--------|-------|----------|--------
opcode|Обязательно|integer|[Код операции](#operations) (`1`, `3`)
merchant_site|Обязательно|integer|Идентификатор сайта ТСП
pan|Условно обязательно|string(19)|Номер банковской карты
expiry|Условно обязательно|string(4)|Срок действия банковской карты
card_token |Условно обязательно|string(40)|Токен карты
cvv2|Условно обязательно|string(4)|CVV2/CVC2 на банковской карте
amount|Обязательно|string(20)|Сумма операции
currency|Обязательно|integer|Валюта суммы операции в цифровом формате согласно ISO 4217
sign|Обязательно|string(64)|[Контрольная сумма переданных параметров](#sign)
card_name|Условно обязательно|string(64)|Имя Покупателя, как указано на карте (латинские буквы)
order_id|Условно обязательно|string(256)|Уникальный номер заказа в системе ТСП
ip|Условно обязательно|string(15)|IP-адрес Покупателя
email|Условно обязательно|string(64)|E-mail Покупателя
country|Условно обязательно|string(3)|Страна Покупателя в формате 3-х буквенных кодов согласно ISO 3166-1
user_device_id|Условно обязательно|string(64)|Уникальный идентификатор устройства Покупателя
city|Условно обязательно|string(64)|Город местонахождения Покупателя
region|Условно обязательно|string(6)|Регион страны формате геокодов согласно ISO 3166-2
address|Условно обязательно|string(64)|Адрес местонахождения Покупателя
phone|Условно обязательно|string(15)|Контактный телефон Покупателя
user_timedate|Опционально|YYYY-MM-DDThh:mm:ss ±hh:mm|Локальное время в системе Покупателя в формате ISO8601, с временной зоной
user_screen_res|Опционально|string(64)|Разрешение экрана Покупателя
user_agent|Опционально|string(256)|Браузер Покупателя
cf1|Опционально|string(256)|Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf2|Опционально|string(256)|Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf3|Опционально|string(256)|Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf4|Опционально|string(256)|Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf5|Опционально|string(256)|Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
product_name|Опционально|string(25)|Описание услуги, которую получает Плательщик.
merchant_uid|Опционально|string(64)|Уникальный идентификатор Покупателя в системе ТСП.
callback_url|Опционально|string(256)|URL отправки [callback](#callback)
cheque|Опционально|string|[Данные для кассового чека по 54-ФЗ](#cheque)
wallet_type|Опционально|string(50)|Тип пополняемого кошелька. Возможные значения: `QIWI`
account_id|Условно обязательно|integer(50)|Номер пополняемого кошелька. **Обязательно указывайте при передаче параметра wallet_type=QIWI**
receiver_name|Опционально|string(30)|Фамилия и имя получателя платежа, разделитель `, ` (запятая и пробел). Сначала указывается фамилия, затем имя. Только латинские буквы, цифры, символы подчеркивания `_` и дефиса `-` (допускаются пробелы в имени).
receiver_pan|Опционально|string(19)|Номер карты получателя денежного перевода. Указывается для операций денежных переводов.
receiver_bank_account|Опционально|string(20)|Номер счета получателя платежа. Указывается для операций денежных переводов.
receiver_bic|Опционально|string(9)|БИК кредитной организации получателя платежа. Указывается для операций денежных переводов.
receiver_wallet|Опционально|string(64)|Номер кошелька получателя платежа. Указывается при пополнении кошелька.
receiver_inn|Опционально|string(12)|ИНН организации-эмитента кошелька. Указывается при пополнении кошелька.
receiver_phone|Опционально|string(15)|Номер телефона получателя платежа. Указывается при пополнении телефона.
upper_commission_taken|Опционально|string|Поле для передачи верхней комиссии.

### Ответ на операцию покупки в случае не 3DS операции {#sale_no3ds}

~~~ json
{
  "txn_id":172001,
  "txn_status":3,
  "txn_type":1,
  "txn_date": "2017-03-09T17:16:06+00:00",
  "card_token": "4d5b363e-a116-35f5-e053-6751080ac38e",
  "card_token_expire": "2018-02-27T21:00:00+00:00",
  "error_code":0,
  "pan": "411111******1111",
  "issuer_name": "QIWI BANK (JSC)",
  "issuer_country": "RUS",
  "amount": 4678.50,
  "currency": 643,
  "auth_code": "2G4923",
  "eci":
}
~~~

Поле ответа|Тип данных|Описание
--------|----------|--------
txn_id | integer | Идентификатор транзакции
txn_status | integer | [Статус транзакции](#txn_status)
txn_type | integer | [Тип транзакции](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Дата транзакции в формате ISO8601 с временной зоной
card_token | string(40) | Токен карты (если функционал токенизации включен для данного сайта)
card_token_expire | YYYY-MM-DDThh:mm:ss±hh:mm | Срок истечения токена карты (если функционал токенизации включен для данного сайта)
error_code | integer | [Код ошибки работы системы](#errors)
pan | string(19) | Номер карты Покупателя
issuer_name | string(40) | Название банка-эмитента
issuer_country | string(3) | Страна банка-эмитента
amount | decimal | Сумма списания
currency | integer | Валюта суммы списания в цифровом формате согласно ISO 4217
auth_code | string(6) | Код авторизации
eci | string(2) | Индикатор E-Commerce операции
is_test| string(4) |Наличие этого параметра со значением `true` указывает на то, что транзакция проведена в [тестовой среде](#test_mode). Реального списания д/с с карты не было.

### Ответ в случае если необходима 3DS аутентификация {#sale_3ds}

~~~ json
{
  "txn_id":172001,
  "txn_status":0,
  "txn_type":1,
  "txn_date": "2017-03-09T17:16:06+00:00",
  "error_code":0,
  "acs_url":"https://test.paymentgate.ru/acs/auth/start.do",
  "pareq":"eJxVUmtvgjAUuG79oClYe51uDcsi2B+ZrJPgKtipGHRnUTgy1w5bVwyf4BfjpuJ........."
}
~~~

Поле ответа|Тип данных|Описание
--------|----------|--------
txn_id | integer | Идентификатор транзакции
txn_status | integer | [Статус транзакции](#txn_status)
txn_type | integer | [Тип транзакции](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Дата транзакции в формате ISO8601 с временной зоной
error_code | integer | [Код ошибки работы системы](#errors)
acs_url | string(1024) | URL адрес банка-эмитента куда необходимо перенаправить Покупателя
pareq | string(4096) | Уникальный идентификатор Покупателя, используемый при дальнейшем его перенаправлении на acs_url.
is_test| string(4) |Наличие этого параметра со значением `true` указывает на то, что транзакция проведена в [тестовой среде](#test_mode). Реального списания д/с с карты не было.

После возврата Покупателя с ссылки перенаправления, сформированной по правилам 3DS,  необходимо отправить запрос для завершения 3DS аутентификации.

## Завершение 3DS аутентификации {#finish_3ds}

После успешного прохождения 3DS аутентификации, ТСП необходимо отправить запрос для завершения проверки.

### Запрос после прохождения 3DS аутентификации {#finish-request}

~~~ json
{
  "opcode":2,
  "merchant_site":99,
  "pares": "eJzVWFevo9iyfu9fMZrzaM0QjWHk3tIiGptgooE3cgabYMKvv3jvTurTc3XOfbkaJMuL............",
  "txn_id": "172001"
}
~~~

Параметр|Условие|Тип данных|Описание
--------|-------|----------|--------
opcode | Обязательно | integer | [Код операции](#operations) (`2`)
merchant_site | Обязательно | integer | Идентификатор сайта ТСП
pares | Обязательно | string(4096) | Результат верификации Покупателя
txn_id | Обязательно | integer | Идентификатор транзакции
sign | Обязательно | string(64) | [Контрольная сумма переданных параметров](#sign)

### Ответ на операцию покупки после 3DS аутентификации {#finish-response}

~~~ json
{
  "txn_id":172001,
  "txn_status":3,
  "txn_type":1,
  "txn_date": "2017-03-09T17:16:06+00:00",
  "card_token": "4d5b363e-a116-35f5-e053-6751080ac38e",
  "card_token_expire": "2018-02-27T21:00:00+00:00",
  "error_code":0,
  "pan": "411111******1111",
  "issuer_name": "QIWI BANK (JSC)",
  "issuer_country": "RUS",
  "amount": 4678.50,
  "currency": 643,
  "auth_code": "2G4923",
  "eci": "5"
}
~~~

Поле ответа|Тип данных|Описание
--------|----------|--------
txn_id | integer | Идентификатор транзакции
txn_status | integer | [Статус транзакции](#txn_status)
txn_type | integer | [Тип транзакции](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Дата транзакции в формате ISO8601 с временной зоной
card_token | string(40) | Токен карты (если функционал токенизации включен для данного сайта)
card_token_expire | YYYY-MM-DDThh:mm:ss±hh:mm | Срок истечения токена карты (если функционал токенизации включен для данного сайта)
error_code | integer | [Код ошибки работы системы](#errors)
pan | string(19) | Номер карты Покупателя
issuer_name | string(40) | Название банка-эмитента
issuer_country | string(3) | Страна банка-эмитента
amount | decimal | Сумма списания
currency | integer | Валюта суммы списания в цифровом формате согласно ISO 4217
auth_code | string(6) | Код авторизации
eci | string(2) | Индикатор E-Commerce операции
is_test| string(4) |Наличие этого параметра со значением `true` указывает на то, что транзакция проведена в [тестовой среде](#test_mode). Реального списания д/с с карты не было.

## Операция подтверждения покупки {#capture}

<aside class="warning">
Не отправляйте повторный запрос на подтверждение, пока не получен ответ от API.  Если соединение разорвалось, отправьте повторный запрос на подтверждение через 3 минуты после первой попытки.
</aside>

### Запрос {#capture-request}

~~~ json
{
  "opcode": 5,
  "merchant_site": 99,
  "txn_id": "172001",
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e9286a9b8f34b236b1bf2587e4268"
}
~~~

Параметр|Условие|Тип данных|Описание
--------|-------|----------|--------
opcode | Обязательно | integer | [Код операции](#operations) (`5`)
merchant_site | Обязательно | integer | Идентификатор сайта ТСП
txn_id | Обязательно | integer | Идентификатор транзакции
sign | Обязательно | string(64) | [Контрольная сумма переданных параметров](#sign)
cheque|Опционально|string|[Данные для кассового чека по 54-ФЗ](#cheque)

### Ответ {#capture-response}

~~~ json
{
  "txn_id":172001,
  "txn_status":3,
  "txn_type":2,
  "txn_date": "2017-03-09T17:16:06+00:00",
  "error_code":0
}
~~~

Поле ответа|Тип данных|Описание
--------|----------|--------
txn_id | integer | Идентификатор транзакции
txn_status | integer | [Статус транзакции](#txn_status)
txn_type | integer | [Тип транзакции](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Дата транзакции в формате ISO8601 с временной зоной
error_code | integer | [Код ошибки работы системы](#errors)
is_test| string(4) |Наличие этого параметра со значением `true` указывает на то, что транзакция проведена в [тестовой среде](#test_mode). Реального списания д/с с карты не было.

## Операция отмены {#cancel}

### Запрос {#cancel-request}

~~~ json
{
  "opcode":6,
  "merchant_site": 99,
  "txn_id": 181001,
  "amount": "700",
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e9286a9b8f34b236b1bf2587e4268"
}
~~~

Параметр|Условие|Тип данных|Описание
--------|-------|----------|--------
opcode | Обязательно | integer | [Код операции](#operations) (`6`)
merchant_site | Обязательно | integer | Идентификатор сайта ТСП
txn_id | Обязательно | integer | Идентификатор транзакции
sign | Обязательно | string(64) | [Контрольная сумма переданных параметров](#sign)
amount | Опционально | string(20) | Сумма операции

### Ответ {#cancel-response}

~~~ json
{
  "txn_id":182001,
  "txn_status":3,
  "txn_type":4,
  "txn_date": "2017-03-09T17:16:06+00:00",
  "error_code":0,
  "amount": 700
}
~~~

Поле ответа|Тип данных|Описание
--------|----------|--------
txn_id | integer | Идентификатор транзакции
txn_status | integer | [Статус транзакции](#txn_status)
txn_type | integer | [Тип транзакции](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Дата транзакции в формате ISO8601 с временной зоной
error_code | integer | [Код ошибки работы системы](#errors)
amount | decimal | Сумма списания
is_test| string(4) |Наличие этого параметра со значением `true` указывает на то, что транзакция проведена в [тестовой среде](#test_mode). Реального возврата д/с на карту не было.

## Операция возврата {#refund}

### Запрос {#refund-request}

~~~ json
{
  "opcode":7,
  "merchant_site": 99,
  "txn_id": 181001,
  "amount": "700",
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e9286a9b8f34b236b1bf2587e4268"
}
~~~

Параметр|Условие|Тип данных|Описание
--------|-------|----------|--------
opcode | Обязательно | integer | [Код операции](#operations) (`7`)
merchant_site | Обязательно | integer | Идентификатор сайта ТСП
txn_id | Обязательно | integer | Идентификатор транзакции
sign | Обязательно | string(64) | [Контрольная сумма переданных параметров](#sign)
amount | Опционально | string(20) | Сумма операции

### Ответ {#refund-response}

~~~ json
{
  "txn_id":182001,
  "txn_status":3,
  "txn_type":3,
  "txn_date": "2017-03-09T17:16:06+00:00",
  "error_code":0,
  "amount": 700
}
~~~

Поле ответа|Тип данных|Описание
--------|----------|--------
txn_id | integer | Идентификатор транзакции
txn_status | integer | [Статус транзакции](#txn_status)
txn_type | integer | [Тип транзакции](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Дата транзакции в формате ISO8601 с временной зоной
error_code | integer | [Код ошибки работы системы](#errors)
amount | decimal | Сумма списания
is_test| string(4) |Наличие этого параметра со значением `true` указывает на то, что транзакция проведена в [тестовой среде](#test_mode). Реального возврата д/с на карту не было.

## Операция выплаты {#payout}

### Запрос {#payout-request}

~~~ json
{
  "opcode": 20,
  "merchant_uid": "10001",
  "merchant_site": 555,
  "card_token": "4d5b363e-a116-35f5-e053-6751080ac38e",
  "txn_id":182001,
  "amount": "4678.50",
  "currency": 643,
  "card_name": "cardholder name",
  "order_id": "order1231231",
  "cf1": "cf1",
  "cf2": "cf2",
  "cf3": "cf3",
  "cf4": "cf4",
  "cf5": "cf5",
  "callback_url": "http://domain.tld/callback_service",
  "success_url": "http://domain.tld/success",
  "decline_url": "http://domain.tld/decline",
  "product_name": "Выплата выигрыша",
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e..........................."
}
~~~

Параметр|Условие|Тип данных|Описание
--------|-------|----------|--------
opcode | Обязательно | integer | [Код операции](#operations) (`20`)
merchant_site | Обязательно | integer | Идентификатор сайта ТСП
amount|Обязательно|string(20)|Сумма операции
currency|Обязательно|integer|Валюта суммы операции в цифровом формате согласно ISO 4217
sign|Обязательно|string(64)|[Контрольная сумма переданных параметров](#sign)
pan|Условно обязательно|string(19)|Номер банковской карты
card_token |Условно обязательно|string(40)|Токен карты
txn_id | Условно обязательно | integer | Идентификатор транзакции (для контроля максимальной суммы выплаты)
card_name|Условно обязательно|string(64)|Имя Покупателя, как указано на карте (латинские буквы)
order_id|Условно обязательно|string(256)|Уникальный номер заказа в системе ТСП
cf1|Опционально|string(256)|Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf2|Опционально|string(256)|Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf3|Опционально|string(256)|Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf4|Опционально|string(256)|Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf5|Опционально|string(256)|Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
product_name|Опционально|string(25)|Описание услуги, которую получает Плательщик.
merchant_uid|Опционально|string(64)|Уникальный идентификатор Покупателя в системе ТСП.
callback_url|Опционально|string(256)|URL отправки [callback](#callback)

### Ответ {#payout-response}

~~~ json
{
  "txn_id":172001,
  "txn_status":3,
  "txn_type":8,
  "txn_date": "2017-03-09T17:16:06+00:00",
  "error_code":0,
  "pan": "411111******1111",
  "amount": 4678.50,
  "currency": 643,
  "auth_code": "2G4923"
}
~~~

Поле ответа|Тип данных|Описание
--------|----------|--------
txn_id | integer | Идентификатор транзакции
txn_status | integer | [Статус транзакции](#txn_status)
txn_type | integer | [Тип транзакции](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Дата транзакции в формате ISO8601 с временной зоной
error_code | integer | [Код ошибки работы системы](#errors)
pan | string(19) | Номер карты
amount | decimal | Сумма выплаты
currency | integer | Валюта суммы списания в цифровом формате согласно ISO 4217
auth_code | string(6) | Код авторизации

## Запрос статуса операции {#opstatus}

### Запрос {#opstatus-request}

~~~ json
{
  "opcode":30,
  "merchant_site": 99,
  "order_id": "41324123412342",
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e9286a9b8f34b236b1bf2587e4268"
}
~~~

Параметр|Условие|Тип данных|Описание
--------|-------|----------|--------
opcode | Обязательно | integer | [Код операции](#operations) (`30`)
merchant_site | Обязательно | integer | Идентификатор сайта ТСП
sign | Обязательно | string(64) | [Контрольная сумма переданных параметров](#sign)
txn_id | Опционально | integer | Идентификатор транзакции
order_id | Опционально | string(256) | Уникальный номер заказа в системе ТСП

### Ответ {#opstatus-response}

~~~ json
{
  "transactions": [
    {
      "error_code": 0,
      "txn_id": 3666050,
      "txn_status": 2,
      "txn_type": 2,
      "txn_date": "2017-03-09T17:16:06+00:00",
      "pan": "400000******0002",
      "amount": 10000,
      "currency": 643,
      "auth_code": "181218",
      "merchant_site": 99,
      "card_name": "cardholder name",
      "card_bank": "",
      "order_id": "41324123412342"
    },
    {
      "error_code": 0,
      "txn_id": 3684050,
      "txn_status": 3,
      "txn_type": 4,
      "txn_date": "2017-03-09T17:16:09+00:00",
      "pan": "400000******0002",
      "amount": 100,
      "currency": 643,
      "merchant_site": 99,
      "card_name": "cardholder name",
      "card_bank": ""
    },
    {
      "error_code": 0,
      "txn_id": 3685050,
      "txn_status": 3,
      "txn_type": 4,
      "txn_date": "2017-03-19T17:16:06+00:00",
      "pan": "400000******0002",
      "amount": 100,
      "currency": 643,
      "merchant_site": 99,
      "card_name": "cardholder name",
      "card_bank": ""
    }
  ],
  "error_code": 0
}
~~~

<aside class="notice">
Возвращается в виде JSON массива.
</aside>

Поле ответа|Тип данных|Описание
--------|----------|--------
txn_id | integer | Идентификатор транзакции
txn_status | integer | [Статус транзакции](#txn_status)
txn_type | integer | [Тип транзакции](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Дата транзакции в формате ISO8601 с временной зоной
error_code | integer | [Код ошибки работы системы](#errors)
pan | string(19) | Номер карты Покупателя в формате 411111******1111
amount | decimal | Сумма списания
currency | integer | Валюта суммы списания в цифровом формате согласно ISO 4217
auth_code | string(6) | Код авторизации
eci | string(2) | Индикатор E-Commerce операции
card_name | string(64) | Имя Покупателя, как указано на карте (латинские буквы)
card_bank | string(64) | Банк-эмитент карты
order_id | string(256) | Уникальный номер заказа в системе ТСП
ip | string(15) | IP-адрес Покупателя
email | string(64) | E-mail Покупателя
country | string(3) | Страна Покупателя в формате 3-х буквенных кодов согласно ISO 3166-1
city | string(64) | Город местонахождения Покупателя
region | string(6) | Регион страны формате геокодов согласно ISO 3166-2
address | string(64) | Адрес местонахождения Покупателя
phone | string(15) | Контактный телефон Покупателя
cf1 | string(256) | Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf2 | string(256) | Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf3 | string(256) | Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf4 | string(256) | Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
cf5 | string(256) | Поля для ввода произвольной информации, дополняющей данные по операции. Например — описание услуг ТСП.
product_name | string(25) | Описание услуги, которую получает Плательщик.

# Подпись запроса {#sign}

~~~ java

public String generateSignature(String data, String secret) {
    try {
        byte[] secretBytes = secret.getBytes("UTF-8");
        Mac hmac = Mac.getInstance("HmacSHA256");
        SecretKeySpec secretKey = new SecretKeySpec(secretBytes, "HmacSHA256");
        hmac.init(secretKey);
        byte[] signBytes = hmac.doFinal(data.getBytes("UTF-8"));
        return Utils.bytesToHex(signBytes);
    } catch (NoSuchAlgorithmException | UnsupportedEncodingException | InvalidKeyException e) {
        throw new SignatureException("Cannot calculate signature", e);
    }
}
~~~

>Для строки параметров { "opcode":3, "merchant_site": 555, "amount": "7.00", "currency": 643 }

~~~java
sign = generateSignature("7.00|643|555|3","secret_key");

Output:
9c878bfbf9baa30c26c8c6206976fc3ed2c036afeabf352f8a045fe331d42d7e
~~~

~~~php
<?php
$hmac = hash_hmac('sha256','7.00|643|555|3','secret_key');
?>

Output:
9c878bfbf9baa30c26c8c6206976fc3ed2c036afeabf352f8a045fe331d42d7e
~~~

~~~shell
user@server:~$ echo -n "7.00|643|555|3" | openssl dgst -sha256  -hmac "secret_key"
(stdin)= 9c878bfbf9baa30c26c8c6206976fc3ed2c036afeabf352f8a045fe331d42d7e
~~~

Каждая операция должна быть подписана. Подпись помещается в параметр `sign` запроса.

Для формирования подписи используется механизм проверки целостности HMAC с хэш-функцией SHA256. Для этого необходим `secret key`, полученный вместе с остальными настройками при подключении мерчанта.

* В качестве разделителей значений параметров используется символ `|`.
* В подписи должны присутствовать значения всех параметров, которые присутствуют в запросе.
* Значения передаются как строки.
* Параметры с пустыми значениями в формировании `sign` не участвуют.
* Формирование подписи происходит только из значений переданных параметров, сами названия параметров в `sign` не участвуют.
* Строка для подписи формируется методом выстраивания параметров в алфавитном порядке. Например, для параметров `{ "opcode":1, "merchant_site": 12345, "amount": "15.00", "currency": 643 }` строка для вычисления подписи будет такой: `15.00|643|12345|1`.

# Уведомления о платежах {#callback}

Существует два метода доставки уведомлений о платеже:

* В процессе выполнения платежа, после получения ответа о списании средств, до показа статусной страницы или ответа ТСП (в зависимости от типа взаимодействия).
* После проведения платежа в фоновом режиме.

Уведомление доставляется на адрес, установленный при подключении мерчанта, или на адрес, переданный в параметре `callback_url` запроса. Уведомление — это POST-запрос, параметры передаются в теле запроса в формате JSON.

<aside class="success">
Уведомление о платеже (callback) отправляется только по протоколу HTTPS и только на 443 порт.
Сертификат должен быть выдан доверенным центром сертификации (напр., Comodo, Verisign, Thawte и т.п.)
</aside>

~~~http
POST /merchant-pay/callback HTTP/1.1
Content-Type: application/json
Host: example.com

{
    "request_id": "d861cxxxx9cayyy",
    "txn_id": 806930407050,
    "txn_date": "2021-10-08T12:12:53+00:00",
    "txn_status": 3,
    "txn_type": 1,
    "issuer_name": "Сбербанк России",
    "issuer_country": "RUS",
    "card_name": "cardholder+name",
    "card_token": "d0f3c937-xxxx-yyyy-zzzz-5614c90b6199",
    "card_token_expire": "2022-05-30T21:00:00+00:00",
    "pan": "400000******0002",
    "amount": 1856,
    "currency": 643,
    "auth_code": "255723",
    "eci": "",
    "error_code": 0,
    "sign": "B9BC3C4A672FAB763519XXXXYYYYZZZZ6DA8BEE9649D6E6A8BD3A1198BE9417532B"
}
~~~

Поле уведомления|Тип данных| Участвует в sign | Описание
--------|----------|------------------|---------
txn_id | integer | + | Идентификатор транзакции
txn_status | integer | + | [Статус транзакции](#txn_status)
txn_type | integer | + | [Тип транзакции](#txn_type)
txn_date |YYYY-MM-DDThh:mm:ss±hh:mm| - | Дата транзакции в формате ISO8601 с временной зоной
error_code | integer | + | [Код ошибки работы системы](#errors)
pan | string(19) | - | Номер карты Покупателя в формате 411111******1111
amount | decimal | + | Сумма списания
currency | integer | + | Валюта суммы списания в цифровом формате согласно ISO 4217
auth_code | string(6) | - | Код авторизации
eci | string(2) | - | Индикатор E-Commerce операции
card_name | string(64) | - | Имя Покупателя, как указано на карте (латинские буквы)
issuer_name | string(64) | - | Название банка-эмитента карты
issuer_country | string(3) | - | Страна банка-эмитента
order_id | string(256) | - | Уникальный номер заказа в системе ТСП. Возвращается в уведомлении, только если был передан в запросе.
ip | string(15) | + | IP-адрес Покупателя
email | string(64) | + | E-mail Покупателя
country | string(3) | - | Страна Покупателя в формате 3-х буквенных кодов согласно ISO 3166-1
city | string(64) | - | Город местонахождения Покупателя
region | string(6) | - | Регион страны формате геокодов согласно ISO 3166-2
address | string(64) | - | Адрес местонахождения Покупателя
phone | string(15) | - | Контактный телефон Покупателя
cf1 | string(256) | - | Поля для ввода произвольной информации, дополняющей данные по операции. Например - описание услуг ТСП.
cf2 | string(256) | - | Поля для ввода произвольной информации, дополняющей данные по операции. Например - описание услуг ТСП.
cf3 | string(256) | - | Поля для ввода произвольной информации, дополняющей данные по операции. Например - описание услуг ТСП.
cf4 | string(256) | - | Поля для ввода произвольной информации, дополняющей данные по операции. Например - описание услуг ТСП.
cf5 | string(256) | - | Поля для ввода произвольной информации, дополняющей данные по операции. Например - описание услуг ТСП.
product_name | string(25) | - | Описание услуги, которую получает Плательщик.
card_token | string(40) | - | Токен карты (если функционал токенизации включен для данного сайта)
card_token_expire | YYYY-MM-DDThh:mm:ss±hh:mm | - | Срок истечения токена карты (если функционал токенизации включен для данного сайта)
sign | string(64) | - | Контрольная сумма переданных параметров. _Передается в верхнем регистре._

В уведомлении присутствует подпись запроса (поле `sign`), который ТСП необходимо проверять на своей стороне для исключения возможности подделки уведомления.

<aside class="notice">
Используется тот же алгоритм создания подписи, что и <a href="#sign">для запросов API</a>. Однако в формировании подписи уведомления участвуют не все поля, а только отмеченные <b>+</b> в столбце <i>Участвует в sign</i> в таблице выше.
</aside>

Для дополнительной уверенности следует принимать уведомления о платежах только с указанных ниже IP-адресов компании QIWI.

* 79.142.16.0/20
* 195.189.100.0/22
* 91.232.230.0/23
* 91.213.51.0/24

Уведомление считается успешно доставленным если сервер ТСП ответил HTTP с кодом состояния `200 OK`.
Таким образом, до момента ответа сервера с кодом состояния `200 OK`, система будет пытаться доставить уведомление через увеличивающиеся интервалы времени в течение суток с момента операции.

<aside class="notice">
Если по какой-либо причине сервер ТСП успешно принял уведомление, но ответил не <i>200 OK</i>, то при повторной попытке доставки он не должен обрабатывать данную транзакцию как новую.
</aside>

<aside class="warning">
Вся ответственность за возможные финансовые потери, случившиеся вследствие отсутствия валидации sign в нелегитимном уведомлении, лежит полностью на ТСП.
</aside>

# Отраслевые данные {#industry_data}

## Авиаданные {#avia-data}

~~~ json
{
  "pan": "4111111111111111",
  "expiry": "1217",
  "cvv2": "002",
  "card_name": "cardholder name",
  "merchant_site": 1234,
  "cf1": "Ghbdtn",
  "opcode": 3,
  "callback_url": "testtest",
  "amount": 100,
  "currency": 643,
  "sign": "9FD874BF9D4483854E25D1D3371D01821AA4814CDFA4FE2033B5EDD4D5DE94C9",
  "industry_data": {
      "type": "avia",
      "pnr": "6Q32R2",
      "tickets": [
          {
              "number": "1",
              "passenger_name": "Ivan Ivanov",
              "amount": 500,
              "amount_total": 700,
              "restricted": false,
              "agency_code": "avi",
              "agency_name": "avia company",
              "airport_code": "air",
              "segments": [
                  {
                      "amount": 500,
                      "service_class": "A",
                      "fare_basis_code": "B",
                      "dest_city": "XYB",
                      "carrier_code": "AI",
                      "stopover_code": "X",
                      "departure_date": "2016-12-30T02:45:00Z",
                      "flight_number": "12545"
                  }
              ]
          }
      ]
  }
}
~~~

Авиаданные передаются в параметре `industry_data` в операциях `auth`, `capture`, `sale` при работе через QIWI PAY API.
В случае, если на момент совершения операции `auth` присутствуют не все необходимые данные, их можно отправить в операции `capture` после получения.

В авиаданных должны присутствовать билеты из брони, а также все сегменты (перелеты) из каждого билета.

Параметр|Условие|Тип данных|Описание
--------|-------|----------|--------
type|Обязательно|string|Типы отраслевых данных (`avia`)
pnr|Опционально|string(6)|Номер брони
tickets|Обязательно|array|Массив билетов
--------|-------|----------|--------
number|Обязательно|string(14)|Номер билета
passenger_name|Обязательно|string(20)|Имя пассажира
amount|Обязательно|decimal|Цена билета без сборов
amount_total|Обязательно|decimal|Цена билета со сборами
restricted|Обязательно|boolean|Restricted Ticked Indicator
agency_code|Обязательно|string(8)|Код агентства
agency_name|Обязательно|string(25)|Название агентства
airport_code|Обязательно|string(3)|Аэропорт вылета (3 буквы по классификации IATA)
segments|Обязательно|array|Массив сегментов
--------|-------|----------|--------
amount|Обязательно|decimal|Номер билета
service_class|Обязательно|string(1)|Класс обслуживания
fare_basis_code|Обязательно|string(6)|Вид тарифа
dest_city|Обязательно|string(3)|Аэропорт назначения (3 буквы по классификации IATA)
carrier_code|Обязательно|string(2)|Код авиакомпании
stopover_code|Обязательно|string(1)|Stop-Over Code
departure_date|Обязательно|YYYY-MM-DDThh:mm:ss±hh:mm|Дата вылета в формате ISO8601 с временной зоной
flight_number|Обязательно|string(5)|Номер рейса, только цифры

# Безопасная сделка {#safe_deal}

Безопасная сделка от QIWI обеспечивает прозрачный и надежный механизм оплаты работ без риска потери денег как для заказчика, так и для исполнителя. Чтобы подключить Безопасную сделку, обратитесь к вашему сопровождающему менеджеру.

Поддерживается два вида безопасной сделки:

* С выплатой на QIWI кошелек
* С выплатой на карты

Авторизация операции производится как с помощью [QIWI PAY API](#api), так и [QIWI PAY WPF](#wpf).

## Работа с подсуммами {#safe_deal_subsums}

> Пример авторизации средств

~~~json
{
  "opcode": 1,
  "merchant_uid": "10001",
  "merchant_site": 555,
  "pan": "4111111111111111",
  "expiry": "1219",
  "cvv2": "123",
  "amount": "450.20",
  "currency": 643,
  "card_name": "cardholder name",
  "order_id": "order1231231",
  "cf1": "79323213232",
  "cf2": "250.10;200.10",
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e"
}
~~~

Для работы по протоколу безопасной сделки вам необходимо разделять сумму авторизации на подсуммы:

* подсумма выплат;
* подсумма комиссий.

**Подсумма выплат** — денежные средства, которые будут выплачены исполнителю при завершении безопасной сделки.

**Подсумма комиссий** — сумма комиссий Площадки, на которой выполняется сделка, и QIWI. Из данной подсуммы QIWI удержит комиссию за проведение операции, а остаток перечислит Площадке.

Подсуммы передаются в [авторизационном запросе](#sale) или в [вызове WPF](#redirect) в поле `cf2` через точку с запятой:

`"cf2": "<подсумма выплаты>;<подсумма комиссий>"`

<aside class="success">Сумма авторизации (<code>amount</code>) должна равняться сумме ее подсумм. Пример:<br><code>"amount": "500.00"</code><br><code>"cf2": "300.00; 200.00"</code>
</aside>

## Безопасная сделка с выплатой на QIWI кошелек {#safe_deal_wallet}

Этот вид безопасной сделки состоит из двух этапов:

* авторизация списания средств через вызов [API](#sale) или [WPF](#redirect) (код операции `3`)
* [подтверждение операции](#capture) (код операции `5`)

На каждом из этапов обязательно указывайте номер QIWI Кошелька в поле `cf1` в формате

`"cf1": "79111111111"`

При подтверждении операции сумма к выплате автоматически зачислится на указанный в запросе QIWI Кошелек.

<aside class="notice">Максимальное время ожидания между авторизацией и подтверждением составляет 5 суток. После этого операция будет автоматически подтверждена или отменена</aside>

> Пример отмены

~~~json
{
  "opcode":6,
  "merchant_site": 10001,
  "txn_id": 181001,
  "amount": "350.00",
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e9286a9b8f34b236b1bf2587e4268",
  "cf2": "250.00;100.00"
}
~~~

Транзакцию можно отменить полностью или частично только до подтверждения операции. Чтобы совершить отмену, выполните [запрос отмены](#cancel) (код операции `6`) и укажите в поле `cf2` подсуммы отменяемой транзакции (в том же формате, что и при авторизации).

<aside class="success">Сумма отмены (<code>amount</code>) должна равняться сумме ее подсумм.</aside>

## Безопасная сделка с выплатой на карту {#safe_deal_card}

Этот вид безопасной сделки состоит из трех этапов:

* авторизация списания средств через вызов [API](#sale) или [WPF](#redirect) (коды операций `1`, `3`)
* [подтверждение операции](#capture) (код операции `5`)
* [выплата](#payout) (код операции `20`)

> Пример выплаты

~~~json
{
  "opcode": 20,
  "merchant_site": "10001",
  "merchant_uid": 555,
  "card_token": "4d5b363e-a116-35f5-e053-6751080ac38e",
  "txn_id":182001,
  "amount": "4678.50",
  "currency": 643,
  "card_name": "cardholder name",
  "order_id": "order1231231",
  "callback_url": "http://domain.tld/callback_service",
  "success_url": "http://domain.tld/success",
  "decline_url": "http://domain.tld/decline",
  "product_name": "Выплата выигрыша",
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e..........................."
}
~~~

После подтверждения операции есть 180 дней для того, чтобы совершить операцию выплаты. Зачисление средств на карту получателя происходит в течение 30 минут.

<aside class="warning">Для организаций без PCI DSS возможно производить выплату только по токенам.</aside>

> Пример отмены

~~~json
{
  "opcode":7,
  "merchant_site": 10001,
  "txn_id": 181001,
  "amount": "350.00",
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e9286a9b8f34b236b1bf2587e4268",
  "cf2": "250.00;100.00"
}
~~~

Вы можете отменить транзакцию полностью или частично до подтверждения операции, а также совершить возврат после подтверждения, если выплата еще не была совершена.

Для отмены или возврата выполните запрос [отмены](#cancel) (код операции `6`) или [возврата](#refund) (код операции `7`) и укажите в поле `cf2` подсуммы отменяемой транзакции (в том же формате, что и при авторизации).

<aside class="success">Сумма отмены (возврата) (<code>amount</code>) должна равняться сумме ее подсумм.</aside>

# Передача чека (54-ФЗ) {#cheque}

Чек передается:

* при перенаправлении на QIWI PAY WPF — в параметре `merchant_cheque`;
* при работе через QIWI PAY API — в параметре `cheque` операций `auth`, `capture`, `sale`.

JSON [структура чека](#json_receipt) должна быть сжата алгоритмом DEFLATE, описанным в [RFC1951](http://www.ietf.org/rfc/rfc1951.txt), а после представлена в ZLIB формате, описанным в [RFC1950](http://www.ietf.org/rfc/rfc1950.txt). Далее результат кодируется в BASE64 и передается в соответствующем параметре.

<aside class="success">
Для активации сервиса приема фискальных данных необходимо сообщить вашему сопровождающему менеджеру ИНН, с которым организация зарегистрирована в сервисе по аренде онлайн касс.
Если используется сервис Атол, то дополнительно необходимо предоставить:<ul><li>email ТСП для печати в чеке</li>
<li>полное название организации</li>
<li>реквизиты ОФД (название, ИНН, url)</li>
<li>login и password для генерации токена</li>
<li>код группы</li></ul>
</aside>

<aside class="notice">
Пока аккаунт находится в <a href="#test_mode">тестовом режиме</a>, чек также будет проходить по тестовому контуру.
</aside>

### Параметры чека {#json_receipt}

~~~json
{
  "seller_id" : 3123011520,
  "cheque_type" : 1,
  "customer_contact" : "foo@domain.tld",
  "tax_system" : 1,
  "positions" : [
    {
      "quantity" : 2,
      "price" : 322.94,
      "tax" : 4,
      "description" : "Товар/Услуга 1",
      "payment_method" : 1,
      "payment_subject" : 1
    },
    {
      "quantity" : 1,
      "price" : 500,
      "tax" : 4,
      "description" : "Товар/Услуга 2",
      "payment_method" : 1,
      "payment_subject" : 1
    }
  ]
}
~~~

~~~ php
<?php
$cheque_json = '{
  "seller_id" : 3123011520,
  "cheque_type" : 1,
  "customer_contact" : "foo@domain.tld",
  "tax_system" : 1,
  "positions" : [
    {
      "quantity" : 2,
      "price" : 322.94,
      "tax" : 4,
      "description" : "Товар/Услуга 1",
      "payment_method" : 1,
      "payment_subject" : 1
    },
    {
      "quantity" : 1,
      "price" : 500,
      "tax" : 4,
      "description" : "Товар/Услуга 2",
      "payment_method" : 1,
      "payment_subject" : 1
    }
  ]
}';


$cheque = base64_encode(gzcompress($cheque_json));
?>

Output:
eJylj0FqwzAQRfc5hdDapJacLNpV71GKUaUpUbEkxxpDTQi05CbJBUK7LPQM8o0qKTSmlKyymcV7mq8/mxkh1EPTQFdrRckdqRivSsaWvCySkytY91Dj0EKy7AR7j87EFeksConJ0Gfn7pUzQts5Normhyheaz94BDMtt85r1M76hB4iIWSTZ3TrXljUOCTFi1/adlrm3yvO57eLM4/piU5AgZedblN6rhT24Tt8hOP4dhMO43v4GnfhMxwJo1O2GAxYrA3gyqlzyz/O908vcLqSZbctLtdm/2svy/L6zvzaznE+zrY/sHaTsw==
~~~

Параметр|Условие|Тип данных|Описание
--------|-------|----------|--------
seller_id|Обязательно|decimal|ИНН организации, для которой пробивается чек
cheque_type|Обязательно|decimal|Признак расчета (тэг `1054`):<br>1.	Приход<br>2.	Возврат прихода<br>3.	Расход<br>4.	Возврат расхода
customer_contact|Обязательно|string(64)|Телефон или электронный адрес покупателя (тэг `1008`)
tax_system|Обязательно|decimal|Система налогообложения (тэг `1055`):<br>0 – Общая, ОСН<br>1 – Упрощенная доход, УСН доход<br>2 – Упрощенная доход минус расход, УСН доход — расход<br>3 – Единый налог на вмененный доход, ЕНВД<br>4 – Единый сельскохозяйственный налог, ЕСН<br>5 – Патентная система налогообложения, Патент
positions|Обязательно|array|Массив товаров
--------|-------|----------|--------
quantity|Обязательно|decimal|Количество предмета расчета (тэг `1023`)
price|Обязательно|decimal|Цена за единицу предмета расчета с учетом скидок и наценок (тэг 1079)
tax|Обязательно|decimal|Ставка НДС (тэг `1199`):<br>3 – с учетом НДС 18% (18/118)<br>4 – с учетом НДС 10% (10/110)<br>5 – ставка НДС 0%<br>6 – НДС не облагается<br>7 – с учетом НДС 20% (20/120)
description|Обязательно|string(128)|Наименование предмета расчета
payment_method|Обязательно|decimal|Признак способа расчёта (тэг `1214`):<br>1 – предоплата 100%. Полная предварительная оплата до момента передачи предмета расчета.<br>2 – предоплата. Частичная предварительная оплата до момента передачи предмета расчета.<br>3 – аванс.<br>4 – полный расчет. Полная оплата, в том числе с учетом аванса (предварительной оплаты) в момент передачи предмета расчета.<br>5 – частичный расчет и кредит. Частичная оплата предмета расчета в момент его передачи с последующей оплатой в кредит.<br>6 – передача в кредит. Передача предмета расчета без его оплаты в момент его передачи с последующей оплатой в кредит.<br>7 – оплата кредита. Оплата предмета расчета после его передачи с оплатой в кредит (оплата кредита).
payment_subject|Обязательно|decimal|Признак предмета расчёта (тэг `1212`):<br>1 – товар. О реализуемом товаре, за исключением подакцизного товара (наименование и иные сведения, описывающие товар).<br>2 – подакцизный товар. О реализуемом подакцизном товаре (наименование и иные сведения, описывающие товар).<br>3 – работа. О выполняемой работе (наименование и иные сведения, описывающие работу).<br>4 – услуга. Об оказываемой услуге (наименование и иные сведения, описывающие услугу).<br>5 – ставка азартной игры. О приеме ставок при осуществлении деятельности по проведению азартных игр.<br>6 – выигрыш азартной игры. О выплате денежных средств в виде выигрыша при осуществлении деятельности по проведению азартных игр.<br>7 – лотерейный билет. О приеме денежных средств при реализации лотерейных билетов, электронных лотерейных билетов, приеме лотерейных ставок при осуществлении деятельности по проведению лотерей.<br>8 – выигрыш лотереи. О выплате денежных средств в виде выигрыша при осуществлении деятельности по проведению лотерей. <br>9 – предоставление результатов интеллектуальной деятельности. О предоставлении прав на использование результатов интеллектуальной деятельности или средств индивидуализации.<br>10 – платеж. Об авансе, задатке, предоплате, кредите, взносе в счет оплаты, пени, штрафе, вознаграждении, бонусе и ином аналогичном предмете расчета.<br>11 – агентское вознаграждение. О вознаграждении пользователя, являющегося платежным агентом (субагентом), банковским платежным агентом (субагентом), комиссионером, поверенным или иным агентом.<br>12 – составной предмет расчета. О предмете расчета, состоящем из предметов, каждому из которых может быть присвоено значение выше перечисленных признаков.<br>13 – иной предмет расчета. О предмете расчета, не относящемуся к выше перечисленным предметам расчета.

# Типы транзакций {#txn_type}

Возможные типы транзакций, возвращаемые в ответе в поле `txn_type`, указаны в таблице.

Тип | Название | Описание
--- | -------- | --------
1 | Purchase | Одношаговая операция оплаты
2 | Purchase | Операция авторизации при двухшаговом сценарии платежа
4 | Reversal | Операция отмены
3 | Refund | Операция возврата
8 | Payout | Операция выплаты (OCT)
0 | Unknown | Неизвестный тип операции

# Статусы транзакции {#txn_status}

Статус | Название | Описание | Возможные операции
------ | -------- | -------- | ------------------
0 | Init | Пройдена базовая верификация данных и начат процесс проведения операции | -
1 | Declined | Операция отклонена | -
2 | Authorized | Выполнена операция авторизации средств (холдирование) | capture, reversal
3 | Captured (Completed) | Подтвержденная операция | reversal
4 | Reconciled | Полностью завершенная финансовая операция | refund
5 | Settled | За данную операцию средства будут выплачены ТСП | refund

<aside class="notice">
В случае, если взаимодействие с банком-эквайером происходит в online режиме, то после операций <code>sale</code>, <code>capture</code> будет возвращен статус операции — <code>4</code>.
Если же взаимодействие с банком-эквайером требует отложенных подтверждений финансовых операций, то после операций <code>sale</code>, <code>capture</code> будет возвращен статус операции — <code>3</code>, а после отправки финансовой информации на запрос статуса операции будет возвращено — <code>4</code>.
</aside>

<aside class="success">
Статус 2 и выше означает, что средства авторизованы и уже можно предоставлять услугу/товар.
</aside>

# Описание ошибок {#errors}

## Структура ответа с ошибкой {#errors-format}

>Запрос с ошибкой карточных данных

~~~json
{
    "opcode": 1,
    "pan": "4",
    "expiry": "1010",
    "cvv2": "1",
    "amount": 4678.5,
    "card_name": "cardholder name",
    "merchant_site": 1000,
    "sign": "9FD874BF9D4483854E25D1D3371D01821AA4814CDFA4FE2033B5EDD4D5DE94C9"
}
~~~

>Ответ

~~~json
{
  "errors": [
    {
      "field": "pan",
      "message": "length of [pan] cannot be less than 13"
    },
    {
      "field": "expiry",
      "message": "card expired"
    },
    {
      "field": "cvv2",
      "message": "length of [cvv2] cannot be less than 3"
    }
  ],
  "error_message": "Validation errors",
  "error_code": 8019
}
~~~

>Запрос с ошибкой (пустой номер транзакции)

~~~json
{
  "opcode" : 6,
  "merchant_site" : "1234",
  "txn_id" : "",
  "sign" : "sadads",
  "amount" : "1000.01"
}
~~~

>Ответ

~~~json
{
  "error_message":"Parsing error",
  "error_code":8018
}
~~~

Ответ с ошибкой имеет следующую структуру:

Поле | Тип | Описание
-----|-------|---------
error_code | Number | [Код ощибки](#codes)
error_message | String | Описание ошибки
errors| Array of Objects | Подробное описание ошибок в параметрах
errors[].field | String | Параметр запроса, вызвавший ошибку
errors[].message | String | Текстовое описание ошибки

## Коды ошибок {#codes}

Код ошибки | Название | Описание
---------- | -------- | --------
0 | No errors | Нет ошибок.
8001 | Internal error | Техническая ошибка банка-эквайера.
8002 | Operation not supported | Операция не поддерживается.
8004 | Temporary error | Сервер занят, повторите запрос позднее.
8005 | Route not found | Не найден маршрут для проведения транзакции.
8006 | Card not supported | Некорректный номер карты.
8008 | No receiver data | Ошибка при передаче данных о получателе.
8018 | Parsing error | Ошибка разбора JSON запроса.
8019 | Validation error | Ошибки валидации входных параметров. Детали отражены в возвращаемом массиве `errors`.
8020 | Amount too big | Превышена сумма для операций reversal, refund.
8021 | Merchant site not found | Неизвестный `merchant_site`.
8022 | Transaction not found | Не найдена транзакция из запроса.
8023 | Transaction expired | Покупатель не смог аутентифицироваться с помощью 3DS в течение 15 минут.
8025 | Opcode is not allowed | Использование данного `opcode` запрещено для данного сайта.
8026 | Incorrect parent transaction | Некорректный статус родительской транзакции.
8027 | Incorrect parent transaction | Некорректный тип родительской транзакции.
8028 | Card expired | Срок действия карты истек.
8051 | Merchant disabled | Данный мерчант не активирован.
8052 | Incorrect transaction state | Некорректный статус транзакции, попытка выполнить операцию `capture` на finish_3ds или reversal.
8054 | Invalid signature | Некорректная подпись запроса.
8055 | Order already payed | Заказ уже оплачен.
8056 | In process | Транзакция по данной карте/заказу уже в процессе проведения.
8057 | Card locked | По данной карте уже проводится транзакция.
8058 | Access denied |
8059 | Currency is not allowed | Данная валюта запрещена.
8060 | Amount too big | Сумма превышает допустимую для выплаты.
8061 | Currency mismatch | Несовпадение валюты выплаты и оригинальной транзакции
8062 | Temporary error  | Сервер занят, повторите запрос позднее
8069 | Quantity limit of transactions is reached | Количество тестовых операций за сутки превышает [допустимое](#test_limit)
8070 | Amount of transaction is bigger than allowed | Сумма тестовой операции превышает [допустимую](#test_limit)
8151 | Authentication failed | Покупатель не смог аутентифицироваться с помощью 3DS.
8152 | Transaction rejected | Транзакция отклонена службой безопасности.
8153 | Reattempt not permitted | Повторный запрос авторизации запрещен на основании полученного ответа от Платежной системы
8160 | Transaction rejected | Ответ эмитента: Платеж отклонен. Попробуйте еще раз.
8161 | Transaction rejected | Ответ эмитента: Платеж отклонен. Попробуйте еще раз.
8162 | Transaction rejected | Ответ эмитента: Платеж отклонен. Попробуйте еще раз.
8163 | Transaction rejected | Ответ эмитента: Платеж отклонен. Обратитесь в поддержку QIWI.
8164 | Transaction rejected | Ответ эмитента: Платеж отклонен. Превышен лимит. Обратитесь в Банк, выпустивший карту.
8165 | Transaction rejected | Ответ эмитента: Платеж отклонен. Проверьте реквизиты платежа.
8166 | Transaction rejected | Ответ эмитента: Платеж отклонен. Проверьте реквизиты карты.
8167 | Transaction rejected | Ответ эмитента: Платеж отклонен. Проверьте реквизиты карты.
8168 | Transaction rejected | Ответ эмитента: Транзакция запрещена. Обратитесь в Банк, выпустивший карту.
8169 | Transaction rejected | Ответ эмитента: Платеж отклонен. Превышен лимит. Обратитесь в Банк, выпустивший карту.
8170 | Transaction rejected | Ответ эмитента: Платеж отклонен. Неверный смс код. Попробуйте ещё раз.

# Данные для тестирования {#test_mode}

Для тестирования операций оплаты используются [производственные URL](#urls) сервисов оплаты.

Для каждого нового ТСП в системе создаются `merchant_site` и по умолчанию для них включается режим тестирования. Также можно запросить переключение в режим тестирования любого своего `merchant_site`, либо добавление нового `merchant_site` в режим тестирования через вашего сопровождающего менеджера.

* В режиме тестирования можно использовать любой номер карты, удовлетворяющий алгоритму Luhn.
* В режиме тестирования из валют (параметр `currency`) разрешен только рубль РФ (`643`).
* CVV в режиме тестирования может быть любым (любые 3 цифры).

<h4>Тестовые номера карт</h4>
<h4 id="visa">
</h4>
<h4 id="mc">
</h4>
<button class="button-popup" id="generate">Получить номера карт</button>

Для тестирования различных вариантов оплаты и ответов необходимо использовать различные сроки действия карты:

* Если месяц срока действия — `02`, то операция будет проведена неуспешно.
* Если месяц срока действия — `03`, то операция будет проведена успешно с задержкой в 3 секунды.
* Если месяц срока действия — `04`, то операция будет проведена неуспешно с задержкой в 3 секунды.
* Во всех остальных случая операция выполняется успешно.

<a name="test_limit"></a>

В тестовой среде установлено ограничение на сумму и количество тестовых операций:

* Максимальная сумма тестовой транзакции — 10 рублей.
* Максимальное количество транзакций в сутки — 100. Учитываются операции за текущие сутки (время по МСК) с суммой каждой операции не более установленного лимита 10 рублей.

Для проведения операции с 3DS необходимо использовать фразу `unknown name` в имени держателя карты.
3DS в тестовом режиме можно проверить только при вводе номера реальной карты.
