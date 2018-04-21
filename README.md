![logo](https://cdn.golangme.com/static/img/carousel/golangroutines.png)


# 版本
| 序号   | 日期         | 修订内容                                     | 修订人    |
| ---- | ---------- | ---------------------------------------- | ------ |
| 16   | 2018-04-21 | 补充描述内容, 增加部分接口 json 例子              | 朱一凡     |
| 15   | 2018-04-18 | 添加支持的币种及小数位数              | 朱一凡     |
| 14   | 2018-02-29 | 回调地址增加手续费扣费币种              | 朱一凡     |
| 13   | 2018-02-29 | 将所有接口的int或者float类型全转为string              | 王东     |
| 12   | 2018-02-28 | transfer 接口增加  unique_id, 以避免重复处理        | 朱一凡    |
| 11   | 2018-02-28 | 返回request_id, 回调增加确认数                    | 朱一凡    |
| 10   | 2018-02-26 | 申请地址时需要传入用户id                            | 朱一凡    |
| 9    | 2018-02-24 | 简化字段名                                    | 朱一凡    |
| 8    | 2018-01-25 | 账单查询入参添加transfer_type，出参删除transfer_type，查询成功账单回参数添加tx_id | 王东     |
| 7    | 2018-01-24 | 账单查询返回确认数                                | 朱一凡    |
| 6    | 2018-01-24 | 分开成功账单与在途账单                              | 朱一凡    |
| 5    | 2018-01-24 | 复用账单查询接口查询 PENDING 状态的账单                 | 王东 朱一凡 |
| 4    | 2018-01-24 | xstar 新需求: 要在未确认时就回调一次 回调增加类型 TRANSFER_PENDING | 朱一凡    |
| 3    | 2018-01-24 | 转账需要指定转出地址, 转账账单查询分开传递转入转出地址             | 王东 朱一凡 |
| 2    | 2018-01-24 | 转账账单查询返回手续费                              | 朱一凡    |
| 1    | 2018-01-24 | sdk 3.0 文档初始版                            | 朱一凡    |




# 关键信息

## 参数类型
接口类型仅包括三种入参类型:
* 字符串型 **String**
* 数字型 **Number**
* 对象 **Object**
* 数组 **Array**

## 鉴权参数
所有接口(包括回调)需要附带以下三个鉴权限参数:

| 名字           | 类型     | 说明            |
| ------------ | ------ | ------------- |
| key          | String |               |
| sign         | String | 生成的签名            |
| request_time | Number | 请求时间戳, 防止重放攻击 |

后面接口中不再重复列出

签名生成规则步骤:

1. 取出 JSON 中所有 key-value 对中的 value, 放入数组 values
1. 将分配的 secret 放入数组 values
1. 对 values 进行字典序排序
1. 使用下划线`_` 连接 values 内的所有值, 得到字符串 content
1. 对 content 进行 md5，得到签名 `sign`
1. 将 `sign` 放入 JSON，完成。

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

| 名称      | 类型     | 说明   |
| ------- | ------ | ---- |
| symbol  | String | 币种   |
| user_id | String | 用户id |
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

| 名称        | 类型     | 说明                                       |
| --------- | ------ | ---------------------------------------- |
| symbol    | String | 币种                                       |
| coins     | String | 转账币数. 精度为小数点后8位                          |
| to        | String | 转入目标地址                                   |
| from      | String | 转出地址                                     |
| unique_id | String | 调用方生成对本操作的唯一id. 出现接口调用超时等情况, 第二次重复调用时需要与第一次id保持一致, 以此避免因超时重复调用导致的重复转账 |

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

针对门罗币的特殊说明:
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

| 名称            | 类型     | 说明                  |
| ------------- | ------ | ------------------- |
| transfer_type | String | 交易类型 IN(转入) OUT(转出) |
| symbol        | String | 币种                  |
| limit         | String | 一页几条                |
| p             | String | 第几页                 |
| start_at      | String | 开始时间(可选)            |
| end_at        | String | 结束时间(可选)            |
| address       | String | 转入地址(可选)            |

若要限定时间, start_at 与 end_at 必须同时传入, 否则当做未传入处理


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


## 在途账单查询

[htts://api.dabank.io/api/v3/transfers/pending](htts://api.dabank.io/api/v3/transfers/pending)

request POST:

| 名称            | 类型     | 说明                  |
| ------------- | ------ | ------------------- |
| transfer_type | String | 交易类型 IN(转入) OUT(转出) |
| symbol        | String | 币种                  |
| limit         | String | 一页几条                |
| p             | String | 第几页                 |
| start_at      | String | 开始时间(可选)            |
| end_at        | String | 结束时间(可选)            |
| address       | String | 转入地址(可选)            |

若要限定时间, start_at 与 end_at 必须同时传入, 否则当做未传入处理


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

## 账单总和

[htts://api.dabank.io/api/v3/transfers/sum](htts://api.dabank.io/api/v3/transfers/sum)

request POST:

| 名称            | 类型     | 说明                  |
| ------------- | ------ | ------------------- |
| symbol        | String | 币种                  |
| start_at      | String | 开始时间(可选)            |
| end_at        | String | 结束时间(可选)            |
| transfer_type | String | 转账类型 IN(转入) OUT(转出) |

response:

| 名称    | 类型     | 说明              |
| ----- | ------ | --------------- |
| coins | String | 转账币数. 精度为小数点后8位 |

## 查询 APP 账户信息

request POST:

| 名称     | 类型     | 说明               |
| ------ | ------ | ---------------- |
| symbol | String | 币种(可选, 不传时查所有币种) |

response:

data 里为 Array

| 名称      | 类型     | 说明            |
| ------- | ------ | ------------- |
| symbol  | String | 币种            |
| balance | String | 余额. 精度为小数点后8位 |

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
| fee_symbol           | String | 手续费扣费币种                                      |

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

接口需要按照正确格式返回成功, 否则会一直重试

| 名字     | 类型     | 说明                       |
| ------ | ------ | ------------------------ |
| result | String | 成功时为 "Success", 失败时为错误信息 |

```json
{  
   "result":"Success"
}
```

# 支持的币种以及小数位数

|          |            | 
|----------|------------| 
| symbol | decimals | 
| USO    | 8          | 
| KEY    | 8         | 
| CRE    | 8         | 
| RUFF   | 8         | 
| CDY    | 8          | 
| THM    | 8         | 
| RED    | 8         | 
| BCH    | 8          | 
| LMC    | 6          | 
| BNB    | 8         | 
| DCON   | 0          | 
| BTK    | 8         | 
| USDT   | 6          | 
| MGD    | 8          | 
| ZRX    | 8         | 
| LTC    | 8          | 
| XMC    | 0          | 
| XMR    | 8         | 
| IOST   | 8         | 
| CFUN   | 8          | 
| EOS    | 8         | 
| ALI    | 8         | 
| BTM    | 8          | 
| BTC    | 8          | 
| QTUM   | 8          | 
| ETH    | 8         | 
