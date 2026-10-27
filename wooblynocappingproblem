// ==WindhawkMod==
// @id            wobbly-windows
// @name          Wobbly Windows (Pure D2D)
// @description   Ultra-smooth Wobbly Windows physics powered by Direct2D.
// @version       1.2.0
// @author        Extracted from Custom Animations Mod
// @include       *.exe
// @compilerOptions -ldwmapi -lgdi32 -ld2d1 -luser32 -lole32 -lcomctl32 -lwinmm
// ==/WindhawkMod==

// ==WindhawkModReadme==
/*
# Wobbly Windows Mod
Provides interactive jelly physics when moving/resizing windows. Powered entirely by Direct2D.

Render loop runs on a dedicated high-priority thread with a high-resolution waitable timer, targeting the monitor's refresh rate. Physics is frame-rate independent so 60 Hz and 144 Hz feel identical.
*/
// ==/WindhawkModReadme==

// ==WindhawkModSettings==
/*
- wobbly_enabled: true
  $name: Enable Wobbly Windows
  $description: Wobble the window when moving or resizing it.

- wobbly_preset: 4
  $name: Wobbly Physics Preset
  $description: Adjust the stiffness and elasticity of the window dragging physics.
  $options:
    - 0: Very jelly / elastic
    - 1: Soft wobble
    - 2: Balanced smooth
    - 3: KDE-like default
    - 4: Very soft / floaty
*/
// ==/WindhawkModSettings==

#include <windows.h>
#include <dwmapi.h>
#include <d2d1.h>
#include <tlhelp32.h>
#include <math.h>
#include <atomic>
#include <mutex>
#include <algorithm>
#include <mmsystem.h>

#define SAFE_RELEASE(p) if (p) { p->Release(); p = NULL; }

#ifndef DWMWA_EXTENDED_FRAME_BOUNDS
#define DWMWA_EXTENDED_FRAME_BOUNDS 9
#endif
#ifndef PW_RENDERFULLCONTENT
#define PW_RENDERFULLCONTENT 2
#endif
#ifndef CREATE_WAITABLE_TIMER_HIGH_RESOLUTION
#define CREATE_WAITABLE_TIMER_HIGH_RESOLUTION 0x00000002
#endif

#define X_TILES 20
#define Y_TILES 20

// -------------------------------------------------------------------------
// High-resolution timing helpers
// -------------------------------------------------------------------------
static double QpcNowMs() {
    static LARGE_INTEGER freq = {};
    if (!freq.QuadPart) QueryPerformanceFrequency(&freq);
    LARGE_INTEGER c; QueryPerformanceCounter(&c);
    return (double)c.QuadPart * 1000.0 / (double)freq.QuadPart;
}

static double GetRefreshRateHz(HWND hwnd) {
    HMONITOR hMon = MonitorFromWindow(hwnd, MONITOR_DEFAULTTONEAREST);
    MONITORINFOEXW mi = { sizeof(mi) };
    if (GetMonitorInfoW(hMon, (MONITORINFO*)&mi)) {
        DEVMODEW dm = { sizeof(dm) };
        if (EnumDisplaySettingsW(mi.szDevice, ENUM_CURRENT_SETTINGS, &dm) && dm.dmDisplayFrequency > 1) {
            return (double)dm.dmDisplayFrequency;
        }
    }
    DWM_TIMING_INFO ti = { sizeof(ti) };
    if (SUCCEEDED(DwmGetCompositionTimingInfo(NULL, &ti)) && ti.rateRefresh.uiNumerator > 0) {
        return (double)ti.rateRefresh.uiNumerator / (double)ti.rateRefresh.uiDenominator;
    }
    return 60.0;
}

// -------------------------------------------------------------------------
// Shared Structs & Configuration
// -------------------------------------------------------------------------
struct Geometry { float x, y, width, height; };
struct ParameterSet { float stiffness, drag, moveFactor, minVelocity, maxVelocity, stopVelocity, minAcc, maxAcc, stopAcc; };

static const ParameterSet set_0 = { 0.15f, 0.80f, 0.10f, 0.0f, 1000.0f, 0.5f, 0.0f, 1000.0f, 0.5f };
static const ParameterSet set_1 = { 0.10f, 0.85f, 0.10f, 0.0f, 1000.0f, 0.5f, 0.0f, 1000.0f, 0.5f };
static const ParameterSet set_2 = { 0.06f, 0.90f, 0.10f, 0.0f, 1000.0f, 0.5f, 0.0f, 1000.0f, 0.5f };
static const ParameterSet set_3 = { 0.03f, 0.92f, 0.20f, 0.0f, 1000.0f, 0.5f, 0.0f, 1000.0f, 0.5f };
static const ParameterSet set_4 = { 0.01f, 0.97f, 0.25f, 0.0f, 1000.0f, 0.5f, 0.0f, 1000.0f, 0.5f };

class ConfigManager {
public:
    static ConfigManager& Instance() { static ConfigManager instance; return instance; }
    std::atomic<bool> wobblyEnabled{true};
    ParameterSet wobblyParams{set_4};
    void Load() {
        wobblyEnabled.store(Wh_GetIntSetting(L"wobbly_enabled") != 0, std::memory_order_relaxed);
        int preset = Wh_GetIntSetting(L"wobbly_preset");
        switch (preset) {
            case 0: wobblyParams = set_0; break;
            case 1: wobblyParams = set_1; break;
            case 2: wobblyParams = set_2; break;
            case 3: wobblyParams = set_3; break;
            case 4: default: wobblyParams = set_4; break;
        }
    }
};

// -------------------------------------------------------------------------
// Global Hooks
// -------------------------------------------------------------------------
using CreateWindowExW_t = HWND(WINAPI*)(DWORD, LPCWSTR, LPCWSTR, DWORD, int, int, int, int, HWND, HMENU, HINSTANCE, LPVOID);
CreateWindowExW_t CreateWindowExW_Original;
using CreateWindowExA_t = HWND(WINAPI*)(DWORD, LPCSTR, LPCSTR, DWORD, int, int, int, int, HWND, HMENU, HINSTANCE, LPVOID);
CreateWindowExA_t CreateWindowExA_Original;

HINSTANCE g_hInstance = NULL;
UINT g_wmAttach = 0;
UINT g_wmDetach = 0;
std::atomic<bool> g_isUnloading{false};

