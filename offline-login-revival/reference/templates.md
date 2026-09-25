# 模板与骨架

按语言无关的伪代码 / 结构给出，落地时替换为项目所用技术栈。所有示例都遵循"版本门控 + 日志 + 可回退"。

## 1. 静态侦察线索工作表

开工前用一张表把 P0 的答案固定下来：

```markdown
## 客户端
- 主程序 / 引擎 / 架构：
- 主模块 SHA-256：
- 文件是否完整？热更缓存位置：

## 三类线索
- 配置表：格式____ 逆回 Excel 方案____ 是否有服务端专用表____
- 协议表：定义位置____ 协议号形式(数字/名字)____ 传输层(wire format)____
- 热更代码：方案(Lua/TS/ILRuntime/…)____ 位置____

## 登录架构
- 直连 / 中台(SDK)：____
- 登录链条（每跳：触发→C2S→S2C→下一步）：
- 服务发现来源（配置 / 下发列表）：____  → 重定向锚点

## 传输栈
- 首包特征：____  TLS？长连接？UDP？
- 域名清单：____
```

## 2. 注入器骨架（x86 示例）

```cpp
// 1) 挂起启动目标
CreateProcessW(exe, cmd, ..., CREATE_SUSPENDED, ..., &pi);

// 2) 等目标模块加载完成（保证在目标初始化前拿到控制权）
for (5000 次) {
  if (已在目标进程中找到 "GameAssembly.dll" / 主模块) break;
  Sleep(1);
}

// 3) 版本门控：校验目标关键模块 SHA-256，不符立即终止
if (HashFile(mainModule) != expectedHash) { TerminateProcess; return; }

// 4) 远程加载 Payload
remotePath = VirtualAllocEx(pid, ..., PAGE_READWRITE);
WriteProcessMemory(pid, remotePath, payloadPath, ...);
hThread = CreateRemoteThread(pid, LoadLibraryW, remotePath);
WaitForSingleObject(hThread, 10000);

// 5) 等 Payload 自报就绪的命名事件（Payload 初始化完 SetEvent）
WaitForSingleObject(readyEvent, 10000);
```

> 若目标对 LoadLibrary 注入有防护，可考虑代理 DLL（如劫持 `xinput`/系统 DLL）等静态注入途径；核心不变：版本门控 + 初始化前介入。

## 3. Hook 骨架（inline 5 字节跳转 + 门控）

```cpp
bool InstallInlineHook(module, rva, trampoline, stolenLen, name) {
  if (HashFile(module) != expectedHash) { Log(name + " refused: hash mismatch"); return false; }
  addr = base + rva;
  if (memcmp(addr, expectedPrologue, stolenLen) != 0) { Log("refused: signature mismatch"); return false; }

  stolen = VirtualAlloc(...);            // 保存原始字节 + 末尾跳回
  memcpy(stolen, addr, stolenLen);
  AppendJumpBack(stolen, addr + stolenLen);

  WriteJump(addr, trampoline, stolenLen); // E9 rel32 + NOP 填充
  FlushInstructionCache(...);
  Log(name + " hook applied");
  return true;
}
```

要点：
- **IAT patch** 适合导入表调用（如 `curl_easy_perform`、`getaddrinfo`）：改 IAT 槽即可，拿到原函数指针；
- **Inline hook** 适合内部函数/导出（如 SDK 登录）：需处理被盗指令与跳回；
- **GetProcAddress 拦截** 可在模块动态取符号时替换（拿外部回调尤其有用）；
- 每个 hook 必须能"拒绝并保持客户端不变"。

## 4. 门控配置模板（bootstrap.ini）

```ini
[redirect]
enabled=1
port=<游戏/长连接端口>
http_port=<HTTP(S) 引导端口>
capture_bugly=0
capture_port=9887

[trust]
certificate=<本地 CA/叶证书路径>
allow_untrusted=1

[sdk]
bypass=1

[debug]
diagnostics=1
```

注入脚本按真实运行参数覆写端口，Payload 启动时读取。

## 5. 服务端伪造响应模板（引导 HTTP）

字段类型（字符串 `"0"` vs 数字 `0`）必须与客户端解析方式一致，否则走未知分支。

