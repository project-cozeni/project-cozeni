# Cozeni Crypto Payment API

Version 1.10 · February 2025

API integration reference for payments, transaction queries, balances, exchange rates, and network fee estimates. Token-generation instructions are in [Authentication](AUTHENTICATION.md).

## Table of Contents

- [Record Of Amendments](#record-of-amendments)
- [General Overview](#general-overview)
- [Payment APIs](#payment-apis)
  - [Pay In](#pay-in)
  - [Payout](#payout)
- [Query APIs](#query-apis)
  - [Pay In Status](#pay-in-status)
  - [Payout Status](#payout-status)
  - [Balance Query](#balance-query)
  - [Exchange Rate](#exchange-rate)
  - [Network Fee Estimation](#network-fee-estimation)
- [References](#references)
  - [Supported Crypto](#supported-crypto)
- [Authentication](AUTHENTICATION.md)

## RECORD OF AMENDMENTS

| DATE | DESCRIPTION OF CHANGE | Version |
| :---- | :---- | :---- |
| Oct 2020 | Document Creation Callback URL added to Pay Out Request  | v1.0 |
| Nov 2020 | Added CANCELLED status on Pay In response  Pay Out Change *requestAmount* parameter to *amount* Change *requestCurrency* parameter to *currency* Remove *convertCurrency* parameter | v1.02 |
| Nov 2020 | Pay Out Initial Payout Status when calling /payouttxn will always be PENDING status (see explanation under PayOut Response Added new status of *PENDING*  Added new resultCode of “01” - Pending Payment | v1.03 |
| Mar 2021 | Pay In Added reference to for requestCurrency Supported crypto table Pay Out Added *payoutCurrency* in the Payout Request API as new attribute to support new crypto Introduced two new result codes - Possible Fraud and Unsupported currency  References Introduce Supported Crypto table | v1.04 |
| Jun 2021 | Updated token terminology to crypto when referencing cryptocurrencies Created new section for query API calls Included public URL to retrieve exchange rates | V1.05 |
| Jul 2021 | Introduced Bitcoin and USDT (TRC) support with new symbol reference | V1.06 |
| Nov 2021 | Introduced Promotional Program for Pay Ins Updates to the Pay In response attributes and callback Introduced conversionRate in response attribute | V1.07 |
| Aug 2022 | Balance Inquiry and Network Fee Estimation Decommissioned support for USDC TRC | V1.08 |
| Feb 2025 | Introduction of LTC, SOL coins | V1.10 |

## GENERAL OVERVIEW

> We require all merchants to complete a full test cycle in our sandbox environment using the information provided below before any production information can be set up and released. Once testing has been completed, contact us to enable your live account.

Throughout this document, placeholders enclosed in `{ }` represent variable values:

- **`{host}`**: The host portion of a URL. Use `test.cozeni.io` for testing and `secure.cozeni.io` for production.
- **`{merchantId}`**: A merchant ID and API key are issued when your account is created. Test and production accounts have different merchant IDs and API keys.
- **`{hashed token}`**: The authorization token supplied in the request header. See [Authentication](AUTHENTICATION.md) for generation instructions.

## PAYMENT APIs

### PAY IN

This API requires the `X-cozeni-txnauthz` request header. See [payment token generation](AUTHENTICATION.md#payment-requests).

| Property | Value |
| :--- | :--- |
| **Description** | Accept payment from an external wallet |
| **HTTP Method** | POST |
| **Request Headers** | `X-cozeni-txnauthz: {hashed token}`<br>`Content-Type: application/json` |
| **URL** | `https://{host}/mxapi/{merchantId}/payintxn` |
| **Request Body** | In JSON format. Following are the attributes for the request body:  |

#### Request Body

| Attribute Name | Required | Description |
| ----- | :---: | ----- |
| accountId | Y | Assigned account ID which is a numeric, 10 digit identifier. |
| requestSalt | Y | Any string of any length. Should be different for every request. Suggest using current time in milliseconds.  |
| requestAmount | Y | The amount of the payment. Numeric, 2 decimal places (e.g. 10.00). |
| requestCurrency | Y | Crypto currency of the payment (ex. USDT). For crypto symbols, please refer to [Supported Crypto](#supported-crypto).  |
| convertCurrency | N | Converted currency in fiat or crypto (e.g. CNY). If passing a crypto symbol, the system will only accept the same crypto symbol as the requestCurrency. Otherwise, not passing this attribute will return the same currency as the requestCurrency. |
| paymentRef | Y | Your own transaction identifier |
| paymentDesc | N | A description of the transaction |
| callbackUrl | N | A URL hosted on your site which we will call to provide an update on payment status (ex. crypto deposit has been confirmed as received). If this attribute is not specified in the request, we will default to the callbackUrl setup in your merchant account. |
| dupCheck | N | Valid values are “true” or “false”. If set to true, we will check if a transaction with the same paymentRef has already been submitted. If a duplicate is found, an error will be returned. If set to false, no duplicate checking will be performed. The default is false. |
| merchantTraceData | N | Custom merchant field to pass data which will be echoed back in the result and in queries. |

##### Sample Request

```json
{
  "accountId": 1000020000,
  "requestSalt": "1234567890123",
  "requestAmount": 10.00,
  "requestCurrency": "USDT",
  "convertCurrency": "CNY",
  "paymentRef": "PR11111",
  "paymentDesc": "test transaction",
  "callbackUrl": "https://yourdomain.com/paymentresult",
  "dupCheck": true,
  "merchantTraceData": "M10001"
}
```

#### Response

There are 2 types of responses that you may get when calling the payment API.

**HTTP STATUS 422** - If the transaction fails authorization or validation, you will get an **HTTP Status 422** (Unprocessable Entity). The response body will contain an error code and an error message. Sample error response:

```json
{
  "msgCode": "E2046",
  "msgText": "Duplicate payment reference"
}
```

**HTTP STATUS 200** - If the transaction passes authorization and validation, you will get an **HTTP Status 200** (OK). The response body will contain a JSON formatted text with the following attributes:

| Attribute | Description |
| :--- | :--- |
| **txnId** | This is the transaction identifier |
| **status** | Possible values are: `COMPLETED`, `PENDING`, `FAILED`, `CANCELLED` |
| **paymentRef** | This is the payment reference you submitted in the request |
| **resultCode** | `00` — Accept payment successful<br>`01` — Accept payment pending<br>`02` — Accepted payment over requested amount<br>`03` — Accepted payment less than requested amount<br>`89` — Accept payment time out<br>`99` — System error |
| **resultMsg** | A description of the result code as specified above |
| **timestamp** | The date/time when transaction was created |
| **requestAmount** | The amount of the requested payment |
| **requestCurrency** | The currency of the requested payment |
| **receivedAmount** | The amount received from the customer. This will initially be zero until payment has completed |
| **convertAmount** | The converted amount is the received amount from the customer. This will initially be zero until payment has completed |
| **convertCurrency** | The currency to convert to upon completion |
| **paymentAddress** | The receiving wallet address for payment |
| **validUntil** | Timestamp when transaction will be void and cancelled |
| **txnAuthz** | System generated authorization code |
| **txnHash** | Unique hash of the transaction based on the specific blockchain. This will initially contain an empty string until payment has completed. |
| **totalCreditAmount** | If a promotional program is **NOT** enabled (See [Promotional Program](#promotional-programs)), this attribute would equal convertAmount. If a promotional program **IS** enabled, this attribute would consist of the convertAmount plus the promoCreditAmount. This will initially be zero until payment has completed. |
| **totalCreditCurrency** | This will be the same as the convertCurrency |

Due to the nature of how cryptocurrencies work, payments using cryptocurrency will be asynchronous. Therefore, the initial response will always be “01 - Accept payment pending” and you will need to provide the transaction details to the customer in order to complete the payment.

##### Sample Response

```json
{
  "txnId": 1360,
  "status": "PENDING",
  "paymentRef": "PR11111",
  "resultCode": "01",
  "resultMsg": "Accept payment pending",
  "timestamp": "2019-09-29T06:48:23.665+0000",
  "requestAmount": 10.00,
  "requestCurrency": "USDT",
  "receivedAmount": 0,
  "convertAmount": 0,
  "convertCurrency": "CNY",
  "paymentAddress": "0x89205A3A3b2A69De6Dbf7f01ED13B2108B2c43e7",
  "validUntil": "2019-09-29T07:48:23.665+0000",
  "txnAuthz": "87854753e5e91d3b82bea0004d519e4fefa81f7d0e89ce8",
  "totalCreditAmount": 0,
  "totalCreditCurrency": "CNY"
}
```

#### Callback

Our system will monitor the paymentAddress for a customer payment. This requires a set number of confirmations on the blockchain before we can deem a payment as complete. At which point, we will perform a callback to the URL you supplied in the “**callbackUrl**” attribute you submitted in the request (or defaulted from merchant settings). The callback will contain HTTP POST parameters corresponding to the [response attributes above](#response). In addition, for a completed payment, there will be the following attributes as part of the callback as outlined below:

| Attribute | Description |
| :--- | :--- |
| **conversionRate** | The exchange rate used at the time of the transaction  |
| **Promotional program fields** | The attributes below exist only if a promotional program is enabled. See [Promotional Program](#promotional-programs). |
| **promoCryptoAmount** | The promotional amount as calculated from the promotional program based on the receivedAmount |
| **promoCreditAmount** | The promotional amount as calculated from the promotional program based on the convertAmount |
| **promoCreditCurrency** | The currency of the promoCreditAmount |
| **promoReference** | The reference to a specific promotional program version |

#### Expired Transactions

There may be cases where the customer is taking too long to complete the authorization or has completely abandoned it. In such cases, the transaction's status would remain PENDING. If a transaction remains PENDING for longer than the ***validUntil*** stated in the response attributes above, we will automatically cancel it and set the resultCode to "89" - Accept payment time out. You will be notified via an HTTP post to the “**callbackUrl**” that you have supplied in the original request (or defined within the merchant settings).

You may also poll our existing [Pay In Status API](#pay-in-status) to check for the status of the transaction if you prefer not to use a callback or in addition thereto. The structure of the data you will get from a query would be the same as what we would have sent you in the callback.

#### Promotional Programs

Our platform allows specific promotional programs to offer a percentage bonus/cashback for deposits ONLY to end customers. This can also be customized for certain cryptocurrency or across all of them providing fine tuning capabilities to further drive incentives.

The system calculates the percentage based on the cryptocurrency amount (receivedAmount) and the responses are reflected in the [response attributes](#response).

For those who are currently integrated with our API, the promo program does not disrupt current responses. You will notice there are additional promo attributes being returned in the response but these would only be applicable if there is a promotional program running.

### PAYOUT

This API requires the `X-cozeni-txnauthz` request header. See [payment token generation](AUTHENTICATION.md#payment-requests).

| Property | Value |
| :--- | :--- |
| **Description** | Send payment to an external wallet |
| **HTTP Method** | POST |
| **Request Headers** | `X-cozeni-txnauthz: {hashed token}`<br>`Content-Type: application/json` |
| **URL** | `https://{host}/mxapi/{merchantId}/payouttxn` |
| **Request Body** | In JSON format. Following are the attributes for the request body:  |

#### Request Body

| Attribute Name | Required | Description |
| ----- | :---: | ----- |
| accountId | Y | Assigned account ID which is a numeric, 10 digit identifier. |
| requestSalt | Y | Any string of any length. Should be different for every request. Suggest using current time in milliseconds.  |
| amount | Y | The fiat amount of the payment. Numeric, 2 decimal places (e.g. 400.00). |
| currency | Y | Conversion currency of the payment. It can be a fiat currency (e.g. CNY) or the same crypto currency as the payoutCurrency (meaning no conversion). |
| payoutCurrency | Y | Crypto currency of the payout. Please refer to [Supported Crypto](#supported-crypto). |
| paymentAddress | Y | Customer’s receiving wallet address (e.g. 0x89205A3A3b2A69De6Dbf7f01ED13B2108B2c43e7) |
| paymentRef | Y | Your own transaction identifier |
| paymentDesc | N | A description of the transaction |
| callbackUrl | N | A URL hosted on your site which we will call to provide an update on payment status. If this attribute is not specified in the request, we will default to the callbackUrl setup in your merchant account. |
| dupCheck | N | Valid values are “true” or “false”. If set to true, we will check if a transaction with the same paymentRef has already been submitted. If a duplicate is found, an error will be returned. If set to false, no duplicate checking will be performed. The default is false. |
| merchantTraceData | N | Custom merchant field to pass data which will be echoed back in the result, the redirect (for 3d) and in queries. |

##### Sample Request

```json
{
  "accountId": 1000020000,
  "requestSalt": "1234567890123",
  "amount": 400.00,
  "payoutCurrency": "USDT",
  "currency": "CNY",
  "paymentAddress": "0x89205A3A3b2A69De6Dbf7f01ED13B2108B2c43e7",
  "paymentRef": "PR11111",
  "paymentDesc": "test transaction",
  "dupCheck": true,
  "merchantTraceData": "M10001"
}
```

#### Response

There are 2 types of responses that you may get when calling the payment API.

**HTTP STATUS 422** - If the transaction fails authorization or validation, you will get an **HTTP Status 422** (Unprocessable Entity). The response body will contain an error code and an error message. Sample error response:

```json
{
  "msgCode": "E2046",
  "msgText": "Duplicate payment reference"
}
```

**HTTP STATUS 200** - If the transaction passes authorization and validation, you will get an **HTTP Status 200** (OK). The response body will contain a JSON formatted text with the following attributes:

| Attribute | Description |
| :--- | :--- |
| **txnId** | This is the transaction identifier |
| **status** | Possible values are: `SUCCESS`, `PENDING`, `FAILED` |
| **paymentRef** | This is the payment reference you submitted in the request |
| **resultCode** | `00` — Send payment successful<br>`01` — Pending payment<br>`90` — Possible Fraud<br>`91` — Unsupported currency<br>`92` — Invalid payment address<br>`93` — Insufficient balance<br>`99` — System error |
| **resultMsg** | A description of the result code as specified above |
| **timestamp** | The date/time when transaction was created |
| **requestAmount** | The amount of the payment |
| **requestCurrency** | The currency of the payment |
| **convertAmount** | The converted amount of the payment |
| **convertCurrency** | The currency of the converted amount |
| **paymentAddress** | The receiving wallet address for payment |

Due to the nature of how cryptocurrencies work, payments using cryptocurrency will be asynchronous. Subject to any issues with the wallet balance or validity of parameters, when sending a payment on the blockchain, the response **will always** be “01 - Pending payment”. If we receive an initial confirmation on the blockchain, we will perform a callback with a SUCCESS status. If there is an error, the callback will be a FAILED status with its respective resultCode.

##### Sample Response

```json
{
  "txnId": 1360,
  "status": "PENDING",
  "paymentRef": "PR11111",
  "resultCode": "01",
  "resultMsg": "Pending payment",
  "timestamp": "2019-09-29T06:48:23.665+0000",
  "requestAmount": 10.00,
  "requestCurrency": "USDT",
  "convertAmount": 68.00,
  "convertCurrency": "CNY",
  "paymentAddress": "0x89205A3A3b2A69De6Dbf7f01ED13B2108B2c43e7",
  "txnHash": "0xdb489e8417c1249d5da5f7153e311fb85a6bb265741353eeb04787aa6ed1dedb"
}
```

#### Callback

Once we send payment to the customer’s payment address, we will also perform a callback to the URL you supplied in the “**callbackUrl**” attribute you submitted in the request (or defaulted from merchant settings). The callback will contain HTTP POST parameters corresponding to the response attributes above.

You may also poll our existing [Payout Status API](#payout-status) to check for the status of the transaction if you prefer not to use a callback or in addition thereto. The structure of the data you will get from a query would be the same as what we would have sent you in the callback.

## QUERY APIs

### PAY IN STATUS

#### By Transaction ID using GET Method

See [token generation](AUTHENTICATION.md#transaction-id-queries) for this endpoint.

| Property | Value |
| :--- | :--- |
| **Description** | Retrieve the payment result by the txnId |
| **HTTP Method** | GET |
| **Request Headers** | `X-cozeni-txnauthz: {hashed token}`<br>`Content-Type: application/json` |
| **URL** | `https://{host}/mxapi/{merchantId}/payintxn/{txnId}` |

##### Response Specifications

Similar to the Pay In API, you could get an HTTP 422 or an HTTP 200 response. The format and attributes of the response body would be the same as those in the callback under Pay In API.

#### By Reference ID using GET Method

See [token generation](AUTHENTICATION.md#reference-id-queries) for this endpoint.

| Property | Value |
| :--- | :--- |
| **Description** | Retrieve the payment result by your payment reference ID |
| **HTTP Method** | GET |
| **Request Headers** | `X-cozeni-txnauthz: {hashed token}`<br>`Content-Type: application/json` |
| **URL** | `https://{host}/mxapi/{merchantId}/payintxn/payref/{paymentRef}` |

##### Response Specifications

Similar to the Pay In API, you could get an HTTP 422 or an HTTP 200 response. The format and attributes of the response body for an HTTP 422 response would be the same as those for the Pay In API. For an HTTP 200 response, however, the response body will be in the format of a list/array. This is because there may be several transactions associated with a single paymentRef if duplicate checking is not enabled. The format and attributes for each entry in the list/array would be the same as those in the callback under Pay In API.

### PAYOUT STATUS

#### By Transaction ID using GET Method

See [token generation](AUTHENTICATION.md#transaction-id-queries) for this endpoint.

| Property | Value |
| :--- | :--- |
| **Description** | Retrieve the payout result by the txnId |
| **HTTP Method** | GET |
| **Request Headers** | `X-cozeni-txnauthz: {hashed token}`<br>`Content-Type: application/json` |
| **URL** | `https://{host}/mxapi/{merchantId}/payouttxn/{txnId}` |

##### Response Specifications

Similar to the Payout API, you could get an HTTP 422 or an HTTP 200 response. The format and attributes of the response body would be the same as those for the Payout API.

#### By Reference ID using GET Method

See [token generation](AUTHENTICATION.md#reference-id-queries) for this endpoint.

| Property | Value |
| :--- | :--- |
| **Description** | Retrieve the payout result by your payment reference ID |
| **HTTP Method** | GET |
| **Request Headers** | `X-cozeni-txnauthz: {hashed token}`<br>`Content-Type: application/json` |
| **URL** | `https://{host}/mxapi/{merchantId}/payouttxn/payref/{paymentRef}` |

##### Response Specifications

Similar to the Payout API, you could get an HTTP 422 or an HTTP 200 response. The format and attributes of the response body for an HTTP 422 response would be the same as those for the Payout API. For an HTTP 200 response, however, the response body will be in the format of a list/array. This is because there may be several transactions associated with a single paymentRef if duplicate checking is not enabled. The format and attributes for each entry in the list/array would be the same as those for the Pay In API.

### BALANCE QUERY

See [token generation](AUTHENTICATION.md#balance-and-network-fee-queries) for this endpoint.

| Property | Value |
| :--- | :--- |
| **Description** | Retrieve the current balance for specified crypto currency |
| **HTTP Method** | POST |
| **Request Headers** | `X-cozeni-txnauthz: {hashed token}`<br>`Content-Type: application/json` |
| **URL** | `https://{host}/mxapi/{merchantId}/balance` |

#### Request Body

| Attribute Name | Required | Description |
| ----- | :---: | ----- |
| requestSalt | Y | Any string of any length. Should be different for every request. Suggest using current time in milliseconds.  |
| cryptoCurrency | Y | The symbol representing a cryptocurrency for which the balance will be retrieved (e.g. BTC, ETH). |

##### Sample Request

```json
{
  "requestSalt": "12345678",
  "cryptoCurrency": "USDT"
}
```

#### Response

Similar to all the other API endpoints, you could get an HTTP 400 or an HTTP 200 response. If the transaction passes authorization and validation, you will get an HTTP Status 200 (OK). The response body will contain a JSON formatted text with the following attributes:

| Attribute | Description |
| :--- | :--- |
| **cryptoCurrency** | Cryptocurrency of the balance |
| **balance** | Current balance amount |
| **asOf** | Date and time of balance |

##### Sample Response

```json
{
  "cryptoCurrency": "USDT",
  "balance": 100.5,
  "asOf": "2022-08-18 16:10:15 UTC"
}
```

### EXCHANGE RATE

In order to retrieve the exchange rate of a supported cryptocurrency to its fiat counterpart, you may use the URL and format outlined below.

| Property | Value |
| :--- | :--- |
| **URL** | `https://{host}/pub/rate/<crypto>/<fiat>` |

`<crypto>` – the symbol representing the [supported crypto](#supported-crypto) (ex. USDT, BTC, etc.)

`<fiat>` – the 3 letter official ISO 4217 standard representing fiat currency (ex. USD, EUR, etc.)

### NETWORK FEE ESTIMATION

See [token generation](AUTHENTICATION.md#balance-and-network-fee-queries) for this endpoint.

| Property | Value |
| :--- | :--- |
| **Description** | Retrieve an estimation of the network fees based on a blockchain |
| **HTTP Method** | POST |
| **Request Headers** | `X-cozeni-txnauthz: {hashed token}`<br>`Content-Type: application/json` |
| **URL** | `https://{host}/mxapi/{merchantId}/networkfees` |

#### Request Body

| Attribute Name | Required | Description |
| ----- | :---: | ----- |
| requestSalt | Y | Any string of any length. Should be different for every request. Suggest using current time in milliseconds.  |
| cryptoCurrency | Y | The symbol representing a cryptocurrency (e.g. BTC, ETH). |
| fiatConvertCurrency | N | The 3 character currency code (e.g. USD) to determine the fiat value equivalent of the network fee. This value will default to USD if not supplied. |

##### Sample Request

```json
{
  "requestSalt": "12345678",
  "cryptoCurrency": "BTC",
  "fiatConvertCurrency": "USD"
}
```

#### Response

| Attribute | Description |
| :--- | :--- |
| **feeCurrency** | The cryptocurrency used to determine the network fee |
| **estimatedFeeAmount** | The estimate value of the network fee |
| **fiatConvertCurrency** | The fiat currency code used to determine the network fee equivalent  |
| **fiatConvertAmount** | The approximate fiat value equivalent of the network fee  |

##### Sample Response

```json
{
  "feeCurrency": "BTC",
  "estimatedFeeAmount": 0.00012600,
  "fiatConvertCurrency": "USD",
  "fiatConvertAmount": 3.00
}
```

## REFERENCES

### Supported Crypto

| Crypto Name | Crypto Symbol |
| :---- | :---- |
| Tether USD (ERC) | USDT |
| Tether USD (TRC) | USDTRX |
| USD Coin (ERC) | USDC |
| Bitcoin | BTC |
| Ethereum | ETH |
| Litecoin | LTC |
| Solana | SOL |
| First Digital USD  | FDUSD |