class D2DEngine {
public:
    static D2DEngine& Instance() { static D2DEngine instance; return instance; }
    ID2D1Factory* Factory() const { return m_factory; }
    void Initialize() {
        if (!m_factory) {
            D2D1CreateFactory(D2D1_FACTORY_TYPE_MULTI_THREADED, __uuidof(ID2D1Factory), reinterpret_cast<void**>(&m_factory));
        }
    }
    void Shutdown() {
        if (m_factory) { m_factory->Release(); m_factory = nullptr; }
    }
private:
    ID2D1Factory* m_factory = nullptr;
};

// -------------------------------------------------------------------------
// Wobbly Controller
// -------------------------------------------------------------------------
struct Pair { float x, y; };
struct WobblyInfos {
    Pair origin[16], position[16], velocity[16], acceleration[16], buffer[16];
    bool constraint[16], wobblying, can_wobble_top, can_wobble_bottom, can_wobble_left, can_wobble_right;
};

class WobblyController {
public:
    static WobblyController& Instance() { static WobblyController inst; return inst; }

    void InitializeIfNeeded();
    void OnEnterSizeMove(HWND hwnd);
    void OnSizingMoving(HWND hwnd, LPRECT r);
    void OnWindowPosChanged(HWND hwnd, WINDOWPOS* wp);
    void OnExitSizeMove(HWND hwnd);
    void Shutdown();
    LRESULT ProcessOverlayMessage(HWND hwnd, UINT msg, WPARAM wp, LPARAM lp);

private:
    std::recursive_mutex m_mutex;
    bool m_engineInitialized = false;
    HWND m_mainHwnd = NULL;
    HWND m_overlayHwnd = NULL;
    int m_capX = 0, m_capY = 0, m_capW = 0, m_capH = 0;
    WobblyInfos m_wwi = {};
    std::atomic<bool> m_isMoving{false};
    std::atomic<bool> m_isSettling{false};
    LONG_PTR m_oldExStyle = 0;
    int m_screenX = 0, m_screenY = 0, m_screenW = 0, m_screenH = 0;
    Geometry m_currentRect = { 0 };

    ID2D1DCRenderTarget* m_rt = nullptr;
    ID2D1BitmapBrush* m_brush = nullptr;
    HBITMAP m_targetBmp = NULL;
    void* m_targetBits = nullptr;
    HDC m_memDC = NULL;
    int m_bmpW = 0, m_bmpH = 0;
    HDC m_screenDC = NULL;

    // Render thread
    HANDLE m_renderThread = NULL;
    HANDLE m_renderEvent = NULL;
    std::atomic<bool> m_renderRunning{false};
    double m_cachedRefreshHz = 60.0;
    bool m_hrTimerOk = false;

    void CaptureWindow(HWND hwnd);
    void UpdateOrigin(float x, float y, float w, float h);
    void InitInfo(float x, float y, float w, float h);
    void StepPhysics(float time);
    void DrawOverlayFrame();
    void CleanupD2D();

    void StartRenderThread();
    void StopRenderThread();
    void TickFrame(double deltaMs);
    static DWORD WINAPI RenderThreadProc(LPVOID lp);

    static void HeightRingLinearMean(Pair* data, Pair* buffer);
    Pair ComputeBezierPoint(float tx, float ty);
};

// -------------------------------------------------------------------------
// Helper Graphics
// -------------------------------------------------------------------------
static ID2D1PathGeometry* CreateQuadGeo(D2D1_POINT_2F p0, D2D1_POINT_2F p1, D2D1_POINT_2F p2, D2D1_POINT_2F p3) {
    ID2D1Factory* factory = D2DEngine::Instance().Factory();
    if (!factory) return nullptr;
    ID2D1PathGeometry* geo = nullptr;
    factory->CreatePathGeometry(&geo);
    if (!geo) return nullptr;
    ID2D1GeometrySink* sink = nullptr;
    geo->Open(&sink);
    sink->BeginFigure(p0, D2D1_FIGURE_BEGIN_FILLED);
    sink->AddLine(p1);
    sink->AddLine(p2);
    sink->AddLine(p3);
    sink->EndFigure(D2D1_FIGURE_END_CLOSED);
    sink->Close();
    sink->Release();
    return geo;
}

static D2D1_POINT_2F BloatPoint(D2D1_POINT_2F p, D2D1_POINT_2F c) {
    float dx = p.x - c.x; float dy = p.y - c.y;
    float len = sqrt(dx*dx + dy*dy);
    if (len < 0.001f) return p;
    return D2D1::Point2F(p.x + (dx/len)*0.5f, p.y + (dy/len)*0.5f);
}

// =========================================================================
// WobblyController implementation
// =========================================================================
static LRESULT CALLBACK OverlayProc(HWND hwnd, UINT msg, WPARAM wp, LPARAM lp) {
    return WobblyController::Instance().ProcessOverlayMessage(hwnd, msg, wp, lp);
}

void WobblyController::InitializeIfNeeded() {
    if (m_overlayHwnd) return;
    if (!g_hInstance) {
        GetModuleHandleExA(GET_MODULE_HANDLE_EX_FLAG_FROM_ADDRESS | GET_MODULE_HANDLE_EX_FLAG_UNCHANGED_REFCOUNT,
                           (LPCSTR)&OverlayProc, (HMODULE*)&g_hInstance);
    }
    WNDCLASSA oc = {0};
    oc.lpfnWndProc = OverlayProc;
    oc.hInstance = g_hInstance;
    oc.lpszClassName = "WobblyOverlayWindow";
    RegisterClassA(&oc);

    m_screenX = GetSystemMetrics(SM_XVIRTUALSCREEN);
    m_screenY = GetSystemMetrics(SM_YVIRTUALSCREEN);
    m_screenW = GetSystemMetrics(SM_CXVIRTUALSCREEN);
    m_screenH = GetSystemMetrics(SM_CYVIRTUALSCREEN);

    m_overlayHwnd = CreateWindowExA(
        WS_EX_TOPMOST | WS_EX_LAYERED | WS_EX_TRANSPARENT | WS_EX_TOOLWINDOW,
        "WobblyOverlayWindow", NULL, WS_POPUP,
        m_screenX, m_screenY, m_screenW, m_screenH,
        NULL, NULL, oc.hInstance, NULL);
    m_engineInitialized = true;
}

