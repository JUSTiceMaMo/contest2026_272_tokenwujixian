---
name: bk7258-ui-ontarget-test
description: "在 BK7258 真机 CP NSH(/dev/ttyUSB0) 上驱动 ai_agent_ui 调试菜单，验证 pet 状态环/粒子流/聊天气泡各页面与 6 种 agent 状态的显示效果。Use when 需要在真机屏幕验证 ai_agent UI 改动、驱动 pet/particle/chat 页面与色条气泡、在 CP 串口操作 ai_agent_ui 调试命令(page/pet/ptt/say/sayc/user/demo)、或排查 flush/SPI/串口相关显示问题。烧录见 bk7258-worktree-build。"
---

# BK7258 真机 UI 测试（ai_agent_ui）

真机屏幕验证 ai_agent UI 各页面/状态的标准流程与指令。目标是**复用**而非每次重新摸索——读本文即接续上次进度，别从零踩坑。

## 背景（路 B 架构，必读）

- BK7258 双核：**CP(CPU0) 拥有 LCD/LVGL/FPU**，**AP(CPU1) 跑 agent/LLM/voice**。
- 真机 UI 由 CP 侧 builtin **`ai_agent_ui`** 渲染（`packages/ai_agent/src/ui_cp/cp_ui_main.c`），自起 LVGL + chat screen + pet_status_view + particle_flow_view。
- AP→CP 的 UI 走 RPMsg（ui_rpmsg_client → ui_rpmsg_server），但**真机上 AP agent 没跑**，所以测试走 **CP 本地调试菜单**（cp_ui_main.c 里加的 help/page/pet/ptt/say/sayc/user/demo）。
- 串口：CP NSH 控制台 = **/dev/ttyUSB0**（CH340，115200）；AP 日志 = /dev/ttyACM1（921600）。

## 测试前准备

1. **烧录**含调试菜单的最新 CP 镜像（流程见 bk7258-worktree-build）。两条硬约束：
   - 当前固件**不含 bootloader-reset patch**：notify_flash.py 的自动 `reboot bootloader` 不生效，必须在 `Getting Bus...` 窗口**手动按 RESET 键（K1，接 CEN，别按 S1/S2/S3）**，直到 `Gotten Bus` / `Writing Flash OK`。
   - 烧录前必须先 `tmux kill-session -t diag` 关掉 picocom（释放 ttyUSB0）。
2. 开串口：`tmux new-session -d -s diag 'picocom /dev/ttyUSB0 -b 115200 --imap lfcrlf'`。
3. 确认 NSH 可交互：发回车看到 `nsh>`；`help` 里 Builtin Apps 应含 `ai_agent_ui`。

## 启动 UI

前台启动（调试菜单只在**前台**能收到命令）：

```
ai_agent_ui
```

启动成功判据（syslog）：`[pet_view] init OK` / `[particle_view] init OK` / `[lvgl_ui] started` / `[ai_agent_ui] UI up; LVGL thread running`。此刻屏幕 = pet 首屏（idle 绿呼吸环）。

> 烧录后板子自动重启，NSH 干净起来，`ai_agent_ui` **不会自动启动**，必须手动敲。若前台启动后 NSH prompt 消失、命令无回显，是它前台占了 stdin——正常。

## 调试命令表

| 命令 | 作用 |
|---|---|
| `help` | 打印命令表 |
| `show` / `chat` | 切聊天气泡屏 |
| `page 0` | pet 状态环首屏 |
| `page 1` | particle 粒子流页 |
| `pet 0..5` | 设置 pet 状态（同时镜像到 particle）：0=idle绿 1=listening青 2=thinking蓝 3=executing橙 4=waiting黄 5=error红 |
| `ptype 0..4` | 切换像素宠物：0=白猫 1=绿青蛙 2=黑猫 3=棕狗 4=粉猪（越界回猫，下一帧状态更新即生效） |
| `ptt 0/1/2` | PTT 状态：0=idle 1=录音中 2=识别中 |
| `user <text>` | 用户右对齐蓝气泡（无 TTS） |
| `say <text>` | Agent 气泡（自动分类色条） |
| `sayc <hex> <text>` | Agent 气泡 + 指定色条颜色 |
| `demo` | 一键轮转 pet 0-5 → particle 0-5 → chat + 4 色条，最后回 pet idle |
| `quit` | 退出 UI 回 NSH（Ctrl-C 无效，前台 builtin 收不到 SIGINT） |

