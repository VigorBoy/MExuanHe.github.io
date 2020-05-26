---
title: JAVA（本地服务器配置微信支付）
date: 2018-3-26 00:32:33
tags:
---
# 前沿
我是一名iOS开发者 最近由于工作的原因 由我来处理服务器后台集成第三方支付的功能 语言是Java 顺水推舟 就学了下 Java语法 

# 配置
需要的工具 ：
- JDK：Java 开发包 [下载地址](https://download.csdn.net/download/tan3739/9476575)
- Eclipse ： 服务器开发工具  [下载地址](https://www.eclipse.org/downloads/)
- Tomcat ：服务器运行环境   [下载地址](http://tomcat.apache.org)

# 创建项目

打开Eclipse 首先要配置下我们的Tomcat     [教程](http://jingyan.baidu.com/article/3065b3b6efa9d7becff8a4c6.html
)

- 创建动态Web项目
![Snip20180410_2.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/16a34cbdc9a4f9ec.png)

配置 Web Project  我的Tomcat是8.0 就选择8.0 
![Snip20180410_5.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/16a34cbdce41c1a2.png)

文件目录
![Snip20180410_9.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/16a34cbdcdd7f2ef.png)

**需要说明下**  
 ```
在WebContent中的lib文件目录下 存放我们的jar 文件
在Java Resources 目录下 存放我们的代码文件
 ```

把这两个工具文件拖入到项目中 方便我们快速开发 工具文件我会放到Github上面 ![Snip20180410_10.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/16a34cbdce7404fa.png)

- 创建接口文件
![Snip20180410_11.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/16a34cbdcded075d.png)

 ```Package``` 是选择哪个文件目录下面
 ```Superclass``` 选择继承```HttpServlet```
![Snip20180410_13.png](https://github.com/MExuanHe/MExuanHe.github.io/raw/master/fancybox/images/16a34cbdce9704bf.png)

接下来就可以写我们的接口代码了

 创建几个参数 把相应的信息填写进去

	// 微信开发平台应用id
	public static String APP_ID = "";
	// 财付通商户号
	public static String PARTNER_ID = "";
	// 商户号对应的密钥
	public static String PARTNER_KEY = "";
	// 统一下单
	public static String URL_UNIFIEDORDER = "";
	// 接收财付通通知的URL
	public static String NOTIFY_URL = "";

在 ```doPost ``` 方法里面处理我们的订单签名

1.填写签名订单信息
 - 初始化``` PrepayIdRequestHandler``` 类  微信写好一个封装案例,你可以根据服务器需求，自己定义网络请求框架
```
PrepayIdRequestHandler handler =new PrepayIdRequestHandler(req, resp);	
```
   - 填写相应的参数
```
// 统一下单的接口(调用微信支付服务器需要的接口)--->公开的
handler.setGateUrl(URL_UNIFIEDORDER);
// 设置密钥
handler.setKey(PARTNER_KEY);
// 设置应用的ID
handler.setParameter("appid", APP_ID);
根据微信文档的要求 填写相应的信息  这里我就不往下写相应的参数信息了 微信文档写的很清楚
唯一想提的是  setParameter 方法就跟iOS里面设置字典一样 都是key和Value
```

2.调用微信统一下单接口(目的：获取prepay_id)

- 调用 ```sendPrepay``` 方法获取 ```prepay_id```  这里微信返回的数据是XML数据格式  ```sendPrepay``` 方法 返回的是一个Map 集合 在iOS里面就叫做 字典
```
Map paramsMap = handler.sendPrepay();
String prepay_id = (String) paramsMap.get("prepay_id");
```
3. 处理接口返回信息进行二次签名
 - 这里需要注意的是 二次签名和一次签名，参数不一样 注意官网文档
```
OrderResult orderResult=new OrderResult();
//清空集合  重新 赋值
handler.clear();
String noncestr = (String) paramsMap.get("noncestr");
String timestamp = WXUtil.getTimeStamp();
// 密钥
handler.setKey(PARTNER_KEY);
// 设置应用的ID
handler.setParameter("appid", APP_ID);
// 预付单ID
handler.setParameter("prepayid", prepay_id);
// 扩展字段
handler.setParameter("package", "Sign=WXPay");
// 商户号
handler.setParameter("partnerid", PARTNER_ID);
// 随机字符串
handler.setParameter("noncestr", noncestr);
// 时间戳
handler.setParameter("timestamp", timestamp);
// 进行二次签名(签名参数不一样)
// 第一次签名：对订单信息签名，获取prepay_id
// 第二次签名：对支付信息进行签名
sign = handler.createMD5Sign();

OrderBean orderBean = new OrderBean();

orderBean.setAppid(APP_ID);
orderBean.setNoncestr(noncestr);
orderBean.setPackageValue("Sign=WXPay");
orderBean.setPartnerid(PARTNER_ID);
orderBean.setPrepayid(prepay_id);
orderBean.setTradeType((String) paramsMap.get("trade_type"));
orderBean.setSign(sign);
orderBean.setTimestamp(timestamp);			
orderResult.setOrderBean(orderBean);
```
 
4. 将model转json数据格式 返回客户单
```
Gson gson=new Gson();
//解析成json字符串
String jsonstr=gson.toJson(orderResult);
resp.getWriter().print(jsonstr);
```

**以上java服务器端微信支付代码完成  客服端的代码我就不贴了**
[dome地址](https://github.com/MExuanHe/WxPay-java)