void WobblyController::Shutdown() {
    if (m_engineInitialized) {
        StopRenderThread();
        if (m_overlayHwnd) {
            SendMessageTimeoutW(m_overlayHwnd, WM_CLOSE, 0, 0, SMTO_ABORTIFHUNG | SMTO_NORMAL, 500, NULL);
            m_overlayHwnd = NULL;
        }
        std::lock_guard<std::recursive_mutex> lock(m_mutex);
        CleanupD2D();
        if (m_rt) { m_rt->Release(); m_rt = nullptr; }
        if (m_memDC) { DeleteDC(m_memDC); m_memDC = NULL; }
        if (m_targetBmp) { DeleteObject(m_targetBmp); m_targetBmp = NULL; }
        if (m_screenDC) { ReleaseDC(NULL, m_screenDC); m_screenDC = NULL; }
    }
}

void WobblyController::CaptureWindow(HWND hwnd) {
    RECT rcWin, rcExt;
    GetWindowRect(hwnd, &rcWin);
    if (DwmGetWindowAttribute(hwnd, DWMWA_EXTENDED_FRAME_BOUNDS, &rcExt, sizeof(rcExt)) != S_OK) rcExt = rcWin;

    int rawW = rcWin.right - rcWin.left;
    int rawH = rcWin.bottom - rcWin.top;
    m_capX = rcExt.left - rcWin.left;
    m_capY = rcExt.top - rcWin.top;
    m_capW = rcExt.right - rcExt.left;
    m_capH = rcExt.bottom - rcExt.top;
    if (rawW <= 0 || rawH <= 0 || m_capW <= 0 || m_capH <= 0) return;

    HDC hdcScreen = GetDC(NULL);
    HDC hdcMem = CreateCompatibleDC(hdcScreen);
    BITMAPINFO bmi = {{sizeof(BITMAPINFOHEADER), rawW, -rawH, 1, 32, BI_RGB}};
    void* rawBits = nullptr;
    HBITMAP hbmRaw = CreateDIBSection(hdcMem, &bmi, DIB_RGB_COLORS, &rawBits, NULL, 0);
    HBITMAP hOldBmp = (HBITMAP)SelectObject(hdcMem, hbmRaw);
    PrintWindow(hwnd, hdcMem, PW_RENDERFULLCONTENT);

    BYTE* pRaw = (BYTE*)rawBits;
    for (int i = 0; i < rawW * rawH; i++) {
        BYTE a = pRaw[i * 4 + 3];
        if (a == 255) continue;
        if (a == 0) { pRaw[i * 4 + 0] = 0; pRaw[i * 4 + 1] = 0; pRaw[i * 4 + 2] = 0; }
        else {
            pRaw[i * 4 + 0] = (pRaw[i * 4 + 0] * a) / 255;
            pRaw[i * 4 + 1] = (pRaw[i * 4 + 1] * a) / 255;
            pRaw[i * 4 + 2] = (pRaw[i * 4 + 2] * a) / 255;
        }
    }

    // --- Tight overlay bounding box (Option A: don't cover the whole screen) ---
    int margin = (std::max)(m_capW, m_capH) / 2 + 64;
    m_screenX = rcExt.left - margin;
    m_screenY = rcExt.top - margin;
    m_screenW = m_capW + margin * 2;
    m_screenH = m_capH + margin * 2;

    if (m_overlayHwnd)
        SetWindowPos(m_overlayHwnd, NULL, m_screenX, m_screenY, m_screenW, m_screenH, SWP_NOZORDER | SWP_NOACTIVATE);

    if (m_bmpW != m_screenW || m_bmpH != m_screenH) {
        if (m_memDC) { DeleteDC(m_memDC); m_memDC = NULL; }
        if (m_targetBmp) { DeleteObject(m_targetBmp); m_targetBmp = NULL; }
        if (m_rt) { m_rt->Release(); m_rt = nullptr; }
        m_bmpW = m_screenW; m_bmpH = m_screenH;
    }

    if (!m_memDC) {
        m_memDC = CreateCompatibleDC(hdcScreen);
        BITMAPINFO bmiScreen = {{sizeof(BITMAPINFOHEADER), m_screenW, -m_screenH, 1, 32, BI_RGB}};
        m_targetBmp = CreateDIBSection(hdcScreen, &bmiScreen, DIB_RGB_COLORS, &m_targetBits, NULL, 0);
        SelectObject(m_memDC, m_targetBmp);
    }

    ID2D1Factory* d2dFactory = D2DEngine::Instance().Factory();
    if (!m_rt && d2dFactory) {
        D2D1_RENDER_TARGET_PROPERTIES rtProps = D2D1::RenderTargetProperties(
            D2D1_RENDER_TARGET_TYPE_SOFTWARE,
            D2D1::PixelFormat(DXGI_FORMAT_B8G8R8A8_UNORM, D2D1_ALPHA_MODE_PREMULTIPLIED),
            0, 0, D2D1_RENDER_TARGET_USAGE_GDI_COMPATIBLE, D2D1_FEATURE_LEVEL_DEFAULT);
        d2dFactory->CreateDCRenderTarget(&rtProps, &m_rt);
    }

    if (m_rt) {
        if (m_brush) { m_brush->Release(); m_brush = nullptr; }
        ID2D1Bitmap* pRawBmp = nullptr;
        D2D1_BITMAP_PROPERTIES bmpProps = D2D1::BitmapProperties(
            D2D1::PixelFormat(DXGI_FORMAT_B8G8R8A8_UNORM, D2D1_ALPHA_MODE_PREMULTIPLIED));
        m_rt->CreateBitmap(D2D1::SizeU(rawW, rawH), rawBits, rawW * 4, bmpProps, &pRawBmp);
        if (pRawBmp) {
            D2D1_BITMAP_BRUSH_PROPERTIES brushProps = D2D1::BitmapBrushProperties(
                D2D1_EXTEND_MODE_CLAMP, D2D1_EXTEND_MODE_CLAMP, D2D1_BITMAP_INTERPOLATION_MODE_LINEAR);
            m_rt->CreateBitmapBrush(pRawBmp, &brushProps, nullptr, &m_brush);
            pRawBmp->Release();
        }
    }

    SelectObject(hdcMem, hOldBmp);
    DeleteObject(hbmRaw);
    DeleteDC(hdcMem);
    ReleaseDC(NULL, hdcScreen);

    m_cachedRefreshHz = GetRefreshRateHz(hwnd);
    if (m_cachedRefreshHz < 30.0) m_cachedRefreshHz = 60.0;
}

