# 无障碍死锁闪退修复 / Accessibility Deadlock Fix

[English](#english) | [简体中文](#简体中文)

<a name="english"></a>
## English

This fork publishes a **patched build** of upstream `Keee7/OBS-for-Android` v0.2.1 that fixes
the crash occurring when an accessibility service is enabled.

* Symptom: with `accessibility_enabled=1`, opening *Settings*, *Add Source*, any source
  *Properties* dialog or the top-right menu aborts the app (`signal 6`).
* Root cause: `QtAndroidPrivate::AndroidDeadlockProtector` is a process-wide, non-reentrant
  flag. The accessibility bridge takes it on the Android UI thread and blocks on the Qt main
  thread; the Qt main thread meanwhile creates a new window surface and
  `QAndroidPlatformOpenGLWindow::eglSurface()` cannot acquire the same flag, so it calls
  `qFatal()` → `abort()`.
* Fix (qtbase 6.11.1, Android platform plugin only):
  1. `androidjniaccessibility.cpp::runInObjectContext()` — dispatch directly when already on
     the object's thread, otherwise post with `Qt::QueuedConnection` and wait at most 120 ms
     (`QSemaphore::tryAcquire`), dropping the query instead of blocking forever.
  2. `qandroidplatformopenglwindow.cpp::eglSurface()` — retry the protector for ~600 ms and
     create the surface afterwards instead of aborting; worst case is a warning plus an
     expose event that retries later.

**Download**: [`v0.2.1-a11y-deadlock-fix`](https://github.com/shiyidzyk/OBS-for-Android/releases/tag/v0.2.1-a11y-deadlock-fix)
(arm64-v8a, debug-signed — run `adb uninstall com.obsproject.studio` first).

**Sources**: branch [`fix/android-a11y-deadlock`](https://github.com/shiyidzyk/OBS-for-Android/tree/fix/android-a11y-deadlock/scripts/qt-android-a11y)
(patch + build helper + README). Upstream PR: [Keee7/OBS-for-Android#12](https://github.com/Keee7/OBS-for-Android/pull/12).

Verified on Android 15 (emulator, same reproduction as on a physical device) with two
accessibility services enabled: settings / add source / colour source / source properties /
menu all render, 22 open-close cycles plus 6 add-source cycles with no abort, no ANR, and no
`FATAL` in logcat. The only trade-off is that an accessibility query which cannot be answered
within 120 ms while a window is being created is dropped.

Once the upstream PR is merged, please use the official release instead of this fork build.

---

<a name="简体中文"></a>
## 简体中文

本 fork 发布了基于上游 `Keee7/OBS-for-Android` v0.2.1 的**修复构建**，用于解决开启无障碍服务后的闪退。

* 现象：`accessibility_enabled=1` 时，打开「设置」「添加源」、任意源「属性」对话框或右上角菜单都会闪退（`signal 6`）。
* 根因：`QtAndroidPrivate::AndroidDeadlockProtector` 是进程级、不可重入的标志。无障碍桥在 Android UI 线程先拿到它并阻塞等待 Qt 主线程；Qt 主线程此时要创建新窗口 Surface，`QAndroidPlatformOpenGLWindow::eglSurface()` 抢不到同一标志，直接 `qFatal()` → `abort()`。
* 修复（qtbase 6.11.1，仅 Android 平台插件）：
  1. `androidjniaccessibility.cpp::runInObjectContext()`：已在对象所属线程时直接执行；否则用 `Qt::QueuedConnection` 投递并最多等待 120 ms（`QSemaphore::tryAcquire`），超时丢弃该次查询而不是无限阻塞。
  2. `qandroidplatformopenglwindow.cpp::eglSurface()`：改为有界重试（约 600 ms）后再创建 Surface，不再 abort；最坏情况只打警告并补发 expose 事件留待后一帧重试。

**下载**：[`v0.2.1-a11y-deadlock-fix`](https://github.com/shiyidzyk/OBS-for-Android/releases/tag/v0.2.1-a11y-deadlock-fix)
（arm64-v8a，debug 签名 —— 安装前请先 `adb uninstall com.obsproject.studio`）。

**源码**：分支 [`fix/android-a11y-deadlock`](https://github.com/shiyidzyk/OBS-for-Android/tree/fix/android-a11y-deadlock/scripts/qt-android-a11y)
（补丁 + 构建脚本 + 说明）。上游 PR：[Keee7/OBS-for-Android#12](https://github.com/Keee7/OBS-for-Android/pull/12)。

已在 Android 15（模拟器，与真机复现一致）并同时启用两个无障碍服务的情况下验证：设置 / 添加源 /
颜色源 / 源属性 / 菜单均可正常渲染，连续 22 次开关设置 + 6 轮添加源无闪退、无 ANR，logcat 无 `FATAL`。
唯一代价是窗口创建瞬间 120 ms 内拿不到结果的无障碍查询会被丢弃。

上游 PR 合并后请改用官方 Release，而不再使用本 fork 构建。
