---
name: bk7258-ui-ontarget-test
description: "在 BK7258 真机屏幕上验证 ai_agent UI 的稳定测试方法与排障。不冻结具体流程——具体界面切换顺序/通过判据以 yzhang/testLoop/ui-screen-transition-order.md、联网与配模型以 yzhang/testLoop/wifi-and-llm-config-flow.md 为持续更新的单一事实源;本 skill 只固化稳定不变的驱动脚手架(两个提示符 nsh-ap>/vela>、wapi/set_llm、烧录清 NVS、板子只在输入时输出、永不 quit)与判定/脱敏流程。Use when 需要在真机按最新流程驱动并判定 UI 状态、做回归、或排查 flush/串口/联网/模型配置/控制台写卡死问题。烧录见 bk7258-worktree-build。"
---

# BK7258 真机 UI 测试(流程驱动,不冻结流程)

一句话定位:**本 skill 固化「怎么稳定地驱动真机、怎么判定、怎么脱敏」,不冻结「具体流程长什么样」。** 具体流程由两份持续更新的文档作为单一事实源,每次测试前先读最新版,再回来用本 skill 的脚手架执行。

## 先读:流程的单一事实源(持续更新)

| 文档(相对本 repo 路径) | 管什么 | 何时读 |
|---|---|---|
| `../../../../yzhang/testLoop/ui-screen-transition-order.md` | 界面切换总顺序、每条支线(A–F)的「触发→预期日志→预期界面(通过判据)」、驱动命令速查、现状待定 | **每次测试前必读**,这是「预期界面/通过判据」的唯一权威 |
| `../../../../yzhang/testLoop/wifi-and-llm-config-flow.md` | 联网(`wapi`)+ 配模型(`set_dns`/`set_llm`)的步骤、判据、坑;BUG-20261021-01 定位全过程与最终结论 | 需要联网/配模型/排查控制台写卡死时读 |
| `../../../../yzhang/cmd.sh` | 烧录 + minicom + wapi + set_llm + ask 的**实际可跑命令**(含密码/密钥) | 抄命令或取凭据时读;只原文引用,**不要把密钥写进本 skill** |

> 流程一直在变(历史上 UI 已从 CP 迁到 AP、命令从 `page/pet/say` 改成 `set_pet/set_page/show_chat`)。**以这两份文档为准,不要凭记忆**;发现文档与真机不符时,先更新文档(它才是事实源),再回来跑。

## 稳定不变的架构/约定(本节才是本 skill 的职责)

- **UI 在 AP**。串口 `/dev/ttyUSB0`(115200)= AP 控制台,只有一个物理串口。
- **两个提示符,命令不互通**:
  - `nsh-ap>`(NuttX shell):`wapi`、`nslookup`、`ping`、`ai_agent`。
  - `vela>`(agent CLI,在 `nsh-ap>` 敲 `ai_agent` 后进入):`set_dns`/`set_llm`/`set_pet`/`set_page`/`show_chat`/`ask`/`rpending`/`config_show`/`net_status` 等。`vela>` **不转发** NSH 命令。
- **联网**:`nsh-ap>` 下 `wapi connect "<SSID>" <pw>`(成功判据 `BK STA got ip4` / `wlan0 ip`)。`set_wifi` 是空壳,别用。
- **配模型**:`vela>` 下先 `set_dns 10.237.25.8` 再 `set_llm <base_url> <model> <api_key>`,然后 `config_show` 校验、`ask` 验证。密钥只从 cmd.sh 取。
- **整片烧录会清空 NVS**:烧完必须重新联网 + 配 key,否则文字/语音支线全跑不了。
- **板子只在收到输入时才输出**:交互或自动化都要**先发 `\r\n` 再等提示符**,否则永远等不到。
- **永不 `quit` 退出 agent**:会卡死、回不到 NSH;要退出/重来 = 板上 RESET(K1)或重烧。

## 基础指令(烧录 → 串口 → 联网 → 配模型 → 启动)

「把板子从刚烧好带到能 `ask`」的最小链路。每条命令后附**标志性节点**——串口上出现它才说明该步成功;密码/密钥照抄 `yzhang/cmd.sh`,本文不写明文。

**0. 烧录**(走 `bk7258-worktree-build`,命令同 cmd.sh 第 1 行)
```sh
full_flow.sh --image <all-app.bin 绝对路径> --worktree <repo> --flash --device /dev/ttyUSB0
```
标志性节点:`connect success` → `Gotten Bus...` → `Writing Flash OK`。烧完板子自动复位、停在 `nsh-ap>`。

**1. 开串口**
```sh
sudo minicom -o -D /dev/ttyUSB0 -b 115200
```
标志性节点:出现 `nsh-ap>` 提示符(空白就先回车——板子只在输入时输出)。

**2. 联网**(`nsh-ap>` 下,密码照抄 cmd.sh)
```sh
wapi connect "<SSID>" <pw>
nslookup www.baidu.com   # 可选:验证 DNS + 出网
```
标志性节点:`BK STA got ip4 ...` 或 `wlan0 ip ...`。