void WobblyController::UpdateOrigin(float x, float y, float w, float h) {
    m_currentRect.x = x; m_currentRect.y = y; m_currentRect.width = w; m_currentRect.height = h;
    float x_length = w / 3.0f, y_length = h / 3.0f;
    Pair origine = { x, y };
    for (int j = 0; j < 4; ++j) {
        for (int i = 0; i < 4; ++i) {
            m_wwi.origin[j * 4 + i] = origine;
            if (i != 2) origine.x += x_length; else origine.x = w + x;
        }
        origine.x = x;
        if (j != 2) origine.y += y_length; else origine.y = h + y;
    }
}

void WobblyController::InitInfo(float x, float y, float w, float h) {
    memset(&m_wwi, 0, sizeof(WobblyInfos));
    UpdateOrigin(x, y, w, h);
    for (int i = 0; i < 16; ++i) m_wwi.position[i] = m_wwi.origin[i];
    m_wwi.can_wobble_top = true; m_wwi.can_wobble_bottom = true;
    m_wwi.can_wobble_left = true; m_wwi.can_wobble_right = true;
    m_wwi.wobblying = true;
}

void WobblyController::HeightRingLinearMean(Pair* data, Pair* buffer) {
    for (int i = 0; i < 16; ++i) {
        int x = i % 4, y = i / 4, weight = 0;
        float sum_x = 0, sum_y = 0;
        for (int dy = -1; dy <= 1; ++dy) for (int dx = -1; dx <= 1; ++dx) {
            int nx = x + dx, ny = y + dy;
            if (nx >= 0 && nx <= 3 && ny >= 0 && ny <= 3) {
                if (dx == 0 && dy == 0) continue;
                sum_x += data[ny * 4 + nx].x; sum_y += data[ny * 4 + nx].y; weight++;
            }
        }
        sum_x += data[i].x * weight; sum_y += data[i].y * weight;
        buffer[i].x = sum_x / (weight * 2.0f); buffer[i].y = sum_y / (weight * 2.0f);
    }
    for (int i = 0; i < 16; ++i) data[i] = buffer[i];
}

void WobblyController::StepPhysics(float time) {
    ParameterSet params = ConfigManager::Instance().wobblyParams;
    float x_length = m_currentRect.width / 3.0f, y_length = m_currentRect.height / 3.0f;
    float acc_sum = 0.0f, vel_sum = 0.0f;

    for (int i = 0; i < 16; ++i) {
        if (m_wwi.constraint[i]) {
            m_wwi.acceleration[i].x = (m_wwi.origin[i].x - m_wwi.position[i].x) * params.stiffness;
            m_wwi.acceleration[i].y = (m_wwi.origin[i].y - m_wwi.position[i].y) * params.stiffness;
        } else {
            float ax = 0, ay = 0; int count = 0; int x = i % 4, y = i / 4;
            if (x < 3) { ax += (m_wwi.position[i + 1].x - m_wwi.position[i].x - x_length) * params.stiffness; ay += (m_wwi.position[i + 1].y - m_wwi.position[i].y) * params.stiffness; count++; }
            if (x > 0) { ax += (m_wwi.position[i - 1].x - m_wwi.position[i].x + x_length) * params.stiffness; ay += (m_wwi.position[i - 1].y - m_wwi.position[i].y) * params.stiffness; count++; }
            if (y < 3) { ax += (m_wwi.position[i + 4].x - m_wwi.position[i].x) * params.stiffness; ay += (m_wwi.position[i + 4].y - m_wwi.position[i].y - y_length) * params.stiffness; count++; }
            if (y > 0) { ax += (m_wwi.position[i - 4].x - m_wwi.position[i].x) * params.stiffness; ay += (m_wwi.position[i - 4].y - m_wwi.position[i].y + y_length) * params.stiffness; count++; }
            m_wwi.acceleration[i].x = count > 0 ? ax / count : 0.0f;
            m_wwi.acceleration[i].y = count > 0 ? ay / count : 0.0f;
        }
    }

    HeightRingLinearMean(m_wwi.acceleration, m_wwi.buffer);

    // Frame-rate independent damping (matches 60 Hz reference feel)
    float damping = powf(params.drag, time * 0.06f);

    for (int i = 0; i < 16; ++i) {
        Pair acc = m_wwi.acceleration[i];
        if (fabsf(acc.x) < params.minAcc) acc.x = 0; if (fabsf(acc.y) < params.minAcc) acc.y = 0;
        if (fabsf(acc.x) > params.maxAcc) acc.x = acc.x > 0 ? params.maxAcc : -params.maxAcc;
        if (fabsf(acc.y) > params.maxAcc) acc.y = acc.y > 0 ? params.maxAcc : -params.maxAcc;

        m_wwi.velocity[i].x = acc.x * time + m_wwi.velocity[i].x * damping;
        m_wwi.velocity[i].y = acc.y * time + m_wwi.velocity[i].y * damping;
        acc_sum += fabsf(acc.x) + fabsf(acc.y);
    }

    HeightRingLinearMean(m_wwi.velocity, m_wwi.buffer);

    for (int i = 0; i < 16; ++i) {
        Pair vel = m_wwi.velocity[i];
        if (fabsf(vel.x) < params.minVelocity) vel.x = 0; if (fabsf(vel.y) < params.minVelocity) vel.y = 0;
        if (fabsf(vel.x) > params.maxVelocity) vel.x = vel.x > 0 ? params.maxVelocity : -params.maxVelocity;
        if (fabsf(vel.y) > params.maxVelocity) vel.y = vel.y > 0 ? params.maxVelocity : -params.maxVelocity;

        m_wwi.position[i].x += vel.x * time * params.moveFactor;
        m_wwi.position[i].y += vel.y * time * params.moveFactor;
        vel_sum += fabsf(vel.x) + fabsf(vel.y);
    }

    if (!m_wwi.can_wobble_top)    { for (int i = 0; i < 4; ++i)   for (int j = 0; j < 3; ++j) m_wwi.position[i + 4*j].y = m_wwi.origin[i + 4*j].y; }
    if (!m_wwi.can_wobble_bottom) { for (int i = 12; i < 16; ++i) for (int j = 0; j < 3; ++j) m_wwi.position[i - 4*j].y = m_wwi.origin[i - 4*j].y; }
    if (!m_wwi.can_wobble_left)   { for (int i = 0; i < 16; i += 4) for (int j = 0; j < 3; ++j) m_wwi.position[i + j].x = m_wwi.origin[i + j].x; }
    if (!m_wwi.can_wobble_right)  { for (int i = 3; i < 16; i += 4) for (int j = 0; j < 3; ++j) m_wwi.position[i - j].x = m_wwi.origin[i - j].x; }

    m_wwi.wobblying = !(acc_sum < params.stopAcc && vel_sum < params.stopVelocity);
}

