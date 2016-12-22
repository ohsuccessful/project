# project
nodejs开发微信扫码支付第二种模式。
微信支付支付都需要什么动动手指头自己搜一下吧！这里不多说。本文只针对微信扫码支付第二种模式废话不多说上代码。
路由：
//微信支付页面路由
router.get('/wxpayhtml',order.wxpayhtml);
//生成二维码图片路由
router.get('/wxpay',order.wxpay);
//完成支付回调路由
router.post('/wxnotify',order.wxnotify);
//判断回调数据是否插入数据库成功路由
router.get('/wxpayfind',order.wxpayfind);
//跳转页面路由
router.get('/wxpay_return',order.wxReturn);
//页面：
<div ng-controller="wxpay">
    <h2>微信支付</h2>
    <img src='请求你的生成二维码路由传一个id:<%-page%>'>
</div>
//js：
app.controller('wxpay', function($scope, $http, $interval,$window) {
     var id = document.URL.split('?')[1].split('=')[1];
     var url = "/order/wxpayfind?id="+id;
     console.log(url)
     var time = $interval(function(){
         $http.get(url).success(function(r) {
             if(r==2){
                 console.log("错了");
                 $interval.cancel();
             }else{
                 if(r==1){
                     var returl = "/order/wxpay_return?id="+id
                     $window.location.href = returl;
                     console.log("完成结束循环");
                     $interval.cancel(time);
                 }else{
                     console.log("没找到");
                 }
             }
         })
     },3000);
})

控制器：
var request = require('request'),
qr = require('qr-image'),
crypto = require('crypto'),
xml2js = require('xml2js'),
parser = new xml2js.Parser(),
ejs = require('ejs'),
fs = require('fs');
 //微信支付页面请求时传进来订单ID然后发送到页面
exports.wxpayhtml = (req, res) => {
    res.render('wxpay',{
        page:req.query.o
    });
}
//微信生成二维码请求时传来订单ID
exports.wxpay = (req, res) => {
    var orderid = req.query.o;
    //根据订单ID查询订单信息
    Order.findById(orderid).populate(['userid', 'addrid']).exec((err, docs) => {
        var _subject = '';
        docs.items.forEach(e => {
           _subject += e.eng + e.part + '|数量：' + e.num + '|'
        })
        _subject = _subject.substr(0,32)+'等多件'
        _body = docs.userid.name
        var data = {
            out_trade_no: docs.id, //订单号
            total_fee: docs.ttl + docs.sship.money, //付款金额
            body: _subject, //描述
            notify_url: 'http://www.ldbauto.com/order/wxnotify/' //回调地址
        };
        console.log(data);
        var code_url = "";//最终付款链接就是这个地址生成二维码
        var url = "https://api.mch.weixin.qq.com/pay/unifiedorder";//请求地址
        var appid = "";//公众账号ID
        var mch_id = "";//商户号
        var notify_url = data.notify_url;//支付成功后post回调地址
        var out_trade_no = data.out_trade_no;//订单号
        //var total_fee = data.total_fee*100;//订单金额
        var total_fee = "1";//金额单位分
        var body = data.body;//商品或订单简单描述
        var nonce_str =  Math.random().toString(36).substr(2, 15);//随机字符串不长于32位
        var formData = "<xml>";
        formData += "<appid>"+appid+"</appid>";
        formData += "<body>"+body+"</body>";
        formData += "<mch_id>"+mch_id+"</mch_id>";
        formData += "<nonce_str>"+nonce_str+"</nonce_str>";
        formData += "<notify_url>"+notify_url+"</notify_url>";
        formData += "<out_trade_no>"+out_trade_no+"</out_trade_no>";
        formData += "<total_fee>"+total_fee+"</total_fee>";
        formData += "<trade_type>NATIVE</trade_type>";
        formData += "<sign>" + paysign(appid,body,mch_id,nonce_str,notify_url, out_trade_no, total_fee, 'NATIVE') + "</sign>";
        formData += "</xml>";
        request(
            {
                url : url,
                method : 'POST',
                body : formData
            }, function (err, response, body)
            {
                if (!err && response.statusCode == 200)
                {
                //生成二维码
                console.log(body);
                    parser.parseString(body, function (err, result) {
                        var result_code = result.xml.result_code[0];
                        if(result_code == "SUCCESS") {
                            code_url = result.xml.code_url[0];
                             var img = qr.image(code_url,{size :10});
                            res.writeHead(200, {'Content-Type': 'image/png'});
                            img.pipe(res);
                        } else if(result_code == "FAIL"){
                            console.log("支付失败");
                            res.json();
                        }
                    });
                }
                }
                );
    })
  }
  //签名
