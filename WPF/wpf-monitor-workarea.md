# WPF 多屏幕工作区域获取方案（含混合 DPI）

## 一、核心结论

| 场景 | 推荐方案 |
|------|----------|
| 单屏，简单居中/限制尺寸 | `SystemParameters.WorkArea` |
| 多屏，需按窗口所在屏幕处理 | `MonitorFromWindow` + `GetMonitorInfo` |
| 多屏，按光标所在屏幕处理 | `MonitorFromPoint` + `GetMonitorInfo` |
| **多屏 + 混合 DPI，把窗口铺满某屏** | **`SetWindowPos` 直接传物理像素（最稳）** |

> ⚠️ **关键坑**：WPF 的逻辑单位 ≠ 物理像素。用 `Screen` 或 P/Invoke 拿到的是**物理像素**，直接赋给 `Window.Left/Top/Width/Height` 在高 DPI 下会偏大。多屏且各屏 DPI 不同时，必须用**目标显示器自身的 DPI** 换算，而不是窗口当前所在屏幕的 DPI。

---

## 二、核心概念区分

| 属性 | 含义 | 是否含任务栏 | 坐标单位 |
|------|------|-------------|----------|
| `SystemParameters.WorkArea` | 主屏工作区域 | ❌ 不含 | 逻辑单位（DIP） |
| `SystemParameters.PrimaryScreenWidth/Height` | 主屏全屏尺寸 | ✅ 含 | 逻辑单位 |
| `SystemParameters.VirtualScreenWidth/Height` | 所有屏幕合并尺寸 | ✅ 含 | 逻辑单位 |
| `Screen.WorkingArea`（WinForms） | 指定屏幕工作区域 | ❌ 不含 | 物理像素 |
| `GetMonitorInfo` 的 `rcWork` | 指定显示器工作区域 | ❌ 不含 | 物理像素 |
| `GetMonitorInfo` 的 `rcMonitor` | 指定显示器全屏区域 | ✅ 含 | 物理像素 |

---

## 三、基础方案：`SystemParameters.WorkArea`

```csharp
using System.Windows;

// 工作区域（逻辑单位，不含任务栏）
Rect workArea = SystemParameters.WorkArea;

double left   = workArea.Left;
double top    = workArea.Top;
double width  = workArea.Width;
double height = workArea.Height;

// 窗口在工作区域居中
this.Left = workArea.Left + (workArea.Width  - this.Width)  / 2;
this.Top  = workArea.Top  + (workArea.Height - this.Height) / 2;
```

**局限**：只返回**主屏**工作区，多屏场景不适用。

---

## 四、WinForms 方案（多屏，需 DPI 换算）

```csharp
using System.Windows;
using WinForms = System.Windows.Forms;   // 避免命名冲突

// 当前窗口所在屏幕
var handle = new System.Windows.Interop.WindowInteropHelper(this).Handle;
var screen = WinForms.Screen.FromHandle(handle);

System.Drawing.Rectangle workArea = screen.WorkingArea; // 物理像素

// 物理像素 → 逻辑单位
var source = PresentationSource.FromVisual(this);
double dpiX = source?.CompositionTarget?.TransformToDevice.M11 ?? 1.0;
double dpiY = source?.CompositionTarget?.TransformToDevice.M22 ?? 1.0;

double left   = workArea.Left   / dpiX;
double top    = workArea.Top    / dpiY;
double width  = workArea.Width  / dpiX;
double height = workArea.Height / dpiY;
```

**局限**：DPI 用的是**窗口当前所在屏幕**的缩放，跨屏查询会偏差。

---

## 五、纯 WPF + P/Invoke 方案（按目标显示器 DPI 换算）

### 5.1 完整封装