Pair WobblyController::ComputeBezierPoint(float tx, float ty) {
    float px[4] = { (1-tx)*(1-tx)*(1-tx), 3*(1-tx)*(1-tx)*tx, 3*(1-tx)*tx*tx, tx*tx*tx };
    float py[4] = { (1-ty)*(1-ty)*(1-ty), 3*(1-ty)*(1-ty)*ty, 3*(1-ty)*ty*ty, ty*ty*ty };
    Pair res = { 0.0f, 0.0f };
    for (int j = 0; j < 4; ++j)
        for (int i = 0; i < 4; ++i) {
            res.x += px[i] * py[j] * m_wwi.position[i + j * 4].x;
            res.y += px[i] * py[j] * m_wwi.position[i + j * 4].y;
        }
    return res;
}

void WobblyController::DrawOverlayFrame() {
    if (!m_rt || !m_brush || !m_memDC) return;

    RECT bindRect = { 0, 0, m_screenW, m_screenH };
    m_rt->BindDC(m_memDC, &bindRect);
    m_rt->BeginDraw();
    m_rt->Clear(D2D1::ColorF(0, 0, 0, 0));
    m_rt->SetAntialiasMode(D2D1_ANTIALIAS_MODE_PER_PRIMITIVE);

    float vLeft = (float)m_screenX;
    float vTop  = (float)m_screenY;

    ID2D1PathGeometry* outlineGeo = nullptr;
    ID2D1Factory* factory = D2DEngine::Instance().Factory();
    if (factory) factory->CreatePathGeometry(&outlineGeo);

    if (outlineGeo) {
        ID2D1GeometrySink* sink = nullptr;
        outlineGeo->Open(&sink);
        Pair startP = ComputeBezierPoint(0.0f, 0.0f);
        sink->BeginFigure(D2D1::Point2F(startP.x - vLeft, startP.y - vTop), D2D1_FIGURE_BEGIN_FILLED);
        for (int x = 1; x <= X_TILES; x++) { Pair p = ComputeBezierPoint((float)x / X_TILES, 0.0f); sink->AddLine(D2D1::Point2F(p.x - vLeft, p.y - vTop)); }
        for (int y = 1; y <= Y_TILES; y++) { Pair p = ComputeBezierPoint(1.0f, (float)y / Y_TILES); sink->AddLine(D2D1::Point2F(p.x - vLeft, p.y - vTop)); }
        for (int x = X_TILES - 1; x >= 0; x--) { Pair p = ComputeBezierPoint((float)x / X_TILES, 1.0f); sink->AddLine(D2D1::Point2F(p.x - vLeft, p.y - vTop)); }
        for (int y = Y_TILES - 1; y >= 0; y--) { Pair p = ComputeBezierPoint(0.0f, (float)y / Y_TILES); sink->AddLine(D2D1::Point2F(p.x - vLeft, p.y - vTop)); }
        sink->EndFigure(D2D1_FIGURE_END_CLOSED);
        sink->Close();

        ID2D1Layer* layer = nullptr;
        m_rt->CreateLayer(&layer);
        D2D1_LAYER_PARAMETERS layerParams = D2D1::LayerParameters();
        layerParams.geometricMask = outlineGeo;
        layerParams.maskAntialiasMode = D2D1_ANTIALIAS_MODE_PER_PRIMITIVE;
        m_rt->PushLayer(&layerParams, layer);

        for (int y = 0; y < Y_TILES; y++) for (int x = 0; x < X_TILES; x++) {
            float tx1 = (float)x / X_TILES, ty1 = (float)y / Y_TILES;
            float tx2 = (float)(x + 1) / X_TILES, ty2 = (float)(y + 1) / Y_TILES;
            Pair bp1 = ComputeBezierPoint(tx1, ty1);
            Pair bp2 = ComputeBezierPoint(tx2, ty1);
            Pair bp3 = ComputeBezierPoint(tx1, ty2);
            Pair bp4 = ComputeBezierPoint(tx2, ty2);
            D2D1_POINT_2F p1 = D2D1::Point2F(bp1.x - vLeft, bp1.y - vTop);
            D2D1_POINT_2F p2 = D2D1::Point2F(bp2.x - vLeft, bp2.y - vTop);
            D2D1_POINT_2F p3 = D2D1::Point2F(bp3.x - vLeft, bp3.y - vTop);
            D2D1_POINT_2F p4 = D2D1::Point2F(bp4.x - vLeft, bp4.y - vTop);
            D2D1_POINT_2F c = D2D1::Point2F((p1.x+p2.x+p3.x+p4.x)/4.0f, (p1.y+p2.y+p3.y+p4.y)/4.0f);
            ID2D1PathGeometry* quadGeo = CreateQuadGeo(BloatPoint(p1, c), BloatPoint(p2, c), BloatPoint(p4, c), BloatPoint(p3, c));
            if (quadGeo) {
                float sx = (float)m_capX + tx1 * m_capW;
                float sy = (float)m_capY + ty1 * m_capH;
                float sw = (float)m_capW / X_TILES;
                float sh = (float)m_capH / Y_TILES;
                float m11 = (p2.x - p1.x) / sw;
                float m12 = (p2.y - p1.y) / sw;
                float m21 = (p3.x - p1.x) / sh;
                float m22 = (p3.y - p1.y) / sh;
                float m31 = p1.x - sx * m11 - sy * m21;
                float m32 = p1.y - sx * m12 - sy * m22;
                m_brush->SetTransform(D2D1::Matrix3x2F(m11, m12, m21, m22, m31, m32));
                m_rt->FillGeometry(quadGeo, m_brush);
                quadGeo->Release();
            }
        }
        m_rt->PopLayer();
        if (layer) layer->Release();
        outlineGeo->Release();
    }
    m_rt->EndDraw();

    if (!m_screenDC) m_screenDC = GetDC(NULL);
    POINT ptS = {0, 0};
    POINT ptW = {m_screenX, m_screenY};
    SIZE sz = {m_screenW, m_screenH};
    BLENDFUNCTION bl = {AC_SRC_OVER, 0, 255, AC_SRC_ALPHA};
    UpdateLayeredWindow(m_overlayHwnd, m_screenDC, &ptW, &sz, m_memDC, &ptS, 0, &bl, ULW_ALPHA);
}

