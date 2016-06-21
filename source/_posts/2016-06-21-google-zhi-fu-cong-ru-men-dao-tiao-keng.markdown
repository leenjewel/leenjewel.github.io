---
layout: post
title: "Google 支付从入门到跳坑"
date: 2016-06-21 14:45:19 +0800
comments: true
categories: Android
---

Google 支付真的不难，难的是由于“你懂的”的国内网络环境导致的复杂的测试流程和 Google Console “天花烂缀”的各种配置项目还有那些“文不达意”的报错信息。

很早写过一篇[《Google Play In-app Billing 踩过的那些坑》](http://leenjewel.github.io/blog/2014/11/21/google-play-in-app-billing-cai-guo-de-na-xie-keng/)，通过网站数据统计发现这篇文章的搜索和阅读量是最大的，后来又通过这篇文章的评论还有其他渠道和不少朋友们交流关于接入 Google 支付时遇到的问题发现大家对于这块儿还是有很多“心虚”的地方，还有一些“坑”我之前也没有说明白，所以我觉得有必要再写一篇来说说 Google 支付。

##先吃两颗定心丸

有两件事一定要知道：

- **Google 支付很简单，一点儿都不难，所以不要头疼，不好害怕，不要压力山大。**

- **当你写好代码完成接入准备测试 Google 支付时，只要顺利弹出了 Google 支付相关的 UI 界面，哪怕是报错提示信息，千万不要怀疑你的代码，只要弹出 UI 界面，你的代码就是对的，问题出在配置上。**

具体如何接入 Google 支付不是这篇文章的主旨，所以就不做详细的介绍了。因为 Google 支付接入真的非常简单。你可以仔细阅读[《官方文档》](https://developer.android.com/google/play/billing/index.html?hl=i)或是我之前写的那篇[《Google Play In-app Billing 踩过的那些坑》](http://leenjewel.github.io/blog/2014/11/21/google-play-in-app-billing-cai-guo-de-na-xie-keng/)再或者干脆阅读 Android SDK 附带的 Google 支付的 Sample 示例工程的源代码都可以帮你快速轻松的完成 Google 支付的编码接入过程。

Samples 示例工程的位置在：

> 你的 Android SDK 目录/extras/google/play_billing/samples

<br/><center>
<img src="http://7rfk33.com1.z0.glb.clouddn.com/google_play_billing_samples.png" alt="Google Play Billing 示例程序位置" />
</center><br/>

如果没有，请记得先打开 `Android SDK Manager` 去下载。

<br/><center>
<img src="http://7rfk33.com1.z0.glb.clouddn.com/google_play_billing_sdkmanager.png" alt="Android SDK Manager" />
</center><br/>

##配置并发布应用内商品

这里也是两点主意：

- **配置完应用内商品一定要发布，使之生效**

- **如果测试的时候需要“翻墙”，诸如使用 VPN 时，一定要保证你的网络环境所对应的国家在发布范围内**

第一条不用多说什么，在 Google Play Developer Console【应用内商品】中配置好商品，完成后的配置看起来是这个样子的就对了。

<br/><center>
<img src="http://7rfk33.com1.z0.glb.clouddn.com/google_play_billing_inapp.png" alt="应用内商品发布" />
</center><br/>

第二条一定要主意。一般在国内的开发者在测试 Google 支付的时候肯定是要“翻墙”的，这里就需要记住，举个例子，如果你使用的是美国的 VPN 进行测试，那么美国一定要在分发的国家或地区范围内，否则是无法进行测试的。

<br/><center>
<img src="http://7rfk33.com1.z0.glb.clouddn.com/google_play_billing_country.png" alt="分发的国家或地区" />
</center><br/>

##上传 APK 并发布应用

这里需要注意的点比较多：

- **APK 包发布到 Beta 或者 Alpha 渠道即可，没必要发布到正式渠道。**

- **如果你的应用状态变为【已发布】说明发布成功了。**

- **一些隐私问题或政策问题会导致你的应用无法通过审核，使用第三方 SDK 或者权限时要多加小心。**

- **你安装到设备上用来测试的 APK 包可以和你上传到 Google Play Developer Console 上的 APK 包不同，但要保证这两个 APK 包使用了相同的签名，这两个 APK 包的 versionCode 要一致。**

- **你测试时使用的网络环境所属的国家和地区一定要在你应用发布的国家或地区范围内。**

首先不要担心你的应用还没有开发完毕，大胆的发布你的应用。发布应用是测试 Google 支付的前提条件，所以请放心大胆的点击 Google Play Developer Console 页面右上角的【发布】按钮来发布你的应用吧。

要想发布你的应用你的上传你的 APK 包，这毋庸置疑。还是那句话，不要担心你的 APK 没有开发完或者还是个半成品，因为你在设备上安装用来测试的 APK 可以和你上传到 Google Play 的 APK 不一样。也就是说你可以先上传一个 APK 包，然后继续你的开发工作，任何开发或修改都不必重新上传你的 APK 包，直接将新生成的 APK 包安装到设备上测试即可。而你要做的就是保证安装到设备上的 APK 包的签名和上传的包的签名一致，`AndroidManifest.xml` 文件中 `versionCode` 也是一致的即可。

之前有个朋友公司的 APK 就遇到了因为接入了 [TalkingData](https://www.talkingdata.com/) 的 SDK 导致违反了 Google 的隐私政策而无法通过审核的问题。所以这里也需要注意，关于 Google 对发布到 Google Play 的应用的政策要求请自行查阅相关文档。

最后一点其实在上面那个章节已经说过了，这里还要重申一下，一定要保证测试时使用的网络环境所在的国家和地区在你应用发布的国家或地区的范围内。这一点非常重要。

## 测试 Google 支付

到这里你距离成功就很近了：

- **你的测试设备上一定要安装了 Google Play Service**

- **封闭测试时，除了要将测试人员的 Google Play 帐号加入封闭测试人员列表，还要让拥有这些测试帐号的人员通过访问生成的特殊链接来确认加入测试列表。**

- **测试支付是不会真的扣除你的任何费用的，但是即便如此你的测试 Google Play 帐号上还是需要绑定一张有国际支付能力的信用卡或银行卡的。**

这里重点说说封闭测试，相对于开放性测试封闭测试的流程稍微复杂一些。首先如图所示：

<br/><center>
<img src="http://7rfk33.com1.z0.glb.clouddn.com/google_play_billing_test.png" alt="封闭测试设置" />
</center><br/>

首先要将想要参与测试的 Google Play 帐号加入到测试人员列表中。这样只有加入到列表中的 Google Play 帐号才能够测试 Google 支付。

这里最需要注意也是最容易被忽略的是在将测试人员的 Google Play 帐号加入到测试人员列表后，一定要记得将下面那个生成好的链接发给参与测试的人员，让他们用浏览器打开这个链接，只有这样测试人员才真的加入了测试列表，才可以真的进行 Google 支付测试。否则在进行支付测试时你将得到`无法购买您要买的商品`错误提示。

## 最后

只要避开上面这些“坑”，你会发现其实 Google 支付真的很简单。如果你不幸掉到其他的“坑”里面了，欢迎分享给我，我们一起填坑。