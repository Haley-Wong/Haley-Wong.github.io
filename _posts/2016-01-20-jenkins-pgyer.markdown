---
layout:     post
title:      "Mac下Jenkins构建+蒲公英分发"
date:       2016-01-20
author:     "Haley_Wong"
catalog:    true
tags:
    - Jenkins
---

在上一篇构建过程中，可能你会遇到一些问题，本篇会针对这些问题讲一下解决方案；另外，因为持续构建完成后，有的公司可能不是用企业证书，需要借助蒲公英、fir.im等分发工具供测试人员安装，所以构建完成后自动上传蒲公英、fir.im也很重要，这里记录一下后续操作。

本篇主要解决的问题如下：
* 1、在command中不使用脚本，直接使用【sh jenkins.sh】。
* 2、创建的项目名称带空格，导致脚本构建失败。（该问题已经更新了上一篇的脚本解决了，主要原因是脚本中的变量（如${APP_NAME}）在使用时没有用""包起来，导致执行出错。）
* 3、构建使用cocoapods的项目如何修改脚本。
* 4、如何在自动构建完成后自动上传到蒲公英服务器。

## 1 如何使用【sh jenkins.sh】

![执行脚本.png](/img/blogs/jenkins-pgyer/img_01.jpg)

因为要上传至蒲公英，构建脚本做了小小的修改,借助一个中间文件获取导出的ipa路径，供上传使用。

```
# 工程名
APP_NAME="HelloJenkins"
# 证书
CODE_SIGN_DISTRIBUTION="iPhone Distribution: SunEee Weilian Technology Development Co., Ltd."
# info.plist路径
project_infoplist_path="./${APP_NAME}/Info.plist"

#取版本号
bundleShortVersion=$(/usr/libexec/PlistBuddy -c "print CFBundleShortVersionString" "${project_infoplist_path}")

#取build值
bundleVersion=$(/usr/libexec/PlistBuddy -c "print CFBundleVersion" "${project_infoplist_path}")

DATE="$(date +%Y%m%d%H%M%S)"
IPANAME="${APP_NAME}_V${bundleShortVersion}_${DATE}.ipa"

#要上传的ipa文件路径
IPA_PATH="$HOME/${IPANAME}"
echo ${IPA_PATH}
echo "${IPA_PATH}">> text.txt

echo "=================clean================="
xcodebuild -target "${APP_NAME}"  -configuration 'Release' clean

echo "+++++++++++++++++build+++++++++++++++++"
xcodebuild -target "${APP_NAME}" -sdk iphoneos -configuration 'Release' CODE_SIGN_IDENTITY="${CODE_SIGN_DISTRIBUTION}" SYMROOT='$(PWD)'

xcrun -sdk iphoneos PackageApplication "./Release-iphoneos/${APP_NAME}.app" -o ~/"${IPANAME}"
```

## 2 项目名称带空格,导致构建失败

已解决，过程就略，见上面新脚本。

## 3 使用cocoapods的项目脚本如何改
使用cocoapods后，因为启动项目的工程文件已经由【xxx.xcodeproj】变为【xxx.xcworkspace】,所以在build时，需要添加【-workspace】和【-scheme】，同时去掉【-target】,如果不修改这些参数，构建会报错也会提示设置这些项。

```
# 工程名
APP_NAME="HelloJenkins"
# 证书
CODE_SIGN_DISTRIBUTION="iPhone Distribution: SunEee Weilian Technology Development Co., Ltd."
# info.plist路径
project_infoplist_path="./${APP_NAME}/Info.plist"

#取版本号
bundleShortVersion=$(/usr/libexec/PlistBuddy -c "print CFBundleShortVersionString" "${project_infoplist_path}")

#取build值
bundleVersion=$(/usr/libexec/PlistBuddy -c "print CFBundleVersion" "${project_infoplist_path}")

DATE="$(date +%Y%m%d%H%M%S)"
IPANAME="${APP_NAME}_V${bundleShortVersion}_${DATE}.ipa"

#要上传的ipa文件路径
IPA_PATH="$HOME/${IPANAME}"
echo ${IPA_PATH}
echo "${IPA_PATH}">> text.txt

echo "=================clean================="
xcodebuild -workspace "${APP_NAME}.xcworkspace" -scheme "${APP_NAME}"  -configuration 'Release' clean

echo "+++++++++++++++++build+++++++++++++++++"
xcodebuild -workspace "${APP_NAME}.xcworkspace" -scheme "${APP_NAME}" -sdk iphoneos -configuration 'Release' CODE_SIGN_IDENTITY="${CODE_SIGN_DISTRIBUTION}" SYMROOT='$(PWD)'

xcrun -sdk iphoneos PackageApplication "./Release-iphoneos/${APP_NAME}.app" -o ~/"${IPANAME}"
```

## 4 添加构建后自动上传蒲公英的脚本

![构建后设置.png](/img/blogs/jenkins-pgyer/img_02.jpg)

![构建后设置脚本.png](/img/blogs/jenkins-pgyer/img_03.jpg)

![构建后待执行的脚本.png](/img/blogs/jenkins-pgyer/img_04.png)

upload.sh脚本与上面jenkins.sh脚本在同一目录。【upload.sh】内容如下：

```
#蒲公英上的User Key
uKey="9743f8cbe9ebef9863912a9d52ac19ce"
#蒲公英上的API Key
apiKey="0419615fa1ebbe8179ee9978abc3d753"
#要上传的ipa文件路径
IPA_PATH=$(cat text.txt)

rm -rf text.txt

#执行上传至蒲公英的命令
echo "++++++++++++++upload+++++++++++++"
curl -F "file=@${IPA_PATH}" -F "uKey=${uKey}" -F "_api_key=${apiKey}" http://www.pgyer.com/apiv1/app/upload
```
>注意：脚本中的uKey和apiKey，是自己的账户对应的userKey和apiKey。

上传成功后，会返回相应的json数据。失败提示，可以参考蒲公英官网说明。

![上传成功返回的json.png](/img/blogs/jenkins-pgyer/img_05.jpg)


