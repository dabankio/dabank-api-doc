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

**keys**

There are two sets of key-pair involved in SHA256-RSA Authentication:

1. You use `app_private_key`(kept as secret) to sign your API requests, and you upload `app_public_key` to Dabank for request verification;
1. You download `dabank_public_key` for callback verification, as Dabank uses `dabank_private_key` to sign callbacks.

**Generation of `app_private_key` and `app_public_key`**

You may use OpenSSL to generate RSA key pairs, note the key length is 2048.

```bash
openssl genrsa -out app_private_key.pem 2048
openssl rsa -in app_private_key.pem -pubout -out app_public_key.pem
```

**Steps to generate `sign`(for API requests)**

`sign` is generated in following steps:

1. extract all keys and values in JSON's key-value pair(except `sign`) in the form of `key=value`, as array `param`;
1. sort `param` in lexical order;
1. join `param` with `&` to obtain string `content`;
1. calculate `sha256` of `content` to get `hashed`;
1. calculate the signature of `hashed` with `app_private_key` using [RSASSA-PKCS1-V1_5-SIGN](https://tools.ietf.org/html/rfc3447#page-33) from RSA PKCS#1 v1.5 to get `rsa_sign`;
1. encode `rsa_sign` with base64 and you get `sign` of your current request.

**Steps to verify `sign`(for Dabank callback verification)**

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

* test-APIs are hosted at `https://api-test.dabank.cf/`.
* formal-APIs are hosted at `https://api.dabank.io/`.

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

## Get Statistics of Your Transactions between a Time Range

```
/api/v3/reconciliation
```

* request:

| key      | type     |  compulsory  | comment   |
| ------ | ------ | ---- | ---------------- |
| start_time | string | ✓   | starting time of the time range(included) |
| end_time | string | ✓   | ending time of the time range(not included) |


```json
{  
  "key":"your_api_id",
  "request_time":"current_time_in_unix_second",
  "sign":"data_signature",
  "start_time": "started_time_in_unix_second",
  "end_time": "ended_time_in_unix_second"
}
```

* response:

| key           | type     | comment            |
| ------- | ------ | ------------- |
| symbol  | string | symbol for coin type identification            |
| fee_symbol | string | symbol in which fee incurs     |
| fee | float64 | fee accumulated during this time range |

The rest keys should be self-documented.

```json
{
    "err_code": "",
    "err_info": "",
    "data": {
        "statistics": [
            {
                "symbol": "XRP",
                "fee_symbol": "XRP",
                "fee": 0.43,
                "in_successful_coin": 34819.302933,
                "in_successful_count": 22,
                "in_pending_coin": 0,
                "in_pending_count": 0,
                "out_successful_coin": 161692.215504,
                "out_successful_count": 43,
                "out_failed_coin": 0,
                "out_failed_count": 0,
                "out_pending_coin": 0,
                "out_pending_count": 0
            },
            {
                            "symbol": "ETH",
                            "fee_symbol": "ETH",
                            "fee": 0.024,
                            "in_successful_coin": 114.48484908,
                            "in_successful_count": 9,
                            "in_pending_coin": 0,
                            "in_pending_count": 0,
                            "out_successful_coin": 39.17130422,
                            "out_successful_count": 6,
                            "out_failed_coin": 0,
                            "out_failed_count": 0,
                            "out_pending_coin": 0,
                            "out_pending_count": 0
            }
        ],
        "realtime_balance": [
            {
                "symbol": "XRP",
                "available_balance": 50000,
                "freeze": 0
            },
            {
                "symbol": "USDT(Omni)",
                "available_balance": 50000,
                "freeze": 0
            },
            {
                "symbol": "BTC",
                "available_balance": 50000,
                "freeze": 0
            },
            {
                "symbol": "ETH",
                "available_balance": 50000,
                "freeze": 0
            }
        ]
    },
    "request_id": 134
}
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
<tr><th>symbol</th><th>token_of</th><th>decimals</th><th>contract_address</th></tr>
<tr><td>AAESP(WCG2)</td><td>WCG</td><td>4</td><td>14166359669915381351</td></tr>
<tr><td>ABB</td><td>ETH</td><td>18</td><td>0xbde6ff7ff944aa8ef554410572dfee184d25302a</td></tr>
<tr><td>ADD(EOS)</td><td>EOS(mainnet)</td><td>4</td><td>eosadddddddd</td></tr>
<tr><td>AIMS</td><td>ETH</td><td>8</td><td>0x9a1c7577a69ea9e296ef99bfb0eb6bedbe352a36</td></tr>
<tr><td>ALI</td><td>ETH</td><td>18</td><td>0x4289c043a12392f1027307fb58272d8ebd853912</td></tr>
<tr><td>ANTC</td><td>ETH</td><td>18</td><td>0xa8953562660855aaa3b2b2af9398e16246dfa597</td></tr>
<tr><td>AXPR</td><td>ETH</td><td>18</td><td>0xc39e626a04c5971d770e319760d7926502975e47</td></tr>
<tr><td>BCD</td><td></td><td>7</td><td></td></tr>
<tr><td>BCH</td><td></td><td>8</td><td></td></tr>
<tr><td>BFDT</td><td>ETH</td><td>18</td><td>0xd2d0f85b690604c245f61513bf4679b24ed64c35</td></tr>
<tr><td>BIDT</td><td>ETH</td><td>18</td><td>0x9206dadafd7555baf1bf004af18911152bf7f55b</td></tr>
<tr><td>BNB</td><td>ETH</td><td>18</td><td>0xb8c77482e45f1f44de1745f52c74426c631bdd52</td></tr>
<tr><td>BTC</td><td></td><td>8</td><td></td></tr>
<tr><td>BTK</td><td>ETH</td><td>18</td><td>0xdb8646f5b487b5dd979fac618350e85018f557d4</td></tr>
<tr><td>BTM</td><td>ETH</td><td>8</td><td>0xcb97e65f07da24d46bcdd078ebebd7c6e6e3d750</td></tr>
<tr><td>CDY</td><td>ETH</td><td>8</td><td>0x10f135ce102eee47dca8fe8e19dca6324dfbd684</td></tr>
<tr><td>CFUN(ERC20)</td><td>ETH</td><td>9</td><td>0xd6ed0e5d7f854b64b5e467a240a6c155c17cc6a2</td></tr>
<tr><td>CRE</td><td>ETH</td><td>18</td><td>0x61f33da40594cec1e3dc900faf99f861d01e2e7d</td></tr>
<tr><td>CTM(WCG2)</td><td>WCG2</td><td>4</td><td>7171868846042689336</td></tr>
<tr><td>CVN</td><td>ETH</td><td>5</td><td>0x62aaf435273bc4baa78dcebd6590042d7e58ba6f</td></tr>
<tr><td>DAILY</td><td>ETH</td><td>8</td><td>0xbc63f11d42860d7e41a4ce856581f0ce15ac4008</td></tr>
<tr><td>DATX</td><td>ETH</td><td>18</td><td>0xabbbb6447b68ffd6141da77c18c7b5876ed6c5ab</td></tr>
<tr><td>DCON</td><td></td><td>0</td><td></td></tr>
<tr><td>DKYC</td><td>ETH</td><td>18</td><td>0x38d1b0d157529bd5d936719a8a5f8379afb24faa</td></tr>
<tr><td>DRT(WCG2)</td><td>WCG2</td><td>4</td><td>7776054229687920460</td></tr>
<tr><td>EGT</td><td>ETH</td><td>18</td><td>0x8e1b448ec7adfc7fa35fc2e885678bd323176e34</td></tr>
<tr><td>EGTY</td><td>ETH</td><td>8</td><td>0x22c422d57638ff48af2e471e6f9fa5dc5b5edbd5</td></tr>
<tr><td>EOS(mainnet)</td><td></td><td>4</td><td>eosio.token</td></tr>
<tr><td>EQT(WCG2)</td><td>WCG2</td><td>4</td><td>17662811370611592334</td></tr>
<tr><td>ETH</td><td></td><td>18</td><td></td></tr>
<tr><td>FCS</td><td>ETH</td><td>18</td><td>0x574b5b500517ccad92c78573c7daf8be8cc0de19</td></tr>
<tr><td>FREC</td><td>ETH</td><td>18</td><td>0x17e67d1cb4e349b9ca4bc3e17c7df2a397a7bb64</td></tr>
<tr><td>GGP</td><td>ETH</td><td>5</td><td>0xcaa8a02ee6f6ed7cc12449041d4426fb10626b6f</td></tr>
<tr><td>GUSD</td><td>ETH</td><td>2</td><td>0x056fd409e1d7a124bd7017459dfea2f387b6d5cd</td></tr>
<tr><td>GVM(WCG2)</td><td>WCG2</td><td>4</td><td>11835391898538536373</td></tr>
<tr><td>HOTC</td><td>ETH</td><td>18</td><td>0x4d09c5e758ca68be27240f29fb681e5a5341ca98</td></tr>
<tr><td>HOTC(HOTCOIN)</td><td>ETH</td><td>0</td><td>0x5ebdc890cdb9312f9c8cd3d8ddc28fe9509f9923</td></tr>
<tr><td>HPB</td><td>ETH</td><td>18</td><td>0x38c6a68304cdefb9bec48bbfaaba5c5b47818bb2</td></tr>
<tr><td>HYB</td><td>ETH</td><td>8</td><td>0x00d53126139c547c7bd4f4285fc3756c2f081ab1</td></tr>
<tr><td>ICX</td><td>ETH</td><td>18</td><td>0xb5a5f22694352c15b00323844ad545abb2b11028</td></tr>
<tr><td>IDM</td><td>ETH</td><td>18</td><td>0x28b85bc9e851271214fe70f433874b9aedc5a94a</td></tr>
<tr><td>IOST</td><td>ETH</td><td>18</td><td>0xfa1a856cfa3409cfa145fa4e20eb270df3eb21ab</td></tr>
<tr><td>IPC</td><td></td><td>8</td><td></td></tr>
<tr><td>ITU</td><td>ETH</td><td>18</td><td>0xe9024599539b135c1eb889c1e70a71241631bc45</td></tr>
<tr><td>JLL</td><td>ETH</td><td>18</td><td>0x5661c46e366570360064ae1a50a17a7a1a8f3236</td></tr>
<tr><td>JOBS</td><td>ETH</td><td>8</td><td>0x994ad6d288f1797ad9e09682970ba474fc3e1fbc</td></tr>
<tr><td>KEY</td><td>ETH</td><td>18</td><td>0x4cc19356f2d37338b9802aa8e8fc58b0373296e7</td></tr>
<tr><td>LBA</td><td>ETH</td><td>8</td><td>0xae2592b477ac500068a7925ddfc9ee448fb58242</td></tr>
<tr><td>LMC</td><td></td><td>6</td><td></td></tr>
<tr><td>LPK</td><td>ETH</td><td>8</td><td>0x2cc71c048a804da930e28e93f3211dc03c702995</td></tr>
<tr><td>LRC</td><td>ETH</td><td>18</td><td>0xef68e7c694f40c8202821edf525de3782458639f</td></tr>
<tr><td>LTC</td><td></td><td>8</td><td></td></tr>
<tr><td>LYS</td><td>ETH</td><td>8</td><td>0xdd41fbd1ae95c5d9b198174a28e04be6b3d1aa27</td></tr>
<tr><td>MAT(WCG2)</td><td>WCG2</td><td>4</td><td>6316460833260624272</td></tr>
<tr><td>MBC</td><td>ETH</td><td>18</td><td>0x9c007b020685387fc93583d8a7eb97506df0f26a</td></tr>
<tr><td>MGD</td><td></td><td>8</td><td></td></tr>
<tr><td>MNR</td><td>ETH</td><td>18</td><td>0x5f824733d130ad85ec5e180368559cc89d14933d</td></tr>
<tr><td>MOS</td><td>ETH</td><td>18</td><td>0x420a43153da24b9e2aedcec2b8158a8653a3317e</td></tr>
<tr><td>MTR(WCG2)</td><td>WCG2</td><td>4</td><td>12642904667336691118</td></tr>
<tr><td>MUXE</td><td>ETH</td><td>18</td><td>0x515669d308f887fd83a471c7764f5d084886d34d</td></tr>
<tr><td>NRM</td><td>ETH</td><td>18</td><td>0x00000b233566fcc3825f94d68d4fc410f8cb2300</td></tr>
<tr><td>NSC</td><td>ETH</td><td>18</td><td>0x827b67176c20edf0f0d3eb76b0c53aa925018f2b</td></tr>
<tr><td>PAX</td><td>ETH</td><td>18</td><td>0x8e870d67f660d95d5be530380d0ec0bd388289e1</td></tr>
<tr><td>PBK</td><td>ETH</td><td>7</td><td>0x077dc3c0c9543df1cdd78386df3204e69e0dd274</td></tr>
<tr><td>PIPS</td><td>ETH</td><td>18</td><td>0x59db9fde270b39a07f38fa3106a760829074c7d9</td></tr>
<tr><td>POVR</td><td>ETH</td><td>18</td><td>0xc26925d537af8b3f315eeaf27113e84875b6f1b9</td></tr>
<tr><td>QOS</td><td>ETH</td><td>4</td><td>0x7b188a8b3a2113621895fb35fc67a779caffa92d</td></tr>
<tr><td>QTUM</td><td></td><td>8</td><td></td></tr>
<tr><td>READ</td><td>ETH</td><td>8</td><td>0x13d0bf45e5f319fa0b58900807049f23cae7c40d</td></tr>
<tr><td>RED</td><td>ETH</td><td>18</td><td>0x76960dccd5a1fe799f7c29be9f19ceb4627aeb2f</td></tr>
<tr><td>REDC</td><td>ETH</td><td>18</td><td>0xb563300a3bac79fc09b93b6f84ce0d4465a2ac27</td></tr>
<tr><td>RUFF</td><td>ETH</td><td>18</td><td>0xf278c1ca969095ffddded020290cf8b5c424ace2</td></tr>
<tr><td>SNCOIN</td><td>ETH</td><td>18</td><td>0x8c211128f8d232935afd80543e442f894a4355b7</td></tr>
<tr><td>TASH</td><td>ETH</td><td>18</td><td>0x002f06abe6995fd0ea4be011c53bfc989fa53ce0</td></tr>
<tr><td>THM</td><td>ETH</td><td>18</td><td>0xf1dd5964eabcc6e86230fa6f222677cfdaaf9f0e</td></tr>
<tr><td>TLT(WCG2)</td><td>WCG2</td><td>4</td><td>2721668346522778691</td></tr>
<tr><td>TRIO</td><td>ETH</td><td>18</td><td>0x8b40761142b9aa6dc8964e61d0585995425c3d94</td></tr>
<tr><td>TRUE</td><td>ETH</td><td>18</td><td>0xa4d17ab1ee0efdd23edc2869e7ba96b89eecf9ab</td></tr>
<tr><td>TRX</td><td>ETH</td><td>6</td><td>0xf230b790e05390fc8295f4d3f60332c93bed42e2</td></tr>
<tr><td>TRXmain</td><td></td><td>6</td><td></td></tr>
<tr><td>USDT</td><td>ETH</td><td>6</td><td>0xdac17f958d2ee523a2206206994597c13d831ec7</td></tr>
<tr><td>USDTK(WCG2)</td><td>WCG2</td><td>2</td><td>11164589766816208741</td></tr>
<tr><td>USDT(Omni)</td><td>BTC</td><td>8</td><td>31</td></tr>
<tr><td>USO</td><td>ETH</td><td>8</td><td>0x6c2428a17bb9074f93563575d9c5c492f268cfb4</td></tr>
<tr><td>VAR</td><td>ETH</td><td>18</td><td>0x828bfe8d626695a75184aa9b35cbdd1ec3eafaae</td></tr>
<tr><td>VNS</td><td>ETH</td><td>18</td><td>0x1e4e36b3f011d862fd70006804da8fcefe89d3d8</td></tr>
<tr><td>WCG2</td><td></td><td>8</td><td></td></tr>
<tr><td>WEN(WCG2)</td><td>WCG2</td><td>4</td><td>8342642094929787676</td></tr>
<tr><td>WOS(WCG2)</td><td>WCG2</td><td>4</td><td>411043316357352222</td></tr>
<tr><td>XLP</td><td>ETH</td><td>2</td><td>0x18017350089c7219f5b9a9b704c459c1a2814063</td></tr>
<tr><td>XMC</td><td></td><td>0</td><td></td></tr>
<tr><td>XMR</td><td></td><td>12</td><td></td></tr>
<tr><td>XNS</td><td>ETH</td><td>18</td><td>0xbf2f1e0e41f1e144bb1d5870e94324b9878dd95b</td></tr>
<tr><td>XRP</td><td></td><td>6</td><td></td></tr>
<tr><td>XRT</td><td>ETH</td><td>18</td><td>0x37d404a072056eda0cd10cb714d35552329f8500</td></tr>
<tr><td>XT</td><td>ETH</td><td>18</td><td>0x528d068ae69e90c90ad090b3c2b0d18241e9e9b5</td></tr>
<tr><td>ZRX</td><td>ETH</td><td>18</td><td>0xe41d2489571d322189246dafa5ebde1f4699f498</td></tr>
</table>

# Version

Dabank v3.4 2019-01

