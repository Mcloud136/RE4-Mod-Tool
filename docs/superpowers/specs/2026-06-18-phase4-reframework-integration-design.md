# RE4 Mod Tool - Phase 4: 3D Preview via REFramework Integration

> **日期：** 2026-06-18
> **状态：** 已批准
> **依赖：** Phase 1-3

## 项目概述

通过REFramework C++ DLL插件实现游戏内3D预览，与桌面Qt工具通过命名管道实时通信。用户在桌面工具编辑颜色，在游戏中即时看到效果。

**技术栈：** C++17 / REFramework Plugin API / ImGui / Qt 6.8 / Named Pipes

---

## 1. 整体架构

### 1.1 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                    RE4 游戏进程                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  re4modtool.dll (REFramework插件)                     │  │
│  │                                                       │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │  │
│  │  │ ImGui UI    │  │ 贴图热替换  │  │ 相机控制    │   │  │
│  │  │ (调色盘)    │  │ (材质系统)  │  │ (自由视角)  │   │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘   │  │
│  │         │                │                │           │  │
│  │         └────────────────┼────────────────┘           │  │
│  │                          │                            │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │              命名管道客户端                      │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            ↕ 命名管道 (双向)
┌─────────────────────────────────────────────────────────────┐
│                    RE4 Mod Tool (桌面应用)                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │  │
│  │  │ Qt调色盘    │  │ 文件管理    │  │ 批量处理    │   │  │
│  │  │ (完整UI)    │  │ (格式转换)  │  │ (预设系统)  │   │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘   │  │
│  │         │                │                │           │  │
│  │         └────────────────┼────────────────┘           │  │
│  │                          │                            │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │              命名管道服务器                      │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 模块职责

| 模块 | 运行位置 | 职责 |
|------|---------|------|
| ImGui UI | 游戏内 | 基础调色盘、实时预览、相机控制 |
| 贴图热替换 | 游戏内 | 监听管道消息，替换游戏材质贴图 |
| 相机控制 | 游戏内 | 自由视角、预设角度、缩放 |
| 命名管道 | 双向 | 消息传递、状态同步 |
| Qt调色盘 | 桌面 | 完整调色盘（HSL、通道、预设） |
| 文件管理 | 桌面 | 格式转换、文件打开/保存 |
| 批量处理 | 桌面 | 批量转换、批量应用 |

### 1.3 数据流

```
用户在Qt调色盘调整颜色
    ↓
Qt调色盘发射 colorChanged 信号
    ↓
MainWindow 通过命名管道发送 COLOR_CHANGED 消息
    ↓
DLL插件接收消息
    ↓
DLL插件更新ImGui显示
    ↓
DLL插件调用 ResourceManager 替换贴图
    ↓
游戏引擎重新渲染
    ↓
用户在游戏中看到效果
```

---

## 2. 命名管道通信协议

### 2.1 管道配置

```
管道名称：\\.\pipe\RE4ModTool
模式：双向 (PIPE_ACCESS_DUPLEX)
缓冲区：64KB
超时：5000ms
```

### 2.2 消息格式

```cpp
struct MessageHeader {
    uint32_t magic;      // 0x5245344D ("RE4M")
    uint32_t version;    // 协议版本 (1)
    uint32_t type;       // 消息类型
    uint32_t size;       // 数据大小（不含头）
};
```

### 2.3 消息类型

```cpp
enum class MessageType : uint32_t {
    // 桌面 → 游戏
    COLOR_CHANGED       = 0x0001,
    TEXTURE_LOADED      = 0x0002,
    CAMERA_PRESET       = 0x0003,
    LIGHT_CHANGED       = 0x0004,
    TEXTURE_FILE_PATH   = 0x0005,
    UNDO_REQUEST        = 0x0006,
    REDO_REQUEST        = 0x0007,
    SAVE_TEXTURE        = 0x0008,
    PING                = 0x0009,
    
    // 游戏 → 桌面
    PONG                = 0x1001,
    TEXTURE_INFO        = 0x1002,
    CAMERA_STATE        = 0x1003,
    GAME_STATUS         = 0x1004,
    PIXEL_COLOR         = 0x1005,
    ERROR               = 0x1006,
};
```

