# Tailscale节点过期导致唯一的回家通道断开

**2026-07-19**

---

## 问题：
- 无法连上家里的服务
- 多个设备包括Tailscale Subnet Router显示已过期

---

## 目标
- 找出不会续期原因
- 如何避免相同的情况

---

## 过程
### 1. 进入Tailscale后台检查设置，查看服务器是否开启了Enble key expiry。
查看后发现是开着的，所以服务器凭证过期。理论上来说应该会自动续期。

### 2. 开启Tailscale 
```
sudo tailscale up --advertise-routes=192.168.50.0/24
``` 

### 3. 查看日志服务器凭证过期那段时间的日志
```
sudo journalctl -u tailscaled | grep -Ei "expired|login|reauth|auth|key|needs"（重点查看6月29号发生了什么）
```
过期时刻：
```
Jun 29 16:13:42 cy-server tailscaled[2036029]: Switching ipn state Running -> NeedsLogin (WantRunning=true, nm=true)
Jun 29 16:13:42 cy-server tailscaled[2036029]: magicsock: SetPrivateKey called (zeroed)
Jun 29 16:13:42 cy-server tailscaled[2036029]: magicsock: closing connection to derp-5 (zero-private-key), age 709h32m19s
Jun 29 16:13:42 cy-server tailscaled[2036029]: magicsock: ReSTUN("link-change-major") ignored; stopped, no private key
```

客户端尝试重新注册：
```
Jun 29 18:13:59 cy-server tailscaled[4105367]: control: RegisterReq: got response; nodeKeyExpired=true, machineAuthorized=true; authURL=false
```

客户端生成新的Node Key：
```
Jun 29 18:13:59 cy-server tailscaled[4105367]: control: Generating a new nodekey.
```

然后Tailscale Control Plane要求服务器重新登陆以获得加入Tailnet的资格：
Jun 29 18:13:59 cy-server tailscaled[4105367]: control: RegisterReq: got response; nodeKeyExpired=false, machineAuthorized=false; authURL=true
Jun 29 18:13:59 cy-server tailscaled[4105367]: control: AuthURL is https://login.tailscale.com/a/XXXXXXXXX

异常点：
```
Jun 29 20:33:58 cy-server tailscaled[4105367]: control: error decoding RegisterResponse with server key mkey:9e5156a4c65121306dd2d8ed8f92cb8d738e2533011344b522c5d28409bc4970 and machine key mkey:36e063784b74153fd3af2582719400c7c5776319556e112f215afb9a93e1d731: unexpected end of JSON input
Jun 29 20:33:58 cy-server tailscaled[4105367]: health(warnable=login-state): error: You are logged out. The last login error was: register request: unexpected end of JSON input
...
Jun 30 19:16:15 cy-server tailscaled[4105367]: health(warnable=login-state): error: You are logged out. The last login error was: register request: http 502: Bad gateway
```
`unexpected end of JSON input` 不是正常的认证失败日志。但是客户端期待收到一份完整的JSON响应，却实际收到的是截断的内容、空响应，或者一个 HTML 错误页。

而后期的 `http 502: Bad gateway` 证明Tailscale Control Plane在节点重新注册过程中返回异常响应（502/不完整JSON），导致重新注册失败。

## 事故推断
1. 6 月 29 日 Node Key 到期（这是预期行为）
2. 客户端发现 Node Key 已过期，尝试重新生成 Node Key来注册（也是预期行为）
3. Control Plane返回异常（502/不完整JSON），导致续期失败（这是异常）
4. 节点进入 NeedsLogin
5. 今天手动执行 tailscale up，重新完成注册

## 修复方案
1. 服务器开启Disable key expiry
2. 未来搭建第二个备用回家方案
3. 像Tailscale官方反馈这个问题
4. 定期检查cy-server这种关键节点是否在线


