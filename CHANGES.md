# Aerial 守护误拉回修复

基于 #1068（极米 Z9X / Android 13 / standard + 守护）的根因定位，修复 `HomeGuardService.kt` 两处缺陷。

## 变更 1：第 579 行 - 去掉 `!usageStatsAllowed(this)` 条件

**原代码**：
```kotlin
if (revived && !diedInFront && !GuardSetup.wasHomeInFront(this) && !usageStatsAllowed(this) &&
    SystemClock.elapsedRealtime() >= BOOT_WINDOW_MS && !fastBootTakeOver
) {
```

**新代码**：
```kotlin
if (revived && !diedInFront && !GuardSetup.wasHomeInFront(this) &&
    SystemClock.elapsedRealtime() >= BOOT_WINDOW_MS && !fastBootTakeOver
) {
```

**原因**：原条件要求 `!usageStatsAllowed(this)`，导致**授权了用量统计的设备反而失去保护**。同机型对照（#845 那台 `usage=false` 保护生效，本机 `usage=true` 保护失效）直接证实了这一点。

现在逻辑：无论是否授权，只要是 `revived`（被杀后复活）且不在前台死亡，都直接返回，不拉回 Aerial。`frontByUsage()` 返回 null（unknown）时，走到第 689 行的新逻辑。

## 变更 2：第 689 行 - 反转 `stockInFront` 的默认值

**原代码**：
```kotlin
val stockInFront = last == null || last.first in stockHomes
```

**新代码**：
```kotlin
// An unknown front is not the stock launcher: Aerial was killed in the background, so its
// record of what was in front died with it. Only bring Home back where usage stats
// positively name a stock launcher.
val stockInFront = last != null && last.first in stockHomes
```

**原因**：`last == null`（unknown）被直接判成「原桌面在前台」。现在改为：**只有明确看到原桌面在前台（`last != null && last.first in stockHomes`）才拉回**。unknown 时不动作。

代价不对等分析：
- unknown 时拉回：把人从视频里拽出来（严重体验损失）
- unknown 时不拉回：用户再按一次主页键即可（微小不便）

## 变更 3：第 671-683 行 - 增强 unknown 时的诊断日志

在 `frontByUsage()` 返回 null 时，记录具体原因（用 fork 的 `foreground(UNKNOWN_FRONT_LOOKBACK_MS)` 实现）：
- `usage stats not allowed`：未授权
- `usage stats: no MOVE_TO_FOREGROUND in 21600s`：窗口内无前台事件
- `usage stats: last front <pkg>/<cls> at <time>`：查到了但被过滤（包名==Aerial 或其它）

**重要性**：本机拿不到 logcat（极米 GMUI 限制），诊断里的 DiagLog 段是唯一证据源。这行日志能让下一次定位直接知道是「没授权」还是「授权了但查不到」。

## 未做的变更

**`android:canRetrieveWindowContent="true"`** —— 故意不改。
- 原因 1：守护是过滤按键的服务，dumpsys 显示 `capabilities=8`（仅过滤），极米语音 `capabilities=9`（读窗口+过滤）。改了也拿不到视频界面的无障碍树（SurfaceView 基本是空的）。
- 原因 2：用户体验成本高（弹「可读取屏幕内容」重权限警告，已开守护用户需重新授权）。
- 原因 3：想保护的「看视频」场景，正是无障碍最难覆盖的。核心矛盾解决靠第 1、2 条逻辑修复。

## 预期效果

| 场景 | 修复前 | 修复后 |
|---|---|---|
| `usage=true` + revived + frontByUsage=null | 拉回 Aerial（误） | **不拉回** |
| `usage=false` + revived + frontByUsage=null | 不拉回（原保护） | **不拉回** |
| `usage=true` + revived + frontByUsage=stock | 拉回（正确） | **拉回** |
| `usage=true` + revived + frontByUsage=app | 拉回（误） | **不拉回** |

## 关联

- #845：同机型同现象（XGIMI MT9660 / Android 13 / fongmi），但那台 `usage=false` 保护生效
- #1068：本 issue，同机型，`usage=true` 暴露了保护分支的覆盖漏洞
