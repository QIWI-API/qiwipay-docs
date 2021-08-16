---
title: QIWI PAY Reference

metatitle: QIWI PAY Reference

metadescription: QIWI PAY service is intended for card payment operations. The service allows Retail Service Provider to accept secure payments from their clients.

category: acquiring

language_tabs:
  - json: JSON
  - php: PHP
  - html: HTML
  - java: Java
  - shell

toc_footers:
  - <a href='/en/'>Home page</a>
  - <a href='mailto:bss@qiwi.com'>Feedback</a>

includes:

search: true
---

 *[WPF]: Web Payment Form - a payment form to enter card data for an item
 *[3DS]: 3-D Secure - a messaging protocol to exchange data during e-commerce transaction for consumer authentication purposes
 *[OCT]: Original Credit Transaction - a financial transaction that credits the Cardholder’s account
 *[API]: Application Programming Interface -  a set of ready-for-use methods that allow the creation of applications which access the features or data of an application or other service
 *[HMAC]: Hash-based message authentication code - a type of message authentication code involving a cryptographic hash function
 *[RSP]: Retail Service Provider
 *[PCI DSS]: Payment Card Industry Data Security Standard -  The Payment Card Industry Data Security Standard – a proprietary information security standard for storing, processing and transmitting credit card data
 *[REST]: Representational State Transfer -  a software architectural pattern for Network Interaction between distributed application components
 *[JSON]: JavaScript Object Notation - a lightweight data-interchange format based on JavaScript
 *[Luhn]: Luhn Algorithm - a checksum formula used for verifying and validating identification numbers against accidental errors
 *[PKCS#10]: RFC2986 Certificate requests standard
 *[HTTPS]: **HTTP** protocol extension to encrypt and enforce security. In HTTPS protocol, data transfer over SSL and TLS cryptographic protocols. In contrast with HTTP on port 80, the TCP 443 port is used by default for HTTPS

<!-- *[MPI]: Merchant Plug-In - a module integrated into QIWI PAY applications to process authentication transactions-->

# Introduction

###### Last update: 2021-06-02 | [Edit on GitHub](https://github.com/QIWI-API/qiwipay-docs/blob/master/qiwipay_en.html.md)

QIWI PAY service is intended for card payment operations. The service allows RSP to accept secure payments from their clients.

QIWI PAY supports the following operations: payment processing, a two-step payment confirmation, refunds, transaction status query.

# Integration Methods

You can use one of the following integration methods on QIWI PAY:

* QIWI PAY WPF — a web payment form on QIWI side. Quick and easy solution for accepting payments with limited functions (only payment transactions).
* QIWI PAY API — fully functional REST-based API for making card payments. All requests are POST-type and the parameters are in JSON body, as well as the responses. RSP implements only the Payment Form Interface.

<aside class="success">
QIWI PAY API requests with full card numbers are allowed by PCI DSS-certified merchants only, as Card Data are processed on the merchant's side.
</aside>


## Service URLs {#urls}

* To use QIWI PAY WPF, one is required to redirect client to the URL:

`https://pay.qiwi.com/paypage/initial`.

* To use QIWI PAY API, one is required to send HTTP POST requests to the URL:

`https://acquiring.qiwi.com/merchant/direct`.


# Operations {#opcode}

To indicate which operation is performed, you need to specify Operation code in `opcode` parameter of each request to API or on loading WPF pay form.

The following operations are accessible through QIWI PAY.

Op.Code | QIWI PAY WPF | QIWI PAY API | Operation | Financial? | Description
------------ | ----------- | ----------- | -------- | --------
1 | + | + | sale | Y | One-step payment
2 | - | + | finish_3ds | Depends on scenario | Return to web-site after 3DS authentication
3 | + | + | auth | N  |Auth request for two-step payments
5 | - | + | capture | Y |Confirm auth for two-step payment
6 | - | + | reversal | N|Cancel payment (funds returned almost immediately)
7 | - | + | refund | Y|Refund payment (funds returned within 30 days)
20 | - | + | payout | Y|Payout operation (OCT)
30 | - | + | status | N|Operation status query
40 | - | + | get_cards_by_token | N | Get linked cards list

<aside class="notice">
Financial operation means that there will be cash flows on bank accounts.
</aside>  

# Operation Methods {#methods}

## QIWI PAY Flows

Two payment flows can be used in QIWI PAY:

* [One-step](#onestep) — `sale` operation
* [Two-step](#twostep) — `auth` operation -> `capture` operation

As a rule, two-step scenario is used when RSP makes verification of service availability after payment is made. Between `auth` operation and `capture` operation, RSP can make `reversal` operation which is not a financial one.

For `sale` operation, RSP can make `reversal` but only until the day ends and not for all acquiring banks. Your support manager can provide details.

To make sure you know which operation may be applied, check transaction status and [statuses table](#txn_status).

<aside class="warning">
The longest period between <code>auth</code> and <code>capture</code> operations is 72 hours. Therefore, if no <code>capture</code> operation comes within 72 hours from RSP since <code>auth</code> is received, it will be performed anyway. 
</aside>

## 3DS {#threeds}

3DS (3-D Secure) is a common notation for technological frameworks of  Visa and MasterCard (Verified By Visa and MasterCard Secure Code, correspondingly). The main goal is to verify the Cardholder's identity by the Issuer before a payment and to prevent card data theft. In practice it work like that. The Cardholder enters his/her card details. The Issuer’s secure web page pops up requesting for verification data (a password or a secret code) to be sent by the Cardholder via SMS. If verification data is valid the payment is processed, if not valid it is rejected.

`sale` operation via QIWI PAY can be 3DS-validated if the Issuer requires 3DS authentication.

QIWI PAY WPF Payment flow with 3DS verification is explained on the following scheme.

<!--img src="/images/3ds_en.png" class="image" /-->
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
Customer->>qp:Send card data from payment web-form
activate Customer
activate qp 
qp->>ips:Send VEReq <br>to verify card for<br>3-D Secure enrollment
activate ips 
ips->>Issuer:Request to ACS<br>to verify card enrollment
activate Issuer 
Issuer-->>ips:Return verify response (VERes)
deactivate Issuer
ips-->>qp:Return verify response (VERes)
deactivate ips
qp->>qp:PAReq generation from VERes
qp-->>Customer:Redirect from PAReq to ACS URL<br>taken from VERes<br>to enter 3-D Secure password
deactivate qp
deactivate Customer
Customer->>Issuer:Redirect to ACS URL
activate Customer 
activate Issuer 
Issuer-->>Customer:One-time password SMS
Customer->>Issuer:Enter one-time password
Issuer->>ips:Send ACS authentication<br>result data
Issuer-->>Customer:Redirect to<br>QIWI PAY page<br>with PARes
deactivate Customer
deactivate Issuer 
Customer->>qp:Redirect to QIWI PAY page
activate Customer 
activate qp 
qp->>qp:PARes validity verification
qp->>qb:Authorization request<br>with PARes data
deactivate qp
deactivate Customer
</div>



# QIWI PAY WPF {#wpf}

RSP only redirects customers to https://pay.qiwi.com/payform. Then customers complete the payment form on QIWI PAY web-site with their card details.

## Payment Flow

QIWI PAY WPF one-step payment flow is explained on the following scheme.

<!--img src="/images/wpf_sale_en.png" class="image" /-->

<div class="mermaid">
sequenceDiagram 
participant Customer
participant Merchant
participant QiwiPay as QIWI PAY 
participant QiwiBank as QIWI Bank
Customer->>Merchant:Placing order,<br>choose card payment
activate Merchant
Merchant-->>Customer:Redirect to payment web-form
deactivate Merchant
Customer->>QiwiPay:Request for payment web-form
activate QiwiPay
QiwiPay-->>Customer:Payment web-form
deactivate QiwiPay
Customer->>QiwiPay:Enter card data and submit form
activate QiwiPay
QiwiPay->>QiwiPay:Initial verification of parameters,<br>creating transaction
QiwiPay->>QiwiPay:Anti-fraud module
rect rgb(237, 243, 255)
Note over Customer, QiwiPay: 3-D Secure
QiwiPay->>QiwiPay:Verify card for 3-D Secure enrollment
QiwiPay-->>Customer:Redirect to ACS URL
deactivate QiwiPay
Customer->>QiwiPay:Return from ACS page<br>with 3-D Secure verification results
activate QiwiPay
end
QiwiPay->>QiwiBank:Authorization request
activate QiwiBank
QiwiBank-->>QiwiPay:Successful authorization
deactivate QiwiBank 
rect rgb(237, 243, 255)
Note over Customer, QiwiPay: Immediate notification
QiwiPay->>Merchant:Notification on operation<br>result (callback)
activate Merchant
Merchant-->>QiwiPay:Confirm receiving<br>notification
deactivate Merchant
end
QiwiPay->>Customer:Status page with operation result
QiwiPay->>Customer:E-mail notification
deactivate QiwiPay
Customer->>Merchant:Return to shop<br>web-site from<br>status page
</div>



## Redirect to QIWI PAY WPF

To redirect, use standard web form HTTP POST data request.

~~~html
<form method="post" action="https://pay.qiwi.com/paypage/initial">
  <input name="opcode" type="hidden" value="1">
  <input name="merchant_site" type="hidden" value="99">
  <input name="currency" type="hidden" value="643">
  <input name="sign" type="hidden" value="bb5c48ea540035e6b7c03c8184f74f09d26e9286a9b8f34b236b1bf2587e4268">
  <input name="amount" type="text">
  <input type="submit" value="Pay">
</form>
~~~

No authorization is required for RSP.

<aside class="notice">
In case of the two-step flow, use only the first operation (<code>auth</code>).
</aside>

Use the following form fields for payment operations. All fields not specified by the customer should be with `hidden` type.

Parameter|Required|Data type|Description
--------|-------|----------|--------
opcode|Yes|integer|Operation code (`1`, `3` only)
merchant_site|Yes|integer|RSP site ID (issued by QIWI)
currency|Yes|integer|Operation currency by ISO 4217
[sign](#sign)|Yes|string(64)|Checksum of the request's parameters
amount|No|decimal|Operation amount
order_id|No|string(256)|Unique order ID assigned by RSP system
email|No|string(64)|Customer E-mail
country|No|string(3)|Customer Country by ISO 3166-1
city|No|string(64)|Customer City of residence
region|No|string(6)|Customer Region (geo code) by ISO 3166-2
address|No|string(64)|Customer Address of residence
phone|No|string(15)|Customer contact phone number
cf1|No|string(256)|Arbitrary information about the operation, such as list of services
cf2|No|string(256)|Arbitrary information about the operation, such as list of services
cf3|No|string(256)|Arbitrary information about the operation, such as list of services
cf4|No|string(256)|Arbitrary information about the operation, such as list of services
cf5|No|string(256)|Arbitrary information about the operation, such as list of services
cf5|No|string(256)|Arbitrary information about the operation, such as list of services
product_name|No|string(25)|Service description for the Customer
merchant_uid|No|string(64)|Unique Customer ID assigned by RSP system
order_expire|No|YYYY-MM-DDThh:mm:ss±hh:mm|Order expiration date by ISO8601 with time zone
callback_url|No|string(256)|[Callback URL](#callback)
success_url|No|string(256)|Redirect URL when payment is successful
decline_url|No|string(256)|Redirect URL when payment is unsuccessful


# QIWI PAY API {#api}

## Authorization {#api_auth}

QIWI PAY API authorizes the request by RSP client certificate validation. Certificate is issued by QIWI for each RSP.

<aside class="success">
To obtain certificate, please submit certificate request in .csr PKCS#10 format to QIWI Support (bss@qiwi.com).
</aside>

## Payment Flow {#flows}

### One-step payment flow {#onestep}

<!--img src="/images/api_sale_en.png" class="image" /-->

<div class="mermaid">
sequenceDiagram
participant Customer
participant Merchant
participant QiwiPay as QIWI PAY
participant QiwiBank as QIWI Bank
Customer->>Merchant:Send card data<br>from payment web-form
activate Merchant
Merchant->>QiwiPay:"sale" operation<br>request
activate QiwiPay
QiwiPay->>QiwiPay:Anti-fraud module
rect rgb(237, 243, 255)
Note over Customer, QiwiPay: 3-D Secure
QiwiPay->>QiwiPay:Check card<br>3-D Secure enrollment
QiwiPay-->>Merchant:Redirect ACS<br>Issuing Bank parameters<br>for Customer
deactivate QiwiPay
Merchant-->>Customer:Redirect to ACS URL
deactivate Merchant
Customer->>Merchant:Return with 3-D Secure<br>verificaiton result
activate Merchant
Merchant->>QiwiPay:"finish_3ds" operation<br>request
activate QiwiPay
end
QiwiPay->>QiwiBank:Authorization request
activate QiwiBank
QiwiBank-->>QiwiPay:Successful authorization
QiwiPay->>QiwiBank:Authorized financial operation
QiwiBank-->>QiwiPay:Confirmed financial operation
deactivate QiwiBank
rect rgb(237, 243, 255)
Note over Customer, QiwiPay: Immediate notification
QiwiPay->>Merchant:Notification on<br>operation result (callback)
activate Merchant
Merchant-->>QiwiPay:Confirm receiving notification
deactivate Merchant
end
QiwiPay->>Merchant:Operation result
QiwiPay->>Customer:E-mail notification
deactivate QiwiPay
Merchant->>Customer:Status page<br>with operation result
deactivate Merchant
</div>


### Two-step payment flow {#twostep}

<!--img src="/images/api_auth_en.png" class="image" /-->

<div class="mermaid">
sequenceDiagram
participant Customer
participant Merchant
participant QiwiPay as  QIWI PAY
participant QiwiBank as QIWI Bank
Customer->>Merchant:Send card data<br>from payment web-form
activate Merchant 
Merchant->>QiwiPay:<br>"auth" operation<br>request 
activate QiwiPay 
QiwiPay->>QiwiPay:Anti-fraud module
rect rgb(237, 243, 255)
Note over Customer, QiwiPay: 3-D Secure
QiwiPay->>QiwiPay:Check card 3-D Secure<br>enrollment
QiwiPay-->>Merchant:Redirect ACS Issuing Bank<br>parameters for Customer
deactivate QiwiPay
Merchant-->>Customer:Redirect to ACS URL
deactivate Merchant
Customer->>Merchant:Return with 3-D Secure<br>verification result
activate Merchant 
Merchant->>QiwiPay:"finish_3ds" operation<br>request
activate QiwiPay 
end
QiwiPay->>QiwiBank: Auhtorization request
activate QiwiBank 
QiwiBank-->>QiwiPay: Successful authorization
deactivate QiwiBank 
rect rgb(237, 243, 255)
Note over Customer, QiwiPay: Immediate notification
QiwiPay->>Merchant:Notification on<br>operation result (callback)
activate Merchant 
Merchant-->>QiwiPay:Confirm receiving notification
deactivate Merchant
end
QiwiPay->>Merchant:Operation result
QiwiPay->>Customer:E-mail notification
deactivate QiwiPay
Merchant->>Customer:Status page<br>with operation result
deactivate Merchant
Note over Merchant: Awaiting for request confirming authorization
Merchant-->>QiwiPay:"capture" operation<br>request
activate Merchant 
activate QiwiPay 
QiwiPay->>QiwiBank:Financial operation<br>after authorization
activate QiwiBank 
QiwiBank-->>QiwiPay:Confirmed financial operation
deactivate QiwiBank
QiwiPay->>Merchant:Notification on<br>operation result (callback)
deactivate QiwiPay
deactivate Merchant
</div>


## Sale Operation {#sale}

### Request

~~~json
{
  "opcode": 1,
  "merchant_uid": "10001",
  "merchant_site": 99,
  "pan": "4111111111111111",
  "expiry": "1219",
  "cvv2": "123",
  "amount": 4678.5,
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
  "user_timedate": "26.08.2016",
  "user_screen_res": 1280,
  "user_agent": "Mozilla FF 312",
  "cf1": "cf1",
  "cf2": "cf2",
  "cf3": "cf3",
  "cf4": "cf4",
  "cf5": "cf5",
  "callback_url": "http://domain.tld/callback_service",
  "product_name": "Flowers",
  "receiver_name": "Ivanov, Ivan",
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e9286a9b8f34b236b1bf2587e4268"
}
~~~

All payment operations (`sale`, `auth`) use the same set of parameters.

Parameter|Required|Data type|Description
--------|-------|----------|--------
opcode|Yes|integer|[Operation code](#opcode) (`1`, `3`)
merchant_site|Yes|integer|RSP site ID (issued by QIWI)
pan|Yes|string(19)|Card number (PAN)
expiry|Yes|string(4)|Card expiry date
card_token |Yes, if available|string(40)|Card token
cvv2|Yes|string(4)|CVV2/CVC2 code
amount|Yes|decimal|Operation amount
currency|Yes|integer|Operation currency by ISO 4217
sign|Yes|string(64)|[Checksum of the request's parameters](#sign)
card_name|Yes, if available|string(64)|Cardholder name as printed on the card (Latin letters)
order_id|Yes, if available|string(256)|Unique order ID assigned by RSP system
ip|Yes, if available|string(15)|Customer IP address
email|Yes, if available|string(64)|Customer E-mail
country|Yes, if available|string(3)|Customer Country by ISO 3166-1
user_device_id|Yes, if available|string(64)|Customer Unique device ID
city|Yes, if available|string(64)|Customer City of residence
region|Yes, if available|string(6)|Customer Region (geo code) by ISO 3166-2
address|Yes, if available|string(64)|Customer Address of residence
phone|Yes, if available|string(15)|Customer contact phone number
user_timedate|No|string(64)|Local time in Customer browser
user_screen_res|No|string(64)|Screen resolution of the Customer's display
user_agent|No|string(256)|Customer browser
cf1|No|string(256)|Arbitrary information about the operation, such as list of services
cf2|No|string(256)|Arbitrary information about the operation, such as list of services
cf3|No|string(256)|Arbitrary information about the operation, such as list of services
cf4|No|string(256)|Arbitrary information about the operation, such as list of services
cf5|No|string(256)|Arbitrary information about the operation, such as list of services
product_name|No|string(25)|Service description for the Customer
merchant_uid|No|string(64)|Unique Customer ID assigned by RSP system
order_expire|No|YYYY-MM-DDThh24:mm:ss|Order expiration date by ISO8601
callback_url|No|string(256)|[Callback URL](#callback)
cheque|No|string|[Receipt data](#cheque)
apple_pay_encoded_payment_token|No|string|Encrypted payment data [from Apple Pay](#applepay)
wallet_type|No|string(50)|Type of a wallet to top up. Possible values: `QIWI`
account_id|No|integer(50)|Number of a wallet to top up. **Required if wallet_type=QIWI is in the operation**
receiver_name|No|string(30)|Last and first name of the payment's receiver separated by `, ` (comma followed by space). Order matters: last name should be before first name.  Only Latin letters, digits, and symbols `_`, `-` (first name can be with spaces).

### Response for card with no 3DS {#sale_no3ds}

~~~json
{
  "txn_id":172001,
  "txn_status":60,
  "txn_type":1,
  "txn_date": "2017-03-09T17:16:06+00:00",
  "card_token": "4d5b363e-a116-35f5-e053-6751080ac38e",
  "card_token_expire": "2018-02-27T21:00:00+00:00",
  "error_code":0,
  "pan": "411111******1111",
  "issuer_name": "QIWI BANK (JSC)",
  "issuer_country": "RUS",  
  "amount": 4678.5,
  "currency": 643,
  "auth_code": "2G4923",
  "eci": "05"
}
~~~

Field|Data type|Description
--------|----------|--------
txn_id | integer | Transaction ID
txn_status | integer | [Transaction status](#txn_status)
txn_type | integer | [Transaction type](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Transaction date, in ISO8601 with time zone
card_token | string(40) | Card token (if tokenization is enabled for the merchant site)
card_token_expire | YYYY-MM-DDThh:mm:ss±hh:mm | Expiry date for the card token (if tokenization is enabled for the merchant site)
error_code | integer | [System error code](#errors)
pan | string(19) | Customer Card number (PAN)
issuer_name | string(40) | Issuer bank name
issuer_country | string(3) | Issuer bank country
amount | decimal | Amount to debit from card account
currency | integer | Operation currency by ISO 4217
auth_code | string(6) | Authorization code
eci | string(2) | Electronic Commerce Indicator
is_test | string(4) | Presence of this parameter with `true` value indicates that transaction is processed in [testing mode](#test_mode) and no card debiting is made

### Response for card with 3DS {#sale_3ds}

~~~json
{
  "txn_id":172001,
  "txn_status":40,
  "txn_type":1,
  "txn_date": "2017-03-09T17:16:06+00:00",
  "error_code":0,
  "acs_url":"https://test.paymentgate.ru/acs/auth/start.do",
  "pareq":"eJxVUmtvgjAUuG79oClYe51uDcsi2B+ZrJPgKtipGHRnUTgy1w5bVwyf4BfjpuJ....."
}
~~~

Field|Data type|Description
--------|----------|--------
txn_id | integer | Transaction ID
txn_status | integer | [Transaction status](#txn_status)
txn_type | integer | [Transaction type](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Transaction date, by ISO8601 with time zone
error_code | integer | [System error code](#errors)
acs_url | string(1024) | Issuing Bank URL where to redirect Customer
pareq | string(4096) | Unique Customer ID to use for `acs_url` redirect.
is_test | string(4) | Presence of this parameter with `true` value indicates that transaction is processed in [testing mode](#test_mode) and no card debiting is made


You need to redirect Customer to 3DS standard redirection URL.

## Finish 3DS authentication {#finish_3ds}

When Customer returns from 3DS standard redirection URL and completes 3DS authentication successfully, you need to send the following request to finish 3DS authentication process.

### Request after successful 3DS authentication

~~~json
{
  "opcode":2,
  "merchant_site":99,
  "pares": "eJzVWFevo9iyfu9fMZrzaM0QjWHk3tIiGptgooE3cgabYMKvv3jvTurTc3XOfbkaJMuL",
  "txn_id": "172001"
}
~~~

Parameter|Required|Data type|Description
--------|-------|----------|--------
opcode | Yes | integer | [Operation Code](#opcode) (`2`)
merchant_site | Yes | integer | RSP site ID (issued by QIWI)
pares | Yes | string(4096) | Customer verification result
txn_id | Yes | integer | Transaction ID
sign | Yes | string(64) | [Checksum of the request's parameters](#sign)

### Response to 3DS authentication result

~~~json
{
  "txn_id":172001,
  "txn_status":60,
  "txn_type":1,
  "txn_date": "2017-03-09T17:16:06+00:00",
  "card_token": "4d5b363e-a116-35f5-e053-6751080ac38e",
  "card_token_expire": "2018-02-27T21:00:00+00:00",
  "error_code":0,
  "pan": "411111******1111",
  "issuer_name": "QIWI BANK (JSC)",
  "issuer_country": "RUS",
  "amount": 4678.5,
  "currency": 643,
  "auth_code": "2G4923",
  "eci": "05"
}
~~~

Field|Data type|Description
--------|----------|--------
txn_id | integer | Transaction ID
txn_status | integer | [Transaction status](#txn_status)
txn_type | integer | [Transaction type](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Transaction date, by ISO8601 with time zone
card_token | string(40) | Card token (if tokenization is enabled for the merchant site)
card_token_expire | YYYY-MM-DDThh:mm:ss±hh:mm | Expiry date for the card token (if tokenization is enabled for the merchant site)
error_code | integer | [System error code](#errors)
pan | string(19) | Customer Card number (PAN)
issuer_name | string(40) | Issuer bank name
issuer_country | string(3) | Issuer bank country
amount | decimal | Amount to debit from card account
currency | integer | Operation currency by ISO 4217
auth_code | string(6) | Authorization code
eci | string(2) | Electronic Commerce Indicator
is_test | string(4) | Presence of this parameter with `true` value indicates that transaction is processed in [testing mode](#test_mode) and no card debiting is made


## Sale Confirmation Operation {#sale_confirm}

### Request

~~~json
{
  "opcode": 5,
  "merchant_site": 99,
  "txn_id": "172001",
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e9286a9b8f34b236b1bf2587e4268"
}
~~~

Parameter|Required|Data type|Description
--------|-------|----------|--------
opcode|Yes|integer|[Operation code](#opcode) (`5`)
merchant_site|Yes|integer|RSP site ID (issued by QIWI)
txn_id | Yes | integer | Transaction ID
cheque|No|string|[Receipt data](#cheque)
sign | Yes | string(64) | [Checksum of the request's parameters](#sign)

### Response

~~~json
{
  "txn_id":172001,
  "txn_status":60,
  "txn_type":1,
  "txn_date": "2017-03-09T17:16:06+00:00",
  "error_code":0
}
~~~

Field|Data type|Description
--------|----------|--------
txn_id | integer | Transaction ID
txn_status | integer | [Transaction status](#txn_status)
txn_type | integer | [Transaction type](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Transaction date, by ISO8601 with time zone
error_code | integer | [System error code](#errors)
is_test | string(4) | Presence of this parameter with `true` value indicates that transaction is processed in [testing mode](#test_mode) and no card debiting is made


## Cancel Operation {#cancel}

### Request

~~~json
{
  "opcode":6,
  "merchant_site": 99,
  "txn_id": 181001,
  "amount": 700,
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e9286a9b8f34b236b1bf2587e4268"
}
~~~

Parameter|Required|Data type|Description
--------|-------|----------|--------
opcode|Yes|integer|[Operation code](#opcode) (`6`)
merchant_site|Yes|integer|RSP site ID (issued by QIWI)
txn_id | Yes | integer | Transaction ID
amount|No|decimal|Operation amount
sign | Yes | string(64) | [Checksum of the request's parameters](#sign)
cheque|No|string|[Receipt data](#cheque)

### Response

~~~json
{
  "txn_id":182001,
  "txn_status":72,
  "txn_type":4,
  "txn_date": "2017-03-09T17:16:06+00:00",
  "error_code":0,
  "amount": 700
}
~~~

Field|Data type|Description
--------|----------|--------
txn_id | integer | Transaction ID
txn_status | integer | [Transaction status](#txn_status)
txn_type | integer | [Transaction type](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Transaction date, by ISO8601 with time zone
error_code | integer | [System error code](#errors)
amount | decimal | Amount to debit
is_test | string(4) | Presence of this parameter with `true` value indicates that transaction is processed in [testing mode](#test_mode) and no real funds return to card is made


## Refund Operation {#refund}

### Request

~~~json
{
  "opcode":7,
  "merchant_site": 99,
  "txn_id": 181001,
  "amount": 700,
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e9286a9b8f34b236b1bf2587e4268"
}
~~~

Parameter|Required|Data type|Description
--------|-------|----------|--------
opcode|Yes|integer|Operation code (`7`)
merchant_site|Yes|integer|RSP site ID (issued by QIWI)
txn_id | Yes | integer | Transaction ID
amount|No|decimal|Operation amount
sign | Yes | string(64) | [Checksum of the request's parameters](#sign)
cheque|No|string|[Receipt data](#cheque)

### Response

~~~json
{
  "txn_id":182001,
  "txn_status":77,
  "txn_type":3,
  "txn_date": "2017-03-09T17:16:06+00:00",
  "error_code":0,
  "amount": 700
}
~~~

Field|Data type|Description
--------|----------|--------
txn_id | integer | Transaction ID
txn_status | integer | [Transaction status](#txn_status)
txn_type | integer | [Transaction type](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Transaction date, by ISO8601 with time zone
error_code | integer | [System error code](#errors)
amount | decimal | Amount to debit
is_test | string(4) | Presence of this parameter with `true` value indicates that transaction is processed in [testing mode](#test_mode) and no real funds return to card is made

## Payout Operation {#payout}

### Request

~~~json
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
  "product_name": "Award payment",
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e..........................."
}
~~~

Parameter|Required|Data type|Description
--------|-------|----------|--------
opcode|Yes|integer|Operation code (`20`)
merchant_site|Yes|integer|RSP site ID (issued by QIWI)
card_token | string(40) | Card token (if tokenization is enabled for the merchant site)
txn_id | Yes | integer | Transaction ID
amount|Yes|decimal|Operation amount
currency|Yes|integer|Operation currency by ISO 4217
card_name|Yes, if available|string(64)|Cardholder name as printed on the card (Latin letters)
order_id|Yes, if available|string(256)|Unique order ID assigned by RSP system
cf1|No|string(256)|Arbitrary information about the operation, such as list of services
cf2|No|string(256)|Arbitrary information about the operation, such as list of services
cf3|No|string(256)|Arbitrary information about the operation, such as list of services
cf4|No|string(256)|Arbitrary information about the operation, such as list of services
cf5|No|string(256)|Arbitrary information about the operation, such as list of services
sign|Yes|string(64)|[Checksum of the request's parameters](#sign)
callback_url|No|string(256)|[Callback URL](#callback)
product_name|No|string(25)|Service description for the Customer
cheque|No|string|[Receipt data](#cheque)


### Response

~~~json
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

Field|Data type|Description
--------|----------|--------
txn_id | integer | Transaction ID
txn_status | integer | [Transaction status](#txn_status)
txn_type | integer | [Transaction type](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Transaction date, by ISO8601 with time zone
error_code | integer | [System error code](#errors)
pan | string(19) | Card number
amount | decimal | Amount to pay
currency | integer | Operation currency by ISO 4217
auth_code | string(6) | Authorization code


## Status Query Operation {#status}

### Request

~~~json
{
  "opcode":30,
  "merchant_site": 99,
  "order_id": "41324123412342",
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e9286a9b8f34b236b1bf2587e4268"
}
~~~

Parameter|Required|Data type|Description
--------|-------|----------|--------
opcode|Yes|integer|Operation code (`30`)
merchant_site|Yes|integer|RSP site ID (issued by QIWI)
txn_id | Yes | integer | Transaction ID
sign|Yes|string(64)|[Checksum of the request's parameters](#sign)
order_id|No|string(256)|Unique order ID assigned by RSP system

### Response

~~~json
{
  "transactions": [
    {
      "error_code": 0,
      "txn_id": 3666050,
      "txn_status": 52,
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
      "txn_status": 72,
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
      "txn_status": 72,
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

Field|Data type|Description
--------|-------|----------|--------
txn_id | integer | Transaction ID
txn_status | integer | [Transaction status](#txn_status)
txn_type | integer | [Transaction type](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | Transaction date, by ISO8601 with time zone
error_code | integer | [System error code](#errors)
pan|string(19)|Masked Card number (PAN), first six and last four digits are unmasked
amount|decimal|Amount to debit
currency|integer|Operation currency by ISO 4217
auth_code | string(6) | Authorization code
eci | string(2) | Electronic Commerce Indicator
card_name|string(64)|Cardholder name as printed on the card (Latin letters)
card_bank|string(64)|Issuing Bank
order_id|string(256)|Unique order ID assigned by RSP system
ip|string(15)|Customer IP address
email|string(64)|Customer E-mail
country|string(3)|Customer Country by ISO 3166-1
city|string(64)|Customer City of residence
region|string(6)|Customer Region (geo code) by ISO 3166-2
address|string(64)|Customer Address of residence
phone|string(15)|Customer contact phone number
cf1|string(256)|Arbitrary information about the operation, such as list of services
cf2|string(256)|Arbitrary information about the operation, such as list of services
cf3|string(256)|Arbitrary information about the operation, such as list of services
cf4|string(256)|Arbitrary information about the operation, such as list of services
cf5|string(256)|Arbitrary information about the operation, such as list of services
product_name|string(25)|Service description for the Customer

# Signature of the Request {#sign}

~~~java
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

>For parameters string amount\|currency\|merchant_site\|opcode

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

You should sign all parameters in each operation request. Add the signature in `sign` parameter for each request.

To calculate signature, use HMAC algorithm with SHA256-hash function. For this, you need `secret key` parameter obtained with other integration settings.

* Separator is `|`.
* Parameters are in alphabetical order, i.e. for parameters `amount|currency|merchant_site|opcode` string for signature verification is: `15.00|643|12345|1`.
* Signed are all the parameters in the request.
* Do not include parameters with empty values
* Only values are signed, not their names.

# Payment Notification {#callback}

There are two types of notification flow:

* During payment processing, when card debit is confirmed and before showing status page or RSP response (depends on what flow is used, WPF or API).
* On background, when payment processed.

Notification goes to `callback_url` parameter from the original request, as a POST request with the following parameters.

<aside class="success">
Callback is sent by HTTPS protocol on 443 port only.
Certificate should be issued by any trusted center of certification (e.g. Comodo, Verisign, Thawte etc)
</aside>


Callback field|Data type| Used for signature | Description
--------|----------|------------------|---------
txn_id | integer | + | Transaction ID
txn_status | integer | + | [Transaction status](#txn_status)
txn_type | integer | + | [Transaction type](#txn_type)
txn_date | YYYY-MM-DDThh:mm:ss±hh:mm | - | Transaction date, by ISO8601 with time zone
error_code | integer | + | [System error code](#errors)
pan | string(19) | - | Customer Card number (PAN), first six and last four digits are unmasked
amount | decimal | + | Amount to debit
currency | integer | + | Operation currency by ISO 4217
auth_code | string(6) | - | Authorization code
eci | string(2) | - | Electronic Commerce Indicator
card_name | string(64) | - | Cardholder name as printed on the card (Latin letters)
card_bank | string(64) | - | Issuing Bank
order_id | string(256) | - | Unique order ID assigned by RSP system. Returned if sent in the original request.
ip | string(15) | + | Customer IP address
email | string(64) | + | Customer E-mail
country | string(3) | - | Customer Country by ISO 3166-1
city | string(64) | - | Customer City of residence
region | string(6) | - | Customer Region (geo code) by ISO 3166-2
address | string(64) | - | Customer Address of residence
phone | string(15) | - | Customer contact phone number
cf1 | string(256) | - | Arbitrary information about the operation, such as list of services
cf2 | string(256) | - | Arbitrary information about the operation, such as list of services
cf3 | string(256) | - | Arbitrary information about the operation, such as list of services
cf4 | string(256) | - | Arbitrary information about the operation, such as list of services
cf5 | string(256) | - | Arbitrary information about the operation, such as list of services
product_name | string(25) | - | Service description for the Customer
card_token | string(40) | - |Card token (if tokenization is enabled for the merchant site)
card_token_expire | YYYY-MM-DDThh:mm:ss±hh:mm | - |Expiry date for the card token (if tokenization is enabled for the merchant site)
sign | string(64) | - | Checksum (signature) of the request

To minimize possible fraud notifications, RSP should verify callback signature located in `sign` field.

<aside class="notice">
Signature algorithm coincides with ordinary QIWI PAY requests, though only the parameters marked above with a plus sign in <i>Used for signature</i> column are taken.
</aside>

To guarantee QIWI origin of the notifications, accept only these IP addresses related to QIWI:

* 79.142.16.0/20
* 195.189.100.0/22
* 91.232.230.0/23
* 91.213.51.0/24

Notification is treated as successfully delivered when RSP server responds with HTTP code `200 OK`.
So far, during the first 24 hours after the operation, the system will continue to deliver notification with incremental time intervals until RSP server responds with HTTP code `200 OK`.

<aside class="notice">
Due to any reason, if RSP accepts notification without HTTP code <i>200 OK</i> then the same transaction on the repeated notification should not be treated as the new one.
</aside>

<aside class="warning">
RSP is solely responsible for possible financial loss when <code>sign</code> is not verified for the illegal notifications.
</aside>

# Industry Data {#industry_data}

## Avia

~~~json
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

Avia data are transferred in `industry_data` parameter for `auth`, `capture`, `sale` operations of QIWI PAY API.
In case of any necessary data are absent on the moment of `auth` operation, it can be sent in `capture` operation after receiving it.

Avia data should include booked tickets as well as all segments (flights) from each ticket.

Parameter|Required|Data type|Description
--------|-------|----------|--------
type|Y|string|Data type (only `avia`)
pnr|N|string(6)|Booking number
tickets|Y|array|Bunch of tickets
--------|-------|----------|--------
number|Y|string(14)|Ticket number
passenger_name|Y|string(20)|Passenger name
amount|Y|decimal|Ticket price without airport fee
amount_total|Y|decimal|Ticket price with airport fee
restricted|Y|boolean|Restricted Ticked Indicator
agency_code|Y|string(8)|Travel agency code
agency_name|Y|string(25)|Travel agency title
airport_code|Y|string(3)|Airport of departure (3 letters by IATA code)
segments|Y|array|Avia segments
--------|-------|----------|--------
amount|Y|decimal|Ticket price for the segment
service_class|Y|string(1)|Flight service class
fare_basis_code|Y|string(6)|Flight fare code
dest_city|Y|string(3)|Airport of destination (3 letters by IATA code)
carrier_code|Y|string(2)|Airline code
stopover_code|Y|string(1)|Stop-Over Code
departure_date|Y|YYYY-MM-DDThh:mm:ss±hh:mm|Departure date, ISO8601 with time zone
flight_number|Y|string(5)|Flight number, digits only


# Safe Purchase {#safe_deal}

Safe Purchase service from QIWI provides a transparent and reliable payment mechanism for work services without the risk of losing money for both the customer and the worker. To connect a Safe Purchase service, contact your support manager.

There are two types of safe purchase:

* With a payout on the QIWI wallet
* With a payout on the cards.

The operation is authorized by both the [QIWI PAY API](#api) and the [QIWI PAY WPF](#wpf).

## Working with partial amounts {#safe_deal_subsums}

> Case Of authorization

~~~json
{
  "opcode": 1,
  "merchant_uid": "10001",
  "merchant_site": 555,
  "pan": "41111111111111111",
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

To use the protocol of a safe purchase, you need to divide the amount of authorization into partial amounts:

* The partial amount of payouts,
* The partial amount of the commissions.

The partial amount of payouts is funds that will be paid to the worker at the end of a safe purchase.

The partial amount of commissions is the amount of commissions of the Marketplace on which the transaction is performed, and QIWI. From this amount, QIWI will hold the commission for the operation service, and the remainder will be transferred to the Marketplace.

The amounts are transmitted in the [authorization request](#sale) or on the [WPF calling](#redirect) in the `cf2` field separated by semicolon: 

`"cf2": "<payout partial amount>; <commissions partial amount>"`

<aside class="success">Amount of authorization (<code>amount</code> value) should equal the sum of its partial amounts. Example:<br><code>"amount": "500.00"</code><br><code>"cf2": "300.00; 200.00"</code>
</aside>

## Safe purchase with a QIWI wallet payout {#safe_deal_wallet}

This type of safe purchase consists of two stages:

* Authorization of charging funds through the request to [API](#sale) or by [calling WPF](#redirect-to-qiwi-pay-wpf) (Operation code `3`).
* [Confirmation of the operation](#sale_confirm) (Operation code `5`).

At each stage, be sure to specify the number of the QIWI Wallet in the `cf1` field of the request in the format

`"cf1": "79111111111"`

If the transaction is confirmed, the amount to be paid will automatically be credited to the QIWI Wallet specified in the request.

<aside class="notice">Max waiting time between authorization and confirmation is 5 days</aside>

> Case Of cancellation

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

The transaction can be cancelled in full or in part only until the transaction is confirmed. To cancel, make a [Cancel request](#cancel) (Operation code `6`) and specify in the `cf2` field a partial amounts of the canceled transaction (in the same format as for authorization). 

<aside class="success">Amount of cancel operation (<code>amount</code> value) should equal the sum of its partial amounts.</aside>

## Safe purchase with a card payout {#safe_deal_card}

This type of safe purchase consists of three stages:

* Authorization of charging funds through the request to [API](#sale) or by [calling WPF](#redirect-to-qiwi-pay-wpf) (Operation codes `1`, `3`).
* [Confirmation of the operation](#sale_confirm) (Operation Code `5`).
* [Payout operation](#payout) (Operation Code `20`).

> Payout Example

~~~json
{
  "opcode": 20,
  "merchant_site": "10001",
  "merchant_uid": 555,
  "card_token": "4d5b363e-a116-35f5-e053-6751080ac38e",
  "txn_id":182001,  "amount": "4678.50",
  "currency": 643,
  "card_name": "cardholder name",
  "order_id": "order1231231",
  "callback_url": "http://domain.tld/callback_service",
  "success_url": "http://domain.tld/success",
  "decline_url": "http://domain.tld/decline",
  "product_name": "Winning Payout"
  "sign": "bb5c48ea540035e6b7c03c8184f74f09d26e..........................."
}
~~~

Once the operation is confirmed, there are 30 days to make the payment operation. Funds are deposited on the recipient's card within three days.

<aside class="warning">Organizations without PCI DSS can only make payments using card tokens.</aside>

> Case Of Cancellation

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

You can cancel the transaction in full or in part before the transaction is confirmed, and make a refund after confirmation if the payment has not yet been made. 

To make cancel or refund an operation, make a [Cancel](#cancel) (Operation code `6`) or [Refund](#refund) (Operation code `7`) requests respectively and specify in the `cf2` field a partial amounts of the cancelled transaction (in the same format as for the authorization). 

<aside class="success">Amount to cancel (<code>amount</code> value) should equal the sum of its partial amounts.</aside>


# Receipt Info {#cheque}

According to Russian Federation law (54-FZ), every merchant should provide online receipts for clients due to tax regulation. QIWI PAY API provides the service for it.

Follow the steps to send receipt for the transaction:

1. Prepare JSON [object of the receipt](#json_receipt)
2. Compress the object by DEFLATE algorithm according to [RFC1951](http://www.ietf.org/rfc/rfc1951.txt)
3. Transform the result to ZLIB format by [RFC1950](http://www.ietf.org/rfc/rfc1950.txt)
4. Send BASE64-encoded obtained string:

    * when using QIWI PAY WPF redirect — in `merchant_cheque` parameter
    * when using QIWI PAY API — in `cheque` parameter for `auth`, `capture`, `sale` operations.
  
<aside class="notice">
When your account is in <a href="#test_mode">test mode</a>, the receipt will be processed in test environment.
</aside>

### JSON description {#json_receipt}

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
      "description" : "Item/Service 1"
    },
    {
      "quantity" : 1,
      "price" : 500,
      "tax" : 4,
      "description" : "Item/Service 2"
    }
  ]
}
~~~

~~~php
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
      "description" : "Item/Service 1"
    },
    {
      "quantity" : 1,
      "price" : 500,
      "tax" : 4,
      "description" : "Item/Service 2"
    }
  ]
}';


$cheque = base64_encode(gzcompress($cheque_json));
?>

Output:
eJyljk0KwjAQRveeImRdtEl1oSvvIVJCGjHQNrWZgqUUFG+iFyi6FDxDeiOT+LcQV25m8d58800zQAhrkaaijGWC0QxFhEYhIRMaBs7xtdhUIoa6EM6SB6w0qMxGuMqBcXAGr5SaJypjMh9CmmC/CGwb61qDyD7hQmkJUuXaoYUlCDV+WrepWA4Saqdo8KJFKblvjygdTsdvbq87+gGJ0LyUhbvuXzJHczNn0/W7kTn1e3PtD+ZiOkSwT7TB73by3T4Jw/+r6bPazuWgvQNvoHPJ
~~~

Parameter|Required|Data type|Description
--------|-------|----------|--------
seller_id|Y|decimal|Organization TIN (taxpayer identification number)
cheque_type|Y|decimal|Receipt type (tag `1054` in the tax document) — one of the following:<br>`1` — Incoming cash<br>`2` — Cash return<br>`3` — Outcoming cash<br>`4` — Outcoming cash return
customer_contact|Y|string(64)|Customer phone or e-mail (tag `1008` in the tax document)
tax_system|Y|decimal|Tax system applied to merchant (tag `1055` in the tax document):<br>`0` – General, "ОСН"<br>`1` – Simplified income, "УСН доход"<br>`2` – Simplified income minus expense, "УСН доход - расход"<br>`3` – United tax to conditional income, "ЕНВД"<br>`4` – United agriculture tax, "ЕСН"<br>`5` – Patent tax system, "Патент"
positions|Y|array|Items
--------|-------|----------|--------
quantity|Y|decimal|Goods quantity (tag `1023` in the tax document)
price|Y|decimal|Price per item of goods, with discounts and markups (tag 1079 in the tax document)
tax|Y|decimal|VAT rate (tag `1199` in the tax document):<br>`1` – 18%<br>`2` – 10%<br>`3` – расч. 18/118<br>`4` – расч. 10/110<br>`5` – 0%<br>`6` – not applicable
description|Y|string(128)|Goods description
payment_method|Y|decimal|Cash type (tag `1214` in the tax document):<br>`1` – payment in advance 100%. Full payment in advance before commodity provision<br>`2`  – payment in advance. Partial payment before commodity provision<br>`3` – prepayment<br>`4` – full payment, taking into account prepayment, with commodity provision<br>`5` – partial payment and credit payment. Partial payment for the commodity at the moment of delivery, with future credit payment.<br>`6`  – credit payment. Commodity is delivered with no payment at the moment and future credit payment is expected.<br>`7` – payment for the credit. Commodity payment after its delivery with future credit payment.
payment_subject|Y|decimal|Payment subject (tag `1212` in the tax document):<br>`1` – commodity except excise commodities (name and other properties describing the commodity).<br>`2` – excise commodity (name and other properties describing the commodity).<br>`3` – work (name and other properties describing the work). .<br>`4` – service (name and other properties describing the service).<br>`5` – gambling rate (in any gambling activities).<br>`6` – gambling prize payment (in any gambling activities)в.<br>`7` – lottery ticket payment (in accepting payments for lottery tickets, including electronic ones, lottery stakes in any lottery activities).<br>`8` – lottery prize payment (n any lottery activities). <br>`9` – provision of rights to use intellectual activity results.<br>`10` – payment (advance, pre-payment, deposit, partial payment, credit, fine, bonus, reward, or any similar payment subject).<br>`11` – agent's commission (in any fee to payment agent, bank payment agent, commissioner or other agent service).<br>`12` – multiple payment subject (in any payment constituent subject to any of the above).<br>`13` – other payment subject not related to any of the above.

# Apple Pay {#applepay}

Apple Pay allows customers to pay for purchases on the site in one touch, without entering the card data. The technology works in mobile apps and the Safari browser on iPhone, iPad, Apple Watch and MacBook. To enable the Apple Pay payment method, you should contact your accompanying manager.

You'll need to integrate with Apple yourself. This integration will allow you to verify the TSP website and receive payment data of the user.

Requirements for TSP to integrate Apple Pay on your web page:

* Developer account in Apple Developer Program
* Use of HTTPS on the Apple Pay page, TLS 1.2 support, valid SSL certificate
* Compliance with [Apple's guidelines](https://developer.apple.com/apple-pay/acceptable-use-guidelines-for-websites/) on Apple Pay on websites
* Using the framework [Apple Pay JS API](https://developer.apple.com/documentation/apple_pay_on_the_web/apple_pay_js_api)

For more information on integration, please visit [Apple's website](https://developer.apple.com/apple-pay/implementation/).

## How to send a payment {#qiwipay_apple}

> Sending Apple Data Example

~~~json
{
  "opcode":1,
  "merchant_site":"123321",
  "order_id":500989,
  "apple_pay_encoded_payment_token":"eNrVV1mPszgW\/S\/1mpGKJSShpX6wgbAEk7AvKmnEFghLSCDghFb\/93EqX31L9zzMvIw0kSLgYpu7nHt8\/MfbJX60+fkmxrf47bc\/3qa8H07d+e23N0n450S\/=",
  "currency":643,
  "amount":"100.00",
  "phone":"79111111111",
  "sign":"d96af29003958983b7ca158083d4c22033eb1a809d4c6"
}
~~~

The process of making an Apple Pay payment in the QIWI PAY API consists of three steps:

1. Creating an Apple Pay payment session and validating TSP in Apple Pay
2. Getting encrypted payment data from Apple Pay
3. Sending a request for a write-off to QIWI

Currently, when Apple Pay payments are processing, the purchase operation is supported by the the following [Operation codes](#sale_types): `sale` (code `1`) and `auth` (code `3`).

To send payment data to QIWI, you must pass the `apple_pay_encoded_payment_token` parameter containing Apple's encrypted payment data in [the request](#sale). The value of the parameter is an encoded string. This parameter (like all other field values in the JSON-body of the request) must be involved in the calculation of the [signature of the request](#sign) (`sign` parameter).

The value of the `apple_pay_encoded_payment_token` field is calculated by the following algorithm:
 
`value = b64_encode_bytes_to_string (compress_bytes_with_zlib (to_bytes (json)))`

~~~json
{
  "paymentData": {
    "data": "TG9yZW0gaXBzdW0gZG9sb3Igc2l0IGFtZX...........................",
    "header": {
      ...
    },
    "signature": "TG9yZW0gaXBzdW0gZG9sb3Igc2l0IGFtZX...........................",
    "version": "EC_v1"
}
}
~~~

> Example of encoding Apple Pay payment token

~~~python
import zlib
import base64

token = 'Hello world'

token_bytes = token.encode('utf-8')
token_deflated = zlib.compress(token_bytes)
token_base64 = base64.b64encode(token_deflated)
token_result = token_base64.decode('utf-8')

resp_base64 = token_result.encode('utf-8')
resp_deflated = base64.b64decode(resp_base64)
resp_bytes = zlib.decompress(resp_deflated)
resp_token = resp_bytes.decode('utf-8')

print('resp_token: ', resp_token)
~~~

~~~shell
>> resp_token: 'Hello world'
~~~

Here `json` is a JSON object. This object must contain all the fields that Apple receives in accordance with the "Top-Level Structure" structure. See [Apple documentation](https://developer.apple.com/library/archive/documentation/PassKit/Reference/PaymentTokenJSON/PaymentTokenJSON.html#//apple_ref/doc/uid/TP40014929-CH8-SW2).

**Step 1** (`to_bytes`) — The `json` object is converted into an array of bytes in accordance with the UTF-8 encoding.

**Step 2** (`compress_bytes_with_zlib`) — The bytes received in the previous step are compressed by the zlib library. When compressing, you must follow [the RFC1950 specification](http://www.ietf.org/rfc/rfc1950.txt) and ensure that the `zlib header` is recorded at the beginning of the compressed stream.

**Step 3** (`b64_encode_bytes_to_string`) — The bytes received in the previous step are encoded in the [base64 format](https://ru.wikipedia.org/wiki/Base64).
 
As a result of all the steps, we get a string with an encoded value, which is framed in the value of the `apple_pay_encoded_payment_token` field.

# Google Pay&trade; Integration {#google}

This section describes Google Pay&trade; integration process into merchant's Payment Form in the QIWI PAY API framework. Payments from Visa and MasterCard payment cards are supported.

1. Implement the requirements of [Google Pay&trade; WEB](https://developers.google.com/pay/api/web) for accepting encrypted payment data. Check [Integration checklist](https://developers.google.com/pay/api/web/guides/test-and-deploy/integration-checklist) and [Brand guidelines](https://developers.google.com/pay/api/web/guides/brand-guidelines).
2. Implement the QIWI PAY [Google data sending requirements](#qiwipay_google). 
3. [Request production access](https://developers.google.com/pay/api/web/guides/test-and-deploy/request-prod-access). Indicate the following data in the request:
    * Tokenization Method — `Gateway`
    * Payment Processor or Gateway — `qiwi`
    * Merchant Name — provided by QIWI Support
4. Google tests your Payment Form and approves the launch. 

## How to send data to Google {#qiwipay_google}

QIWI PAY API supports Google Pay&trade; payment processing for `sale` (opcode: 1) and `auth` (opcode: 3) [sale operations](#sale).

~~~json
{
  "opcode": 1,
  "merchant_site": "123456",
  "order_id": 1969843,
  "google_pay_encoded_payment_token": "eJxVUtuK2zAQfd+vCHpeWN2sy0If3GShS9mS0hb8YChjabx........",
  "currency": 643,
  "amount": "1000.00",
  "country": "RUS",
  "ip": "1.2.3.4",
  "user_timedate": "2020-01-29T14:14:47+00:00",
  "product_name": "Payment amount 100000 copecs",
  "sign": "de4f8390ee8afb05797057ca79d20a416512f..."
}
~~~

> Example of encoding Google Pay payment token

~~~python
import zlib
import base64

token = 'Hello world'

token_bytes = token.encode('utf-8')
token_deflated = zlib.compress(token_bytes)
token_base64 = base64.b64encode(token_deflated)
token_result = token_base64.decode('utf-8')

resp_base64 = token_result.encode('utf-8')
resp_deflated = base64.b64decode(resp_base64)
resp_bytes = zlib.decompress(resp_deflated)
resp_token = resp_bytes.decode('utf-8')

print('resp_token: ', resp_token)
~~~

~~~shell
>> resp_token: 'Hello world'
~~~

Add `google_pay_encoded_payment_token` field to each sale request. Field value is an encoded string and it should participate in [sign](#sign) verification.

The field value is calculated by the following algorithm:

`b64_encode_bytes_to_string (compress_bytes_with_zlib (to_bytes (JSON)))`

<aside class="notice">
<code>JSON</code> object structure is described in <a href="https://developers.google.com/pay/api/web/guides/resources/payment-data-cryptography">Payment data cryptography for merchants</a> document.
</aside>


**Step 1** (`to_bytes`) — Convert `JSON` object to bytes array UTF-8 encoded.
 
**Step 2** (`compress_bytes_with_zlib`) — Compress the bytes from Step 1 by zlib library. You should follow the specification [RFC1950](http://www.ietf.org/rfc/rfc1950.txt) and provide `zlib header` record into the beginning of compressed data pipe.
 
**Step 3** (`b64_encode_bytes_to_string`) — `Base64` encode the bytes from Step2.

As a result of these Steps, you obtain a string with encoded value which is the required `google_pay_encoded_payment_token` field value.

## Response to sale operation

If Google Pay&trade; payment token contained 3DS cryptogram, then QIWI PAY API response completely corresponds to the [sale operation response](#sale_no3ds) in case of no 3DS operation.

If Google Pay&trade; payment token does not contain 3DS cryptogram, then QIWI PAY API response completely corresponds to the [sale operation response](#sale_3ds) when the user needs 3DS authentication. 


# Transaction Types {#txn_type}

Transaction types returned in `txn_type` parameter are in the following table.

Type | Name | Description
--- | -------- | --------
1 | Purchase | One-step payment
2 | Purchase | Authorization for two-step payment
4 | Reversal | Cancel operation
3 | Refund | Refund operation
8 | Payout | Payout operation (OCT)
0 | Unknown | Unknown operation type

# Transaction Statuses {#txn_status}

Status | Name | Description | Possible operations
------ | -------- | -------- | ------------------
0 | Init | Base verification is passed and operation begins to process | -
1 | Declined | Operation declined | -
2 | Authorized | Authorization is passed (hold) | `capture`, `reversal`
3 | Captured (Completed) | Confirmed operation | `reversal`
4 | Reconciled | Completely finished financial operation | `refund`
5 | Settled | RSP is paid for this operation | `refund`

<aside class="notice">
If interaction with Acquiring bank is online, then status <i>4</i> is returned in case of <code>sale</code>, <code>capture</code> operations.

If interaction with Acquiring bank is made via postponed approval of financial operations, then status <i>3</i> is returned after <code>sale</code>, <code>capture</code> operations, while status <i>4</i> is returned after sending financial information on status query operation.
</aside>

<aside class="success">
Status <i>2</i> and higher means that funds are authorized and you may already provide service/goods.
</aside>

# Errors Description {#errors}

## Response with errors


>Request with errors in card data

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


>Response

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
  "error_code": 8024
}
~~~

> Request with empty txn_id

~~~json
{

  "opcode" : 6,
  "merchant_site" : "1234",
  "txn_id" : "",
  "sign" : "sadads",
  "amount" : "1000.01"
}
~~~

>Response

~~~json
{
  "error_message":"Parsing error",
  "error_code":8006
}
~~~

Response with errors has the following structure:

Field | Type | Description
-----|-------|---------
error_code | Number | [Error code](#codes)
error_message | String | Error description
errors| Array of Objects | Detailed description of errors
errors[].field | String | Name of the request's parameter caused the error
errors[].message | String | Error text description

## Error codes {#codes}

Error code | Name | Description
---------- | -------- | --------
0 | No errors | No errors
8001 | Internal error | Acquiring Bank technical error
8002 | Operation not supported | Operation not supported
8004 | Temporary error | Server is busy, repeat later
8005 | Route not found | Route for transaction processing not found
8006 | Parsing error | Unable to process input request, incorrect format
8018 | Transaction not found | Transaction not found
8019 | Incorrect [opcode](#opcode) |
8020 | Amount too big | `reversal`, `refund` operations amount exceeded
8021 | Merchant site not found | Unknown merchant with `merchant_site_id`
8022 | Transaction not found | Transaction from the request not found.
8023 | Transaction expired | Customer failed to authorize on 3DS within 15 min
8025 | Opcode is not allowed | Using this [opcode](#opcode) value forbidden for this merchant site
8026 | Incorrect parent transaction | Incorrect status of the parent transaction
8027 | Incorrect parent transaction | Incorrect type of the parent transaction
8028 | Card expired | Card valid date is expired
8051 | Merchant disabled | This merchant is not activated.
8052 | Incorrect transaction state | Incorrect transaction status, attempt for `capture` after `finish_3ds` or `reversal`
8054 | Invalid signature | Invalid signature of the request
8055 | Order already paid |
8056 | In process | Transaction on this card/order is processing now.
8057 | Card locked | Transaction is already processing upon this card
8058 | Access denied |
8059 | Currency is not allowed |
8060 | Amount too big | Amount is higher than allowed for payout.
8061 | Currency mismatch | Payout currency and original transaction currency are not the same
8069 | Quantity limit of transactions is reached | The number of testing operations per day exceeds the [limit](#test_limit)
8070 | Amount of transaction is bigger than allowed | Test operation amount exceeds the [allowed value](#test_limit)
8151 | Authentification failed | Customer failed to authenticate on 3DS.
8152 | Transaction rejected | Transaction rejected by security service.
8160 | Transaction rejected | Issuer response: Payment rejected. Try again
8161 | Transaction rejected | Issuer response: Payment rejected. Try again
8162 | Transaction rejected | Issuer response: Payment rejected. Try again
8163 | Transaction rejected | Issuer response: Payment rejected. Contact QIWI Support
8164 | Transaction rejected | Issuer response: Payment rejected. Limit exceeded. Address to Issuing Bank
8165 | Transaction rejected | Issuer response: Payment rejected. Check payment details
8166 | Transaction rejected | Issuer response: Payment rejected. Check card details
8167 | Transaction rejected | Issuer response: Payment rejected. Check card details
8168 | Transaction rejected | Issuer response: Transaction forbidden. Address to Issuing Bank
8169 | Transaction rejected | Issuer response: Payment rejected. Limit exceeded. Address to Issuing Bank

# Test Data {#test_mode}

By default, for each new RSP a `merchant_site` record is created in QIWI PAY service in test environment only. You can ask your support manager to transfer any of your  `merchant_site` to test environment, or add new `merchant_site` in test environment.

* To make tests for payment operations, you may use any card number satisfying to Luhn algorithm.
* In test environment, you may use only ruble (`643` code) for the `currency` parameter.
* CVV for payments in test environment may be arbitrary (any 3 digits).

Use [production URLs](#urls) for sending test requests.

<h4>Test card numbers</h4>
<h4 id="visa">
</h4>
<h4 id="mc">
</h4>
<button class="button-popup" id="generate">Get more cards</button>

To make tests of various payment methods and responses, use different expiry dates:

* If month of expiry date is `02`, then operation is treated as unsuccessful.
* If month of expiry date is `03`, then operation will process successfully with 3 seconds timeout.
* If month of expiry date is `04`, then operation will process unsuccessfully with 3 seconds timeout.
* In all other cases, operation is treated as successful.

<a name="test_limit"></a>
Test environment has restrictions on the total amount and number of operations. By default, maximum amount of test transaction is 10 rubles. Maximum number of test transactions is 100 per day (MSK time zone). Only test transactions within allowed amount are taken into account.

To process 3DS operation, use `unknown name` as card holder name.

3DS in test environment may be properly tested on real card number.
