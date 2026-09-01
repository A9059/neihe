# 云端编译 6.1 GKI 内核（无痕教程）

这个仓库通过 GitHub Actions 在**云端自动编译**一个 Android 14 的 6.1 GKI 内核（aarch64），
用的就是 **Google 官方内核源码 + Google 官方编译脚本**。你不需要在本机装任何 Linux。

---

### 你需要做的（只需 3 步）

1. **登录 GitHub**（你已有账号）
2. 点网页右上角 `+` → **New repository**：
   - Repository name：随便，比如 `gki-build`
   - 选 **Public 或 Private 都可以**
   - 勾选 **"Add a README file"**
   - 点 **Create repository**
3. 把本目录下这些文件**上传**到仓库根目录：
   - `.github/workflows/build-kernel.yml`
   - `README.md`

   **上传方式**：仓库主页 → **Add file** → **Upload files** →
   把你电脑上的 `.github` 文件夹和 `README.md` 拖进去（文件夹结构要保留：`.github/workflows/build-kernel.yml`）。

---

### 触发编译

上传完会自动触发（因为 push 到 main）。也可以手动触发并**控制是否关闭模块签名**：

1. 仓库页 → **Actions** → 左侧 **Build 6.1 GKI Kernel (aarch64)**
2. 点 **Run workflow** → 会看到一个复选框：
   - `Disable CONFIG_MODULE_SIG / FORCE (demo cheat vendor style)`
   - **勾上它**再点绿色按钮 = 复刻外挂厂商"关闭模块强制签名+改版本号"的操作
   - **不勾** = 编译原版 GKI（模块签名开启，LOCALVERSION 保持默认）
3. 点 **Run workflow** 即可开始

---

### 看结果

- Actions 页里点进去能看到**完整编译日志**（相当于真实的内核编译输出）
- 编译完成后产物作为 **Artifact**：网页右上角导航栏
  `Actions` → 对应 build 的 Summary 页 → **Artifacts** → 下载 `gki-6.1-kernel.zip`（或 `gki-6.1-kernel-no-sig.zip`）
- 产物里包含：
  - `Image`（未压缩内核）、`Image.gz`（gzip 压缩）、`Image.lz4`（LZ4 压缩，**就是你手机 boot 里那种**）
  - `modules/`（可用的内核模块）
  - `vmlinux`（带符号的完整内核，**可直接丢进 IDA/Ghidra 做静态分析**）
  - `System.map`、`Module.symvers`（符号表，调试/开发驱动用）
- 日志里会打印 **`Linux version ...`** 完整版本串（就是 `/proc/version` 里显示的），以及 **vermagic**（模块加载校验用的魔数）

---

### 说明

- 这个产物是 **Google 官方 GKI 通用内核**（android14-6.1 分支），用的是默认签名配置。
- 它**不能直接刷入**天玑9300手机——刷入还需要厂商 vendor 设备树/私有驱动（之前分析过）。
  本教程的目标是 **让你亲手走通"从源码到内核"的完整编译流程**，理解内核是怎么出来的。
- 想进一步实验"关闭模块签名/改 LOCALVERSION"：
  - 直接在 Actions 页面勾选 `disable_module_sig` 再跑一次
  - 或者手动编辑 `build-kernel.yml`，在 build 步骤前加两行（原理同上）：
    ```bash
    echo 'CONFIG_MODULE_SIG=n' >> common/arch/arm64/configs/gki_defconfig
    echo 'CONFIG_MODULE_SIG_FORCE=n' >> common/arch/arm64/configs/gki_defconfig
    echo 'CONFIG_LOCALVERSION="-mybuild"' >> common/arch/arm64/configs/gki_defconfig
    ```
  - 改完重新 Run workflow 就能看到版本后缀变成 `-mybuild`——这正是分析过的魔改内核改版本名的手法。

---

### 常见问题

| 现象 | 原因/解决 |
|---|---|
| 编译超时（>180 min） | 首次下载 pinned clang ~1.5GB，GitHub 网络波动；重新 Run workflow 即可 |
| 产物里没 `vmlinux` | GKI 默认可能不产出 vmlinux，用 `strings Image` 也能看到版本串 |
| 想用别的分支（如 android13-6.1） | 修改 workflow 里两处 `android14-6.1` 为目标分支名 |

---

### 关键知识点对照（复习用）

| 你在本地分析到的 | 本编译里对应位置 |
|---|---|
| `6.1.145-android14-11-maybe-dirty` | `android14-6.1` 分支最新 HEAD 的版本号 |
| `LOCALVERSION="-@Coolpak@..."` | `CONFIG_LOCALVERSION`，workflow 里可改 |
| `CONFIG_MODULE_SIG_FORCE` | `gki_defconfig` 里默认 `y`，勾选 input 会改成 `n` |
| `clang 17.0.2 (10087095...)` | `build/build.sh` 自动下载的 pinned toolchain |
| `vmlinux` 里的版本串 | `strings vmlinux \| grep 'Linux version'` |

---

生成的 zip 在 `C:\Users\Administrator\AppData\Local\Temp\opencode\GKI-Build.zip`，解压后把里面的 `.github` 和 `README.md` 上传即可。