void WobblyController::CleanupD2D() {
    if (m_brush) { m_brush->Release(); m_brush = nullptr; }
}

// -------------------------------------------------------------------------
// Render thread
// -------------------------------------------------------------------------
void WobblyController::StartRenderThread() {
    if (m_renderRunning.load()) return;
    m_renderRunning.store(true);
    m_renderEvent = CreateEventW(NULL, TRUE, FALSE, NULL);
    m_renderThread = CreateThread(NULL, 0, RenderThreadProc, this, 0, NULL);
    if (m_renderThread) SetThreadPriority(m_renderThread, THREAD_PRIORITY_HIGHEST);
}

void WobblyController::StopRenderThread() {
    if (!m_renderRunning.load()) return;
    m_renderRunning.store(false);
    if (m_renderEvent) SetEvent(m_renderEvent);
    if (m_renderThread) {
        WaitForSingleObject(m_renderThread, 2000);
        CloseHandle(m_renderThread);
        m_renderThread = NULL;
    }
    if (m_renderEvent) { CloseHandle(m_renderEvent); m_renderEvent = NULL; }
}

DWORD WINAPI WobblyController::RenderThreadProc(LPVOID lp) {
    auto* self = (WobblyController*)lp;
    timeBeginPeriod(1);

    HANDLE timer = CreateWaitableTimerExW(NULL, NULL, CREATE_WAITABLE_TIMER_HIGH_RESOLUTION, TIMER_ALL_ACCESS);
    self->m_hrTimerOk = (timer != NULL);
    if (!timer) {
        Wh_Log(L"Wobbly: high-resolution timer unavailable (err=%lu), falling back", GetLastError());
        timer = CreateWaitableTimerW(NULL, FALSE, NULL);
    } else {
        Wh_Log(L"Wobbly: high-resolution timer created");
    }

    double last = QpcNowMs();
    while (self->m_renderRunning.load()) {
        double frameMs = 1000.0 / self->m_cachedRefreshHz;
        LARGE_INTEGER due;
        due.QuadPart = -(LONGLONG)(frameMs * 10000.0);
        SetWaitableTimer(timer, &due, 0, NULL, NULL, FALSE);

        HANDLE waits[2] = { timer, self->m_renderEvent };
        DWORD w = WaitForMultipleObjects(2, waits, FALSE, 100);
        if (w == WAIT_OBJECT_0 + 1 || !self->m_renderRunning.load()) break;

        double now = QpcNowMs();
        double delta = now - last;
        last = now;
        if (delta > 64.0) delta = 64.0;
        if (delta <= 0.0) continue;

        self->TickFrame(delta);
    }

    CancelWaitableTimer(timer);
    CloseHandle(timer);
    timeEndPeriod(1);
    return 0;
}

void WobblyController::TickFrame(double deltaMs) {
    // FPS log (once per second)
    static double s_fpsLastLog = 0.0;
    static int s_fpsCount = 0;
    s_fpsCount++;
    double nowLog = QpcNowMs();
    if (nowLog - s_fpsLastLog >= 1000.0) {
        Wh_Log(L"Wobbly FPS: %d (target %.1f Hz, hrTimer=%d)", s_fpsCount, m_cachedRefreshHz, (int)m_hrTimerOk);
        s_fpsCount = 0;
        s_fpsLastLog = nowLog;
    }

    std::lock_guard<std::recursive_mutex> lock(m_mutex);
    if (!m_isMoving.load() && !m_isSettling.load()) return;

    if (!IsWindow(m_mainHwnd)) {
        m_isMoving.store(false);
        m_isSettling.store(false);
        if (m_overlayHwnd) ShowWindow(m_overlayHwnd, SW_HIDE);
        CleanupD2D();
        m_mainHwnd = NULL;
        // note: StopRenderThread() cannot be called from inside the thread; do it from the caller side.
        return;
    }
    if (!(GetWindowLongPtrW(m_mainHwnd, GWL_EXSTYLE) & WS_EX_LAYERED)) {
        SetWindowLongPtrW(m_mainHwnd, GWL_EXSTYLE, m_oldExStyle | WS_EX_LAYERED);
        SetLayeredWindowAttributes(m_mainHwnd, 0, 1, LWA_ALPHA);
    }

    const double kSubstep = 4.0;
    double remaining = deltaMs;
    while (remaining > 0.0) {
        double dt = remaining > kSubstep ? kSubstep : remaining;
        StepPhysics((float)dt);
        remaining -= dt;
    }

    DrawOverlayFrame();

    if (m_isSettling.load() && !m_wwi.wobblying) {
        m_isSettling.store(false);
        if (m_overlayHwnd) ShowWindow(m_overlayHwnd, SW_HIDE);
        SetLayeredWindowAttributes(m_mainHwnd, 0, 255, LWA_ALPHA);
        if (!(m_oldExStyle & WS_EX_LAYERED))
            SetWindowLongPtrW(m_mainHwnd, GWL_EXSTYLE, m_oldExStyle);
        CleanupD2D();
        m_mainHwnd = NULL;
    }
}