function paysign(appid,body,mch_id,nonce_str,notify_url,out_trade_no,total_fee,trade_type) {
    var ret = {
        appid: appid,
        body: body,
        mch_id: mch_id,
        nonce_str: nonce_str,
        notify_url:notify_url,
        out_trade_no:out_trade_no,
        total_fee:total_fee,
        trade_type:trade_type
    };
    var string = raw(ret);
    var key = "";//微信支付密匙自己设定的
    stringA = string + '&key='+key;
    return crypto.createHash('md5').update(stringA,'utf8').digest('hex');
};
//拼接字符串
function raw(args) {
    var keys = Object.keys(args);
    keys = keys.sort()
    var newArgs = {};
    keys.forEach(function (key) {
    newArgs[key.toLowerCase()] = args[key];
    });
    var string = '';
    for (var k in newArgs) {
        string += '&' + k + '=' + newArgs[k];
    }
    string = string.substr(1);
    return string;
};
//微信支付完成回调
//可能你回收不到微信post过来的xml数据后边说
exports.wxnotify = function(req, res) {
    res.status(200).end();
    var body = req.body;
    console.log(body)
    if(body.xml.result_code == "SUCCESS"){
        if(body.xml.return_code == "SUCCESS"){
            console.log("支付成功")
            let _paytype = {
               ptype: '微信',
                info: {
                    email: body.xml.transaction_id,
                    id: body.xml.openid,
                 }
            }
            let orderid = body.xml.out_trade_no;
            console.log(_paytype,orderid);
            var p =   Order.findOneAndUpdate({
                _id:orderid,
                status: 0
            }, {
                status: 1,
                paydate: new Date(),
                paytype: _paytype,
                ttl_fee: body.xml.total_fee
            }, {
                upsert: true
            }).populate('addrid').exec((err, docs) => {
            })
        }
    }else{
        console.log("支付失败")
    }
    var data = "{xml:{return_code:'SUCCESS',return_msg:'OK'}";
    return data;
    //返回试了很多种方法还是给我一直请求MD不管了
}
 //微信支付成功跳转页面
 //支付页面从一开始就js一直隔几秒定时请求数据库支付状态改变停止请求跳转页面
exports.wxpayfind = function(req,res){
    var id = req.query.id;
    Order.find({_id:id,status:1}).populate('addrid').exec((err,docs)=>{
       if(err){
            res.josn(2);
            res.render('404')
        }else {
            if(docs.length == 1){
                res.json(1);
                return
            }else{
                res.json(0)
                return
            }
        }
    })
}
//跳转页面
exports.wxReturn = (req,res)=>{
   var id = req.query.id;
    console.log(id);
    Order.findOne({_id:id}).populate('addrid').exec((err,docs)=>{
        if(err){
            res.render('404')
        }else{
            if(docs){
                console.log(docs)
                res.render('alipay-success', {
                    page: docs,
                    part:req.cookies.part
                });
            }else{
                res.redirect('/')
            }
        }
    })
}
xml数据接收不到你需要在app.js加载一个模块
 bodyParser = require('body-parser');
 require('body-parser-xml')(bodyParser);
  app.use(bodyParser.xml({
     Limit: '2MB',
     xmlParseOptions: {
         normalize: true,
     normalizeTags: true,
     explicitArray: false
     }
 }));
