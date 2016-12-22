# project
nodejs开发微信扫码支付第二种模式。
微信支付支付都需要什么动动手指头自己搜一下吧！这里不多说。本文只针对微信扫码支付第二种模式废话不多说上代码。
路由：
//微信支付页面路由
router.get('/wxpayhtml',order.wxpayhtml);
//生成二维码路由
router.get('/wxpay',order.wxpay);
//完成支付回调路由
router.post('/wxnotify',order.wxnotify);
//判断回调数据是否插入数据库成功页面跳转路由
router.get('/wxpayfind',order.wxpayfind);
//跳转页面路由
router.get('/wxpay_return',order.wxReturn);

控制器：
request = require('request'),
qr = require('qr-image'),
crypto = require('crypto'),
xml2js = require('xml2js'),
parser = new xml2js.Parser(),
ejs = require('ejs'),
fs = require('fs');

