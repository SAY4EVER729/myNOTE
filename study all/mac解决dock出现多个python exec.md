

## 背景

`pnpm start`启动项目的时候程序坞总会出现多个python 的exec进程

![[Pasted image 20250108235841.png]]

## 原因

测试/升级/重启/更新项目版本 均无效果

发现是 MacOS Sequoia 15.0 的通病

## 解决方法

- navigate to where the python installation files are
	如果是 brew 下载，navigate to:  `/opt/homebrew/Frameworks/Python.framework/Versions/3.12/Resources/Python.app/Contents/Info.plist

- `sudo chmod 666 Info.plist`

- 编辑该`Info.plist`文件，在文件最后一个`</dict>`后面添加：
```
<key>LSUIElement</key>

<true/>
```


## 相关链接

https://forums.developer.apple.com/forums/thread/764130

https://leancrew.com/all-this/2014/01/stopping-the-python-rocketship-icon/