### 2.4 消息数据结构

```cpp
struct ColorChangedData {
    int32_t r, g, b, a;
    bool applyToAll;
};

struct TextureFilePathData {
    char path[260];
    char format[8];
};

struct CameraPresetData {
    enum Preset : uint32_t {
        FRONT = 0, BACK = 1, LEFT = 2, RIGHT = 3,
        TOP = 4, BOTTOM = 5, FREE = 6,
    };
    Preset preset;
    float distance;
    float fov;
};

struct LightChangedData {
    float ambientIntensity;
    float directionalIntensity;
    float directionalAngleX, directionalAngleY;
    float pointLightIntensity;
    float pointLightX, pointLightY, pointLightZ;
};

struct TextureInfoData {
    uint32_t width, height, format, mipLevels;
    char name[64];
};

struct CameraStateData {
    float positionX, positionY, positionZ;
    float rotationX, rotationY, rotationZ;
    float fov;
};

struct GameStatusData {
    bool isPaused, isPhotoMode, isLoaded;
    char currentScene[64];
};

struct PixelColorData {
    int32_t x, y, r, g, b, a;
};

struct ErrorData {
    int32_t code;
    char message[256];
};
```

---

## 3. DLL插件设计

### 3.1 插件结构

```
re4modtool_dll/
├── CMakeLists.txt
├── include/
│   └── reframework/
│       ├── API.h
│       └── API.hpp
├── src/
│   ├── plugin.cpp              # 插件入口
│   ├── plugin.h
│   ├── imgui_ui.h/.cpp         # ImGui界面
│   ├── texture_swapper.h/.cpp  # 贴图热替换
│   ├── camera_controller.h/.cpp # 相机控制
│   ├── pipe_client.h/.cpp      # 命名管道客户端
│   └── rendering/
│       ├── d3d11.h/.cpp        # D3D11渲染
│       └── d3d12.h/.cpp        # D3D12渲染
└── imgui/                      # ImGui源码
```

### 3.2 插件入口

```cpp
// plugin.cpp
extern "C" __declspec(dllexport) void reframework_plugin_required_version(
    REFrameworkPluginVersion* version) {
    version->major = REFRAMEWORK_PLUGIN_VERSION_MAJOR;
    version->minor = REFRAMEWORK_PLUGIN_VERSION_MINOR;
    version->patch = REFRAMEWORK_PLUGIN_VERSION_PATCH;
}

extern "C" __declspec(dllexport) bool reframework_plugin_initialize(
    const REFrameworkPluginInitializeParam* param) {
    API::initialize(param);
    
    const auto functions = param->functions;
    functions->on_present(on_present);
    functions->on_message((REFOnMessageCb)on_message);
    functions->on_device_reset(on_device_reset);
    
    // 启动命名管道客户端
    PipeClient::get().start();
    
    return true;
}
```

### 3.3 ImGui界面

