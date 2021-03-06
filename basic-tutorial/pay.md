# 微信支付

本文对应实现例子为[mp-book-demo-basic](https://github.com/TencentCloudBase/mp-book-demo-basic)中的 `微信支付` 功能，可以通过该例子体验下单、支付、退款等支付类的功能。

<p align="center">
    <img src="../assets/cloudbase-qr.png" width="500px">
    <p align="center">扫码体验</p>
</p>

## 准备工作

微信支付，最主要的包括订单创建、发起支付、订单查询、申请退款、退款查询等等，本章会围绕这些最基本的功能去讲述如何基于云开发实现微信支付。

开发前，建议先阅读以下相关的文档：

1. [微信支付小程序文档](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=7_10&index=1)，该文档，尤其建议你先阅读 “开发前必读”、“业务说明”（如果小程序还没绑定微信支付） 和 “业务流程”。
2. [微信小程序支持接口](https://developers.weixin.qq.com/miniprogram/dev/api/open-api/payment/wx.requestPayment.html)

## 体验 DEMO

本章的案例代码，是在 [miniprogram-demo](https://github.com/TencentCloudBase/mp-book-demo-basic)。

将代码下载下来之后，遵循以下步骤可以在微信开发者工具里体验：

1. 在云开发中创建好至少一个环境

2. 在 `app.js` 中，如果使用默认环境，则没有改动。如果需要使用非默认环境，则需要在 `wx.cloud.init` 中传入环境id，如下：

<p align="center">
    <img src="../assets/user1.png" width="800px">
    <p align="center">云开发获取环境ID</p>
</p>

```js
wx.cloud.init({
  env: 'xxxx', // 环境 id
  traceUser: true
});
```

如果使用非默认环境，在云函数的 `cloud.init` 处，也需要补充环境id。

3. 在云函数根目录 `cloud/functions` 中找到函数 `pay`，在 `config` 目录中新建 `index.js`，配置环境ID、微信支付商户号、微信支付商户密钥、和双向证书中的 `apiclient_cert.p12`，该证书放也放在 `config` 目录中，用 `npm` 安装两个云函数的依赖，并上传该函数。

<p align="center">
    <img src="../assets/pay1.png" width="800px">
    <p align="center">微信支付获取要超级管理员才能获取证书</p>
</p>

4. 找到云函数 `pay-message`，在 `config` 目录中新建 `index.js`，配置小程序 `AppSecret` 和模板 id（选择合适的模板，合适的字段即可），用 `npm` 安装两个云函数的依赖，然后上传该函数。

<p align="center">
    <img src="../assets/user4.png" width="800px">
    <p align="center">小程序获取密钥</p>
</p>

<p align="center">
    <img src="../assets/pay2.png" width="800px">
    <p align="center">取得模板 id</p>
</p>

5. 在云开发数据库中，新建两个 `colleciton`，第一个名为 `goods`，权限设置为 `所有用户可读，仅创建者及管理员可写`，而另一个名为 `orders`，权限保持不变。

6. 在云开发存储中，上传 `cloud/storage` 中的图片到 `goods` 目录下，然后复制文件 `fileID`，并填入 `cloud/database/goods.json` 中。

<p align="center">
    <img src="../assets/pay3.png" width="800px">
    <p align="center">上传商品照片</p>
</p>

<p align="center">
    <img src="../assets/pay4.png" width="800px">
    <p align="center">将图片 fileId 填入 `goods.json` 中</p>
</p>

7. 在 `collections` `goods` 中，导入 `cloud/database/goods.json`。


## 订单创建

在真正发起支付之前，通常都需要创建好一个订单，这个订单需要在微信支付的服务中创建，也最好在自己小程序的服务中存一份，方便后续的发起支付。毕竟用户很多时候并不是马上支付的，可能会想放在购物车里，等双11的时候让老公帮忙清空呢。

在 DEMO 里，我们有三个页面，分别是 `pay-list`（商品列表）, `pay-order` （订单列表） 和 `pay-result` （订单详情）。我们在商品列表里，直接点击"下单"，就会帮我们调用云函数 `pay` 进行订单的创建。 云函数 `pay` 是一个复合的云函数，通过 `switch` 对支付的不同操作类型进行分支处理，创建订单的类型是 `unifiedorder`。

如下面我们截选了部份云函数 `pay` 的代码。我们借助 [`wx-js-utils`](https://github.com/lcxfs1991/wx-js-utils) 封装好微信支付的能力，初始化的时候，我们要填入小程序  `appId`, 微信支付商店号, 微信支付密钥等参数。值得注意的是，由于某些敏感操作如申请退款需要使用[证书](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=4_3)，因此我们也需要带上证书并进行读取，但对于云函数来说，由于服务器里有内置 CA 证书，因此不需要填写 `caFileContent` 参数。 

我们根据微信支付[统一下单](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=9_1)的文档，先拼凑好请求参数。需要注意的参数是 `notify_url`，由于目前云函数还没支持 `http` 触发器和公开的外网地址（即将支持），因此这里可以先随便填一个，可以是自有业务的地址。后面支付成功后，建议由小程序端主动去触发订单查询并更新状态。

```js

const {
  WXPay,
  WXPayConstants,
  WXPayUtil
} = require('wx-js-utils');

// 此处省略其它代码


const pay = new WXPay({
    appId: APPID, // 小程序 appID
    mchId: MCHID, // 微信支付商户号
    key: KEY, // 微信支付密钥
    certFileContent: CERT_FILE_CONTENT, // 微信支付证书
    timeout: TIMEOUT, //  超时时间
    signType: WXPayConstants.SIGN_TYPE_MD5, // 加密方式
    useSandbox: false // 不使用沙箱环境
  });

  // 此处省略其它代码

// 拼凑订单参数
const curTime = Date.now();
const tradeNo = `${goodId}-${curTime}`; // 自这义的trade number
const body = good.name; // 商品名作为 body 内容
const spbill_create_ip = ip.address() || '127.0.0.1'; //  获取服务器的 ip 地址
const notify_url = 'http://www.qq.com'; //'云函数暂时没有外网地址和HTTP触发起，暂时随便填个地址。建议填自己站点的域名'
const total_fee = good.price; // 商品的价格
const time_stamp = '' + Math.ceil(Date.now() / 1000); // 订单创建的时间
const out_trade_no = `${tradeNo}`;
const sign_type = WXPayConstants.SIGN_TYPE_MD5; // 加密方式

let orderParam = {
    body,
    spbill_create_ip,
    notify_url,
    out_trade_no,
    total_fee,
    openid: OPENID,
    trade_type: 'JSAPI',
    timeStamp: time_stamp,
};

// 在微信支付服务端生成该订单
const {
    return_code,
    ...restData 
} = await pay.unifiedOrder(orderParam);
```


## 发起支付与订单查询

发起支付的过程，除了后台接口，还涉及到小程序的 `wx.requestPayment` 接口。下面代码截选自 `pay-result` 订单详情页面，进入页面后，会先通过 `getOrder` 方法查询订单并获取订单所有数据，然后用户点击支付后，会根据订单的数据拼凑参数，然后调用 `wx.requestPayment` 发起支付，再去请求云函数 `pay`，在 `payorder` 处理分支中，进行订单信息的更新。

```js
// 获取订单详情
async getOrder() {
const {result} = await wx.cloud.callFunction({
    name: 'pay',
    data: {
    type: 'orderquery',
    data: {
        out_trade_no: this.data.out_trade_no
    }
    }
})

const data = result.data || {}

this.setData({
    order: data
}, () => {
    // 如果状态是退款中，则每次进来都会查询一下退款情况
    if (data.status === 3) {
    this.queryRefund()
    }
})
},

// 发起支付
pay() {
const orderQuery = this.data.order
const out_trade_no = this.data.out_trade_no

const {
    time_stamp,
    nonce_str,
    sign,
    prepay_id,
    body,
    total_fee
} = orderQuery

wx.requestPayment({
    timeStamp: time_stamp,
    nonceStr: nonce_str,
    package: `prepay_id=${prepay_id}`,
    signType: 'MD5',
    paySign: sign,
    success: async () => {
    wx.showLoading({
        title: '正在支付',
    })

    wx.showToast({
        title: '支付成功',
        icon: 'success',
        duration: 1500,
        success: async () => {
        this.getOrder()

        await wx.cloud.callFunction({
            name: 'pay',
            data: {
            type: 'payorder',
            data: {
                body,
                prepay_id,
                out_trade_no,
                total_fee
            }
            }
        })
        wx.hideLoading()
        }
    })
    },
    fail() {}
})
},
```

这里截取了云函数 `pay` 的 `payorder` 处理分支，主要就是通过 `pay.orderQuery` 查询微信服务端，如果支付成功，就会更新订单数据，并取发送一条模板消息告诉用户支付成功（模板消息详细教程请参考[模板消息和统一服务消息](basic-tutorial/message.md)教程）。

```js
// 进行微信支付及更新订单状态
case 'payorder': {
    const {
    out_trade_no,
    prepay_id,
    body,
    total_fee
    } = data

    const {return_code, ...restData} = await pay.orderQuery({
    out_trade_no
    })

    if (restData.trade_state === 'SUCCESS') {
    await orderCollection
        .where({out_trade_no})
        .update({
        data: {
            status: 1,
            trade_state: restData.trade_state,
            trade_state_desc: restData.trade_state_desc
        }
        })

    // console.log('======restData======');
    // console.log(restData);

    const curDate = new Date()
    const time = `${curDate.getFullYear()}-${curDate.getMonth() + 1}-${curDate.getDate()} ${curDate.getHours()}:${curDate.getMinutes()}:${curDate.getSeconds()}`
    try {
        const messageResult = await cloud.callFunction({
        name: 'pay-message',
        data: {
            formId: prepay_id,
            openId: OPENID,
            appId: APPID,
            page: `/pages/pay-result/index?id=${out_trade_no}`,
            data: {
            keyword1: {
                value: out_trade_no // 订单号
            },
            keyword2: {
                value: body // 物品名称
            },
            keyword3: {
                value: time// 支付时间
            },
            keyword4: {
                value: (total_fee / 100) + '元' // 支付金额
            }
            }
        }
        })
        console.log('=======message=========')
        console.log(messageResult)
    } catch (e) {
        console.log('===========')
        console.log(e)
    }
    }

    return {
    code: return_code === 'SUCCESS' ? 0 : 1,
    data: restData
    }
}
```

## 申请退款

申请退款是针对已经支付过的订单，而且这个属于敏感操作，需要使用双向证书，在本文开头已经有所提及。如下面代码片段，也是通过云函数 `pay` 的 `refund` 操作分支申请退款，而且退款不是马上生效的，可能会有一定的延迟。

```js
// 申请退款，但不会马上退
async refund() {
wx.showLoading({
    title: '正在申请退款',
})

const result = await wx.cloud.callFunction({
    name: 'pay',
    data: {
    type: 'refund',
    data: {
        out_trade_no: this.data.out_trade_no
    }
    }
})

wx.hideLoading()

if (!result.code) {
    const order = this.data.order
    order.trade_state_desc = '正在退款'
    order.status = 3
    order.trade_state = 'REFUNDING'

    this.setData({
    order
    })
} else {
    this.showToast({
    title: result.message,
    icon: 'none'
    })
}
},
```

云函数中，主要需要的参数是 `out_trade_no` 和  `total_fee`，然后分别作为退款的订单号和退款金额传给 `pay.refund` 方法进行退款申请即可。

```js
// 申请退款
case 'refund': {
    const {out_trade_no} = data
    const orders = await orderCollection.where({out_trade_no}).get()

    console.log(orders)

    if (!orders.data.length) {
    return {
        code: 1,
        message: '找不到订单'
    }
    }

    const order = orders.data[0]
    const {
    total_fee,
    } = order

    const result = await pay.refund({
    out_trade_no,
    total_fee,
    out_refund_no: out_trade_no,
    refund_fee: total_fee
    })

    const {return_code} = result

    if (return_code === 'SUCCESS') {
    try {
        await orderCollection.where({out_trade_no}).update({
        data: {
            trade_state: 'REFUNDING',
            trade_state_desc: '正在退款',
            status: 3
        }
        })
    } catch (e) {
        return {
        code: 1,
        mesasge: e.message
        }
    }
    } else {
    return {
        code: 1,
        mesasge: '退款失败，请重试'
    }
    }

    return {
    code: 0,
    data: {}
    }
}
```

## 退款查询

申请退款也是有 `notify_url`  回调参数的，不过是可选项。但由于一样的原因（不支持http触发和公网地址），因此我们也只好让用户在小程序端主动触发更新订单信息。

```js
// 查询退款情况
async queryRefund() {
const {result} = await wx.cloud.callFunction({
    name: 'pay',
    data: {
    type: 'queryrefund',
    data: {
        out_trade_no: this.data.out_trade_no
    }
    }
})

// 退款成功，则更新本地数据状态
if (!result.code && result.data) {
    const data = result.data
    const order = this.data.order
    order.trade_state_desc = data.trade_state_desc
    order.status = data.status
    order.trade_state = data.trade_state

    this.setData({
    order
    })
}
},
```

云函数 `pay` 中查询并更新退款订单。

```js
// 查询退款情况
case 'queryrefund': {
    const {out_trade_no} = data

    const result = await pay.refundQuery({
    out_trade_no
    })

    const {refund_status_0, return_code} = result

    if (return_code === 'SUCCESS' && refund_status_0 === 'SUCCESS') {
    try {
        await orderCollection.where({out_trade_no}).update({
        data: {
            trade_state: 'REFUNDED',
            trade_state_desc: '已退款',
            status: 4
        }
        })

        return {
        code: 0,
        data: {
            trade_state: 'REFUNDED',
            trade_state_desc: '已退款',
            status: 4
        }
        }
    } catch (e) {
        return {
        code: 0
        }
    }
    } else {
    return {
        code: 0
    }
    }

    // eslint-disable-next-line no-unreachable
    return {
    code: 0
    }
}
```