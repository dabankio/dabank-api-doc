![logo](../LOGO.svg)


# 关键信息

## 参数类型
接口类型仅包括4种参数类型:
* 字符串型 **String**
* 数字型 **Number**
* 对象 **Object**
* 数组 **Array**

## 鉴权参数
所有接口(包括回调)需要附带以下三个鉴权限参数:

| 名字           | 类型     | 说明            |
| ------------ | ------ | ------------- |
| key          | String | 身份识别关键字       |
| sign         | String | 生成的签名         |
| request_time | Number | 请求时间戳, 防止重放攻击 |

后面接口中不再重复列出

签名生成规则步骤:

1. 取出 JSON 中所有 key-value 对中的 value, 放入数组 values
2. 将分配的 secret 放入数组 values
3. 对 values 进行字典序排序
4. 使用下划线`_` 连接 values 内的所有值, 得到字符串 content
5. 对 content 进行 md5，得到签名 `sign`
6. 将 `sign` 放入 JSON，完成。

## 结构说明

返回均包在以下结构的 data 中

| 名称         | 类型              | 说明                       |
| ---------- | --------------- | ------------------------ |
| err_code   | String          | 错误编码, 用于 i18n. (正确时为空字符) |
| err_info   | String          | 错误说明. (正确时为空字符)          |
| request_id | Number          | 请求id, 用于定位该请求            |
| data       | Object 或者 Array | 返回值                      |

```json
{ 
    "err_code": "ERR_ACCT_NOT_EXIST",
    "err_info": "account not exist",
    "data": {}
}
```

后面接口只列出 data 中内容

##  json 风格

参看上面的例子, 属性名遵照以下风格:

* 一律小写
* 下划线分割单词

# 接口列表

接口一律使用 `POST` 方式交互

## 申请地址

