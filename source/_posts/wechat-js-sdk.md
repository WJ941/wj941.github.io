---
title: wechat-js-sdk
date: 2020-12-25 09:12:33
tags:
---

# 微信JS-SDK接入
最近项目里遇到需要手机端H5使用扫码的需求，解决方法是使用微信的JSSDK调用微信扫一扫接入。微信调用扫一扫需要通过调用wx.config方法注入权限验证配置。下面说明如果获取JSSDK的权限。

### 申请微信公众号
在微信公众号的注册页面申请注册，由于是个人开发测试接口，选择订阅号就行。
{% asset_img wx-apply.png %}



注册完成之后在开发--开发者工具页面选择 公众平台测试账号
{% asset_img wx-test-account.png %}


里面有测试的appId和appSecret, 配置安全域名，本地开发时如何配置虚假域名后面说明。
{% asset_img js-sdk-security-host.png %}

### 配置虚假域名（windos）
修改c:/Windows/System32/drivers/etc/hosts文件,把127.0.0.1改成自己定义的域名。
{% asset_img modify-host-file.png %}


下载代理软件Charles, 配置代理端口，以8888为例。
{% asset_img proxy-setting.png %}


确保手机和电脑连接在统一局域网内，修改手机WiFi的代理配置，地址改成电脑的局域网ip地址，端口为代理软件配置的地址。
手机通过配置的域名访问前端开发页面。

### 后端生成signature

看下微信的[wx.config接口](https://developers.weixin.qq.com/doc/offiaccount/OA_Web_Apps/JS-SDK.html#4)
```javascript
wx.config({
  debug: true, // 开启调试模式,调用的所有api的返回值会在客户端alert出来，若要查看传入的参数，可以在pc端打开，参数信息会通过log打出，仅在pc端时才会打印。
  appId: '', // 必填，公众号的唯一标识
  timestamp: , // 必填，生成签名的时间戳
  nonceStr: '', // 必填，生成签名的随机串
  signature: '',// 必填，签名
  jsApiList: [] // 必填，需要使用的JS接口列表
});
```
wx.config接口需要[signature](https://developers.weixin.qq.com/doc/offiaccount/OA_Web_Apps/JS-SDK.html#62)这是需要通过jsapi_ticket经过hash之后生成的，所以需要后端提供一个生成signature的接口。


后端获取公众号的access_token，[微信官方文档](https://developers.weixin.qq.com/doc/offiaccount/Basic_Information/Get_access_token.html)。
1. 通过GET请求接口`https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=APPID&secret=APPSECRET`，拿到返回的access_token
```json
{"access_token":"ACCESS_TOKEN","expires_in":7200}
```
2. 通过GET请求：`https://api.weixin.qq.com/cgi-bin/ticket/getticket?access_token=ACCESS_TOKEN&type=jsapi`，拿到返回的ticket:
```json
{
  "errcode":0,
  "errmsg":"ok",
  "ticket":"bxLdikRXVbTPdHSM05e5u5sUoXNKd8-41ZO3MhKoyN5OfkWITDGgnr2fwJ0m9E8NYzWKVZvdVtaUgWvsdshFKA",
  "expires_in":7200
}
```
3. 对noncestr， timestamp, jsapi_ticket, url四个值做sha1哈希，微信需要按字段名排序后拼接成字符串，就是js代码写就是：
```javascript
var str = `jsapi_ticket=${ticket}&noncestr=${noncestr}&timestamp=${ts}&url=${url}`;
```
url需要是配置的JS接口安全域名下的url。


整个生成signature的nodejs代码如下：
```javascript
// app.js
const request = require('request');
const cors = require('cors')
const express = require('express')
const crypto = require('crypto');
const bodyParser = require('body-parser');
const app = express()
const port = 3000

app.use(cors());
app.use(bodyParser.json());

app.get('/', (req, res) => {
  res.send('Hello World!')
})



const appId = '申请的公众号的appId'
const appSecret = '申请的公众号的appSecret';


const createNonceStr = () => Math.random().toString(36).substr(2, 15);
const createTimeStamp = () => parseInt(new Date().getTime() / 1000) + '';
const calcSignature = function (ticket, noncestr, ts, url) {
    const str = `jsapi_ticket=${ticket}&noncestr=${noncestr}&timestamp=${ts}&url=${url}`;
    return crypto.createHash('sha1').update(str).digest('hex')
}

app.post('/getWX', function (req, res) {
    const url = req.body.url; // 初始化jsdk的页面url，如果是单页应用记得截掉url的#部分

    let promise = new Promise((resolve, reject) => {
            // 第一步，通过appId和appSecret 获取access_token
            request(`https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=${appId}&secret=${appSecret}`, function (error, response, body) {
                if (!error && response.statusCode == 200) { 
                    let access_token = JSON.parse(body).access_token;
                    console.log('第一步获取access_token：', access_token);
                    resolve(access_token);
                } else {
                    reject(error);
                }
                });
        });
    
    promise.then(access_token => {
        // 第二步，通过第一步的access_token获取票据ticket
        return new Promise((resolve, reject) => {
            request(`https://api.weixin.qq.com/cgi-bin/ticket/getticket?access_token=${access_token}&type=jsapi`, function (error, response, body) {
                if (!error && response.statusCode == 200) {
                    let ticket = JSON.parse(body).ticket;
                    console.log('第二步获取ticket：',ticket);
                    resolve(ticket);
                } else {
                    reject(error);
                }
            });
        });
        
    }).then(ticket => {

        const noncestr = createNonceStr(); // 随机字符串
        const timestamp = createTimeStamp(); // 时间戳
        const signature = calcSignature(ticket, noncestr, timestamp, url);  // 通过sha1算法得到签名
        const wxConfig = {
            appId: appId,
            noncestr: noncestr,
            timestamp: timestamp,
            signature: signature,
        }
        console.log({ ...wxConfig, url });
        res.send(wxConfig);
    }).catch(error =>{
        console.log(error);
    });
});

app.listen(port, () => {
    console.log(`Example app listening at http://localhost:${port}`)
})
```