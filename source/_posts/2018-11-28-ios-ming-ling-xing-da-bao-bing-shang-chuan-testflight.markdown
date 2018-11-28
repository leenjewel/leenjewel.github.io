---
layout: post
title: "iOS 命令行打包并上传 TestFlight"
date: 2018-11-28 13:40:51 +0800
comments: true
categories: iOS
---

网络上教大家如何用命令行的方式构建 iOS ipa 包并上传 TestFlight 的文章非常多了，最近也正好用到这些来结合 Jenkins 做持续集成。亲手实践了一把也找了很多相关的资料，发现大多成文较早，都或多或少的有些问题，所以这里也把自己最近一次实践做个总结。需要注意的点都标注上了。

## 环境

我用的基本上都是最新的 iOS 开发环境，简单罗列

- Mac OS X 10.14.1 Mojave

- Xcode 10.1(10B61)

- CocoaPods 1.6.0.beta.2

> **这里 CocoaPods 之所以用了 Beta 版本是因为目前为止稳定版和最新的 Xcode 10 兼容性上有些问题。当然了，在更新 CocoaPods Beta 版本的时候发现 gem 命令更新针对 Mac OS X 10.14 也有了改变，具体更新 CocoaPods 的命令是：**
> 
> ```
> sudo gem install -n /usr/local/bin cocoapods --pre
> ```

## 构建 Archive

先给出具体的命令

```
/usr/bin/xcodebuild -scheme {{YOUR SCHEME}} -configuration Release -derivedDataPath derivedData -archivePath {{YOUR ARCHIVE OUTPUT PATH}} -destination generic/platform=iOS clean archive
```

如果你使用了 CocoaPods 做你的 iOS 工程依赖管理，那么你的 iOS 工程中会存在一个 `xcworkspace` 因此构建的命令也会稍显不一样，要增加一个参数项指定你的 `xcworkspace` 位置

```
/usr/bin/xcodebuild -workspace {{YOUR XCWORKSPACE PATH}} -scheme {{YOUR SCHEME}} -configuration Release -derivedDataPath derivedData -archivePath {{YOUR ARCHIVE OUTPUT PATH}} -destination generic/platform=iOS clean archive
```

## 导出 IPA

上一步构建 Archive 并不难，我自己卡在导出 IPA 包这步卡的比较久，主要是一直没有搞正确 `exportOptionsPlist` 这个 plist 文件的写法。

还是先给出具体的命令


```
/usr/bin/xcodebuild -exportArchive -exportOptionsPlist {{YOUR PLIST FILE PATH}} -archivePath {{YOUR ARCHIVE OUTPUT PATH}} -exportPath {{YOUR IPA OUTPUT PATH}}
```

> **导出 ipa 包最关键的是要把 exportOptionsPlist 参数指定的 plist 文件写正确。这里有个敲门，那就是你可以先使用 Xcode 做一次 Archive 然后将构建成功的工程通过 Xcode 的 Organizer 做一次发布。当然，不要选择直接发布到 TestFlight 而是在选择 `Distribute App` 后要选择 `iOS App Store` 然后选择 `Export` 而不是 upload**

![1、打开 Xcode Orgnaizer]({{ site.url }}/assets/ios_command_build_ipa_upload_testflight/xcode_open_orgnaizer.png)

![2、选择 Distribute App]({{ site.url }}/assets/ios_command_build_ipa_upload_testflight/orgnaizer_distribute_app.png)

![3、选择 iOS App Store]({{ site.url }}/assets/ios_command_build_ipa_upload_testflight/orgnaizer_ios_appstore.png)

![4、选择 Export]({{ site.url }}/assets/ios_command_build_ipa_upload_testflight/orgnaizer_export_app.png)

![5、得到 ExportOptions.plist 文件]({{ site.url }}/assets/ios_command_build_ipa_upload_testflight/export_options_plist.png)

在你用 Xcode 导出的可以上传到 TestFlight 的包中你可以找到一个名为 `ExportOptions.plist` 的文件。使用这个文件作为命令行导出 ipa 是 `exportOptionsPlist` 参数的文件输入就一定不会错了。

顺便给出我导出的这个 plist 文件的内容示意

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>destination</key>
	<string>export</string>
	<key>method</key>
	<string>app-store</string>
	<key>signingStyle</key>
	<string>automatic</string>
	<key>stripSwiftSymbols</key>
	<true/>
	<key>teamID</key>
	<string>{{YOUR TEAM ID}}</string>
	<key>uploadBitcode</key>
	<true/>
	<key>uploadSymbols</key>
	<true/>
</dict>
</plist>
```

> `{{YOUR TEAM ID}}` 就是开发者账号里那个 10 位的 team id

## 关于证书

构建、导出、上传 TestFlight 都需要证书这个是基本概念。这里要顺带提上一句的是证书放置的位置。在 `钥匙串访问` 程序里证书可以放置的位置有三个：

- 登陆

- 本地项目

- 系统

> **请将出包上传 TestFlight 用的证书放置在【登陆】这个位置，千万不要放置在【系统】中，否则每次出包系统会弹出对话框让你输入用户登录认证的用户名和密码，这样是无法做到自动化出包的。**

## 上传 TestFlight

最后一步上传 TestFlight 就没什么难点了，直接给出命令

```
/Applications/Xcode.app/Contents/Applications/Application Loader.app/Contents/Frameworks/ITunesSoftwareService.framework/Versions/A/Support/altool --upload-app -t ios -f {{YOUR IPA FILE PATH}} -u {{YOUR ITUNESCONNECT USERNAME}} -p {{YOUR ITUNESCONNECT PASSWORD}}
```
