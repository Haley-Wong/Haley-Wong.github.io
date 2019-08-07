---
layout:     post
title:      "Mac下Jenkins+SVN+Xcode构建持续导出环境"
date:       2016-01-19
author:     "Haley_Wong"
catalog:    true
tags:
    - Jenkins
---

每一次新版本要发布，都被测试部门催成狗，测试部也耐不住了，想自己打包，研发只管提交代码，听到这个消息，还是很开心的，终于不用打包了。跟同事折腾Jenkins三天，终于正常导出ipa包了！！

因为网上教程多是依靠Github，而且很多是在Jenkins中配置Xcode参数，相当的麻烦，我们是用Shell 脚本，非常的easy。在这里记录下环境搭建的过程，希望能帮他人减少一点坑。

# 1 安装Jenkins
[Jenkins](http://baike.baidu.com/link?url=BaaxH2yX4xBX7JNTOZvhPhe78o8ABVVo_ZZlsIyitogh9ziX9QpaGt0qR4ykuo-YQerCe_PeJjHevYQQCFr-Sa)是基于Java开发的一种[持续集成](http://baike.baidu.com/view/5253255.htm)工具。所以呢，要使用Jenkins必须使用先安装JDK。
#### JDK安装
JDK [下载地址](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
![](/img/blogs/jenkins-SVN-Xcode/img_01.png)
安装JDK的过程略，别说你不会安装（如有不会安装的，自行百度）。

#### Jenkins安装
Jenkins [下载地址](http://jenkins-ci.org/)

![](/img/blogs/jenkins-SVN-Xcode/img_02.png)
点击图中 Mac OS X，会自动下载【jenkins-1.644.pkg】

安装过程略（双击jenkins-1.644.pkg后，下一步就OK了）。

> 注意： 
> 1. Jenkins 安装成功后,会创建一个Jenkins用户，而Jenkins的工作区间默认是在【/用户/共享/Jenkins/Home/jobs】目录下，可以用Finder-->前往，进入。
> 2. Jenkins目录下的文件夹的读写权限只对Jenkins用户开放，所以后面apple证书等必须在Jenkins用户下安装，项目的ipa导出也得在Jenkins用户下操作。（或者用管理员权限修改该目录针对用户的权限)
> 3. Jenkins的使用是每一个用户都可以使用，所以有可能导致构建版本的时候报错，还是老老实实在Jenkins用户下操作吧。

#### 测试Jenkins安装成功
打开浏览器，输入[http://localhost:8080](http://localhost:8080)，如果能够正常打开Jenkins，则Jenkins安装成功。

# 2 安装Jenkins插件
Jenkins里有相当多的插件，使用什么工具就安装什么工具的插件。

比如我们这里使用SVN，就安装SVN的插件，如果你使用Git就安装Git的插件。

因为我已经安装了SVN，所以安装插件的过程就用Git来演示。

![安装插件第一步.png](/img/blogs/jenkins-SVN-Xcode/img_03.png)

![安装插件第二步.png](/img/blogs/jenkins-SVN-Xcode/img_04.png)

![第三步，搜索，安装插件.png](/img/blogs/jenkins-SVN-Xcode/img_05.png)

![第四步，安装过程.png](/img/blogs/jenkins-SVN-Xcode/img_06.png)

![第五步，查看已安装插件.png](/img/blogs/jenkins-SVN-Xcode/img_07.png)

# 3 Xcode以及开发证书设置
因为要使用Xcode命令，所以必须保证Xcode command Line已安装。

#### 3.1 设置apple development 证书

在原来Xcode开发所在用户下，导出发布证书，如果要打企业包（299刀）和公司/个人版包(99刀）,则两种证书都要导出，然后拷贝到Jenkins用户环境下，双击安装到Mac 的钥匙串中。

![证书设置第一步.png](/img/blogs/jenkins-SVN-Xcode/img_08.png)

![证书设置第二步.png](/img/blogs/jenkins-SVN-Xcode/img_09.png)

> 注意：因为用户访问钥匙串中的证书需要权限，而用jenkins构建时，不管是用Xcode插件配置还是shell 脚本，都不能输入用户密码，所以必须设置证书的【访问控制】为允许所有应用程序访问此项目。

#### 3.2 安装mobileprovision描述文件

同样需要在Jenkins用户下，安装好打包需要的手机描述文件。

# 4 配置构建项目

下面讲解构建项目的配置，可以使用本地的项目，也可以使用SVN上的项目（只需要填入svn上工程地址即可），然后输入shell 脚本就可以开始构建了。

### 4.1 使用本地项目构建
步骤如下：

![配置项目，第一步.png](/img/blogs/jenkins-SVN-Xcode/img_10.png)

点击OK，在【/用户/共享/Jenkins/Home/jobs】目录下会生成HelloJenkins的目录。

![配置项目第二步.png](/img/blogs/jenkins-SVN-Xcode/img_11.png)

![配置第三步.png](/img/blogs/jenkins-SVN-Xcode/img_12.png)

其他的设置项，均不用设置，只需要设置下脚本即可，脚本详细的内容如下：

```
# 工程名
APP_NAME="HelloJenkins"
# 证书
CODE_SIGN_DISTRIBUTION="iPhone Distribution: XXXXXXXXXXXX"
# info.plist路径
project_infoplist_path="./${APP_NAME}/Info.plist"

if [ ! -f "$project_infoplist_path" ]
then
echo "*************************************"
echo "***       plist文件路径错误！    ****"
echo "***       plist文件路径错误！    ****"
echo "*************************************"
exit
fi

#取版本号
bundleShortVersion=$(/usr/libexec/PlistBuddy -c "print CFBundleShortVersionString" "${project_infoplist_path}")

#取build值
bundleVersion=$(/usr/libexec/PlistBuddy -c "print CFBundleVersion" "${project_infoplist_path}")

DATE="$(date +%Y%m%d%H%M%S)"
IPANAME="${APP_NAME}_V${bundleShortVersion}_${DATE}.ipa"

# 导出路径
IPA_PATH=~/"${IPANAME}"

echo "=================clean================="
xcodebuild -target "${APP_NAME}"  -configuration 'Release' clean

echo "+++++++++++++++++build+++++++++++++++++"
xcodebuild -target "${APP_NAME}" -sdk iphoneos -configuration 'Release' CODE_SIGN_IDENTITY="${CODE_SIGN_DISTRIBUTION}" SYMROOT='$(PWD)'

xcrun -sdk iphoneos PackageApplication "./Release-iphoneos/${APP_NAME}.app" -o "${IPA_PATH}"

if [ -f "$IPA_PATH" ]
then
echo "*************************************"
echo "*             iPa 导出成功          *"
echo "*             iPa 导出成功          *"
echo "*             iPa 导出成功          *"
echo "*             iPa 导出成功          *"
echo "*************************************"
echo "安装文件路径:${IPA_PATH}"
#要上传到蒲公英的ipa文件路径
echo "${IPA_PATH}">> text.txt
else
echo "*************************************"
echo "*             iPa 导出失败          *"
echo "*             iPa 导出失败          *"
echo "*             iPa 导出失败          *"
echo "*             iPa 导出失败          *"
echo "*************************************"
echo "安装文件路径:${IPA_PATH}"
fi
```


> 注意1：【-o ~/$IPANAME】表示导出的ipa文件在当前用户的目录下，即【/用户/共享/Jenkins/】下。
其中CODE_SIGN_IDENTITY="iPhone Distribution: xxxxxxxxxx"是你打包使用的证书在钥匙串中的常用名称。
导出的ipa，叫【HelloJenkins_V1.2_20160118.ipa】。

> 注意2：如果如上图【配置项目第二步.png】那样，在xcodeproj相同目录下，新建一个sh脚本文件，用【sh xxx.sh】命令的话，见下一篇介绍。
如果你的项目中用到了cocoapods,那脚本有几个参数需要调整一下，详情见下一篇。

> 注意3（2016.02.17更新）：CODE_SIGN_IDENTITY 这个属性可以不设置，直接设置profile就可以了，编译时会自动去匹配对应的CODE_SIGN_IDENTITY，需要注意的是设置profile时，设置的是其UUID值。例如【PROVISIONING_PROFILE='f035763e-e847-4db8-ac10-0004809fdc90'】

点击保存，然后点击左侧菜单，立即构建，即可开始构建。

![立即构建.png](/img/blogs/jenkins-SVN-Xcode/img_13.png)

![构建成功.png](/img/blogs/jenkins-SVN-Xcode/img_14.png)

![构建结果.png](/img/blogs/jenkins-SVN-Xcode/img_15.png)

### 4.2 使用svn地址构建

第一步，新建项目，与上面的一样。

第二步，不用将工程拷贝到jobs目录下了，直接在配置里源码管理那一栏设置svn地址

![SVN配置.png](/img/blogs/jenkins-SVN-Xcode/img_16.png)

> 这里如果想要构建svn 上某个版本的工程，只需要再路径后面加上@版本号即可。
例如：http://192.168.1.1:8999/svn/iOS/TestDemo@150。


第三步，设置shell 脚本，与上面的一样。
第四步，立即构建即可。

> 提示：构建成功后，还有一些选项可以设置，比如自动上传到蒲公英或者fir.im，或者邮件通知等。
还可以设置构建触发器，设置在某个时刻自动构建等条件。因为这些设置都挺简单的，而我们目前还未用到，大家自行研究一下吧。


