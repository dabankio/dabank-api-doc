![logo](https://cdn.golangme.com/static/img/carousel/golangroutines.png)


# 版本
| 序号   | 日期         | 修订内容                                     | 修订人    |
| ---- | ---------- | ---------------------------------------- | ------ |
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


# 鉴权参数
所有接口(包括回调)需要附带以下三个鉴权限参数:

| 名字           | 类型     | 说明            |
| ------------ | ------ | ------------- |
| key          | String |               |
| sign         | String | 签名            |
| request_time | Number | 请求时间戳, 防止重放攻击 |
后面接口中不再重复列出

# 返回错误码

返回均包在以下结构的 data 中

| 名称         | 类型              | 说明                       |
| ---------- | --------------- | ------------------------ |
| err_code   | String          | 错误编码, 用于 i18n. (正确时为空字符) |
| err_info   | String          | 错误说明. (正确时为空字符)          |
| request_id | Number          | 请求id, 用于定位该请求            |
| data       | Object or Array | 返回值                      |

```json
{ 
    "err_code": "ERR_ACCT_NOT_EXIST",
    "err_info": "account not exist",
    "data": {}
}
```

后面接口只列出 data 中内容

## 申请地址

`/api/v3/address`

request POST:

| 名称      | 类型     | 说明   |
| ------- | ------ | ---- |
| symbol  | String | 币种   |
| user_id | String | 用户id |

response:

| 名称      | 类型     | 说明   |
| ------- | ------ | ---- |
| address | String | 地址   |

## 转账

`/api/v3/transfer`

request POST:

| 名称        | 类型     | 说明                                       |
| --------- | ------ | ---------------------------------------- |
| symbol    | String | 币种                                       |
| coins     | String | 转账币数. 精度为小数点后8位                          |
| to        | String | 转入目标地址                                   |
| from      | String | 转出地址                                     |
| unique_id | String | 调用方生成对本操作的唯一id. 出现接口调用超时等情况, 第二次重复调用时需要与第一次id保持一致, 以此避免因超时重复调用导致的重复转账 |


response:

| 名称          | 类型     | 说明                                       |
| ----------- | ------ | ---------------------------------------- |
| transfer_id | String | 交易id. 请存储, 用于交易确认后回调关联                   |
| status      | String | TRANSFER_SUCCESSFUL(成功) 或 TRANSFER_PENDING(在途) |

针对门罗币的特殊说明:
- 如果客户提币地址要求 address+paymenid 的形式, `to` 参数设置成 `address$payment_id` 即可， 即用`$`连接 `to` 和 `paymentid`, 例如:`9wHNWjJ1Gnw7Yw1RjDLNiJ8yJw91xFCz4NvXCcQjzceiGqFXDcXuCqPckgi7pn1LA4FZ5EaAUd19meb8GXxCp3iFT2yZViw$662f879677b96d0919db4ae069d985e5e3dcf10c8429ed395a9a6e798bb4f9d1`
- 如果不设置`paymentid`, 则不带$符号, 直接传入集成地址

## 成功账单查询

`/api/v3/transfers/success`

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

`/api/v3/transfers/pending`

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

`/api/v3/transfers/sum`

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

# SDK回调接口

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

response:

| 名字     | 类型     | 说明                       |
| ------ | ------ | ------------------------ |
| result | String | 成功时为 "Success", 失败时为错误信息 |

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
