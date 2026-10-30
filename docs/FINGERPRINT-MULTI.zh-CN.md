# 一个指纹库装多根手指（实验性）

> **状态：实验性，未随组件发布。** 下面的行为是在 liuqin 上使用本地修改的 FPC1264 TOD 驱动、录入 provider 和 FP3→裸数据库提取工具测出来的。这些改动不在本仓库发布的组件里，发布组件仍然拒绝录入第二根手指。本文记录方案和实测结果，供后续工作参考。

FPC1264 的模板存在 TEE 里的一个不透明数据库中，而不是 fprintd 写出的文件里。这个数据库有 5 个槽位，TA 支持在已有库上追加手指。问题出在上层 libfprint/fprintd：它默认"一根手指一个模板文件"。

## 为什么一根手指一个文件不行

fprintd 1.94.5 的 `device.c: verify_start()` 在设备不支持 `IDENTIFY` 时只取 `gallery[0]`，重试也只重试这一份模板。于是存在两个模板文件时，验证永远从第一根手指开始，第二根无论按多少次都不会被匹配。

另一条路是让驱动实现 `IDENTIFY`，但 `fpi_device_identify_report()` 每个动作只允许上报一次，驱动无法简单地"逐个库尝试，再上报匹配的那个"。

## 一个库 + 改写手指号

验证过的方案只保留一个模板文件：

```text
录入新手指
  -> TOD 驱动采集样本
  -> provider 从已存储的 FpPrint 中抽出裸 OEM 数据库
  -> oem_runtime --enrol-existing <raw-db> 在 TA 里追加一个槽位
  -> 驱动把返回的手指号改写回库里原来的手指号（2）
  -> fprintd 认为还是同一个 print，覆盖同一个模板文件
```

这样 fprintd 的 gallery 始终只有 1 项，`verify_start()` 拿到的就是包含所有手指的数据库。匹配仍由 TA 完成，主机侧拿不到可比较的模板。

与发布组件不同的三处：

| 部件 | 改动 |
| --- | --- |
| TOD 驱动 | 追加成功后用 `fp_print_set_finger()` 把手指号改回已存储的值，保持 print 身份不变 |
| 录入 provider | 走 `--bio-pipeline-existing` / `--enrol-existing`，而不是单指路径；已有模板时不再直接拒绝 |
| 提取工具 | 把 fprintd 存储的 FP3 容器还原成 `fpi-data` 裸数据库再交给 TA；TA 不接受 FP3 容器本身，会报 `db_import_remaining_mismatch` |

最后一点是第一次失败的原因：把 772,578 字节的 FpPrint 文件直接喂给 TA，最后一片 1458 字节返回 `0xe853009f`；换成 772,495 字节的裸数据库就通过了。

## 录入第二根手指

装好上述部件后，用户流程仍然在「设置」里：

1. 先关掉已打开的「设置」窗口，再重新打开 **设置 → 系统 → 用户 → 指纹登录**。
2. 用正常的密码对话框授权，按提示录入新手指，按一下抬一下，约 20 次。
3. 页面上仍然只有一行 `left-index-finger`。这是正常的：新手指被追加进同一个 print，存储的手指号被改写了。
4. 锁屏后用两根手指分别解锁。

第二根录入成功后，模板文件从 772,578 字节增长到约 1,162,014 字节。journal 里能看到追加和规范化的手指号：

```text
OEM enroll database extended under stored finger 2 (requested finger 7)
native_enrol=OK database=extended canonical_finger=2
```

## 限制

- **5 个槽位。** TA 分配 5 个指纹槽，第 6 次追加返回 `0xe56004c4`。同一根手指重复录会白占槽位。
- **设置里只有一行。** fprintd 只知道一个 print，界面固定显示最初那根手指的名字，所有手指共用这一行。
- **删除就是整库删除。** 删掉这一行会把追加过的所有手指一起删掉，不能单独删某一根。这比留下孤儿模板干净，但不方便。
- **每次成功解锁模板都会变。** OEM fprintd 补丁会做自适应更新，不能用文件哈希判断是否被篡改或重新录入。
- **设置页会占用设备。** 打开指纹登录页面时 `gnome-control-center` 会长期 claim 指纹设备，`sudo` 和锁屏可能抢不到而回落到密码，关掉设置页再试。

## 回滚

恢复发布版的单指组件时也要恢复对应的模板备份；实测步骤：

```sh
systemctl stop fprintd
tar xzf /root/fpc-backup-2026-10-30-pre-round2/liuqin-fpc-oem-101725.tar.gz -C /usr/local/lib
systemctl start fprintd
```

实验前先备份 `/var/lib/fprint` 和 `/usr/local/lib/liuqin-fpc-oem`。要删掉追加的手指，只能整库重新录入。

## 来源

逆向和修改后的组件是为这台平板开发的。TA 协议部分基于本仓库的 OEM 组件，不分发任何原厂二进制。两根手指的实测结果来自 10-27 的构建（`build=20261027-round2 database=per-account extract=1`）。
