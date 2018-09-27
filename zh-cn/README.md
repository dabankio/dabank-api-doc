![logo](../LOGO.svg)

# Dabank

Dabank（大棒客）是一个商用加密货币托管服务，支持几十个币种和很多类型的token。

Dabank能让你快速地为你的客户提供加密货币转入、存储以及转出服务。

Dabank也提供冷钱包业务。

# 综述

## 数据交换

应用（即接入方）与Dabank进行双向通讯：

* 应用通过调用Dabank API来完成转入地址申请、转出申请等业务；
* Dabank回调应用，为应用提供相关信息，如有新的转入，转入/转出被网络确认等。

## 请求方式

所有的API请求使用HTTP POST方法，并且需要附带请求头`Content-Type`，值为`application/json; charset=utf-8`.

## 基本数据类型

应用与Dabank的通讯在绝大部分情况使用字符串作为基本数据类型，请注意，整形（int）和浮点（float）同样以字符串的形式编码。

## API和回调的签名

### SHA256-RSA签名方案

所有接口（包括回调）需要附带以下三个鉴权限参数:

| 名字           | 类型     | 说明            |
| ------------ | ------ | ------------- |
| key          | string | 身份识别关键字       |
| sign         | string | SHA256-RSA风格的签名         |
| request_time | string | 当前的UNIX秒 |

* 应用和Dabank的密钥

本方案中，有两对密钥参与签名和验证：

1. 应用使用`app_private_key`（应用视为机密）签署API请求，Dabank使用应用上传的`app_public_key`进行验证；
1. 应用从Dabank处下载`dabank_public_key`来验证来自于Dabank的回调，Dabank使用`dabank_private_key`对这些回调进行签署。

* `app_private_key`和`app_public_key`的生成

应用可以使用OpenSSL来生成RSA密钥对，私钥的长度要求为2048：

```bash
openssl genrsa -out app_private_key.pem 2048
openssl rsa -in app_private_key.pem -pubout -out app_public_key.pem
```

* 签名生成步骤

签名按如下方式生成：

