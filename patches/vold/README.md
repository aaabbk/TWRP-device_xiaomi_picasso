# vold 补丁：fscrypt policy v1/v2 双兼容

## 用途
同一台机器上可能同时存在两种 fscrypt 数据：

- 官方包 / 早期 ROM 格式化的数据：**policy v1**
- 移植包（Android 13+）格式化的数据：**policy v2**（可能还带 metadata 加密）

fscrypt 的 policy 版本在**格式化那一刻**固定，之后刷机、升级都不会重新加密。
所以 recovery 不能只看 fstab（fstab 只描述"新格式化会用什么"），必须按磁盘上实际策略装 key。

## 改动内容
1. `PolicyDetect.h`（新增）
   - 用 `FS_IOC_GET_ENCRYPTION_POLICY` 的返回码判断版本：
     `0` = v1，`EINVAL` = v2（内核拒绝把 v2 填进 v1 结构体），`ENODATA` = 未加密
   - 探测顺序 `/data/system_de` → `/data/misc` → `/data/media` → `/data/system_ce`
2. `KeyUtil.cpp`
   - `installKey()` 按探测到的版本装 key（探测不到才用 fstab 给的版本）
   - 正常情况下只装一次
   - 探测成功但安装失败 → **直接报错，不换版本**（磁盘策略才是准的，换版本只会掩盖真实问题）
   - 探测失败（只能靠 fstab 猜）且安装失败 → 才换另一种版本重试一次
   - v1 用 8 字节 descriptor、v2 由内核计算 identifier
3. `FsCrypt.cpp`
   - `get_data_file_encryption_options()` 解析完 fstab 后，用探测到的版本覆盖 `options.version`
4. `Decrypt.cpp`
   - 新增 `Resolve_Spblob_Id()`：Android 14+ 把合成密码状态文件改成按随机
     **protector id** 命名（`<id>.spblob/.pwd/.secdis`），不再用 `locksettings.db` 里的
     `sp-handle`。这里先试老 handle，不行就扫 `spblob/` 目录，优先选同时有
     `.pwd` 和 `.spblob` 的（LSKF 保护器）
   - `Get_Password_Type()` 与 `Decrypt_User_Synth_Pass()` 都改用解析出的 id，
     keystore 别名同源，避免 v1 ROM 与 A14+ ROM 的命名差异
   - 结果缓存到 `s_cached_spblob_id`：TWRP 中途会卸载/重挂 /data，第二次解析可能
     `locksettings.db` 读不到、目录 ENOENT，此时复用缓存而不是退化成空 handle
   - `Ensure_Data_Accessible()`：访问 spblob 前若 `/data/system_de` 不可见，就重新
     调一次 `Decrypt_DE()` 装回 DE key 再试
5. `MetadataCrypt.cpp`
   - `keydirectory` 不存在时（没有 metadata 加密的数据）直接挂载 /data 并返回成功，
     让 FBE 层继续走普通解密流程，避免官方包被 metadata 分支卡住

## 应用方式
在 TWRP 源码根目录执行：

```bash
cd system/vold
git apply /path/to/device/xiaomi/picasso/patches/vold/vold-policy-detect-header.patch
git apply /path/to/device/xiaomi/picasso/patches/vold/vold-fscrypt-v1v2.patch
```

仓库自带的 `.github/workflows/Build TWRP picasso.yml` 已经包含 "Apply vold patches" 步骤，
云编译无需手工操作。本地编译若跳过该步骤，补丁不会生效。

## 验证
补丁里的诊断日志**故意用 printf 而不是 LOG(INFO)**：recovery 里的 vold 走 logd，
`LOG(INFO)` 只会进 `adb logcat`，不会出现在 `/tmp/recovery.log` 里；printf 才能直接落盘。

刷入后在 recovery 里查看 `/tmp/recovery.log`，应能看到：

```
twrp_policy_detect: /data/system_de -> version 2
twrp_policy_detect: /data/misc -> version 0
twrp_install_key: mountpoint=/data fstab_version=1 wrapped=1
twrp_install_key: detected policy v2, overriding fstab v1
twrp_spblob: spblob_path=/data/system_de/0/spblob/ legacy_handle='a595d274077350b'
twrp_spblob: entry 0.handle
twrp_spblob: entry <protector id>.pwd
twrp_spblob: using protector id <protector id> (has .pwd and .spblob)
```

若只看到 `twrp_policy_detect` 全为 0，说明探测拿不到策略（DE 还没挂载或内核不支持该 ioctl）；
若完全没有 `twrp_install_key`，说明 key 安装流程根本没走到。

以下是正常解密成功的日志：

```
Detected fscrypt policy v2 on /data/system_de
Data is encrypted with fscrypt policy v2, overriding fstab version v1
Data is encrypted with fscrypt policy v2, overriding fstab version v1      (KeyUtil)
Installed fscrypt key with ref ...
```

`ls /data/system_de/0` 能看到 `spblob` 等明文名字即为解密成功。
