# issue #245：WAN 重拨后 IPv6 流量统计失效

上游 issue：<https://github.com/timsaya/luci-app-bandix/issues/245>

## 现象

WAN 口掉线重拨后，bandix 不再统计 IPv6 流量，IPv4 一切正常；手动 `/etc/init.d/bandix restart` 后恢复。

## 根因

IPv6 的本地网段信息只在进程启动时写入一次 eBPF 映射，之后永不刷新。

1. `bandix/src/command.rs` 的 `run()` 启动时调用一次 `SubnetInfo::from_interface(options.iface())`。
2. 其内部通过 `get_interface_ipv6_info()`（`bandix/src/utils.rs`，执行 `ip -6 addr show <iface>` 解析）取接口的 IPv6 地址与前缀长度。
3. `create_module_contexts()` 把这些前缀写进 `IPV6_SUBNET_INFO` 映射后，持有映射的局部变量随即被 drop，此后再无任何写入点。
   对照之下，邻居表/设备列表有 30 秒的后台刷新任务，子网信息没有。
4. 内核侧 `bandix-ebpf/src/utils.rs` 的 `is_subnet_ipv6()` 逐条比对映射里的前缀；
   `modules/traffic/mod.rs` 的 `monitor_traffic_v6()` 只在 `src_is_local || dst_is_local` 为真时才累加计数。

WAN 重拨后 ISP 通过 DHCPv6-PD 下发的前缀可能变化，br-lan 拿到新的 GUA，而映射里仍是旧前缀，
于是新前缀的包源和目的都被判为「非本地」，整包从统计中丢弃。
LAN 的 IPv4 网段不随 WAN 变化，所以只有 IPv6 受影响——这正对应「只有 IPv6 挂掉、重启服务就恢复」。

触发条件是重拨后前缀确实发生了变化。若 ISP 每次下发同一前缀则不会复现，这解释了为何不是所有人都遇到。

## 修复

分支 `fix/ipv6-subnet-refresh`，只改 `bandix/src/command.rs`：

- 把原先内联的映射写入逻辑抽成 `write_ipv6_subnets()`；
- 保留 `IPV6_SUBNET_INFO` 的映射句柄，交给一个 30 秒周期的后台任务，重新读取接口 IPv6 地址，
  仅在地址集合真正变化时才重写映射（避免每轮无谓的系统调用与日志）；
- 写入顺序改为「先写新条目、再清理多余槽位」，避免刷新瞬间出现所有网段都未配置的空窗。

未改动 eBPF 侧、`build_release.sh`、`install_dependencies.sh`。
`DeviceManager` 持有的 `SubnetInfo` 同样是启动时快照，但它只用到其中的 MAC 与 IPv4 字段，不受此问题影响，故未改动。

## 验证

1. 用 `dist/bandix` 替换设备上的 `/usr/bin/bandix`（先备份原文件），`/etc/init.d/bandix restart`。
2. 确认 IPv6 流量正常计数。
3. 手动断开并重拨 WAN，**不要重启 bandix**，等 30 秒以上。
4. `logread | grep -i "IPv6 addresses of"`：前缀变化时应出现
   `IPv6 addresses of br-lan changed, reconfiguring subnet info maps...`，
   随后是新的 `Configured IPv6 subnet N: .../64 ...` 若干行。
5. 观察 IPv6 流量是否在无需重启的情况下恢复计数。

若前缀重拨后未变化，第 4 步不会有日志输出，这属于预期行为（此时原版也不会出问题）。

## 回退

把备份的原 `bandix` 二进制放回 `/usr/bin/bandix` 并重启服务即可。

## 构建

`.github/workflows/build-x86_64.yml` 只编译 `x86_64-unknown-linux-musl` 单架构并上传 artifact，
用于快速出验证包；上游原有的 `build bandix.yml` 发版流程未改动。
需要其他架构时，在 fork 上手动触发该 workflow 并调整 target，或本地执行 `./build_release.sh`。