1. 将JSON中的所有key-value对（`sign`除外）记为`key=value`形式的字符串，放入数组`param`；
1. 对`param`进行字典排序；
1. 使用`&`连接`param`内的所有值, 得到字符串`content`；
1. 对`content`进行SHA256，得到哈希`hashed`；
1. 使用[RSASSA-PKCS1-V1_5-SIGN](https://tools.ietf.org/html/rfc3447#page-33)以及`app_private_key`对`hashed`签名，得到`rsa_sign`；
1. 使用base64对`rsa_sign`进行编码，得到`sign`；
1. 将`sign`放入 JSON，完成。

### HMAC风格的签名方案（已废弃）

HMAC风格的签名方案已被废弃，仅作为现有应用的过渡期兼容方案，Dabank在将来会停止支持本鉴权方案。

所有接口（包括回调）需要附带以下三个鉴权限参数:

| 名字           | 类型     | 说明            |
| ------------ | ------ | ------------- |
| key          | string | 身份识别关键字       |
| sign         | string | HMAC风格的签名         |
| request_time | string | 当前的UNIX秒 |

这些参数在接口文档中不再重复列出。

* 签名生成步骤

1. 取出JSON中所有 key-value 对（`sign`除外）中的value，放入数组`values`；
1. 将从Dabank取得的`secret`放入数组`values`；
1. 对`values`进行字典序排序；
1. 使用下划线`_`连接`values`内的所有值，得到字符串`content`；
1. 对`content`进行 md5，得到签名`sign`；
1. 将`sign`放入 JSON，完成。

## API返回值结构

返回均包在以下结构的JSON中：

| 名称         | 类型              | 说明                       |
| ---------- | --------------- | ------------------------ |
| err_code   | string          | 错误编码 |
| err_info   | string          | 错误说明   |
| request_id | int          | 请求id   |
| data       | object/array | 返回值                      |

举例：

```json
{ 
    "err_code": "ERR_ACCT_NOT_EXIST",
    "err_info": "account not exist, check your key",
    "data":null
}
```

后面接口只列出 data 中内容。

##  json 风格

参看上面的例子, 属性名遵照`snake_case`，即:

* 一律小写
* 下划线分割单词

# 接口列表

Dabank API位于`https://api.dabank.io/`.

## 转入地址申请

```
/api/v3/address
```

* request:

| 名称      | 类型     | 必传   | 说明   |
| ------- | ------ | ---- | ---- |
| symbol  | string | ✓    | 币种   |
| user_id | string | ✓    | 可以唯一标识您的用户的ID |


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

| 名称      | 类型     | 说明   |
| ------- | ------ | ---- |
| address | string | 提供给您的用户的转入地址   |

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

## 转出提币申请

```
/api/v3/transfer
```

* request:

| 名称        | 类型     | 必传   | 说明                                       |
| --------- | ------ | ---- | ---------------------------------------- |
| symbol    | string | ✓    | 币种                                       |
| coins     | string | ✓    | 转账币数  |
| to        | string | ✓    | 提币目标地址                                   |
| from      | string | ✓    | 提币发起地址                                     |
| unique_id | string | ✓    | 调用方生成对本操作的唯一ID，如果应用打算重试同一笔交易，需要提供相同的`unique_id` |

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

| 名称          | 类型     | 说明                                       |
| ----------- | ------ | ---------------------------------------- |
| transfer_id | string | 交易ID，请存储, 用于交易确认后回调关联                   |
| status      | string | TRANSFER_SUCCESSFUL（成功） 或 TRANSFER_PENDING（在途） |



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

## 查询应用服务套餐内各币种的余额

```
/api/v3/appAccount
```

* request:

| 名称     | 类型     | 必传   | 说明               |
| ------ | ------ | ---- | ---------------- |
|  |  |  |  |

```json
{  
  "key":"bigzhu",
  "request_time":"1524290015",
  "sign":"xxxx",
  "symbol":"ETH"
}
```

* response:

data 里为 Array

| 名称      | 类型     | 说明            |
| ------- | ------ | ------------- |
| symbol  | string | 币种            |
| address | string | 应用的企业地址            |
| balance | string | 该币种的可用余额 |

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

## 验证地址正确性

```
/api/v3/checkAddress
```

验证指定地址是否为合法的地址。
请注意，“合法”的地址未必是“正确”的地址，地址的正确性应该由您的用户自己来保证。

* request:

| 名称      | 类型     | 必传   | 说明   |
| ------- | ------ | ---- | ---- |
| symbol  | string | ✓    | 币种   |
| address | string | ✓    | 待验证地址 |

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

| 名称      | 类型     | 说明                          |
| ------- | ------ | --------------------------- |
| verify  | string | 验证结果，`Success`为合法，`Failure`为非法 |
| err_msg | string | `Failure`时，这里会提供更多信息    |

```json
{  
  "verify":"Failure",
  "err_msg":"ETH address should be started with \"0x\""
}
```

## BCH新老地址转换

对于Bitcoin Cash(BCH)，Dabank一律使用新版CashAddr，
旧版的地址在提币时会被自动转换。

Dabank知晓部分交易所还不支持提币到CashAddr地址，
您可使用本接口转换Dabank提供的充币地址，
供您的用户进行BCH充币。

```
/api/v3/bchAddrConvert
```

* request:

| 名称    | 类型   | 必传 | 说明     |
| ------- | ------ | ---- | -------- |
| address | string | ✓    | BCH新版或旧版地址 |

```json
{  
  "key":"your_api_id",
  "request_time":"1524290015",
  "sign":"data_signature",
  "address":"1BpEi6DfDAUFd7GtittLSdBeYJvcoaVggu"
}
```

* response:

| 名称    | 类型   | 说明                                 |
| ------- | ------ | ------------------------------------ |
| legacy_addr  | string | 旧版BCH钱包地址（类似于比特币） |
| cash_addr | string | 新版BCH钱包地址（以`bitcoincash:`开头，请注意，`bitcoincash:`是地址的一部分）   |

```json
{  
  "legacy_addr":"1BpEi6DfDAUFd7GtittLSdBeYJvcoaVggu",
  "cash_addr":"bitcoincash:qpm2qsznhks23z7629mms6s4cwef74vcwvy22gdx6a"
}
```

# 回调

以下三种情况，Dabank会回调应用进行通知:

* 转出，网络确认数变化
* 转出，网络确认转账成功
* 转入，网络确认数变化
* 转入，网络确认转账成功

## 安全须知

应用须验证回调签名，以确认请求确实来自于Dabank而非伪造，签名的验证请参考“API和回调的签名”章节。

## Dabank的HTTP POST回调

| 名字            | 类型     | 说明                                       |
| ------------- | ------ | ---------------------------------------- |
| transfer_id   | string | 交易id                                     |
| symbol        | string | 币种                                       |
| confirms      | string | 确认数(仅 TRANSFER_PENDING 状态有效)             |
| tx_id         | string | txid                                     |
| transfer_at   | string | 交易发起时间                                   |
| confirm_at    | string | 交易确认时间                                   |
| status        | string | TRANSFER_SUCCESSFUL(成功) TRANSFER_PENDING(在途) |
| transfer_type | string | 转账类型 IN(转入) OUT(转出)                      |
| to            | string | 转入地址                                     |
| from          | string | 转出地址, 类型为转入(IN)时可能为空                     |
| coins         | string | 转账币数. 精度为小数点后8位                          |
| fee           | string | 手续费                                      |
| fee_symbol    | string | 手续费扣费币种                                  |

* 转出，网络确认数变化的例子：

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

* 转出，网络确认转账成功的例子：

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

* 转入的例子：

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

## 应用对Dabank的HTTP Response

接口需要按照正确格式返回成功，否则会一直重试，重试间隔时间为2的重试次数次幂，单位为秒。

| 名字     | 类型     | 说明                       |
| ------ | ------ | ------------------------ |
| result | string | 成功时为 "Success", 失败时为错误信息 |

```json
{  
   "result":"Success"
}
```

# 附注

## 门罗币（XMR）的地址说明

* 如果客户提币地址要求 address+paymenid 的形式, `to` 参数设置成 `address$payment_id` 即可， 即用`$`连接 `to` 和 `paymentid`, 例如:`9wHNWjJ1Gnw7Yw1RjDLNiJ8yJw91xFCz4NvXCcQjzceiGqFXDcXuCqPckgi7pn1LA4FZ5EaAUd19meb8GXxCp3iFT2yZViw$662f879677b96d0919db4ae069d985e5e3dcf10c8429ed395a9a6e798bb4f9d1`
* 如果不设置`paymentid`, 则不带$符号, 直接传入集成地址

## EOS的地址说明

* 充币地址申请

申请到的地址分为`主地址`和`子地址`两个部分，两个部分在`address`字段中返回，以`$`拼接（与门罗类似），形如`主地址$子地址`。

字符集：

* 主地址的字符集为小写a-z，数字1-5，长度1-12位
* 子地址字符集为ASCII，但Dabank提供的子地址为纯数字

* 用户充币

Dabank在回调时会提供`主地址$子地址`

* 用户提币

应用为用户提供`地址`和`转账附言(memo)`两个输入框，分别对应`主地址`和`子地址`。

应用以`主地址$子地址`的形式调用Dabank接口。

* 地址的特殊情况

* 子地址为空时，记为`主地址`
* 子地址中含有`$`时，照常以`主地址$子地址`形式传递，因为主地址的字符集不含`$`，我们可以安全约定第一个`$`为分隔符，其后的所有内容为子地址

* 地址校验

地址校验接口只校验主地址的有效性，出于便利考虑，应用可以传递`$子地址`部分，这部分不会参与校验。

# 支持的币种以及小数位数

注：币种小数位超过8位的支持到8位，低于等于8位的支持实际的小数位数。

<table border="0" style="border-collapse:collapse; text-align: center;">
<tr><th>symbol</th><th>cn_name</th><th>decimals</th><th>token_of</th></tr>
<tr><td>BCD</td><td>Bitcoin Diamond</td><td>7</td><td>主网</td></tr>
<tr><td>BCH</td><td>比特币现金</td><td>8</td><td>主网</td></tr>
<tr><td>BTC</td><td>比特币</td><td>8</td><td>主网</td></tr>
<tr><td>DCON</td><td>DCON</td><td>0</td><td>主网</td></tr>
<tr><td>EOS(mainnet)</td><td>EOS(mainnet)</td><td>4</td><td>主网</td></tr>
<tr><td>ETH</td><td>以太坊</td><td>18</td><td>主网</td></tr>
<tr><td>LMC</td><td>邻萌宝</td><td>6</td><td>主网</td></tr>
<tr><td>LTC</td><td>莱特币</td><td>8</td><td>主网</td></tr>
<tr><td>MGD</td><td>MGD</td><td>8</td><td>主网</td></tr>
<tr><td>QTUM</td><td>量子</td><td>8</td><td>主网</td></tr>
<tr><td>TRXmain</td><td>波场主网</td><td>6</td><td>主网</td></tr>
<tr><td>WCG</td><td>WCG</td><td>8</td><td>主网</td></tr>
<tr><td>XMC</td><td>XMC</td><td>0</td><td>主网</td></tr>
<tr><td>XMR</td><td>XMR</td><td>12</td><td>主网</td></tr>
<tr><td>DRT(WCG)</td><td>DRT(WCG)</td><td>4</td><td>WCG</td></tr>
<tr><td>AIMS</td><td>HighCastle Token</td><td>8</td><td>ETH</td></tr>
<tr><td>ALI</td><td>AiLink Token</td><td>18</td><td>ETH</td></tr>
<tr><td>ANTC</td><td>AntCoin</td><td>18</td><td>ETH</td></tr>
<tr><td>BFDT</td><td>BFDToken</td><td>18</td><td>ETH</td></tr>
<tr><td>BNB</td><td>币安币</td><td>18</td><td>ETH</td></tr>
<tr><td>BTK</td><td>BitcoinToken</td><td>18</td><td>ETH</td></tr>
<tr><td>CDY</td><td>CDY</td><td>8</td><td>ETH</td></tr>
<tr><td>CFUN(ERC20)</td><td>CFun Token</td><td>9</td><td>ETH</td></tr>
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
<tr><td>READ</td><td>阅读币</td><td>8</td><td>ETH</td></tr>
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
<tr><td>ADD(EOS)</td><td>ADD(EOS)</td><td>4</td><td>EOS(mainnet)</td></tr>
<tr><td>USDT(Omni)</td><td>USDT（比特币）</td><td>8</td><td>BTC</td></tr></table>

# 版本

Dabank v3.4-Beta 2018-09-27

