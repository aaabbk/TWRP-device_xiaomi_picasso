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
   - `installKey()` 按探测到的版本装 key（探测不到才用 fstab 给的版本），不重复安装
   - v1 用 8 字节 descriptor、v2 由内核计算 identifier
3. `FsCrypt.cpp`
   - `get_data_file_encryption_options()` 解析完 fstab 后，用探测到的版本覆盖 `options.version`
4. `MetadataCrypt.cpp`
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
刷入后在 recovery 里查看 `/tmp/recovery.log`：

```
Detected fscrypt policy v2 on /data/system_de
Data is encrypted with fscrypt policy v2, overriding fstab version v1
Data is encrypted with fscrypt policy v2, overriding fstab version v1      (KeyUtil)
Installed fscrypt key with ref ...
```

`ls /data/system_de/0` 能看到 `spblob` 等明文名字即为解密成功。
