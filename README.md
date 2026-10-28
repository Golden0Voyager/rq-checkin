# RQrun 自动签到（领 RQ 币）

把 RQrun APP 里「首页 → 签到」这件事搬到命令行，一条命令完成签到，可挂到定时任务里每天自动跑。

## 一、签到奖励规则（官方口径）

| 奖励类型 | 规则 |
|---|---|
| 首签奖励 | 首次签到一次性 +5 RQ 币（每账号仅一次） |
| 常规奖励 | 连续签到满 7 天后，**每 3 天**结算一次：连续 <30 天 +1 币；30~60 天 +2 币；>60 天 +3 币 |
| 目标奖励 | 88 天一轮，第 88 天 +25 币；从第 2 轮起每轮再加 5 币，单轮上限 50 币 |

关键点：**断签会清零周期**，所以自动化签到真正的价值是「不断签」。

## 二、原理

RQrun 的签到不是纯 APP 内逻辑，而是一个 H5 页面：`https://www.rq.run/user/SignIn/`。
网页端完整流程（已实测确认接口存在）：

1. `GET  /Home/Login/login.html` —— 拿 `RQWEBSESSID` 会话
2. `GET  /Home/Tool/verify/rand/<随机数>` —— 取登录图形验证码（160×50 PNG，4 位数字）
3. `POST /Home/Login/password_login` —— 账号 + 密码 + 图形验证码
4. `GET  /user/SignIn/` —— 打开签到页，会话落地
5. `GET  /User/SignIn/sign_verify_code` —— **签到专用**图形验证码
6. `POST /User/SignIn/sign_in` —— 提交 `codes=<验证码>`

返回码含义：`1` 签到成功 / `10009` 今日已签 / `10011` 验证码错误（可重试）。

所以每天自动化只需要一次自动登录 + 一次自动签到，**两道图形验证码都由 ddddocr 本地识别**。

## 三、安装

```bash
cd rq-checkin
python3 -m pip install requests ddddocr
```

> `ddddocr` 体积较大（含 onnx 模型，约 130MB 依赖），只需装一次。
> 不想装 OCR 也能用：加 `--manual`，脚本会把验证码弹出来让你手输。

## 四、使用

**方式 A：账号密码（推荐，可完全无人值守）**

```bash
python3 rq_checkin.py --username 你的邮箱或手机号 --password 密码 --save
```

`--save` 会把凭据写入同目录 `config.json`（权限 600），之后直接 `python3 rq_checkin.py` 即可。

**方式 B：复用浏览器会话（微信登录/没有密码的账号）**

浏览器登录 www.rq.run → F12 → Application → Cookies → 复制 `RQWEBSESSID`：

```bash
python3 rq_checkin.py --cookie "RQWEBSESSID=xxxxxxxx"
```

**方式 C：自动读取 Chrome 会话（当前采用，最省事）**

浏览器登录一次后，脚本直接从本机 Chrome 的 Cookie 库里解密取出 `RQWEBSESSID`，不需要手工复制：

```bash
python3 rq_checkin.py --from-chrome
```

`get_rq_cookie.py` 只读 `host like '%rq.run%'` 的记录，其它站点的 Cookie 一条都不碰；
解密用的是 Keychain 里的 `Chrome Safe Storage` 密钥，全程不接触你的密码。

会话有效期约 30 天，到期后需要重新在浏览器登录一次。

**其他参数**

```bash
python3 rq_checkin.py --dry-run    # 只验证登录和签到页可达性，不提交签到
python3 rq_checkin.py --json       # 输出机器可读结果，便于接通知/监控
python3 rq_checkin.py --manual     # 人工输入验证码
```

环境变量同样可用：`RQ_USERNAME` / `RQ_PASSWORD` / `RQ_COOKIE`。

## 五、执行方式

**当前由 GitHub Actions 执行**（私有仓库 `Golden0Voyager/rq-checkin`），本机不再定时跑。

| 项 | 值 |
|---|---|
| 工作流 | `.github/workflows/rq-checkin.yml` |
| 定时 | 北京 08:10 主跑 / 20:10 兜底（UTC `10 0 * * *`、`10 12 * * *`） |
| 凭据 | Secret `RQ_COOKIE`，格式 `RQWEBSESSID=xxxx` |
| 验证码 | ddddocr 本地识别，单次最多重试 12 轮 |
| 保活 | WorkBuddy 自动化「RQrun 签到守护」每 8 小时巡检：工作流被 GitHub 自动停用会重新 enable，当天无成功运行会补一次 dispatch |

不依赖 Chrome 的跑法（与 Actions 一致）：

```bash
RQ_COOKIE="RQWEBSESSID=xxx" python3 rq_checkin.py --json
```

本机手动跑（自动读 Chrome 会话）：

```bash
python3 rq_checkin.py --from-chrome --json
```

退出码：`0` 正常（含今日已签）· `2` 无凭据 · `3` 会话失效 · `4` 签到失败 · `5` 触发风控。


## 六、文件说明

| 文件 | 作用 |
|---|---|
| `rq_checkin.py` | 主脚本（登录 + 签到） |
| `get_rq_cookie.py` | 从 Chrome 解密读取 RQ 会话 |
| `config.json` | 账号密码或 Cookie（权限 600） |
| `.rq_state.json` | 缓存的网页会话，避免每次重新登录 |
| `checkin.log` | 每次运行的日志 |

## 七、实测记录

- 登录图形验证码识别：15 次样本 15 次通过（ddddocr + 形近字符归一化）
- 完整签到链路：跑通。2026-10-26 实测签到页显示「今日已签到」「连续签到 2 天」，累计 9 天
- 会话有效期：约 30 天（本次 2026-11-25 到期）
- `--json` 输出：`{"ok":true,"status":10009,"message":"...","streak":2,"signed_today":true,"total_days":9,"nickname":"..."}`

## 八、风险提示

RQrun 服务条款里写了「通过作弊工具或非正常手段参与活动，官方有权取消资格、冻结已得权益」。
脚本只是把你手动点的那一下换成程序点，不伪造数据、不刷量、不操作他人账号，实际风险很低；
但如果介意，建议只用 `--manual` 半自动模式，或干脆继续手动签。**不要多账号批量跑。**