```cpp
void on_present() {
    if (!g_initialized) {
        initialize_imgui();
        g_initialized = true;
    }
    
    // NewFrame
    ImGui_ImplDX11_NewFrame(); // 或 DX12
    ImGui_ImplWin32_NewFrame();
    ImGui::NewFrame();
    
    // 调色盘窗口
    if (ImGui::Begin("RE4 Mod Tool - Color Editor")) {
        // RGB滑块
        static int r = 128, g = 128, b = 128, a = 255;
        ImGui::SliderInt("Red", &r, 0, 255);
        ImGui::SliderInt("Green", &g, 0, 255);
        ImGui::SliderInt("Blue", &b, 0, 255);
        ImGui::SliderInt("Alpha", &a, 0, 255);
        
        // 颜色预览
        ImVec4 color(r/255.0f, g/255.0f, b/255.0f, a/255.0f);
        ImGui::ColorEdit4("Preview", (float*)&color);
        
        // 应用按钮
        if (ImGui::Button("Apply to Texture")) {
            TextureSwapper::get().applyColor(r, g, b, a);
        }
        
        ImGui::Separator();
        
        // 相机控制
        ImGui::Text("Camera Controls");
        if (ImGui::Button("Front")) CameraController::get().setPreset(CameraPresetData::FRONT);
        ImGui::SameLine();
        if (ImGui::Button("Back")) CameraController::get().setPreset(CameraPresetData::BACK);
        ImGui::SameLine();
        if (ImGui::Button("Left")) CameraController::get().setPreset(CameraPresetData::LEFT);
        ImGui::SameLine();
        if (ImGui::Button("Right")) CameraController::get().setPreset(CameraPresetData::RIGHT);
        
        // 连接状态
        ImGui::Separator();
        bool connected = PipeClient::get().isConnected();
        ImGui::Text("Desktop Tool: %s", connected ? "Connected" : "Disconnected");
    }
    ImGui::End();
    
    // 渲染
    ImGui::EndFrame();
    ImGui::Render();
    g_d3d11.render_imgui(); // 或 g_d3d12
}
```

### 3.4 贴图热替换

```cpp
class TextureSwapper {
public:
    static TextureSwapper& get() {
        static TextureSwapper instance;
        return instance;
    }
    
    void applyColor(int r, int g, int b, int a) {
        auto& api = API::get();
        auto rm = api.resource_manager();
        
        // 获取当前场景中的角色材质
        auto scene_mgr = api.get_native_singleton("via.SceneManager");
        // ... 查找角色GameObject
        // ... 获取Renderer组件
        // ... 获取Material
        // ... 替换贴图颜色
    }
    
    void loadTexture(const std::string& path) {
        auto& api = API::get();
        auto rm = api.resource_manager();
        
        // 使用ResourceManager加载新贴图
        auto texture = rm->create_resource(
            api.tdb()->find_type("via.render.Texture"),
            std::wstring(path.begin(), path.end())
        );
        
        // 替换到材质上
        applyTextureToMaterial(texture);
    }
    
private:
    void applyTextureToMaterial(sdk::Resource* texture) {
        // 查找目标材质并替换贴图
        // 具体实现需要根据RE4的材质结构
    }
};
```

### 3.5 相机控制

```cpp
class CameraController {
public:
    static CameraController& get() {
        static CameraController instance;
        return instance;
    }
    
    void setPreset(CameraPresetData::Preset preset) {
        auto& api = API::get();
        auto scene_mgr = api.get_native_singleton("via.SceneManager");
        
        // 获取相机
        // 设置位置和旋转
        switch (preset) {
            case CameraPresetData::FRONT:
                setPosition(0, 1.5f, -3.0f);
                setRotation(0, 0, 0);
                break;
            case CameraPresetData::BACK:
                setPosition(0, 1.5f, 3.0f);
                setRotation(0, 180, 0);
                break;
            // ... 其他预设
        }
    }
    
    void setPosition(float x, float y, float z) {
        // 通过REFramework API设置相机位置
    }
    
    void setRotation(float x, float y, float z) {
        // 通过REFramework API设置相机旋转
    }
    
    void setFov(float fov) {
        // 设置视场角
    }
};
```

### 3.6 命名管道客户端