// -------------------------------------------------------------------------
// Public events
// -------------------------------------------------------------------------
void WobblyController::OnEnterSizeMove(HWND hwnd) {
    if (!ConfigManager::Instance().wobblyEnabled.load(std::memory_order_relaxed)) return;
    if (IsZoomed(hwnd) || IsIconic(hwnd)) return;

    std::lock_guard<std::recursive_mutex> lock(m_mutex);
    if ((m_isMoving.load() || m_isSettling.load()) && m_mainHwnd != NULL && m_mainHwnd != hwnd) return;

    if (!m_engineInitialized) InitializeIfNeeded();
    m_mainHwnd = hwnd;

    RECT rcExt;
    if (DwmGetWindowAttribute(hwnd, DWMWA_EXTENDED_FRAME_BOUNDS, &rcExt, sizeof(rcExt)) != S_OK) GetWindowRect(hwnd, &rcExt);

    if (!m_isSettling.load() && !m_isMoving.load()) {
        CaptureWindow(hwnd);

        m_oldExStyle = GetWindowLongPtrW(hwnd, GWL_EXSTYLE);
        SetWindowLongPtrW(hwnd, GWL_EXSTYLE, m_oldExStyle | WS_EX_LAYERED);
        SetLayeredWindowAttributes(hwnd, 0, 1, LWA_ALPHA);

        InitInfo((float)rcExt.left, (float)rcExt.top,
                 (float)(rcExt.right - rcExt.left), (float)(rcExt.bottom - rcExt.top));
    } else {
        UpdateOrigin((float)rcExt.left, (float)rcExt.top,
                     (float)(rcExt.right - rcExt.left), (float)(rcExt.bottom - rcExt.top));
        for (int i = 0; i < 16; ++i) m_wwi.constraint[i] = false;
    }

    POINT pt; GetCursorPos(&pt);
    float x_inc = (rcExt.right - rcExt.left) / 3.0f;
    float y_inc = (rcExt.bottom - rcExt.top) / 3.0f;
    int indx = (int)((pt.x - rcExt.left) / x_inc + 0.5f);
    int indy = (int)((pt.y - rcExt.top)  / y_inc + 0.5f);
    if (indx < 0) indx = 0; if (indx > 3) indx = 3;
    if (indy < 0) indy = 0; if (indy > 3) indy = 3;
    m_wwi.constraint[indy * 4 + indx] = true;

    m_isMoving.store(true);
    m_isSettling.store(false);

    ShowWindow(m_overlayHwnd, SW_SHOWNOACTIVATE);
    StartRenderThread();
}

void WobblyController::OnSizingMoving(HWND hwnd, LPRECT r) {
    std::lock_guard<std::recursive_mutex> lock(m_mutex);
    if (m_isMoving.load() && m_mainHwnd == hwnd) {
        RECT rExt, rWin;
        if (DwmGetWindowAttribute(hwnd, DWMWA_EXTENDED_FRAME_BOUNDS, &rExt, sizeof(rExt)) != S_OK) GetWindowRect(hwnd, &rExt);
        GetWindowRect(hwnd, &rWin);
        int dx = rExt.left - rWin.left;
        int dy = rExt.top - rWin.top;
        float w = (float)(r->right - r->left) - (rWin.right - rExt.right) - dx;
        float h = (float)(r->bottom - r->top) - (rWin.bottom - rExt.bottom) - dy;
        UpdateOrigin((float)r->left + dx, (float)r->top + dy, w, h);
    }
}

void WobblyController::OnWindowPosChanged(HWND hwnd, WINDOWPOS* wp) {
    std::lock_guard<std::recursive_mutex> lock(m_mutex);
    if (hwnd != m_mainHwnd || m_isMoving.load()) return;
    if (m_isSettling.load() && wp) {
        if ((wp->flags & SWP_NOMOVE) && (wp->flags & SWP_NOSIZE)) return;
        RECT rExt, rWin;
        if (DwmGetWindowAttribute(hwnd, DWMWA_EXTENDED_FRAME_BOUNDS, &rExt, sizeof(rExt)) == S_OK && GetWindowRect(hwnd, &rWin)) {
            int dx = rExt.left - rWin.left;
            int dy = rExt.top - rWin.top;
            float w = (float)wp->cx - (rWin.right - rExt.right) - dx;
            float h = (float)wp->cy - (rWin.bottom - rExt.bottom) - dy;
            UpdateOrigin((float)wp->x + dx, (float)wp->y + dy, w, h);
        }
    }
}

void WobblyController::OnExitSizeMove(HWND hwnd) {
    std::lock_guard<std::recursive_mutex> lock(m_mutex);
    if (m_isMoving.load() && m_mainHwnd == hwnd) {
        m_isMoving.store(false);
        m_isSettling.store(true);
        RECT rcExt;
        if (DwmGetWindowAttribute(hwnd, DWMWA_EXTENDED_FRAME_BOUNDS, &rcExt, sizeof(rcExt)) != S_OK) GetWindowRect(hwnd, &rcExt);
        UpdateOrigin((float)rcExt.left, (float)rcExt.top,
                     (float)(rcExt.right - rcExt.left), (float)(rcExt.bottom - rcExt.top));
    }
}

LRESULT WobblyController::ProcessOverlayMessage(HWND hwnd, UINT msg, WPARAM wp, LPARAM lp) {
    if (msg == WM_CLOSE) {
        DestroyWindow(hwnd);
        return 0;
    }
    return DefWindowProc(hwnd, msg, wp, lp);
}

// -------------------------------------------------------------------------
// Subclassing & hooks (unchanged)
// -------------------------------------------------------------------------
LRESULT CALLBACK WobblySubclassProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam, UINT_PTR uIdSubclass, DWORD_PTR dwRefData) {
    if (msg == g_wmDetach) {
        RemoveWindowSubclass(hwnd, WobblySubclassProc, uIdSubclass);
        return DefSubclassProc(hwnd, msg, wParam, lParam);
    }
    else if (msg == WM_ENTERSIZEMOVE) WobblyController::Instance().OnEnterSizeMove(hwnd);
    else if (msg == WM_SIZING || msg == WM_MOVING) WobblyController::Instance().OnSizingMoving(hwnd, (LPRECT)lParam);
    else if (msg == WM_WINDOWPOSCHANGED) WobblyController::Instance().OnWindowPosChanged(hwnd, (WINDOWPOS*)lParam);
    else if (msg == WM_EXITSIZEMOVE) WobblyController::Instance().OnExitSizeMove(hwnd);
    else if (msg == WM_NCDESTROY) RemoveWindowSubclass(hwnd, WobblySubclassProc, uIdSubclass);
    return DefSubclassProc(hwnd, msg, wParam, lParam);
}

