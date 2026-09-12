---
name: xiaoai-speaker-llm
description: "小爱音箱(MiGPT)接入大模型: Node部署、小米登录验证、开机自启。"
version: 0.1.0
author: Xingzhewujiang2017
license: MIT
platforms: [windows]
metadata:
  hermes:
    tags: [migpt, xiaomi, llm, integration]
    related_skills: []
---

# 小爱音箱接入大模型 (MiGPT)

把小米小爱音箱(推荐 Pro 型号)接入任意大模型(DeepSeek/通义/豆包/OpenAI兼容), 让语音问答走大模型, 并配置 Windows 开机自启。核心 = MiGPT 开源项目 + 破解小米异地登录验证 + 计划任务常驻。

## When to Use

- 用户想给小爱音箱接入大模型/让音箱变聪明
- MiGPT 部署、小米账号登录验证失败(70016)、开机自启配置
- 音箱曾接入过, token/密码失效需重新配置

Don't use for: 小度/天猫精灵/HomePod(项目不支持); 刷机改装(那是 open-xiaoai 的方案, 不同项目)。

## Prerequisites

- Windows + 小爱音箱 Pro(最稳, 其他多数型号也可)
- 一个 OpenAI 兼容的模型 key(base_url + api_key + 模型名)
- 小米账号: **小米ID(纯数字, 在 account.xiaomi.com 个人信息页, 不是手机号/邮箱)** + 密码
- Node.js ≥18(推荐用 winget 装 LTS)

## How to Run (部署一条龙)

1. 安装并验证环境:
   `winget install --id OpenJS.NodeJS.LTS -e`(可能弹 UAC 授权)
   `node -v && npm -v`
2. 建项目并装 mi-gpt:
   `mkdir ~/migpt && cd ~/migpt && npm init -y && npm install mi-gpt`
   把 package.json 的 `"main"` 改成 `"type": "module"`(否则 .mjs 的 export 报错)
3. 若 import mi-gpt 报 `Named export 'PrismaClient' not found`:
   `cd node_modules/mi-gpt && npx prisma generate`(修复 Prisma CJS 问题, 装完必做)
4. 写配置文件见下方 步骤A/B; 启动脚本见 步骤C; 验证见 步骤D
5. 开机自启见 步骤E
6. **计划任务创建需管理员**, 提权命令会弹 UAC, 须用户点"是"

## Quick Reference

| 用途 | 命令 |
|---|---|
| 验证模型能加载 | `node --input-type=module -e "import('mi-gpt').then(m=>console.log(Object.keys(m.default)))"` |
| 前台诊断登录 | `timeout 40 node --env-file=.env diag.mjs` |
| 验证 key 已加载 | `node --env-file=.env -e "console.log(!!process.env.OPENAI_API_KEY)"` |
| 写 .mi.json | `node write_clean.mjs` |
| 跑计划任务 | `schtasks /run /tn MiGPT` |
| 停任务 | `schtasks /end /tn MiGPT` |
| 清 node 进程 | `powershell -NoProfile -Command "Get-Process node | Stop-Process -Force"` |

## Procedure

### 步骤A .env 配置(模型)
```
OPENAI_MODEL=deepseek-v4.1-flash
OPENAI_API_KEY=sk-...
OPENAI_BASE_URL=https://来个/聚合服务/v1
```
- **模型名必须与聚合服务的 /models 返回的完全一致**(含大小写/连字符/无空格)。先 `curl $BASE/models -H "Authorization: Bearer $KEY"` 拉真实列表, 别照抄显示名(如 `DeepSeek V4.1 flash` 不等于 `deepseek-v4.1-flash`)
- **node 不会自动读 .env**, 启动必须带 `--env-file=.env`(见 步骤C/步骤E), 否则报 `OPENAI_API_KEY missing`

### 步骤B .migpt.js(小米账号+音箱)
```js
export default {
  speaker: {
    userId: "<小米ID>",   // 纯数字, account.xiaomi.com 个人信息页可查
    password: "<你的密码>",
    did: "<音箱DID或名称>",  // 首次跑会自动打印, 或填米家里的名称
  }
};
```

### 步骤C start.js(启动脚本)
```js
import { MiGPT } from "mi-gpt";
const client = MiGPT.create({
  speaker: { userId: "...", password: "...", did: "..." },
});
console.log("🚀 MiGPT 启动中...");
await client.start();
```
启动: `node --env-file=.env start.js`(**必带 --env-file**)

### 步骤D 小米登录: 破解异地验证(核心难点)

**背景**: mi-gpt v4 用 `mi-service-lite`, 登录逻辑是 `userId + md5(密码)` 直连小米 `serviceLoginAuth2`。首次在新设备登录会触发"异地登录安全验证" → 报 `70016 登录验证失败` 且 captchaUrl=null(纯密码被拒)。