```cpp
class PipeClient {
public:
    static PipeClient& get() {
        static PipeClient instance;
        return instance;
    }
    
    void start() {
        m_thread = std::thread([this]() {
            while (m_running) {
                if (!connect()) {
                    std::this_thread::sleep_for(std::chrono::seconds(1));
                    continue;
                }
                
                while (m_connected && m_running) {
                    processMessages();
                }
            }
        });
    }
    
    void stop() {
        m_running = false;
        if (m_thread.joinable()) m_thread.join();
    }
    
    bool isConnected() const { return m_connected; }
    
    void send(MessageType type, const void* data, uint32_t size) {
        std::lock_guard<std::mutex> lock(m_mutex);
        
        MessageHeader header;
        header.magic = 0x5245344D;
        header.version = 1;
        header.type = static_cast<uint32_t>(type);
        header.size = size;
        
        DWORD bytesWritten;
        WriteFile(m_pipe, &header, sizeof(header), &bytesWritten, nullptr);
        if (size > 0 && data) {
            WriteFile(m_pipe, data, size, &bytesWritten, nullptr);
        }
    }
    
private:
    bool connect() {
        m_pipe = CreateFileA(
            "\\\\.\\pipe\\RE4ModTool",
            GENERIC_READ | GENERIC_WRITE,
            0, nullptr, OPEN_EXISTING, 0, nullptr
        );
        
        if (m_pipe == INVALID_HANDLE_VALUE) {
            return false;
        }
        
        m_connected = true;
        return true;
    }
    
    void processMessages() {
        MessageHeader header;
        DWORD bytesRead;
        
        if (!ReadFile(m_pipe, &header, sizeof(header), &bytesRead, nullptr)) {
            m_connected = false;
            return;
        }
        
        if (header.magic != 0x5245344D) {
            m_connected = false;
            return;
        }
        
        std::vector<uint8_t> data(header.size);
        if (header.size > 0) {
            ReadFile(m_pipe, data.data(), header.size, &bytesRead, nullptr);
        }
        
        handleMessage(static_cast<MessageType>(header.type), data.data(), header.size);
    }
    
    void handleMessage(MessageType type, const void* data, uint32_t size) {
        switch (type) {
            case MessageType::COLOR_CHANGED: {
                auto* color = static_cast<const ColorChangedData*>(data);
                TextureSwapper::get().applyColor(color->r, color->g, color->b, color->a);
                break;
            }
            case MessageType::TEXTURE_FILE_PATH: {
                auto* path = static_cast<const TextureFilePathData*>(data);
                TextureSwapper::get().loadTexture(path->path);
                break;
            }
            case MessageType::CAMERA_PRESET: {
                auto* preset = static_cast<const CameraPresetData*>(data);
                CameraController::get().setPreset(preset->preset);
                break;
            }
            case MessageType::PING: {
                send(MessageType::PONG, nullptr, 0);
                break;
            }
            // ... 其他消息处理
        }
    }
    
    HANDLE m_pipe = INVALID_HANDLE_VALUE;
    std::thread m_thread;
    std::mutex m_mutex;
    bool m_running = true;
    bool m_connected = false;
};
```

---

## 4. 桌面工具集成

### 4.1 命名管道服务器

```cpp
// src/network/pipeserver.h
class PipeServer : public QObject {
    Q_OBJECT
    
public:
    explicit PipeServer(QObject* parent = nullptr);
    ~PipeServer();
    
    void start();
    void stop();
    bool isConnected() const;
    
    void sendColorChanged(int r, int g, int b, int a);
    void sendTexturePath(const QString& path);
    void sendCameraPreset(int preset);
    void sendLightChanged(const LightSettings& settings);
    void sendPing();
    
signals:
    void connected();
    void disconnected();
    void textureInfoReceived(int width, int height, int format);
    void cameraStateReceived(float x, float y, float z, float rx, float ry, float rz);
    void pixelColorReceived(int x, int y, int r, int g, int b, int a);
    void gameStatusReceived(bool paused, bool loaded);
    void errorReceived(int code, const QString& message);
    
private:
    void processMessages();
    void handleMessage(MessageType type, const void* data, uint32_t size);
    
    HANDLE m_pipe = INVALID_HANDLE_VALUE;
    std::thread m_thread;
    std::mutex m_mutex;
    bool m_running = false;
    bool m_connected = false;
};
```

### 4.2 MainWindow集成