```csharp
using System;
using System.Runtime.InteropServices;
using System.Windows;
using System.Windows.Interop;

public static class MonitorHelper
{
    [DllImport("user32.dll")]
    private static extern IntPtr MonitorFromWindow(IntPtr hwnd, uint dwFlags);

    [DllImport("user32.dll")]
    private static extern IntPtr MonitorFromPoint(POINT pt, uint dwFlags);

    [DllImport("user32.dll")]
    private static extern bool GetMonitorInfo(IntPtr hMonitor, ref MONITORINFO lpmi);

    [DllImport("user32.dll")]
    private static extern bool GetCursorPos(out POINT pt);

    // 获取指定显示器的 DPI（Win8.1+）
    [DllImport("shcore.dll")]
    private static extern int GetDpiForMonitor(IntPtr hmonitor, int dpiType,
        out uint dpiX, out uint dpiY);

    private const uint MONITOR_DEFAULTTONEAREST = 0x00000002;
    private const int  MDT_EFFECTIVE_DPI = 0;

    [StructLayout(LayoutKind.Sequential)]
    private struct POINT { public int X, Y; }

    [StructLayout(LayoutKind.Sequential)]
    private struct RECT { public int Left, Top, Right, Bottom; }

    [StructLayout(LayoutKind.Sequential)]
    private struct MONITORINFO
    {
        public int cbSize;
        public RECT rcMonitor;
        public RECT rcWork;
        public uint dwFlags;
    }

    /// <summary>
    /// 获取窗口所在显示器的工作区域（WPF 逻辑单位，按该显示器 DPI 换算）
    /// </summary>
    public static Rect GetWorkArea(Window window)
    {
        var hwnd = new WindowInteropHelper(window).Handle;
        var hmon = MonitorFromWindow(hwnd, MONITOR_DEFAULTTONEAREST);

        var mi = new MONITORINFO { cbSize = Marshal.SizeOf<MONITORINFO>() };
        GetMonitorInfo(hmon, ref mi);

        double scale = GetMonitorScale(hmon);   // 用目标显示器的 DPI
        var rc = mi.rcWork;

        return new Rect(
            rc.Left / scale,
            rc.Top  / scale,
            (rc.Right  - rc.Left) / scale,
            (rc.Bottom - rc.Top ) / scale);
    }

    /// <summary>
    /// 获取光标所在显示器的工作区域（WPF 逻辑单位）
    /// </summary>
    public static Rect GetWorkAreaFromCursor()
    {
        GetCursorPos(out POINT pt);
        var hmon = MonitorFromPoint(pt, MONITOR_DEFAULTTONEAREST);

        var mi = new MONITORINFO { cbSize = Marshal.SizeOf<MONITORINFO>() };
        GetMonitorInfo(hmon, ref mi);

        double scale = GetMonitorScale(hmon);
        var rc = mi.rcWork;

        return new Rect(
            rc.Left / scale,
            rc.Top  / scale,
            (rc.Right  - rc.Left) / scale,
            (rc.Bottom - rc.Top ) / scale);
    }

    private static double GetMonitorScale(IntPtr hmon)
    {
        try
        {
            if (GetDpiForMonitor(hmon, MDT_EFFECTIVE_DPI, out uint dpiX, out _) == 0)
                return dpiX / 96.0;
        }
        catch { /* Win8 以下无此 API */ }
        return 1.0;
    }
}
```

### 5.2 使用

```csharp
Rect wa = MonitorHelper.GetWorkArea(this);
this.Left   = wa.Left + 100;
this.Top    = wa.Top  + 100;
this.Width  = 800;
this.Height = 600;
```

**优点**：不依赖 WinForms；DPI 按目标显示器换算，多屏混合 DPI 下更准。

---

## 六、多屏 + 混合 DPI 最稳方案：`SetWindowPos` 传物理像素

**绕开 WPF 逻辑单位体系**，完全在 Win32 层操作，多屏混合 DPI 下最不容易出错。

