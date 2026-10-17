# 03 跳槽，使用codex
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  

<br>

<br>

## 00 前言
***
事情的起因是什么呢，做一个PPT给我恶心到了，使用了各种skill，但是大肥鱼不给力，codewhale也不给力。  
入魔了，所以考虑入坑codex，Claude Code容易被封号，codex说是更综合，视觉也更强，入坑了。  
以后涉及视觉的，交给codex，不涉及视觉的，考虑依旧使用之前的codewhale或者后续用别的。  

<br>

<br>

## 01 安装codex
***
访问下载链接：  
- x64 (Intel/AMD处理器)：https://persistent.oaistatic.com/codex-app-prod/ChatGPT-x64.msix
- ARM64 (ARM处理器)：https://persistent.oaistatic.com/codex-app-prod/ChatGPT-arm64.msix

下载对应系统安装包,我是Windows，所以本篇博客是在Windows上的过程。  
下载安装包后双击运行。  

<br>

<br>

## 02 解决启动无窗口问题
***
禁用Sparkle更新器启动，在PowerShell窗口中执行：
```powershell
# 1. 再次确保没有残留进程
Get-Process ChatGPT -ErrorAction SilentlyContinue | Stop-Process -Force
Start-Sleep -Seconds 2

# 2. 在当前会话中临时禁用 Sparkle 更新器
$env:CODEX_SPARKLE_ENABLED = "false"

# 3. 通过系统接口启动应用
$pkg = Get-AppxPackage -Name OpenAI.Codex
$exe = Join-Path $pkg.InstallLocation "app\ChatGPT.exe"
Invoke-CommandInDesktopPackage -PackageFamilyName $pkg.PackageFamilyName -AppId "App" -Command $exe
```
窗口应该可以如愿出现。  
后续使用下面步骤，让更新器不再捣乱：  
1. 按 Win + R，输入 sysdm.cpl 回车。
2. 切换到“高级”选项卡，点击“环境变量”。
3. 在“用户变量”区域，点击“新建”。
4. 变量名填：CODEX_SPARKLE_ENABLED，变量值填：false。
5. 点击确定保存。

<br>

<br>

## 03 获取账号
***
这一部分需要你从官网注册一个账号了。  
电话号码可以使用虚拟接码平台，可以试试https://hero-sms.com/cn/purchases/numbers。  

<br>

<br>

## 04 配置API
***
我这里是使用Kimi为例。  
https://platform.kimi.com/docs/guide/codex-kimi  
这是官方的教程。  
我们Win+R，输入`%USERPROFILE%\.codex\config.toml`。
将下面三行替换前两行：
```text
model = "kimi-k3"
model_provider = "kimi"
model_context_window = 1048576
```
并在最后添加：
```text
[model_providers.kimi]
name = "Kimi"
base_url = "https://api.moonshot.cn/v1"
env_key = "KIMI_API_KEY"
wire_api = "responses"
```
然后退出后重新进入即可。  
注意Kimi支持的等级为：low、high、max

<br>

<br>

## 05 验证
***
输入：
```text
你是谁
```
如果回复你就说明成功了。  
我的回复是：  
```text

```

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>