```cpp
// 在MainWindow中添加管道服务器
class MainWindow : public QMainWindow {
    // ... 现有代码 ...
    
private:
    PipeServer* m_pipeServer;
    
    void setupPipeServer() {
        m_pipeServer = new PipeServer(this);
        
        // 连接信号
        connect(m_pipeServer, &PipeServer::connected, this, [this]() {
            statusBar()->showMessage("Game connected");
        });
        
        connect(m_pipeServer, &PipeServer::disconnected, this, [this]() {
            statusBar()->showMessage("Game disconnected");
        });
        
        // 颜色变化 → 发送到游戏
        connect(m_rgbSliders, &RGBSliderWidget::colorChanged, this,
            [this](const QColor& color) {
                if (m_pipeServer->isConnected()) {
                    m_pipeServer->sendColorChanged(
                        color.red(), color.green(), color.blue(), color.alpha()
                    );
                }
            });
        
        // 接收游戏像素颜色
        connect(m_pipeServer, &PipeServer::pixelColorReceived, this,
            [this](int x, int y, int r, int g, int b, int a) {
                m_rgbSliders->setColor(QColor(r, g, b, a));
            });
        
        m_pipeServer->start();
    }
};
```

---

## 5. 构建配置

### 5.1 DLL项目CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.21)
project(re4modtool_dll LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)

# ImGui源码
set(IMGUI_SOURCES
    imgui/imgui.cpp
    imgui/imgui_demo.cpp
    imgui/imgui_draw.cpp
    imgui/imgui_tables.cpp
    imgui/imgui_widgets.cpp
    imgui/imgui_impl_win32.cpp
    imgui/imgui_impl_dx11.cpp
    imgui/imgui_impl_dx12.cpp
)

# 插件源码
set(PLUGIN_SOURCES
    src/plugin.cpp
    src/imgui_ui.cpp
    src/texture_swapper.cpp
    src/camera_controller.cpp
    src/pipe_client.cpp
    src/rendering/d3d11.cpp
    src/rendering/d3d12.cpp
)

add_library(re4modtool_dll SHARED ${IMGUI_SOURCES} ${PLUGIN_SOURCES})

target_include_directories(re4modtool_dll PRIVATE
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_SOURCE_DIR}/imgui
)

target_link_libraries(re4modtool_dll PRIVATE
    d3d11.lib
    d3d12.lib
    dxgi.lib
)

# 输出到REFramework插件目录
set_target_properties(re4modtool_dll PROPERTIES
    OUTPUT_NAME "re4modtool"
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_SOURCE_DIR}/output"
)
```

### 5.2 安装目录结构

```
RE4游戏目录/
├── dinput8.dll              # REFramework
├── reframework/
│   ├── plugins/
│   │   └── re4modtool.dll   # 我们的插件
│   └── autorun/
└── ...
```

---

## 6. 开发计划

### 阶段1：基础设施（1-2周）
- 创建DLL项目结构
- 集成ImGui
- 实现REFramework插件入口
- 实现命名管道通信

### 阶段2：ImGui界面（1-2周）
- 实现基础调色盘UI
- 实现相机控制面板
- 实现连接状态显示

### 阶段3：贴图热替换（1-2周）
- 研究RE4材质系统
- 实现贴图加载和替换
- 实时预览效果

### 阶段4：桌面工具集成（1-2周）
- 实现命名管道服务器
- 集成到MainWindow
- 双向通信测试

### 阶段5：高级功能（1周）
- 光照调整
- 相机预设
- 取色器

**总开发周期：6-8周**

---

## 7. 已知限制

1. **需要游戏运行** — DLL插件依赖REFramework和游戏进程
2. **仅支持RE4** — 针对RE4的材质系统优化
3. **贴图替换范围** — 需要逆向具体的材质和贴图槽位
4. **性能开销** — ImGui渲染和管道通信有少量性能开销
5. **调试困难** — 需要附加到游戏进程调试

---

## 8. 未来扩展

1. **支持更多游戏** — RE2/RE3重置版
2. **游戏内完整调色盘** — 在ImGui中实现更多编辑功能
3. **实时3D预览窗口** — 在桌面工具中显示游戏画面
4. **自动化脚本** — Lua脚本支持
5. **社区预设分享** — 在线预设库