### 色条色值（`sayc` 第二个参数）

| 分类 | 色值 |
|---|---|
| 蓝 structured | `0x3a7bd5` |
| 绿 conclusion | `0x4caf50` |
| 橙 jira | `0xff9800` |
| 灰 plain | `0x9e9e9e` |

## 踩坑清单（必读，每条都踩过）

1. **烧录自动复位（已含 patch，但 ai_agent_ui 会污染注入）**：2026-10-14 起固件已含 bootloader-reset patch，`notify_flash.py` 自动注入 `reboot bootloader` **零人工**生效（两次实测 `Gotten Bus`→`Writing Flash OK`）。但**烧录前必须确保 ai_agent_ui 没在前台/后台跑**——它持续打 `[lv_nuttx_lcd] flush #N` 日志刷屏，会污染注入字节（`reboot bootloader` 到 NSH 变乱码 `command not found`），导致 `GetBus fail`。正确顺序：`kill -9 <ai_agent_ui pid>`（`ps` 里 COMMAND=ai_agent_ui 的 Task pid）→ 确认 NSH 干净 → 再烧。若已 GetBus fail，板子其实没被擦，重试前先 ps 确认无 ai_agent_ui。
2. **TX 掉线**：若发命令无任何回显、只有板子→host 的日志（idle rotate/adc release）在滚，是 host→board TX 掉线。软重插 CH340（`echo 0/1 > /sys/bus/usb/devices/1-2/authorized`）**无效**，要物理拔插 USB 断电。烧录脚本的 `do_reset_signal` 也会顺带复位板子恢复串口。
3. **`show` 不重置 idle 让位**：`lvgl_ui_channel_show` 不调用 `pet_idle_reset`。若上次消息已超 8s，`show` 后约 1s 内就被 idle 切回 pet。**正确做法：直接发气泡（user/say/sayc），不手动 show**；气泡经 async 添加时才重置 idle tick（`chat_view_add_message_async_cb` 内 `pet_idle_reset()`）。
4. **气泡添加不打日志**：`chat_view_add_message` 静默，无 syslog。判断气泡是否生效**看屏幕**，别靠日志。`help`/`pet` 有日志；`show`/`user`/`say` 通常没有（除非触发 `loading chat screen`）。
5. **idle 让位/轮转**：chat 屏空闲 8s 自动回 pet（`PET_IDLE_YIELD_MS=8000`）；idle 时 pet↔particle 每 10s 轮转（`ROTATE_INTERVAL_MS=10000`）。发气泡会暂停轮转。
6. **异步时序**：所有 UI 命令经 `lv_async_call` 调度到 LVGL 线程执行，日志有延迟。`demo` 的 chat 部分是异步排队，demo 主循环立即返回，chat 屏在 demo 结束后才真正 load（`set_page(0)` 先切回 pet，随后 chat 异步 load 又抢前台）。
7. **flush 慢（根因已定位）**：新镜像 flush **1.9~9.8 flush/s** 是 defconfig `CONFIG_BK7258_LCD1_SPI_BITBANG=y`（bit-bang 软件翻转 GPIO，天然慢）所致，**不是**增量构建带偏，与 cp_ui_main.c 改动无关；旧镜像 145/s 是 HW SPI。回快档：defconfig 去 `CONFIG_BK7258_LCD1_SPI_BITBANG`、改 `CONFIG_BK7258_LCD1_SPI_HW=y`（bk7258_spi_hw.c 已存在，第 60 行 `#ifdef CONFIG_BK7258_LCD1_SPI_HW`），再刷 build-dir defconfig + 增量构建。`rate:` 日志判据：~145=HW SPI 正常，~2~10=bit-bang。**2026-10-14 现状**：已切回 HW SPI+`CONFIG_SPI=y`，宠物首屏 97×97 精灵区域实测 **19.7 flush/s**（每 flush ~17ms、约 1 MB/s≈8Mbps SPI 吞吐）；对像素猫脸的慢动画（呼吸/眨眼/上视下视/抖动）已够流畅，未深挖 SPI 时钟上探（145/s 是更高时钟或更小区域的历史数据）。
8. **中文经串口发送**：`tmux send-keys -t diag 'sayc 0x4caf50 值得合入，建议采纳' Enter` 中文可正常传入（UTF-8），picocom `--imap lfcrlf` 不影响。