```jsonc
// 公网 IP 探测：返回假 IP，让 SDK 认为网络可用
"203.0.113.1"

// 版本检查：与本地 assetmap 版本一致 => 无需下载
{ "errornu": "0", ..., "tar_version": "<本地版本>", ... }

// 服务器列表：关键 —— host/port 指向本地
{ "errornu": "0", "errordesc": "",
  "root": {
    "notice": { "open": 0, "desc": "" },
    "item": [ { "name": "...", "serverIndex": 1, "groupid": "1",
                "status": 1, "host": "127.0.0.1", "port": <本地端口>,
                "recommend_weight": 1 } ]
  } }

// 会话票据：给出 host/port
{ "errornu": "0", "pid": "<本地账号>", "feignRoleId": "1",
  "host": "127.0.0.1", "port": <本地端口> }
```

## 6. 协议表模板

把抓包 / 反编译得到的协议整理成可检索的表。**这是读/写数据包的唯一依据。**

| 协议号 / 名 | 方向 | 触发条件 | 字段（号:类型） | 嵌套 | 证据级别 | 备注 |
|---|---|---|---|---|---|---|
| `player.Login` | C2S | 建连后 | 1:string Pid, 2:int Timestamp, 5:msg SampleInfo | SampleInfo | 字节级 | 走自研帧+protobuf |
| `player.GetUserList` | C2S | Login 应答 ok 后 | — | — | 运行时验证 | |
| `TRetLogin` | S2C | 应答 player.Login | 1:string Ret, 2:string FeignRoleId | — | 字节级 | |
| ... | ... | ... | ... | ... | ... | ... |

**证据级别**：`推断` < `静态确认` < `运行时验证` < `字节级/进程级测试`。

## 7. 登录握手逐跳记录表

| # | 触发条件 | C2S 消息 | S2C 响应/推送 | 客户端 handler | 证据级别 | 备注 |
|---|---|---|---|---|---|---|
| 1 | SDK 登录成功回调 | — | `root.item[]` | 列表解析 | 运行时确认 | host/port 本地化 |
| 2 | 选服后进入 | `player.Login` | `TRetLogin` | 收到 Ret=ok | 字节级 | 应用层头 + protobuf |
| ... | ... | ... | ... | ... | ... | ... |

## 8. 时序契约记录模板

```
触发初始化的请求：user.UserLogin
  响应的 handler：  _ReceiveUserLogin -> SendLuaEvent(LoginOk)
初始化的消费方：    LoginLogic:_LoginOk (读 m_TypeNumMap)
                    LoginStage:_LoginOk -> guideManager:init() (读 GUIDE_DONE_STAGES)
=> 契约：user.UpdateUserInfo / guide.GuideInfo 必须在 user.UserLogin 应答之前推送
回归验证：<测试名>
```

## 9. 一键闭环脚本结构（伪脚本）

```
1. [可选] 构建 native payload / 服务端
2. 清理残留进程与端口
3. 生成 TLS 材料（本地 CA + 叶证书）
4. 启动服务端（HTTP 引导 + 游戏登录端点），等待 ready JSON
5. 启动 TLS 代理（若走代理方案），等待 ready
6. 写 bootstrap.ini（端口/证书/开关）并注入客户端
7. 跟踪 Payload 日志，退出时清理所有子进程
```

## 10. 版本适配器结构

```jsonc
{
  "clients": [
    {
      "id": "jp-1.4.0",
      "mainModuleHash": "<sha256>",
      "hooks": {
        "sdkLogin":    { "module": "new_sdk.dll",      "rva": "0x3A850",  "sig": "A1" },
        "sdkManager":  { "module": "GameAssembly.dll", "rva": "0x2D1870", "sig": "80 3D" },
        "unityTls":    { "module": "UnityPlayer.dll",  "rva": "0x8E1573", "sig": "8B 47 34 5F 5E 5D C3" }
      },
      "messages": { "login": "player.Login", "userLogin": "user.UserLogin" }
    }
  ]
}
```

业务代码只读适配器，不含 `if (region == ...)` 之类的判断。

## 11. 配置表工具链约定

- 明确"打包格式 ⇄ Excel ⇄ 强类型类"三向转换脚本，做到可逆、可重复；
- 配置加密常见为可逆变换（如 XOR），确认后写进工具链文档；
- 服务端需要但客户端没有的配置（服务端专用表），单独维护并注明来源（自填 / 沿用客户端）；
- 配置目录由工具可重复生成，不手写常量。
