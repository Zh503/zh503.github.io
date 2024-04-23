---
title: Navicat无限试用
date: 2024-04-23 19:28:10
tags:
  - 小脚本
categories:
  - 实用工具 
---

 navicat16破解可能出现被植入木马的可能性(git仓库里很多都有),吾爱大神给出了一个无限试用脚本,原理是修改注册表,可以避免这种情况.

```
@echo off
set dn=Info
set rp=HKEY_CURRENT_USER\Software\Classes\CLSID
:: reg delete HKEY_CURRENT_USER\Software\PremiumSoft\NavicatPremium\Registration14XCS /f  %针对navicat15%
reg delete HKEY_CURRENT_USER\Software\PremiumSoft\NavicatPremium\Registration16XCS /f
reg delete HKEY_CURRENT_USER\Software\PremiumSoft\NavicatPremium\Update /f
echo 查找中.....
for /f "tokens=*" %%a in ('reg query "%rp%"') do (
 echo %%a
for /f "tokens=*" %%l in ('reg query "%%a" /f "%dn%" /s /e ^|findstr /i "%dn%"') do (
  echo 正在删除：%%a
  reg delete %%a /f
)
)
echo 完成重置！

pause
exit
```

复制代码,放到文档里,改后缀为bat,运行,完事
