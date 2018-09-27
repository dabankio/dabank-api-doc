![logo](./LOGO.svg)


# Dabank

Dabank is a commercial cloud-based white-labeled cryptocurrency wallet service,
supporting a wide range of coins.

With Dabank API, you are able to provide your customers with deposit,
vault and withdrawal with grace and speed.

# General

## Communication

You ("app") and Dabank communicate in a bi-directional way:

* You call APIs to apply for deposit address, trigger a withdrawal, etc
* Dabank calls back to you for transaction status update, like new deposit arrival, deposit confirmation, withdrawal confirmation, etc

## Data Type

Data of input and output only consists of following basic type:

* string - most data type in use (mind all numbers(int/float) are represented in string format)

Array and Object are used on top of basic type.

## Request

All APIs uses HTTP POST method, with `Content-Type`header `application/json; charset=utf-8`.

## Authentication

### SHA256-RSA Authentication

For every API following, these inputs are needed in request for authentication:

| key           | type     | comment            |
| ------------ | ------ | ------------- |
| key          | string | key provides identity       |
| sign         | string | SHA256-RSA data signature    |
| request_time | string | current unix second |

* keys

There are two sets of key-pair involved in SHA256-RSA Authentication:

1. You use `app_private_key`(kept as secret) to sign your API requests, and you upload `app_public_key` to Dabank for request verification;
1. You download `dabank_public_key` for callback verification, as Dabank uses `dabank_private_key` to sign callbacks.

* Generation of `app_private_key` and `app_public_key`

You may use OpenSSL to generate RSA key pairs, note the key length is 2048.

```bash
openssl genrsa -out app_private_key.pem 2048
openssl rsa -in app_private_key.pem -pubout -out app_public_key.pem
```

* Steps to generate `sign`(for API requests)

`sign` is generated in following steps:

