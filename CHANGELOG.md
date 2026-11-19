# 修改日志（Fork）

本仓库是 [Tohi-GT7](https://github.com/masterzhang233/Tohi-GT7) **1.0.2**（提交
`25ad6e3fb7a2dcbf0c40c426f6c7ecc534a747de`）的 fork，仅增加方向盘力反馈持续性的修复，
其余代码与 1.0.2 保持一致。

## 2026-11-19 — `1.0.2-ffb1`：修复力反馈中途消失

### 问题现象

- 游戏：Gran Turismo 7，PPSA01317，app ver 01.710.000。
- 方向盘：SIMAGIC Alpha EVO Sport（`3670:0500`），在 `wheel.json` 中模拟为 Fanatec GT DD Pro。
- 比赛中力反馈正常，但在约 35–150 秒后**突然完全消失**；车辆仍可正常驾驶。
- 暂停（>2 秒）再继续后，力反馈立刻恢复，并可能再次消失。
- 模拟器日志显示游戏全程仍在以约 60 Hz 发送恒力更新（`01 08`），并非游戏停发。

### 根因

问题在宿主侧的 haptic 调用链，不在游戏、不在底座硬件：

- `SDL_UpdateHapticEffect()` 在 Windows/DirectInput 后端只调用
  `IDirectInputEffect_SetParameters()`，**不会重新启动效果**；只有
  `SDL_RunHapticEffect()` 会调用 `IDirectInputEffect_Start()`。
- 原实现只在效果首次创建或重建时 `SDL_RunHapticEffect()` 一次。设备端（底座固件或
  DirectInput 驱动）中途把效果停掉后，宿主不会再次 `Start`，因此力反馈永久消失。
- 暂停超过 2 秒会触发 `SDL_StopHapticEffects()` 并清空播放状态；恢复时宿主重建效果并
  重新 `Run`，所以暂停/继续可以临时恢复。

### 修复内容

只修改 `src/libs/wheelHost.cpp`：

- `SlotApplied` 增加 `last_run_ns` / `run_failed`，记录每个槽位上次启动时间。
- 在主循环中，对所有“应该正在播放”的设备效果，每 **1 秒**重新调用一次
  `SDL_RunHapticEffect()`。设备停了会被重新拉起；仍在播放的效果重复 `Start` 无副作用
  （GT7 发送的效果都是无限时长）。
- 宿主计算的恒力（`m_render_id`）同样每秒重新 `Run` 一次。
- `DestroySlot()` / `StopRender()` 清理对应状态。
- 重新启动失败只记录一次；恢复时记录一次，避免日志刷屏。
- 没有修改游戏文件，也没有修改 Tohi 的 `wheelFanatec.cpp`、`libPad.cpp` 等其他源码。

补丁统计：

```
 src/libs/wheelHost.cpp | 73 +++++++++++++++++++++++++++++++++++++++++++-------
 1 file changed, 64 insertions(+), 9 deletions(-)
```

### 验证结果

- 修复前：比赛中 35–150 秒力反馈消失；暂停/继续可恢复。
- 修复后：连续比赛 5 分钟以上力反馈持续存在，没有中途消失；中文显示正常
  （Noto Sans CJK SC 回退方案保持不变）。
- 已知现象：路面/胎噪带来的“细碎”路感在 1.0.2 原版中同样存在，与本修复无关。

### 构建

源码：Tohi-GT7 1.0.2，提交 `25ad6e3`。工具链：Clang-CL / MSVC、CMake、Ninja、Vulkan SDK。

```powershell
cmake -S <source> -B <build> -G Ninja `
      -DCMAKE_BUILD_TYPE=Release `
      -DCMAKE_C_COMPILER=clang-cl `
      -DCMAKE_CXX_COMPILER=clang-cl `
      -DKYTY_BUILD_LAUNCHER=OFF
cmake --build <build> --target kyty_emulator --parallel
```

构建需要 Tohi 仓库声明的子模块；Noto CJK 字体由 CMake 自动复制到输出目录的 `fonts/`。
运行时把 `fonts/NotoSansCJKsc-Regular.otf` 与 `kyty_emulator.exe` 放在同一目录结构中即可。

### 升级与回滚

- 升级：直接用本 fork 构建的 `kyty_emulator.exe` 替换原版即可，配置和存档不需要改动。
- 回滚：换回 Tohi 1.0.2 原版 `kyty_emulator.exe`。

### 备注

- 每秒重新启动意味着设备端停掉效果后，最坏情况下约有 1 秒的力反馈空档；如果希望更平滑，
  可以把 `ReassertIntervalNs` 从 `1'000'000'000` 调到 `250'000'000`（250 ms）后重新编译。
- 该修复只处理“设备端停止效果”这一类丢失；如果游戏自身停止发送力反馈，仍由原有的
  `StaleForceNs` 看门狗处理。