static bool IsInternalModClass(HWND hwnd) {
    char className[256];
    if (GetClassNameA(hwnd, className, sizeof(className))) {
        if (strcmp(className, "WobblyOverlayWindow") == 0) return true;
    }
    return false;
}

HWND WINAPI CreateWindowExW_Hook(DWORD dwExStyle, LPCWSTR lpClassName, LPCWSTR lpWindowName, DWORD dwStyle, int X, int Y, int nWidth, int nHeight, HWND hWndParent, HMENU hMenu, HINSTANCE hInstance, LPVOID lpParam) {
    HWND hwnd = CreateWindowExW_Original(dwExStyle, lpClassName, lpWindowName, dwStyle, X, Y, nWidth, nHeight, hWndParent, hMenu, hInstance, lpParam);
    if (hwnd && (dwStyle & WS_CHILD) == 0 && !IsInternalModClass(hwnd)) SetWindowSubclass(hwnd, WobblySubclassProc, 0, 0);
    return hwnd;
}

HWND WINAPI CreateWindowExA_Hook(DWORD dwExStyle, LPCSTR lpClassName, LPCSTR lpWindowName, DWORD dwStyle, int X, int Y, int nWidth, int nHeight, HWND hWndParent, HMENU hMenu, HINSTANCE hInstance, LPVOID lpParam) {
    HWND hwnd = CreateWindowExA_Original(dwExStyle, lpClassName, lpWindowName, dwStyle, X, Y, nWidth, nHeight, hWndParent, hMenu, hInstance, lpParam);
    if (hwnd && (dwStyle & WS_CHILD) == 0 && !IsInternalModClass(hwnd)) SetWindowSubclass(hwnd, WobblySubclassProc, 0, 0);
    return hwnd;
}

static LRESULT CALLBACK CrossThreadHookProc(int nCode, WPARAM wParam, LPARAM lParam) {
    if (nCode == HC_ACTION) {
        CWPSTRUCT* pCwp = (CWPSTRUCT*)lParam;
        if (pCwp->message == g_wmAttach) SetWindowSubclass(pCwp->hwnd, WobblySubclassProc, 0, 0);
        else if (pCwp->message == g_wmDetach) RemoveWindowSubclass(pCwp->hwnd, WobblySubclassProc, 0);
    }
    return CallNextHookEx(NULL, nCode, wParam, lParam);
}

static BOOL CALLBACK AttachSubclassEnumProc(HWND hwnd, LPARAM lParam) {
    LONG_PTR style = GetWindowLongPtrW(hwnd, GWL_STYLE);
    if ((style & WS_CHILD) == 0) {
        DWORD tid = GetWindowThreadProcessId(hwnd, NULL);
        HHOOK hHook = SetWindowsHookEx(WH_CALLWNDPROC, CrossThreadHookProc, NULL, tid);
        if (hHook) {
            SendMessageTimeoutW(hwnd, g_wmAttach, 0, 0, SMTO_ABORTIFHUNG | SMTO_NORMAL, 500, NULL);
            UnhookWindowsHookEx(hHook);
        }
    }
    return TRUE;
}

static BOOL CALLBACK DetachSubclassEnumProc(HWND hwnd, LPARAM lParam) {
    LONG_PTR style = GetWindowLongPtrW(hwnd, GWL_STYLE);
    if ((style & WS_CHILD) == 0) SendMessageTimeoutW(hwnd, g_wmDetach, 0, 0, SMTO_ABORTIFHUNG | SMTO_NORMAL, 100, NULL);
    return TRUE;
}

void EnumerateAndSubclass(BOOL attach) {
    HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPTHREAD, 0);
    if (hSnapshot != INVALID_HANDLE_VALUE) {
        THREADENTRY32 te; te.dwSize = sizeof(THREADENTRY32);
        if (Thread32First(hSnapshot, &te)) {
            DWORD pid = GetCurrentProcessId();
            do {
                if (te.th32OwnerProcessID == pid) {
                    EnumThreadWindows(te.th32ThreadID, attach ? AttachSubclassEnumProc : DetachSubclassEnumProc, 0);
                }
            } while (Thread32Next(hSnapshot, &te));
        }
        CloseHandle(hSnapshot);
    }
}

// -------------------------------------------------------------------------
// Mod lifecycle
// -------------------------------------------------------------------------
BOOL Wh_ModInit() {
    Wh_Log(L"Wobbly Windows Mod Loading (v1.2.0)...");
    ConfigManager::Instance().Load();
    if (!g_hInstance) {
        GetModuleHandleExA(GET_MODULE_HANDLE_EX_FLAG_FROM_ADDRESS | GET_MODULE_HANDLE_EX_FLAG_UNCHANGED_REFCOUNT,
                           (LPCSTR)&Wh_ModInit, (HMODULE*)&g_hInstance);
    }
    g_isUnloading.store(false, std::memory_order_relaxed);

    WCHAR exePath[MAX_PATH];
    if (GetModuleFileNameW(NULL, exePath, MAX_PATH)) {
        _wcslwr_s(exePath);
        if (wcsstr(exePath, L"dwm.exe") || wcsstr(exePath, L"csrss.exe") ||
            wcsstr(exePath, L"lsass.exe") || wcsstr(exePath, L"winlogon.exe")) return FALSE;
    }

    D2DEngine::Instance().Initialize();
    g_wmAttach = RegisterWindowMessageW(L"WobblyWindows_Attach");
    g_wmDetach = RegisterWindowMessageW(L"WobblyWindows_Detach");

    Wh_SetFunctionHook((void*)CreateWindowExW, (void*)CreateWindowExW_Hook, (void**)&CreateWindowExW_Original);
    Wh_SetFunctionHook((void*)CreateWindowExA, (void*)CreateWindowExA_Hook, (void**)&CreateWindowExA_Original);

    EnumerateAndSubclass(TRUE);
    return TRUE;
}

void Wh_ModSettingsChanged() {
    ConfigManager::Instance().Load();
}

void Wh_ModUninit() {
    Wh_Log(L"Wobbly Windows Mod Unloading...");
    EnumerateAndSubclass(FALSE);
    g_isUnloading.store(true, std::memory_order_relaxed);
    WobblyController::Instance().Shutdown();
    if (g_hInstance) {
        UnregisterClassA("WobblyOverlayWindow", g_hInstance);
        g_hInstance = NULL;
    }
    D2DEngine::Instance().Shutdown();
}