## 验证成功判据

- 屏幕按命令在 pet / particle / chat 三页之间切换。
- pet 6 状态颜色/形态变化可见（绿/青/蓝/橙/黄/红）。
- chat 屏显示 1 条 user 右对齐蓝气泡 + 4 条色条 Agent 卡片（蓝/绿/橙/灰）。
- 日志 `[lv_nuttx_lcd] flush rc=0`（无渲染报错）。

## 测试后收尾

- `quit` 退出 ai_agent_ui 回 NSH（或直接 `tmux kill-session -t diag` 断串口）。

## 最近测试状态（每次测试后更新本节）

- **日期**：2026-10-15（第八轮：PTT 胶囊下移贴边 + 移除"关闭"按钮）
- **烧录镜像**：all-app.bin SHA-256 `92f07c2f85be2e566a0bcccf5da27c9f9268ce3e8a5b29b46878632aa0c2e030`（CP 含本轮改动；AP 复用旧构建）。
- **本轮改动**：① **移除"关闭"按钮**——屏幕非触屏，"关闭"纯装饰无触摸功能，整块删除（`CLOSE_BTN_TEXT` 宏、`close_btn` 字段、`close_btn_event_cb` 声明+函数体、`close_cx/close_cy` 布局参数、创建块、3 处 `close_btn=NULL` 清理、`lvgl_ui_channel_stop_sync()` 死代码，全部清干净，`grep close_btn/stop_sync/CLOSE_BTN` 零残留）；② **PTT 胶囊下移贴边**——`bot_m 9%→4%`（胶囊底 y 240→252，距圆盘底缘 260 仅 8px，底角 (82,252)=√(38²+104²)=110.7 < R=112 仍安全）；`top_m 22%→14%`（关闭按钮腾出的顶部空间回收，chat_list 顶 y 85→67，chat_list 高 104→134px，可多显示气泡）。
- **caption/环重叠根因（上轮结论，保留）**：MiSans 16 字体 `line_height=20`，caption 实际高 20px（不是字号 16px）。`lv_obj_align(LV_ALIGN_CENTER,0,dy)` 对齐 caption 中心，顶边 = dy-10。dy=88 时顶边 y=226 → 净空 3px。
- **已验证（日志层）**：烧录 `Writing Flash OK`（零人工，自动注入 `reboot bootloader` 生效——**前提是先 `quit` 掉前台 `ai_agent_ui` 让 NSH 干净**，否则注入字节被 ai_agent_ui 吃掉 → `GetBus fail`）。`ai_agent_ui` 启动后 `page 0/1`、`pet 1/3`、`ptt 1/2/0`、`user`、`sayc` 全走一遍，**零 `flush rc!=0`、零 assert/panic/ERROR**；`loading chat screen` 正常；idle pet↔particle 每 10s 轮转正常；HW SPI ~28 flush/s（particle 页 `rate: 100 flushes in ~3500ms`，rate 日志 Y 值仍随累计数高估，真实看 `100/Xms`）。
- **待肉眼确认**：① "请说"胶囊是否明显靠下、贴近圆盘底缘（底 y≈252）；② 关闭按钮是否彻底消失（右上角无"关闭"胶囊）；③ 顶部空出的空间是否让气泡列表更舒展。
- **待办（后续轮）**：继续系统性几何审查（气泡换行边界、长文本多行换行、particle 粒子是否遮挡 caption 文案等）。
