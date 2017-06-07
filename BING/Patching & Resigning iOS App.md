
Patching and Re-Signing iOS Apps
==

[原文地址](http://www.vantagepoint.sg/blog/85-patching-and-re-signing-ios-apps)

在未越狱的设备上运行修改后的iOS二进制文件有时会是一个值得探讨的议题，尤其是当你的越狱手机变砖后被迫升级到一个未越狱版本（我知道这不会发生在我或者任何人身上）。举个例子，你可以使用这种技术来动态分析app。亦或者你需要做一些GPS欺骗绕过锁区问题来捕捉在非洲的Pokemon，但是又担心被越狱检测抓到。无论怎样，你都可以使用接下来的流程来重签名一个被修改后的app，并能运行在你自己的未越狱设备上。记住，这种技术仅仅对非app store发布的二进制文件有效(**app store上发布的文件都是FairPlay加密的**)。

感谢Apple混乱的配置及代码签名系统，重签名一个app比想象中更具有挑战性。除非你获取到完全正确的配置文件和代码签名头，否则iOS会拒绝运行。这就需要你学习一大堆概念：**证书，BundleID，应用ID，Team标识**，以及他们是如何被Apple的构建工具捆绑在一起的。总之，让系统运行一个不是通过默认方式（Xcode）来构建的应用程序是一个艰巨的过程。

我们要使用的工具集包括：[optool](https://github.com/alexzielenski/optool)，苹果的构建工具和一些shell命令。这个方法的灵感来源于[Vincent Tan's Swizzle project](https://github.com/vtky/Swizzler2/wiki)的resign script。另一种使用不同工具重打包的方式可参考[NCC group描述的](https://www.nccgroup.trust/au/about-us/newsroom-and-events/blogs/2016/october/ios-instrumentation-without-jailbreak/)。

为完成以下一系列的步骤，先从[OWASP Mobile Testing Guide 仓库](https://github.com/OWASP/owasp-mstg/blob/master/Crackmes/iOS/Level_01/)下载UnCrackable iOS App Level 1。我们的目标是使UnCrackable应用在启动过程中加载FridaGadget.dylib，使我们能使用Frida来分析它。

### 获取开发者配置文件以及证书

由Apple签名的配置文件**provisioning profile**是一个plist文件，它能将你在一台或多台设备上的代码签名证书**code signing certificate**加入白名单。换种说法，苹果明确允许你的app运行在某些环境下，比如在被选中设备上运行调试。**provisioning profile**也罗列了你的app所被授予的权限。**code signing certificate**包含了你实际用来签名的私钥。

有两种法师来获取证书和配置文件，这取决于你是否已注册成为一个iOS开发者。

#### 使用一个iOS开发者账户：
如果你之前使用Xcode开发或者部署过iOS应用，说明你已经有自己的代码签名证书了。使用**security**工具来列出你的签名标识：

```
$ security find-identity -p codesigning -v
1) 61FA3547E0AF42A11E233F6A2B255E6B6AF262CE "iPhone Distribution: Vantage Point Security Pte. Ltd."
2) 8004380F331DCA22CC1B47FB1A805890AE41C938 "iPhone Developer: Bernhard Müller (RV852WND79)"

```
已注册的开发者能够从苹果开发者门户网站获取到配置文件。[这里涉及到第一次创建App ID，然后发布配置文件来允许这个App ID运行在你的设备上](https://developer.apple.com/library/content/documentation/IDEs/Conceptual/AppDistributionGuide/MaintainingProfiles/MaintainingProfiles.html)。出于重打包的目的，你选择了哪个App ID并不重要，你甚至可以重用一个已经存在的。关键是要有一个匹配的配置文件。确认你创建的是一个开发配置文件，而不是分发文件，因为你将需要能够附加一个调试器到app。

#### 使用一个常规的iTunes账户：
很幸运，即使你不是一个付费开发者，苹果会发布一个免费的开发配置文件给你。你能够通过Xcode，使用你的Apple账户来获取配置文件：简单创建一个空的iOS项目，从app容器中提取**embedded.mobileprovision**。[NCC blog](https://www.nccgroup.trust/au/about-us/newsroom-and-events/blogs/2016/october/ios-instrumentation-without-jailbreak/)详细的解释了这个过程。

当你已经获取到配置文件后，你能够使用**security**工具检查文件的内容。除了被允许的证书和设备，你能够从文件中找到授予app的权限。你之后做代码签名的时候需要这些，所以像下图一样把这些提取到一个独立的plist文件。看一下整个文件的内容，检查是否和期望的一样。

```
$ security cms -D -i AwesomeRepackaging.mobileprovision > profile.plist
$ /usr/libexec/PlistBuddy -x -c 'Print :Entitlements' profile.plist > entitlements.plist
$ cat entitlements.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
<key>application-identifier</key>
<string>LRUD9L355Y.sg.vantagepoint.repackage</string>
<key>com.apple.developer.team-identifier</key>
<string>LRUD9L355Y</string>
<key>get-task-allow</key>
<true/>
<key>keychain-access-groups</key>
<array>
<string>LRUD9L355Y.*</string>
</array>
</dict>
</plist>
```
记住，应用标示符(App ID)，是由TeamID(LRUD9L355Y)和Bundle ID (sg.vantagepoint.repackage)组成的。这个配置文件只对使用这个指定app id的app有效。**"get-task-allow"**key也是非常重要的，当设置为"true"，其他的进程，比如debugging server,将被允许附加到app（因此，在分发文件中这个值会被设置成"false"）。

### 其他准备工作
为了能让我们的app在启动时加载一个附加的库，我们需要一些插入额外加载命令到主执行文件的Mach-O头的方式。我们使用[optool](https://github.com/alexzielenski/optool)来自动化这个过程：

```
$ git clone https://github.com/alexzielenski/optool.git
$ cd optool/
$ git submodule update --init --recursive
```
我们还将使用[ios-deploy](https://github.com/phonegap/ios-deploy)，一个用于脱离Xcode来部署和调试iOS app的工具。（如果你还没有node.js，你可以通过homebrew来安装）

```
$ npm install -g ios-deploy
```
在接下来的例子中，你还需要**FridaGadget.dylib**:

```
$ curl -O https://build.frida.re/frida/ios/lib/FridaGadget.dylib
```
除了上面所列出的工具，我们还将使用OS X和Xcode下的标准工具（确认你已经安装了Xcode命令行开发者工具）。

### 打补丁，重打包，重签名
Time to get serious! 理所当然的，IPA文件实际上是ZIP压缩文件，所以可以使用任意zip工具解压。然后，复制**FridaGadget.dylib**到app目录，并且使用optool添加加载命令到"UnCrackable Level 1"的二进制文件。

```
$ unzip UnCrackable_Level1.ipa
$ cp FridaGadget.dylib Payload/UnCrackable\ Level\ 1.app/
$ optool install -c load -p "@executable_path/FridaGadget.dylib" -t Payload/UnCrackable\ Level\ 1.app/UnCrackable\ Level\ 1
Found FAT Header
Found thin header...
Found thin header...
Inserting a LC_LOAD_DYLIB command for architecture: arm
Successfully inserted a LC_LOAD_DYLIB command for arm
Inserting a LC_LOAD_DYLIB command for architecture: arm64
Successfully inserted a LC_LOAD_DYLIB command for arm64
Writing executable to Payload/UnCrackable Level 1.app/UnCrackable Level 1..
```
如此明显的篡改肯定会使主执行文件的代码签名失效，所以这不能在未越狱设备上运行。你还需要替换配置文件，并使用配置文件中的证书对主执行文件和FridaGadgt.dylib签名。

首先，我们将自己的配置文件添加到包里：

```
$ cp AwesomeRepackaging.mobileprovision Payload/UnCrackable\ Level\ 1.app/embedded.mobileprovision\
```
接下来，我们需要确认Info.plist文件中的BundleID是否和配置文件中的匹配。原因是因为**codesign**会在签名的时候从Info.plist中读取BundleID，错误的值将会导致一个无效的签名。

```
$ /usr/libexec/PlistBuddy -c "Set :CFBundleIdentifier sg.vantagepoint.repackage" Payload/UnCrackable\ Level\ 1.app/Info.plist
```
最后，使用**codesign**工具来重签名这两个二进制文件：

```
$ rm -rf Payload/F/_CodeSignature
$ /usr/bin/codesign --force --sign 8004380F331DCA22CC1B47FB1A805890AE41C938 Payload/UnCrackable\ Level\ 1.app/FridaGadget.dylib
Payload/UnCrackable Level 1.app/FridaGadget.dylib: replacing existing signature
$ /usr/bin/codesign --force --sign 8004380F331DCA22CC1B47FB1A805890AE41C938 --entitlements entitlements.plist Payload/UnCrackable\ Level\ 1.app/UnCrackable\ Level\ 1
Payload/UnCrackable Level 1.app/UnCrackable Level 1: replacing existing signature
```
### 安装及运行App
现在你已经准备好运行修改后的app了。根据以下命令在设备上部署并运行app。

```
$ ios-deploy --debug --bundle Payload/UnCrackable\ Level\ 1.app/
```
如果一切顺利，app会附加上lldb，在调试模式下启动。Frida现在也应该能够附加到app。你可以通过**frida-ps**命令来确认：

```
$ frida-ps -U
PID Name
--- ------
499 Gadget
```
你现在能像平常一样使用Frida来分析app了。或许它并不像它名字的含义一样**Uncrackable**(不能被破解)？

### 问题排除
如果遇到错误（通常都会），最有可能是配置文件和代码签名头不匹配。在这种情况下，最有帮助的是翻阅[官方文档](https://developer.apple.com/library/content/documentation/IDEs/Conceptual/AppDistributionGuide/MaintainingProfiles/MaintainingProfiles.html)，并弄明白整个系统是如何工作的。另外我还找到一个有用的资源：
[苹果的关于权限问题的解决](https://developer.apple.com/library/content/technotes/tn2415/_index.html)