1. extract all keys and values in JSON's key-value pair(except `sign`) in the form of `key=value`, as array `param`;
1. sort `param` in lexical order;
1. join `param` with `&` to obtain string `content`;
1. calculate `sha256` of `content` to get `hashed`;
1. calculate the signature of `hashed` with `app_private_key` using [RSASSA-PKCS1-V1_5-SIGN](https://tools.ietf.org/html/rfc3447#page-33) from RSA PKCS#1 v1.5 to get `rsa_sign`;
1. encode `rsa_sign` with base64 and you get `sign` of your current request.

* Steps to verify `sign`(for Dabank callback verification)

1. extract all keys and values in JSON's key-value pair(except `sign`) in the form of `key=value`, as array `param`;
1. sort `param` in lexical order;
1. join `param` with `&` to obtain string `content`;
1. calculate `sha256` of `content` to get `hashed`;
1. verify the signature of `hashed` with `dabank_public_key` using [RSASSA-PKCS1-V1_5-VERIFY](https://tools.ietf.org/html/rfc3447#page-34) from RSA PKCS#1 v1.5


### HMAC-style Authentication(Deprecated)

HMAC-style Authentication is deprecated, which will be dropped in future.

For every API following, these inputs are needed in request for authentication:

| key           | type     | comment            |
| ------------ | ------ | ------------- |
| key          | string | key provides identity       |
| sign         | string | HMAC-style data signature    |
| request_time | string | current unix second |

* Steps to generate `sign`(for API requests and Dabank callback verification)

`sign` is generated in following steps:

1. extract all values except `sign` in JSON's key-value pair to array `values`;
1. put your `secret`(obtained from control panel of Dabank) into `values`;
1. sort `values` in lexical order;
1. join `values` with underline `_` to obtain string `content`;
1. calculate md5 of `content` and you get `sign` of your current request.

Documentations on above-mentioned authentication input are omitted
from now on for brevity. 

## Response

All responses are boxed in following object:

| key           | type     | comment            |
| ---------- | --------------- | ------------------------ |
| err_code   | string          | code of error, empty when everything is fine |
| err_info   | string          | description for error |
| request_id | int     | id for request booking, 0 if it is not booked   |
| data       | object/array | output of API         |

Example:

```json
{ 
    "err_code": "ERR_ACCT_NOT_EXIST",
    "err_info": "account not exist, check your key",
    "data":null
}
```

##  Param Style

Keys of JSON in request and response are in lower snake case, like `snake_case`.

## Status of Transactions

Status of transactions, including deposit and withdrawal, could be:

* `TRANSFER_PENDING`:
  * APIs response: the withdrawal has been handled and will be broadcasted on cryptocurrency network soon
  * withdrawal callback: the withdrawal has been broadcasted on network, but needs more confirmations
  * deposit callback: the deposit has been detected, but needs more confirmations from cryptocurrency network
* `TRANSFER_SUCCESSFUL`:
  * APIs response: the withdrawal is handled off-chain instantly, in most case, address `from` and `to` are both in Dabank
  * withdrawal callback: the withdrawal has enough confirmations from cryptocurrency network and can be considered stable
  * deposit callback: the deposit has enough confirmations from cryptocurrency network and can be considered stable
* `TRANSFER_INVALID`:
  * withdrawal callback: unusually, the withdraw is later rejected or considered invalid for various reasons, e.g. risk management, cryptocurrency network down, network fork
  * deposit callback: seldom, the deposit is considered invalid(most likely due to a network fork), note you may get several `TRANSFER_PENDING` callbacks before this.

# APIs

APIs are hosted at `https://api.dabank.io/`.

## Deposit Address Application

```
/api/v3/address
```

* request:

| key      | type     |  compulsory  | comment   |
| ------- | ------ | ---- | ---- |
| symbol  | string | ✓    | symbol for coin type identification   |
| user_id | string | ✓    | unique id to distinguish user, needs to be the same for one specific user |

```json
{  
   "key":"your_api_id",
   "request_time":"1524290015",
   "sign":"data_signature",
   "symbol":"BTC",
   "user_id":"1234"
}
```

* response:

| key           | type     | comment            |
| ------- | ------ | ---- |
| address | string | address for deposit   |

```json
{
   "err_code":"",
   "err_info":"",
   "data":{  
      "address":"0x12345678910388342390012323"
   },
   "request_id":233049
}
```

## Trigger a withdrawal

```
/api/v3/transfer
```

* request:

| key      | type     |  compulsory  | comment   |
| --------- | ------ | ---- | ---------------------------------------- |
| symbol    | string | ✓    | symbol for coin type identification  |
| coins     | string | ✓    | amount to withdraw  |
| to        | string | ✓    | destination address |
| from      | string | ✓    | deposit address of the user withdrawing |
| unique_id | string | ✓    | generated by you, unique for one specific transaction, use the same one if you are re-trying this withdraw for some reason |

```json
{  
   "unique_id":"must_be_unique_use_the_same_one_when_retrying",
   "to":"0x12345678910388342390012323",
   "sign":"data_signature",
   "coins":"2.12345678",
   "symbol":"ETH",
   "from":"0xabc45678910388342390012323",
   "request_time":"1524298182",
   "key":"your_api_id"
}
```

* response:

| key           | type     | comment            |
| ----------- | ------ | ---------------------------------------- |
| transfer_id | string | internal transaction ID provided by Dabank, which you use to identity specific transaction  |
| status      | string | status of transaction |



```json
{  
   "err_code":"",
   "err_info":"",
   "data":{  
      "status":"TRANSFER_PENDING",
      "transfer_id":123
   },
   "request_id":233114
}
```


## Get Balance of Your Supported Symbols

```
/api/v3/appAccount
```

* request:

This API needs only basic auth param.

| key      | type     |  compulsory  | comment   |
| ------ | ------ | ---- | ---------------- |
|  |  |  |  |

```json
{  
  "key":"your_api_id",
  "request_time":"1524290015",
  "sign":"data_signature"
}
```

* response:

| key           | type     | comment            |
| ------- | ------ | ------------- |
| symbol  | string | symbol for coin type identification            |
| address | string | your app exclusive address            |
| balance | string | your real-time balance |

```json
[
  {
  "symbol":"ETH",
  "address":"your_app_exclusive_eth_address",
  "balance":"234.5"
  },
  {
  "symbol":"BTC",
  "address":"your_app_exclusive_btc_address",
  "balance":"2.5"
  }
]
```

## Validate An Address

Validate an address to ensure it is not malformed,
note a "valid" address does not mean it is the "correct" address,
the correction of an address as withdrawal destination is guaranteed by your users themselves.

```
/api/v3/checkAddress
```

* request:

| key      | type     |  compulsory  | comment   |
| ------- | ------ | ---- | ---- |
| symbol  | string | ✓    | symbol for coin type identification   |
| address | string | ✓    | address to be checked |

```json
{  
  "key":"your_api_id",
  "request_time":"1524290015",
  "sign":"data_signature",
  "symbol":"ETH",
  "address":"12345678910388342390012321"
}
```

* response:

| key           | type     | comment            |
| ------- | ------ | --------------------------- |
| verify  | string | Validation result, `Success` means valid, `Failure` means invalid |
| err_msg | string | when `Failure`, here lies the validation detail   |

```json
{  
  "verify":"Failure",
  "err_msg":"ETH address should be started with \"0x\""
}
```

## BCH CashAddr/Legacy Address Convention

For Bitcoin Cash(BCH), Dabank uses CashAddr across service,
your users may use Legacy Address as withdrawal destination,
in which case we shall convert them to CashAddr automatically.

We understand that there are lots of exchange not supporting withdrawal to
CashAddr, so you may use this API to convert the deposit address Dabank
provides to Legacy format, in favor for users in need.

```
/api/v3/bchAddrConvert
```

* request:

| key      | type     |  compulsory  | comment   |
| ------- | ------ | ---- | -------- |
| address | string | ✓    | BCH CashAddr or Legacy Address |

```json
{  
  "key":"your_api_id",
  "request_time":"1524290015",
  "sign":"data_signature",
  "address":"1BpEi6DfDAUFd7GtittLSdBeYJvcoaVggu"
}
```

* response:

| key           | type     | comment            |
| ------- | ------ | ------------------------------------ |
| legacy_addr  | string | your input in Legacy Address format |
| cash_addr | string | your input in CashAddr format(starting with `bitcoincash:` which is a inseparable part of address, do not drop that) |

```json
{  
  "legacy_addr":"1BpEi6DfDAUFd7GtittLSdBeYJvcoaVggu",
  "cash_addr":"bitcoincash:qpm2qsznhks23z7629mms6s4cwef74vcwvy22gdx6a"
}
```

# Callbacks

You need to setup an endpoint to receive callbacks from Dabank,
which is triggered in these cases:

* deposit
  * a new deposit arrival, pending or already confirmed
  * the confirmation count of a pending deposit changes
  * the confirmation count of a previously pending deposit changes, which makes it confirmed
* withdrawal
  * the confirmation count of a pending withdrawal changes
  * the confirmation count of a previously pending withdrawal changes, which makes it confirmed
  
## Security Note

Dabank uses your secret key to generate data signature in the way mentioned in APIs chapter.
Check `sign` to avoid phishing, see chapter "Authentication".

## HTTP POST Request from Dabank

| key           | type     | comment            |
| ------------- | ------ | ---------------------------------------- |
| transfer_id   | string | internal transaction ID provided by Dabank, which you use to identity specific transaction |
| symbol        | string | symbol for coin type identification |
| confirms      | string | confirmation count(not applicable for off-chain internal transaction in Dabank) |
| tx_id         | string | transaction ID on cryptocurrency network   |
| transfer_at   | string | unix second of transaction initialization  |
| confirm_at    | string | unix second of transaction final confirmation  |
| status        | string | status of transaction |
| transfer_type | string | `IN` for incoming deposit, `OUT` for outgoing withdrawal  |
| to            | string | transaction destination address  |
| from          | string | transaction address of departure, may be empty when deposit |
| coins         | string | amount of transaction, whose max precision is 8 decimals  |
| fee           | string | fee charged to you for this specific transaction  |
| fee_symbol    | string | the symbol of fee charged, normally we charge the mainnet coin for token withdrawal on that network |

* Callback request example on a withdrawal:

```json
{  
   "transfer_id":"1365850",
   "symbol":"BTC",
   "tx_id":"7cee2e2055c85d40ab27f713d1bbcd7db2d07a8ab54483e7c423ed6cdc689007",
   "confirms":"2",
   "transfer_at":"1524298179",
   "confirm_at":"1524299426",
   "transfer_type":"OUT",
   "status":"TRANSFER_PENDING",
   "to":"1A3x2pU6gJKJAzc6XiZHKLhkBWSH6QkQo8",
   "from":"1G2BLexgX5dYwuzagBuSEp43Wy7Jgmm1mw",
   "coins":"2.73604069",
   "fee":"0.0008",
   "fee_symbol":"BTC",
   "request_time":"1524299426",
   "key":"your_api_id",
   "sign":"data_signature"
}
```

* Callback request example on a successful withdrawal:

```json
{  
   "transfer_id":"1365848",
   "symbol":"MGD",
   "tx_id":"bc2929b14ea303e198517d1550e30e1b8125f5f16fb373f7407f8a48cf605c7a",
   "confirms":"3",
   "transfer_at":"1524293497",
   "confirm_at":"1524294491",
   "transfer_type":"OUT",
   "status":"TRANSFER_SUCCESSFUL",
   "to":"MHBLixzhMs4RCfudx2jttU1SesxkjLxUrF",
   "from":"MPfFsz7GTSi4vQ5XEbMVRAZ67tDd6sQD21",
   "coins":"999.9",
   "fee":"0.001",
   "fee_symbol":"MGD",
   "request_time":"1524294491",
   "key":"your_api_id",
   "sign":"123"
}
```

* Callback request example on a successful deposit:

```json
{  
   "transfer_id":"1365849",
   "symbol":"ETH",
   "tx_id":"0x4ce2767bb3d039a5c62860cf51aec489ab1e287e62e9c60d1723186aee105bc7",
   "confirms":"13",
   "transfer_at":"1524296032",
   "confirm_at":"1524296243",
   "transfer_type":"IN",
   "status":"TRANSFER_SUCCESSFUL",
   "to":"0xc03db2681719eb3d41aa8ef5b23e11341fba57f0",
   "from":"",
   "coins":"2.997",
   "fee":"0",
   "fee_symbol":"ETH",
   "request_time":"1524296243",
   "key":"your_api_id",
   "sign":"123"
}
```

## Your HTTP Response to Dabank

You need to response to HTTP request from Dabank with following content:

| key           | type     | comment            |
| ------ | ------ | ------------------------ |
| result | string | `Success` when everything is fine, you may put error message when there is a problem |

```json
{  
   "result":"Success"
}
```

Dabank will retry the callback if you does not response with `Success`,
the gap between each retry is `2^n` in second, where `n` is the times of retry.

# Notes

## Monero Address

When withdrawing, user may provide a payment ID.
In that case, you should join destination address and payment ID
with `$` as `to` to call API, like following:

```
address: 9wHNWjJ1Gnw7Yw1RjDLNiJ8yJw91xFCz4NvXCcQjzceiGqFXDcXuCqPckgi7pn1LA4FZ5EaAUd19meb8GXxCp3iFT2yZViw
payment ID: 662f879677b96d0919db4ae069d985e5e3dcf10c8429ed395a9a6e798bb4f9d1

to = address + $ + payment ID

to: 9wHNWjJ1Gnw7Yw1RjDLNiJ8yJw91xFCz4NvXCcQjzceiGqFXDcXuCqPckgi7pn1LA4FZ5EaAUd19meb8GXxCp3iFT2yZViw$662f879677b96d0919db4ae069d985e5e3dcf10c8429ed395a9a6e798bb4f9d1
```

If there is no payment ID, use destination address as `to` directly.

## EOS Address

Dabank use memo to distinguish deposit of EOS and its tokens.
For example, `myeosaddress$9045` means the user need to send EOS
to `myeosaddress` with memo `9045`, where `$` is the separator.

When a user is withdrawing, if a memo is provided,
it should be attached to the destination address, with the separator `$`
like above mentioned.

# Symbols Supported

Note: max decimals APIs and callbacks supported is 8. 

<table border="0" style="border-collapse:collapse; text-align: center;">
<tr><th>symbol</th><th>name</th><th>decimals</th><th>is token of</th></tr>
<tr><td>BCD</td><td>Bitcoin Diamond</td><td>7</td><td>mainnet coin</td></tr>
<tr><td>BCH</td><td>Bitcoin Cash</td><td>8</td><td>mainnet coin</td></tr>
<tr><td>BTC</td><td>Bitcoin</td><td>8</td><td>mainnet coin</td></tr>
<tr><td>DCON</td><td>DCON</td><td>0</td><td>mainnet coin</td></tr>
<tr><td>EOS(mainnet)</td><td>EOS(mainnet)</td><td>4</td><td>mainnet coin</td></tr>
<tr><td>ETH</td><td>Ethereum</td><td>18</td><td>mainnet coin</td></tr>
<tr><td>LMC</td><td>Lomocoin</td><td>6</td><td>mainnet coin</td></tr>
<tr><td>LTC</td><td>Litecoin</td><td>8</td><td>mainnet coin</td></tr>
<tr><td>MGD</td><td>MassGrid</td><td>8</td><td>mainnet coin</td></tr>
<tr><td>QTUM</td><td>Qtum</td><td>8</td><td>mainnet coin</td></tr>
<tr><td>TRXmain</td><td>Tron mainnet</td><td>6</td><td>mainnet coin</td></tr>
<tr><td>WCG</td><td>World Crypto Gold</td><td>8</td><td>mainnet coin</td></tr>
<tr><td>XMC</td><td>XMC</td><td>0</td><td>mainnet coin</td></tr>
<tr><td>XMR</td><td>XMR</td><td>12</td><td>mainnet coin</td></tr>
<tr><td>DRT(WCG)</td><td>DRT(WCG Token)</td><td>4</td><td>WCG</td></tr>
<tr><td>AIMS</td><td>HighCastle Token</td><td>8</td><td>ETH</td></tr>
<tr><td>ALI</td><td>AiLink Token</td><td>18</td><td>ETH</td></tr>
<tr><td>ANTC</td><td>AntCoin</td><td>18</td><td>ETH</td></tr>
<tr><td>BFDT</td><td>BFDToken</td><td>18</td><td>ETH</td></tr>
<tr><td>BNB</td><td>Binance Token</td><td>18</td><td>ETH</td></tr>
<tr><td>BTK</td><td>Bitcoin Token</td><td>18</td><td>ETH</td></tr>
<tr><td>CDY</td><td>CDY</td><td>8</td><td>ETH</td></tr>
<tr><td>CRE</td><td>CybereitsToken</td><td>18</td><td>ETH</td></tr>
<tr><td>DKYC</td><td>Data Know Your Customer</td><td>18</td><td>ETH</td></tr>
<tr><td>EGT</td><td>Egretia</td><td>18</td><td>ETH</td></tr>
<tr><td>EGTY</td><td>EgtyChain</td><td>8</td><td>ETH</td></tr>
<tr><td>HOTC</td><td>HOTchain</td><td>18</td><td>ETH</td></tr>
<tr><td>HOTC(HOTCOIN)</td><td>HotCoin</td><td>0</td><td>ETH</td></tr>
<tr><td>HPB</td><td>HPBCoin </td><td>18</td><td>ETH</td></tr>
<tr><td>HYB</td><td>Hybrid</td><td>8</td><td>ETH</td></tr>
<tr><td>ICX</td><td>ICON</td><td>18</td><td>ETH</td></tr>
<tr><td>IDM</td><td>IDMONEY</td><td>18</td><td>ETH</td></tr>
<tr><td>IOST</td><td>IOSToken</td><td>18</td><td>ETH</td></tr>
<tr><td>JLL</td><td>JLL</td><td>18</td><td>ETH</td></tr>
<tr><td>KEY</td><td>SelfKey</td><td>18</td><td>ETH</td></tr>
<tr><td>LBA</td><td>LightBitAtom</td><td>8</td><td>ETH</td></tr>
<tr><td>LPK</td><td>Kripton</td><td>8</td><td>ETH</td></tr>
<tr><td>LRC</td><td>Loopring</td><td>18</td><td>ETH</td></tr>
<tr><td>LYS</td><td>LIGHTYEARS</td><td>8</td><td>ETH</td></tr>
<tr><td>MBC</td><td>MicroBusinessCoin</td><td>18</td><td>ETH</td></tr>
<tr><td>MOS</td><td>Moses</td><td>18</td><td>ETH</td></tr>
<tr><td>MUXE</td><td>MUXE Token</td><td>18</td><td>ETH</td></tr>
<tr><td>NRM</td><td>Neuromachine</td><td>18</td><td>ETH</td></tr>
<tr><td>POVR</td><td>POVCoin Token</td><td>18</td><td>ETH</td></tr>
<tr><td>READ</td><td>READ Token</td><td>8</td><td>ETH</td></tr>
<tr><td>RED</td><td>REDToken</td><td>18</td><td>ETH</td></tr>
<tr><td>RUFF</td><td>RUFF</td><td>18</td><td>ETH</td></tr>
<tr><td>THM</td><td>THM</td><td>18</td><td>ETH</td></tr>
<tr><td>TRUE</td><td>TRUE Token</td><td>18</td><td>ETH</td></tr>
<tr><td>USDT</td><td>Tether USD</td><td>6</td><td>ETH</td></tr>
<tr><td>USO</td><td>Ubiquity</td><td>8</td><td>ETH</td></tr>
<tr><td>VAR</td><td>Variant</td><td>18</td><td>ETH</td></tr>
<tr><td>VNS</td><td>VenusToken</td><td>18</td><td>ETH</td></tr>
<tr><td>XNS</td><td>SUNX</td><td>18</td><td>ETH</td></tr>
<tr><td>XRT</td><td>XRT Token</td><td>18</td><td>ETH</td></tr>
<tr><td>XT</td><td>XstarToken</td><td>18</td><td>ETH</td></tr>
<tr><td>ZRX</td><td>0x Protocol Token</td><td>18</td><td>ETH</td></tr>
<tr><td>ADD(EOS)</td><td>ADD(EOS Token)</td><td>4</td><td>EOS(mainnet)</td></tr>
<tr><td>USDT(Omni)</td><td>USDT(Omni Layer)</td><td>8</td><td>BTC</td></tr></table>

# Version

Dabank v3.4-Beta 2018-09-27