**解法(一次性建立设备信任, 之后纯密码即可登录)**:
1. 用已登录小米账号的浏览器调试实例(本机保持登录态的调试 Chrome(如 ChromeDebugProfile + remote-debugging-port), 见 launch-debug-chrome)连接
2. 通过 CDP `Network.getAllCookies` 抓 `passToken`、`deviceId`、`serviceToken`、`cUserId`
3. 写入 `.mi.json`(v4 的存储格式, 需含匹配的 deviceId+passToken):
   `{ miiot: {userId, password, sid:"xiaomiio", deviceId, pass:{passToken}}, mina: {...sid:"micoapi"} }`
4. 用这个 .mi.json 跑一次 → 服务启动成功 → mi-service-lite 登录后**自动把最新 session 回写进 .mi.json**
5. 之后可把 .mi.json 精简为纯 `userId+password+deviceId`(去掉 pass/serviceToken), 小米已信任此设备, 纯密码登录即可

> 关键认知: Chrome 抓的 passToken 绑定**浏览器自己的 deviceId**, 必须把**同一对 deviceId+passToken**一起写进 .mi.json, 否则仍触发验证。首次建立信任后 token 不再是必需。

### 步骤E 开机自启(计划任务 + 隐藏窗口)

**要点**: InteractiveToken 计划任务若直接前台跑 node, 会把 cmd 窗口常驻在桌面(碍眼)。**用 wscript 隐藏控制台启动 node**, 服务后台跑、桌面不弹窗口。

1. 隐藏启动器 `migpt-hidden.vbs`(放 hermes/scripts, 无 TTY/隐藏窗口下启动):
```vbs
' 用法: wscript.exe migpt-hidden.vbs <MiGPT目录>
' sh.Run 第2参数 0=隐藏窗口; 第3参数 False=不等待
Set sh = CreateObject("WScript.Shell")
migptDir = WScript.Arguments(0)
cmd = "cmd /c cd /d " & Chr(34) & migptDir & Chr(34) & " && node --env-file=.env start.js <NUL >> migpt.log 2>&1"
sh.Run cmd, 0, False
```
   - **`<NUL` 必加**: 无 TTY 下 MiGPT 内部读 stdin 会报 `stdin is not a tty` 退出
   - **cd /d 必加**: wscript 的 CurrentDirectory 不传给 cmd 子进程, 须在命令里显式 cd
2. 自启 bat `migpt-start.bat`(计划任务调用, 只负责触发 wscript):
```bat
@echo off
cd /d %USERPROFILE%
wscript.exe "<scripts目录>\migpt-hidden.vbs" "%USERPROFILE%\<项目目录>"
exit /b 0
```
   - **别在 bat 里直接前台跑 node**(会弹 cmd 窗口), 必须经 wscript 隐藏
3. 建任务(需管理员, 弹 UAC):
```
schtasks /create /tn MiGPT /xml "<项目目录>\migpt-task.xml" /f
```
   XML 需 InteractiveToken + LogonTrigger(登录时), 仿现有 Hermes_Gateway 任务的 XML
4. 验证: `schtasks /run /tn MiGPT` → 看 migpt.log 出现 `Speaker ✅ 服务已启动...` → 确认 node 进程恰 1 个, 且桌面无 cmd 窗口
5. **清理重复实例**: 多次测试可能残留多个 node, 会抢同一音箱, 用 Stop-Process 清干净再 /run

## Pitfalls

- **OPENAI_API_KEY missing**: 忘了 `--env-file=.env`; node 不会自动读 .env
- **70016 登录验证失败且 captchaUrl=null**: 首次异地验证被拒, 走 步骤D 建立信任非重试(重试会触发小米防爆破锁号1小时)
- **PrismaClient not found**: npm 装完没跑 `npx prisma generate`(在 mi-gpt 包目录里跑)
- **stdin is not a tty**: bat 后台/计划任务环境缺 TTY, 启动命令加 `<NUL`(或交互 PTY 跑)
- **桌面弹 cmd 窗口**: 计划任务 InteractiveToken 直接前台跑 node 会常驻 cmd 窗口; 用 wscript 隐藏启动(sh.Run, 0, False), bat 只触发 wscript
- **模型 404/model_not_found**: 模型名与 /models 列表不一致(大小写/空格/连字符)
- **多实例抢音箱**: taskkill node 清理; 计划任务 MultipleInstancesPolicy 设 IgnoreNew
- **Access is denied 建任务**: 无管理员, 提权会弹 UAC 需用户点"是"
- **token 有效期**: 首次信任后纯密码+自动回写, token 不再是必需; 仅改密码或换设备才需重走 步骤D
- **调度触发 debug wrong**: mi-gpt v4 不读旧文档的 passToken .mi.json 结构(那是旧版), v4 存 {userid,password,deviceid,sid} 由 mi-service-lite 维护

## Verification

- `migpt.log` 出现 `Speaker ✅ 服务已启动...`
- `schtasks /query /tn MiGPT` 状态 Running/Ready
- node 进程数恰为 1
- 对音箱说"小爱同学, 请..."能收到大模型风格回答