```csharp
[DllImport("user32.dll")]
private static extern bool SetWindowPos(IntPtr hWnd, IntPtr hWndInsertAfter,
    int X, int Y, int cx, int cy, uint uFlags);

[DllImport("user32.dll")]
private static extern IntPtr MonitorFromWindow(IntPtr hwnd, uint dwFlags);

[DllImport("user32.dll")]
private static extern bool GetMonitorInfo(IntPtr hMonitor, ref MONITORINFO lpmi);

private const uint MONITOR_DEFAULTTONEAREST = 0x00000002;
private const uint SWP_NOZORDER   = 0x0004;
private const uint SWP_NOACTIVATE = 0x0010;

[StructLayout(LayoutKind.Sequential)]
private struct RECT { public int Left, Top, Right, Bottom; }

[StructLayout(LayoutKind.Sequential)]
private struct MONITORINFO
{
    public int cbSize;
    public RECT rcMonitor;
    public RECT rcWork;
    public uint dwFlags;
}

/// <summary>
/// 把窗口铺满其所在显示器的工作区（物理像素，直接 Win32 设置）
/// </summary>
public static void FillWorkArea(Window window)
{
    var hwnd = new WindowInteropHelper(window).Handle;
    var hmon = MonitorFromWindow(hwnd, MONITOR_DEFAULTTONEAREST);

    var mi = new MONITORINFO { cbSize = Marshal.SizeOf<MONITORINFO>() };
    GetMonitorInfo(hmon, ref mi);

    var rc = mi.rcWork;   // 直接用物理像素，无需 DPI 换算
    SetWindowPos(hwnd, IntPtr.Zero,
        rc.Left, rc.Top,
        rc.Right  - rc.Left,
        rc.Bottom - rc.Top,
        SWP_NOZORDER | SWP_NOACTIVATE);
}
```

**优势**：

- 无需 DPI 换算
- 多屏混合 DPI 下不会偏差
- 可精确铺满指定显示器的工作区

---

## 七、多屏注意事项

1. **副屏坐标为负是正常的**：副屏位于主屏左侧/上方时，`rcWork.Left/Top` 为负。WPF 的 `Window.Left/Top` 也是虚拟屏坐标系（主屏左上角为原点），可直接赋负值。

2. **混合 DPI 是最大坑**：
   - 用 `PresentationSource.FromVisual(window).TransformToDevice` 拿的是**窗口当前屏幕**的缩放
   - 查询**其他屏幕**时会用错 DPI
   - 解决：用 `GetDpiForMonitor` 取目标显示器的 DPI，或直接用 `SetWindowPos`

3. **Per-Monitor DPI Aware V2**：建议在 `app.manifest` 中声明，.NET 4.6.2+ 支持，可让 WPF 正确处理多屏不同 DPI：

   ```xml
   <application xmlns="urn:schemas-microsoft-com:asm.v3">
     <windowsSettings>
       <dpiAwareness xmlns="http://schemas.microsoft.com/SMI/2016/WindowsSettings">
         PerMonitorV2
       </dpiAwareness>
     </windowsSettings>
   </application>
   ```

4. **获取句柄时机**：`new WindowInteropHelper(window).Handle` 必须在窗口 `SourceInitialized` 之后调用才有效。

---

## 八、方案选型速查

| 需求 | 方案 | 是否需 DPI 换算 |
|------|------|----------------|
| 单屏简单布局 | `SystemParameters.WorkArea` | 否 |
| 多屏，不想引 WinForms | P/Invoke `GetMonitorInfo` + `GetDpiForMonitor` | 是（按目标屏） |
| 多屏，允许引 WinForms | `Screen.FromHandle` + DPI 换算 | 是 |
| **多屏 + 混合 DPI，铺满某屏** | **`SetWindowPos` 物理像素** | **否（推荐）** |
| 全屏覆盖任务栏 | `WindowStyle=None` + `WindowState=Maximized` | 视情况 |

---

## 九、一句话总结

> 多屏场景 P/Invoke 方案完全可用，但**混合 DPI** 下最稳妥的做法是：**用 `SetWindowPos` 在 Win32 层直接以物理像素设置窗口位置和大小**，彻底绕开 WPF 逻辑单位换算，避免各屏 DPI 不一致带来的偏差。