**3. 启动 agent**(`nsh-ap>` 下)
```sh
ai_agent
```
标志性节点:提示符变 `vela>`;日志 `[pet_view] init OK` / `[pet_view] pet screen shown`。

**4. 配模型**(`vela>` 下,key 照抄 cmd.sh)
```sh
set_dns 10.237.25.8
set_llm http://model.mify.ai.srv deepseek/deepseek-flash <api_key>
config_show
```
标志性节点:`[llm] LLM config updated atomically: model.mify.ai.srv/v1/chat/completions (model: deepseek/deepseek-flash)`。

**5. 验证**
```sh
ask 你好
```
标志性节点:`[llm] OpenAI API with tools (...)` = 请求真的发出去了。

> 整片烧录会清空 NVS,所以第 2、4 步每次烧完都必须重做。

## 稳定可靠的测试脚手架

1. **烧录**:走 `bk7258-worktree-build` skill 的 `full_flow.sh --flash`(交互终端,不要后台/重定向)。烧录会顺带复位板子。
2. **开串口**:`sudo minicom -o -D /dev/ttyUSB0 -b 115200`(或 picocom)。串口被占用时先 `sudo fuser -v /dev/ttyUSB0` / `sudo pkill -x minicom`。
3. **驱动**:按 flow doc 的「驱动命令速查」执行。核心命令速览(以文档为准,此处仅供快速回忆):
   - `ai_agent`(`nsh-ap>` 下启动,出 `vela>`)
   - `set_pet 0..5`(0 待机/1 听音/2 思考/3 执行/4 等待/5 错误)、`set_page 0|1`、`show_chat`
   - `ask <文本>`(真实 agent 生命周期)、`rpending <tool> [hint]`(假审批,用 S2/S3 决定)
   - `voice_start`/`voice_stop` 只驱动音频,**不驱动 UI**;语音 UI 支线**必须物理按 S1**
   - `config_show`/`net_status` 查支线前置
4. **判定**:每条步骤同时核 **预期日志 + 预期界面**(判据以 flow doc 为准)。日志级硬判据:无 `flush rc!=0`、无 `assert`/`UsageFault`/`Panic by remote core`、无 `LLM unavailable`(除非故意测 E 支线)。
5. **自动化**(可选):pyserial 直接开 `/dev/ttyUSB0`(别经 minicom);先写 `\r\n` 再等提示符;`set_llm` 整行从 cmd.sh 原样下发,避免脚本里再存一份 key。
6. **脱敏**:落到文件/日志里的一切密钥都要先打码再存/提交(见 `contest-log-redact` skill,必须 `--verify` 输出 `CLEAN` 且退出码 0 才提交)。

## 排障速查(架构级,不易随流程变)

| 现象 | 结论 / 处理 |
|---|---|
| 发命令无回显,只有板子→host 日志在滚 | host→board **TX 掉线**;物理拔插 USB 重建 TX,别盲刷 |
| `quit` 后卡死 | agent 卡死回不到 NSH;复位/重烧,下次别再敲 quit |
| `set_wifi` 没反应 | AP 上 `set_wifi` 是空壳;用 `wapi connect` |
| `nslookup: ... Unreachable: 0100000a` | resolver 还在用默认 DNS 10.0.0.1;先 `wapi connect` 成功,进 `vela>` 后 `set_dns` |
| `ifconfig` 空 | AP 是纯 usrsock 客户端,网卡在 CP;属正常,看 `net_status`/`wlan0 ip` |
| flush 率异常慢(~2–10 flush/s) | defconfig 误切 `SPI_BITBANG`;应 HW SPI + `CONFIG_SPI=y`(正常 ~20–145 flush/s) |
| 在线 `ask` 后 ~20s 整机静默 | 历史 BUG-20261021-01,真根因=RPMSG 控制台写 `wait=true` 无限阻塞,已修(`uart_rpmsg_dmasend` 改 `wait=false`)。若仍复现见 wifi-and-llm-config-flow.md §4.4 下一嫌疑(usrsock `wait=true`) |
| 中文经串口发不进去 | UTF-8 直接发即可;picocom `--imap lfcrlf` 不影响 |

## 流程变了怎么办(保持本 skill 不过时的机制)

1. **先改文档**:把新流程/新判据写进 `ui-screen-transition-order.md` / `wifi-and-llm-config-flow.md`(它们是事实源)。
2. **本 skill 尽量不动**:只要架构约定(两个提示符、串口、`wapi`/`set_llm`、烧录清 NVS、板子只在输入时输出、永不 `quit`)没变,本 skill 就不需要改。
3. **只有「脚手架」变了才改本 skill**:串口节点、烧录方式、脱敏流程、或上表排障条目有变化时,更新对应小节,并在此处记一条变更。
4. **不要**把最新测试状态 / 某轮镜像 SHA 硬写进本 skill——那是 `ui-screen-transition-order.md` 末尾「现状待定」该管的事,写进来必然过期。
