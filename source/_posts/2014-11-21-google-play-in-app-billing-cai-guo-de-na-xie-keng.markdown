---
layout: post
title: "Google Play In-app Billing 踩过的那些坑"
date: 2014-11-21 15:14:24 +0800
comments: true
categories: Android
---
最近在做的一款游戏针对海外发行，要上 Google Play，所以支付这块儿要接入 Google Play 。因为我们是免费 App + 应用内支付，所以 Google Play 这块儿只接入 In-app 类型的支付方式，接下来我准备吐槽了。

### 大环境

国内做 Google Play 相关的开发外围难度因素可想而知。具体原因相比大家都知道，所以那个什么墙什么的我就不多说了。这里已经无力吐槽了。

### 流程

这里简单说一下 Google Play In-app Billing 支付的流程。具体的建议看[官方文档](https://developer.android.com/google/play/billing/index.html?hl=d)最靠谱。Google Play 没有可重复购买商品这个概念，所有的“商品/充值档”用户成功购买过一次之后就不允许再次购买了。所以为了实现像应用内支付充值这种可重复购买的“商品/充值档”，Google Play 提供了一个[“消耗”借口(Consuming In-app Products)](https://developer.android.com/google/play/billing/api.html#consume)。用户购买完商品后，调一下“消耗”接口，这样用户下次就可以继续购买了。

### 写代码

还是就看[官方文档](https://developer.android.com/google/play/billing/index.html?hl=d)是最靠谱的，In-app Billing 的 API 有个 v2 版本和 v3 版本，v2 版本已经不支持了，直接整 v3 版本的吧。怎么开发，怎么写代码这块儿没什么好说的，看着文档写，基本都不会错。这里我只掉进坑里去一次，说说。

### ProductID 这个坑

在发起购买时需要调用 [getBuyIntent(int apiVersion, String packageName, String sku, String type, String developerPayload)](https://developer.android.com/google/play/billing/billing_reference.html#getBuyIntent)这个方法，sku 就是充值档ID（也就是 productId）。因为我们的游戏是夸 iOS 和 Android 平台的，在做 iOS 支付的时候，配置的充值档ID都是 Bundle Identifier + xxx 的格式，比如：

```
com.abc.def.product1
com.abc.def.product2
```

在 Google Play Developer Console 配置充值档时，为了统一，我们配置的和 iOS 一样的 productId，也同样是为了统一，我们 Android 项目配置的 package name 也和 iOS 配置的 Bundle Identifier 是一样的，所以我就掉坑里面了。

看到 [getBuyIntent(int apiVersion, String packageName, String sku, String type, String developerPayload)](https://developer.android.com/google/play/billing/billing_reference.html#getBuyIntent) 这个方法需求一个 sku 还需求一个 packageName ，我当时**错误**的认为 sku 和 packageName 分开传，所以**错误**的写成了

```java
// 假设要充值的充值档 ID 为 com.abc.def.product1 
// **注意** 这样写是错误的！！！
getBuyIntent(3, "com.abc.def", "product1", "inapp", "");
```

结果在调用 [getSkuDetails(int apiVersion, String packageName, String type, in Bundle skusBundle)](https://developer.android.com/google/play/billing/billing_reference.html#getSkuDetails) 方法时就一直返回空结果，告诉我找不到对应的商品。这里正确的做法就是**严格**按照你 Google Play Developer Console 里配置的 ProductId 来写，配置的是什么值，就传什么值。

```java
// 假设要充值的充值档 ID 为 com.abc.def.product1 
getBuyIntent(3, "com.abc.def", "com.abc.def.product1", "inapp", "");
```

### 测试

总的来说，Google Play In-app Billing 支付接入的开发算是比较简单的，步骤不多，也比较容易理解。最让人头疼的是测试，特别是你人在大陆，那就是难上加难了~坑略多。

### 上传测试 APK

首先你要在 Google Play Developer Console 里面为你要测试的 APP 新建一个应用，然后上传你要测试的 APP 的 APK 包。这里有两点注意：

> * 上传的 APK 包必须要有签名，而且不能用 Debug 签名。
> * 上传的 APK 包体积不能超过 50MB 超过的话要做分包（分包打算下回单开一篇来讲）。

Google Play Developer Console 一个应用可以对应发布三个频道，正式版、Beta版和Alpha版，我们测试用的 APK 只要上传到 Beta版或 Alpha版频道就好。

### 发布你的应用

看到“发布”这个词你可能会慌一下：“怎么，我的应用还没做完呢，怎么能发布呢？”。不要担心，这里你只上传了你的测试 APK 包到 Beta 或 Alpha 频道，把应用发布了，普通用户也是无法下载的。发布是必须做的，如果你只处于默认的“草稿”状态，是根本没办法测试支付功能的。

这里吐个槽，当你发布了你的应用后 Google Play 不会立即让它生效。仔细看你的 Google Play Developer Console 页面，你会发现 Google 提示你要等一等才会发布成功，等待的时间是按“小时”为单位的，没辙，耐心等待吧。

### 准备测试帐号

上面说了，发布应用后要等，到底要等到什么时候呢？在你的 Google Play Console 页面你对应发布的频道那里会有个“管理测试人员列表”的超链接，点开会弹出一个弹出框，在弹出框里有个标题是“与测试者分享以下链接”，下面有个 URL 链接，形如：

```
https://play.google.com/apps/testing/xxxxxxx
```

的链接，用浏览器点开这个链接你会发现如果它跳转到了 Google Play 应用商店并能看到你的测试应用了，说明你已经发布成功了。

什么？你一直看不到 Google Play 应用商店里你发布的应用？那说明你当前登录到 Google Play 应用商店的帐号既不是你这个应用的开发者帐号，也不是你这个应用的测试组帐号。

成为开发者需要在 Google Play Developer Console 里面设置，这个就不多讲了。主要提一下怎么成为测试帐号。

首先你要到 [Google Group](https://groups.google.com) 去建立一个新的论坛，然后回到 Google Play Console 页面你对应发布的频道，还是点击“管理测试人员列表”，在弹出的弹出框里将你刚刚建立好的 Google Group 群组的 Email 填写进去。这样只要你邀请进入这个 Google Group 的人员都是这个应用的测试人员了。

### 在真机上安装要测试的APP

要测试 Google Play In-app Billing 支付，一定要在真实的设备上测试，而且还要保证设备上装了 Google Play 国内一般装了制定 Android 系统的手机都不会默认安装 Google Play 需要你去网上搜一艘 Google Play 的安装包。我当时用的是 Google 的 Nexus 7 测试的，系统用的 Google 原生 4.4.4。

这里你可能还有个疑问：“我在测试我的 APP 时肯定会经常做一些修改，或要加断点 Debug，我总不能修改一次就发个 APK 包到 Google Play Developer Console 吧？”。这里你可以放心，你完全没必要用上传的 APK 来测试，你只要保证

> 安装到真机上的测试 APP 签名和上传到 Google Play 的 APK 包的签名一致

### 搞定 Google Play

好了，如果目前你手拿着安装好测试 APP 的真机设备，设备上安装有 Google Play ，Google Play 上登录了你的开发组或测试组人员帐号，你的应用已经成功发布了，而刚好你此时人不是在大陆，那么恭喜你，你已经可以开始测试你的支付了。

如果你上述工作都做好了，可是你人在大陆，那么你就“万事俱备，只欠东风”了。

先吐槽，在 Google Play Developer Console 的应用发布国家列表中是不允许选择“中国”的，Google Play 在大陆也是不允许支付的。如果你用你的设备打开 Google Play 应用商店看到的满眼都是免费应用，一个付费应用都没有，那么“恭喜”你，目前 Google Play 认为你是个大陆用户，你是不允许付费的，自然你就没办法测试你的支付流程了。

网上查一圈，发现不少人给出解决方案，都挺复杂的。什么又要先把设备给越狱拿到 root 权限啦，什么又要安装第三方破解 Google Play 的软件啦，还有什么需要插个国外的 SIM 卡了，据本人亲测根本没那么那么费劲，你只要有个 VPN 即可，你懂的。

根据我本人拿着手头的 Nexus 7 亲测，只要你在设备上连上 VPN（也有人说要 VPN 对应的国家要涵盖在你要测试应用对应发布的国家范围内，这点我没有亲测，我只知道当时我的 VPN 是加拿大，而加拿大也是在我测试的应用对应的发布国家内）。再打开 Google Play 应用商店，如果这时候你发现你能看到付费应用了，这说明你的“东风”也来了。

如果连上 VPN 后在 Google Play 应用商店还是看不到其他付费应用的话，先尝试去设置那里删除 Google Play 的缓存数据，如果还不行据说需要将你的设备恢复一下出厂设置再连上 VPN 就可以了。

打开你的测试 APP 点击支付，如果弹出 Google Play 的支付弹出框，说明流程都走通了。最后说一句，要想付费成功你的 Google Play 帐号必须绑定有海外支付能力的信用卡或者有海外支付能力的 Paypal 账户，这个只能你自己想办法了。

祝玩得愉快~
