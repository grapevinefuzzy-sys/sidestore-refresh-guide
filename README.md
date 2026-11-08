# SideStore Refresh Repair

An error-driven troubleshooting runbook and installable agent skill for restoring SideStore refresh on modern iOS, with special coverage for iOS 26.4+ Remote Pairing.

It routes each visible symptom to the matching branch instead of repeatedly reinstalling apps or revoking certificates. The arrows below are diagnostic layers, not an instruction to perform every step:

```text
expired SideStore / WDA
  → certificate ownership (only when explicitly reported)
  → pure Remote Pairing
  → LocalDevVPN endpoint
  → current dynamic _remotepairing._tcp port
  → scoped app refresh
  → WDA refresh (only when requested)
```

## Error labels covered

- `WebDriverAgentRunner is no longer available`
- `Operation Failed`
- `The current pairing file is invalid or missing`
- `.rppairing UDID not found`
- `Pairing Required: No valid pairing file has been loaded`
- `DeviceEndpointNotInitialized`
- `Remote endpoint 10.7.0.1 is configured, but TCP port poll failed`
- stale or cached `RemotePair Port`
- LocalDevVPN routing failures
- SideStore certificate/private-key ownership warnings
- Apple sign-in error `-80045`

## The short version

1. Identify which app expired. If SideStore still opens and only WDA expired, refresh only WDA. Use iLoader only to bootstrap an expired SideStore while preserving its data, team, and bundle identity.
2. Enter the certificate branch only if SideStore explicitly reports that it lacks the matching private key.
3. On iOS 26.4+, generate a **pure Remote Pairing / RPPairing** file for the current device only when SideStore reports a missing, invalid, or unusable pairing file. Do not reset working pairing merely to make it fresh, and do not fall back to Lockdown merely because it produces a larger file.
4. When that branch is required, validate the replacement locally before resetting SideStore's stored pairing, then import it, set Remote Pairing as preferred, and verify that `rppairing` is active.
5. Only if the pairing file is active and the endpoint TCP poll fails, discover `_remotepairing._tcp.local` in SideStore's Network Discovery and copy its **current SRV port** into Connection Config.
6. Only when the active LocalDevVPN profile exposes the current phone at `10.7.0.1`, keep that Device IP, turn SideStore's `Use Local VPN` off for the manual configuration, and require `Reachable: Yes`. Stop if multiple devices or profiles make the target ambiguous. Inspect and record the prior VPN state before connecting LocalDevVPN.
7. Refresh only the apps the user placed in scope. Use **Refresh All** only when every listed app is intended, then refresh WDA when WDA is in scope.

The Remote Pairing port is dynamic. `49152` is a common default, not a value to hardcode.

## Important safety rules

- Never request, upload, inspect, or commit pairing files. They contain private cryptographic material; transfer them only between trusted local tools.
- Never publish a UDID, HostID, Wi-Fi MAC, Apple ID, password, verification code, certificate private key, `.p12`, or provisioning profile.
- Never revoke certificates in bulk. Revocation is irreversible and can disable other signed apps.
- Never delete SideStore, WDA, or their data as a routine troubleshooting step.
- Stop before resetting iPhone trust or network settings unless the user explicitly authorizes that exact action.
- The separate **Wireless Pairing / Advertising server** page is not Network Discovery; its temporary port is not the iPhone Remote Pairing service port.

These privacy rules are hard stops: user approval does not make raw pairing material, private keys, passwords, codes, or unredacted identifiers safe to publish. The bundled skill also prevents an agent from expanding a narrow refresh failure into destructive account or device changes.

## Repository layout

```text
SKILL.md                  Agent workflow and safety boundaries
agents/openai.yaml        Skill display metadata
references/error-map.md   Exact error-to-action table and primary sources
```

## 中文速览

这个仓库把 SideStore 刷新失败拆成四层：证书所有权、Remote Pairing、VPN 路由、动态端口。每出现一个新错误，只处理对应的一层，不重新撤销证书、重装 App 或重置手机。

最容易漏掉的修复是：当 `rppairing` 已经生效，却出现 `TCP port poll failed` 时，前往 SideStore 的 **Network Discovery** 查找 `_remotepairing._tcp.local` 当前端口，再写入 **Connection Config**。这个端口会变化，不能长期写死为 `49152`。

## Evidence and scope

The Remote Pairing and port-discovery steps were tested on iOS 26. Menu labels and VPN routes can vary by release. Check the settings shown on the device before editing them.

Primary material is linked in [`references/error-map.md`](references/error-map.md), including SideStore's official docs and the separate GitHub issues that document pure Remote Pairing, LocalDevVPN routing, and stale-port rediscovery.

This project is not affiliated with Apple, SideStore, iLoader, LocalDevVPN, or `idevice_pair`.

## License

MIT