[htts://api.dabank.io/api/v3/address](htts://api.dabank.io/api/v3/address)

* request:

| 名称      | 类型     | 必传   | 说明   |
| ------- | ------ | ---- | ---- |
| symbol  | String | ✓    | 币种   |
| user_id | String | ✓    | 用户id |
```json
{  
   "key":"bigzhu",
   "request_time":"1524290015",
   "sign":"xxxx",
   "symbol":"TRX",
   "user_id":"1234"
}
```

response:

| 名称      | 类型     | 说明   |
| ------- | ------ | ---- |
| address | String | 地址   |

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

## 转账

[htts://api.dabank.io/api/v3/transfer](htts://api.dabank.io/api/v3/transfer)

request POST:

| 名称        | 类型     | 必传   | 说明                                       |
| --------- | ------ | ---- | ---------------------------------------- |
| symbol    | String | ✓    | 币种                                       |
| coins     | String | ✓    | 转账币数. 精度为小数点后8位                          |
| to        | String | ✓    | 转入目标地址                                   |
| from      | String | ✓    | 转出地址                                     |
| unique_id | String | ✓    | 调用方生成对本操作的唯一id. 出现接口调用超时等情况, 第二次重复调用时需要与第一次id保持一致, 以此避免因超时重复调用导致的重复转账 |

```json
{  
   "unique_id":"123",
   "to":"123",
   "sign":"98277e8d22205579a68e736e36832dcd",
   "coins":"2.54957926",
   "symbol":"ETH",
   "from":"123",
   "request_time":"1524298182",
   "key":"bigzhu"
}
```

response:

| 名称          | 类型     | 说明                                       |
| ----------- | ------ | ---------------------------------------- |
| transfer_id | String | 交易id. 请存储, 用于交易确认后回调关联                   |
| status      | String | TRANSFER_SUCCESSFUL(成功) 或 TRANSFER_PENDING(在途) |

针对门罗币(XMR)的特殊说明:
- 如果客户提币地址要求 address+paymenid 的形式, `to` 参数设置成 `address$payment_id` 即可， 即用`$`连接 `to` 和 `paymentid`, 例如:`9wHNWjJ1Gnw7Yw1RjDLNiJ8yJw91xFCz4NvXCcQjzceiGqFXDcXuCqPckgi7pn1LA4FZ5EaAUd19meb8GXxCp3iFT2yZViw$662f879677b96d0919db4ae069d985e5e3dcf10c8429ed395a9a6e798bb4f9d1`
- 如果不设置`paymentid`, 则不带$符号, 直接传入集成地址

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


## 成功账单查询

[htts://api.dabank.io/api/v3/transfers/success](htts://api.dabank.io/api/v3/transfers/success)

request POST:

| 名称            | 类型     | 必传   | 说明                  |
| ------------- | ------ | ---- | ------------------- |
| transfer_type | String | ✓    | 交易类型 IN(转入) OUT(转出) |
| symbol        | String | ✓    | 币种                  |
| limit         | String | ✓    | 一页几条                |
| p             | String | ✓    | 第几页                 |
| start_at      | String | ✗    | 开始时间(可选)            |
| end_at        | String | ✗    | 结束时间(可选)            |
| address       | String | ✗    | 转入地址(可选)            |

若要限定时间, start_at 与 end_at 必须同时传入, 否则当做未传入处理
```json
{  
   "key":"bigzhu",
   "request_time":"1524290015",
   "sign":"xxxx",
   "transfer_type":"OUT",
   "symbol":"ETH",
   "limit":"20",
   "p":"1",
   "start_at":"1522553241",
   "end_at":"1524539191",
   "address":"0x12345678910388342390012323"
}
```

response:

| 名称        | 类型     | 说明                    |
| --------- | ------ | --------------------- |
| total     | String | 总共几条                  |
| transfers | Array  | 账单列表, 里面存放 transer 对象 |

transer 对象结构: 

| 名称           | 类型     | 说明              |
| ------------ | ------ | --------------- |
| transfer_id  | String | 交易id            |
| symbol       | String | 币种              |
| to           | String | 转入地址            |
| from         | String | 转出地址            |
| coins        | String | 转账币数. 精度为小数点后8位 |
| confirmed_at | String | 交易完成时间          |
| tx_id        | String | 链上交易的唯一标示       |
| fee          | String | 手续费             |

```json
{  
   "total":"202",
   "transfers":[
    {
     "transfer_id":"123",
     "symbol":"ETH",
     "to":"0x12345678910388342390012323",
     "from":"0x12345678910388342390012321",
     "coins":"2.54957926",
     "confirmed_at":"1524539191",
     "tx_id":"0x4ce2767bb3d039a5c62860cf51aec489ab1e287e62e9c60d1723186aee105bc7",
     "fee":"0.001"
    },
     {
     "transfer_id":"124",
     "symbol":"ETH",
     "to":"0x12345678910388342390012323",
     "from":"0x12345678910388342390012321",
     "coins":"3.2326",
     "confirmed_at":"1524539191",
     "tx_id":"0x4ce2767bb3d039a5c62860cf51aec489ab1e287e62e9c60d1723186aee105bc7",
     "fee":"0.001"
    },
  ]
}
```

## 在途账单查询

[htts://api.dabank.io/api/v3/transfers/pending](htts://api.dabank.io/api/v3/transfers/pending)

request POST:

| 名称            | 类型     | 必传   | 说明                  |
| ------------- | ------ | ---- | ------------------- |
| transfer_type | String | ✓    | 交易类型 IN(转入) OUT(转出) |
| symbol        | String | ✓    | 币种                  |
| limit         | String | ✓    | 一页几条                |
| p             | String | ✓    | 第几页                 |
| start_at      | String | ✗    | 开始时间(可选)            |
| end_at        | String | ✗    | 结束时间(可选)            |
| address       | String | ✗    | 转入地址(可选)            |

若要限定时间, start_at 与 end_at 必须同时传入, 否则当做未传入处理

```json
{  
   "key":"bigzhu",
   "request_time":"1524290015",
   "sign":"xxxx",
   "transfer_type":"OUT",
   "symbol":"ETH",
   "limit":"20",
   "p":"1",
   "start_at":"1522553241",
   "end_at":"1524539191",
   "address":"0x12345678910388342390012323"
}
```


response:

| 名称        | 类型     | 说明                    |
| --------- | ------ | --------------------- |
| total     | String | 总共几条                  |
| transfers | Array  | 账单列表, 里面存放 transer 对象 |

transer 对象结构: 

| 名称          | 类型     | 说明              |
| ----------- | ------ | --------------- |
| transfer_id | String | 交易id            |
| symbol      | String | 币种              |
| to          | String | 转入地址            |
| from        | String | 转出地址            |
| coins       | String | 转账币数. 精度为小数点后8位 |
| transfer_at | String | 交易发起时间          |
| fee         | String | 手续费             |
| confirms    | String | 当前确认数           |

```json
{  
   "total":"32",
   "transfers":[
    {
     "transfer_id":"123",
     "symbol":"ETH",
     "to":"0x12345678910388342390012323",
     "from":"0x12345678910388342390012321",
     "coins":"2.54957926",
     "transfer_at":"1524539191",
     "fee":"0.001",
     "confirms":"1"
    },
    {
     "transfer_id":"124",
     "symbol":"ETH",
     "to":"0x12345678910388342390012323",
     "from":"0x12345678910388342390012321",
     "coins":"3.2326",
     "transfer_at":"1524539191",
     "fee":"0.001",
     "confirms":"1"
   }
  ]
}
```
## 账单总和

[htts://api.dabank.io/api/v3/transfers/sum](htts://api.dabank.io/api/v3/transfers/sum)

request POST:

| 名称            | 类型     | 必传   | 说明                  |
| ------------- | ------ | ---- | ------------------- |
| symbol        | String | ✓    | 币种                  |
| start_at      | String | ✗    | 开始时间(可选)            |
| end_at        | String | ✗    | 结束时间(可选)            |
| transfer_type | String | ✓    | 转账类型 IN(转入) OUT(转出) |

```json
{  
   "key":"bigzhu",
   "request_time":"1524290015",
   "sign":"xxxx",
   "symbol":"ETH",
   "start_at":"1522553241",
   "end_at":"1524539191",
   "transfer_type":"OUT"
}
```

response:

| 名称    | 类型     | 说明              |
| ----- | ------ | --------------- |
| coins | String | 转账币数. 精度为小数点后8位 |

```json
{  
   "coins":"2233.434944"
}
```

## 查询 APP 账户信息

[htts://api.dabank.io/api/v3/appAccount](htts://api.dabank.io/api/v3/appAccount)

request POST:

| 名称     | 类型     | 必传   | 说明               |
| ------ | ------ | ---- | ---------------- |
| symbol | String | ✓    | 币种(可选, 不传时查所有币种) |

```json
{  
  "key":"bigzhu",
  "request_time":"1524290015",
  "sign":"xxxx",
  "symbol":"ETH"
}
```

response:

data 里为 Array

| 名称      | 类型     | 说明            |
| ------- | ------ | ------------- |
| symbol  | String | 币种            |
| address | String | 地址            |
| balance | String | 余额. 精度为小数点后8位 |

```json
[
  {
  "symbol":"ETH",
  "address":"0x12345678910388342390012321",
  "balance":"234.5"
  },
  {
  "symbol":"BTC",
  "address":"0x123456789103883423900123242",
  "balance":"2.5"
  },
]
```

## 验证钱包地址正确性

[htts://api.dabank.io/api/v3/checkAddress](htts://api.dabank.io/api/v3/checkAddress)

request POST:

| 名称      | 类型     | 必传   | 说明   |
| ------- | ------ | ---- | ---- |
| symbol  | String | ✓    | 币种   |
| address | String | ✓    | 钱包地址 |

```json
{  
  "key":"bigzhu",
  "request_time":"1524290015",
  "sign":"xxxx",
  "symbol":"ETH",
  "address":"0x12345678910388342390012321"
}
```

response:

| 名称      | 类型     | 说明                          |
| ------- | ------ | --------------------------- |
| verify  | String | 验证结果，成功(Success)失败(Failure) |
| err_msg | String | 错误信息,verify成功时,此字段为空        |

```json
{  
  "verify":"Failure",
  "err_msg":"err info"
}
```

## 转换BCH地址

[htts://api.dabank.io/api/v3/bchAddrConvert](htts://api.dabank.io/api/v3/bchAddrConvert)

request POST:

| 名称    | 类型   | 必传 | 说明     |
| ------- | ------ | ---- | -------- |
| address | String | ✓    | BCH新版或旧版地址 |

```json
{  
  "key":"bigzhu",
  "request_time":"1524290015",
  "sign":"xxxx",
  "address":"1BpEi6DfDAUFd7GtittLSdBeYJvcoaVggu"
}
```

response:

| 名称    | 类型   | 说明                                 |
| ------- | ------ | ------------------------------------ |
| legacy_addr  | String | 旧版BCH钱包地址（类似于比特币） |
| cash_addr | String | 新版BCH钱包地址（以`bitcoincash:`开头，请注意，`bitcoincash:`是地址的一部分）   |

```json
{  
  "legacy_addr":"1BpEi6DfDAUFd7GtittLSdBeYJvcoaVggu",
  "cash_addr":"bitcoincash:qpm2qsznhks23z7629mms6s4cwef74vcwvy22gdx6a"
}
```

#  

# 回调

以下三种情况, 会回调接口进行通知:

* 转出, 确认数变化
* 转出, 转账成功后
* 转入

request POST:

| 名字            | 类型     | 说明                                       |
| ------------- | ------ | ---------------------------------------- |
| transfer_id   | String | 交易id                                     |
| symbol        | String | 币种                                       |
| confirms      | String | 确认数(仅 TRANSFER_PENDING 状态有效)             |
| tx_id         | String | txid                                     |
| transfer_at   | String | 交易发起时间                                   |
| confirm_at    | String | 交易确认时间                                   |
| status        | String | TRANSFER_SUCCESSFUL(成功) TRANSFER_PENDING(在途) |
| transfer_type | String | 转账类型 IN(转入) OUT(转出)                      |
| to            | String | 转入地址                                     |
| from          | String | 转出地址, 类型为转入(IN)时可能为空                     |
| coins         | String | 转账币数. 精度为小数点后8位                          |
| fee           | String | 手续费                                      |
| fee_symbol    | String | 手续费扣费币种                                  |

* 转出, 确认数变化

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
   "fee":"0.001",
   "fee_symbol":"BTC",
   "request_time":"1524299426",
   "key":"bigzhu",
   "sign":"123"
}
```
* 转出, 转账成功后

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
   "key":"bigzhu",
   "sign":"123"
}
```

* 转入

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
   "key":"bigzhu",
   "sign":"123"
}
```

response:

接口需要按照正确格式返回成功，否则会一直重试，重试间隔时间为2的重试次数次幂，单位为秒。

| 名字     | 类型     | 说明                       |
| ------ | ------ | ------------------------ |
| result | String | 成功时为 "Success", 失败时为错误信息 |

```json
{  
   "result":"Success"
}
```

# 支持的币种以及小数位数

以下数据截止2018年09月07日，共61个（币种小数位超过8位的支持到8位，低于等于8位的支持实际的小数位数）：

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
<tr><td>BTM</td><td>Bytom</td><td>8</td><td>ETH</td></tr>
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

请参阅[提交历史](https://github.com/dabankio/dabank-api-doc/commits/master)。

