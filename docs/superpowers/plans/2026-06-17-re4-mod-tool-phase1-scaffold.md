# RE4 Mod Tool - Phase 1: Project Scaffold & Core Infrastructure

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Set up the build system, Qt application skeleton, and core infrastructure (logger, settings, file/image/math utilities) that all subsequent modules depend on.

**Architecture:** CMake + vcpkg builds a Qt 6.8 Widgets application with DirectX 11/12 support. Core utilities (logger, settings, file I/O) are header/source pairs in `src/core/` and `src/utils/`, tested via Google Test.

**Tech Stack:** C++17, Qt 6.8, CMake 3.21+, vcpkg, Google Test, DirectXTex, DirectXMath

**Spec Reference:** `docs/superpowers/specs/2026-06-17-re4-mod-tool-design.md` — Sections 1, 8.2, 8.3, 9.3, 10, 11

---

## File Structure (Phase 1)

| File | Responsibility |
|------|---------------|
| `CMakeLists.txt` | Root build config: Qt6, DirectX, vcpkg, tests |
| `vcpkg.json` | Dependency manifest for vcpkg |
| `CMakePresets.json` | Build presets for Debug/Release |
| `.gitignore` | C++, Qt, CMake, vcpkg ignores |
| `src/main.cpp` | Application entry point |
| `src/core/application.h/.cpp` | QApplication subclass, global init/shutdown |
| `src/core/logger.h/.cpp` | Multi-level file+console logger with rotation |
| `src/core/settings.h/.cpp` | QSettings wrapper for app preferences |
| `src/utils/fileutils.h/.cpp` | File I/O helpers, path manipulation, temp files |
| `src/utils/imageutils.h/.cpp` | QImage helpers: channel extraction, format conversion |
| `src/utils/mathutils.h/.cpp` | Math helpers: clamp, lerp, color space conversions |
| `tests/unit/test_logger.cpp` | Logger unit tests |
| `tests/unit/test_settings.cpp` | Settings unit tests |
| `tests/unit/test_fileutils.cpp` | FileUtils unit tests |
| `tests/unit/test_imageutils.cpp` | ImageUtils unit tests |
| `tests/unit/test_mathutils.cpp` | MathUtils unit tests |
| `tests/CMakeLists.txt` | Test build config |

---

### Task 1: Initialize Git Repository and .gitignore

**Files:**
- Create: `.gitignore`

- [ ] **Step 1: Create .gitignore**

```
# Build directories
build/
out/
cmake-build-*/

# CMake
CMakeFiles/
CMakeCache.txt
cmake_install.cmake
Makefile

# vcpkg
vcpkg_installed/

# Qt
*.pro.user
*.qbs.user
*.moc
moc_*.cpp
moc_*.h
qrc_*.cpp
ui_*.h
*.autosave

# C++ objects and binaries
*.o
*.obj
*.so
*.dll
*.dylib
*.exe
*.out
*.a
*.lib

# IDE
.vs/
.vscode/
*.suo
*.user
*.sln.docstates
.idea/
*.swp
*~

# OS
Thumbs.db
.DS_Store

# Test output
Testing/
*.gcda
*.gcno
*.gcov
```

- [ ] **Step 2: Initialize git repo**

Run:
```bash
cd D:/wxmuma
git init
git add .gitignore
git commit -m "chore: initialize repository with .gitignore"
```

- [ ] **Step 3: Verify**

Run: `git log --oneline`
Expected: one commit "chore: initialize repository with .gitignore"

---

### Task 2: CMake Root Configuration with vcpkg

**Files:**
- Create: `CMakeLists.txt`
- Create: `vcpkg.json`
- Create: `CMakePresets.json`

- [ ] **Step 1: Create vcpkg.json dependency manifest**

```json
{
  "$schema": "https://raw.githubusercontent.com/microsoft/vcpkg-tool/main/docs/vcpkg.schema.json",
  "name": "re4modtool",
  "version": "1.0.0",
  "description": "RE4 Remake costume mod color customization tool",
  "dependencies": [
    "directxtex",
    "gtest"
  ]
}
```

- [ ] **Step 2: Create CMakePresets.json**

```json
{
  "version": 6,
  "configurePresets": [
    {
      "name": "vcpkg-debug",
      "displayName": "Debug (vcpkg)",
      "generator": "Visual Studio 17 2022",
      "architecture": "x64",
      "binaryDir": "${sourceDir}/build/debug",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug",
        "CMAKE_TOOLCHAIN_FILE": "${sourceDir}/vcpkg/scripts/buildsystems/vcpkg.cmake"
      }
    },
    {
      "name": "vcpkg-release",
      "displayName": "Release (vcpkg)",
      "generator": "Visual Studio 17 2022",
      "architecture": "x64",
      "binaryDir": "${sourceDir}/build/release",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Release",
        "CMAKE_TOOLCHAIN_FILE": "${sourceDir}/vcpkg/scripts/buildsystems/vcpkg.cmake"
      }
    }
  ],
  "buildPresets": [
    {
      "name": "debug",
      "configurePreset": "vcpkg-debug"
    },
    {
      "name": "release",
      "configurePreset": "vcpkg-release"
    }
  ]
}
```

- [ ] **Step 3: Create root CMakeLists.txt**

```cmake
cmake_minimum_required(VERSION 3.21)
project(RE4ModTool
    VERSION 1.0.0
    DESCRIPTION "RE4 Remake costume mod color customization tool"
    LANGUAGES CXX
)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

# ── Qt 6 ────────────────────────────────────────────────
find_package(Qt6 REQUIRED COMPONENTS Core Gui Widgets 3DCore 3DRender 3DExtras)
qt_standard_project_setup()

# ── DirectXTex ──────────────────────────────────────────
find_package(directxtex CONFIG REQUIRED)

# ── Application target ──────────────────────────────────
set(APP_SOURCES
    src/main.cpp
    src/core/application.cpp
    src/core/logger.cpp
    src/core/settings.cpp
    src/utils/fileutils.cpp
    src/utils/imageutils.cpp
    src/utils/mathutils.cpp
)

set(APP_HEADERS
    src/core/application.h
    src/core/logger.h
    src/core/settings.h
    src/utils/fileutils.h
    src/utils/imageutils.h
    src/utils/mathutils.h
)

add_executable(${PROJECT_NAME} WIN32 ${APP_SOURCES} ${APP_HEADERS})

target_include_directories(${PROJECT_NAME} PRIVATE ${CMAKE_SOURCE_DIR}/src)

target_link_libraries(${PROJECT_NAME} PRIVATE
    Qt6::Core
    Qt6::Gui
    Qt6::Widgets
    Qt6::3DCore
    Qt6::3DRender
    Qt6::3DExtras
    Microsoft::DirectXTex
)

# ── Tests ───────────────────────────────────────────────
enable_testing()
add_subdirectory(tests)
```

- [ ] **Step 4: Verify CMake configuration**

Run:
```bash
cd D:/wxmuma
cmake --preset vcpkg-debug
```
Expected: CMake configures successfully (it will fail to build until sources exist, but configure should pass with warnings about missing source files — that's expected at this point).

- [ ] **Step 5: Commit**

```bash
git add CMakeLists.txt vcpkg.json CMakePresets.json
git commit -m "chore: add CMake build system with vcpkg and Qt6/DirectX dependencies"
```

---

### Task 3: Application Entry Point and Application Class

**Files:**
- Create: `src/main.cpp`
- Create: `src/core/application.h`
- Create: `src/core/application.cpp`

- [ ] **Step 1: Create Application header**

```cpp
// src/core/application.h
#pragma once

#include <QApplication>
#include <memory>

namespace re4 {

class Logger;
class Settings;

class Application : public QApplication {
    Q_OBJECT

public:
    Application(int& argc, char** argv);
    ~Application() override;

    static Application* instance();
    Logger* logger() const;
    Settings* settings() const;

private:
    void initializeLogger();
    void initializeSettings();

    std::unique_ptr<Logger> m_logger;
    std::unique_ptr<Settings> m_settings;
};

} // namespace re4
```

- [ ] **Step 2: Create Application implementation**

```cpp
// src/core/application.cpp
#include "core/application.h"
#include "core/logger.h"
#include "core/settings.h"
#include <QDir>

namespace re4 {

Application::Application(int& argc, char** argv)
    : QApplication(argc, argv)
{
    setApplicationName("RE4ModTool");
    setApplicationVersion("1.0.0");
    setOrganizationName("RE4ModTool");
    setOrganizationDomain("re4modtool.local");

    initializeLogger();
    initializeSettings();
}

Application::~Application() = default;

Application* Application::instance()
{
    return static_cast<Application*>(QApplication::instance());
}

Logger* Application::logger() const { return m_logger.get(); }
Settings* Application::settings() const { return m_settings.get(); }

void Application::initializeLogger()
{
    m_logger = std::make_unique<Logger>();
    m_logger->setLogFile(
        QDir(QStandardPaths::writableLocation(QStandardPaths::AppDataLocation))
            .filePath("re4modtool.log"));
    m_logger->setLogLevel(Logger::Debug);
    m_logger->info("Application started, version: " + applicationVersion());
}

void Application::initializeSettings()
{
    m_settings = std::make_unique<Settings>();
}

} // namespace re4
```

- [ ] **Step 3: Create main.cpp**

```cpp
// src/main.cpp
#include "core/application.h"
#include <QMainWindow>
#include <QLabel>

int main(int argc, char* argv[])
{
    re4::Application app(argc, argv);

    // Temporary main window — replaced in later phases
    QMainWindow window;
    window.setWindowTitle("RE4 Mod Tool v1.0.0");
    window.resize(1280, 800);
    window.setCentralWidget(new QLabel("RE4 Mod Tool — Under Construction"));
    window.show();

    return app.exec();
}
```

- [ ] **Step 4: Build and verify the skeleton compiles**

Run:
```bash
cd D:/wxmuma
cmake --build build/debug --config Debug --target RE4ModTool
```
Expected: Build succeeds (warnings about missing logger/settings implementations are expected — those files are stubs at this point).

- [ ] **Step 5: Commit**

```bash
git add src/main.cpp src/core/application.h src/core/application.cpp
git commit -m "feat: add application entry point and Application class skeleton"
```

---

### Task 4: Logger System (TDD)

**Files:**
- Create: `tests/unit/test_logger.cpp`
- Create: `tests/CMakeLists.txt`
- Create: `src/core/logger.h`
- Create: `src/core/logger.cpp`

- [ ] **Step 1: Create tests/CMakeLists.txt**

```cmake
find_package(GTest CONFIG REQUIRED)

add_executable(re4modtool_tests
    unit/test_logger.cpp
    unit/test_settings.cpp
    unit/test_fileutils.cpp
    unit/test_imageutils.cpp
    unit/test_mathutils.cpp
)

target_include_directories(re4modtool_tests PRIVATE ${CMAKE_SOURCE_DIR}/src)

target_link_libraries(re4modtool_tests PRIVATE
    GTest::gtest
    GTest::gtest_main
    Qt6::Core
    Qt6::Gui
    Microsoft::DirectXTex
)

# Link the non-main sources (everything except main.cpp)
target_sources(re4modtool_tests PRIVATE
    ${CMAKE_SOURCE_DIR}/src/core/logger.cpp
    ${CMAKE_SOURCE_DIR}/src/core/settings.cpp
    ${CMAKE_SOURCE_DIR}/src/utils/fileutils.cpp
    ${CMAKE_SOURCE_DIR}/src/utils/imageutils.cpp
    ${CMAKE_SOURCE_DIR}/src/utils/mathutils.cpp
)

add_test(NAME UnitTests COMMAND re4modtool_tests)
```

- [ ] **Step 2: Write the failing logger tests**

```cpp
// tests/unit/test_logger.cpp
#include <gtest/gtest.h>
#include "core/logger.h"
#include <QTemporaryDir>
#include <QFile>
#include <QTextStream>

using namespace re4;

class LoggerTest : public ::testing::Test {
protected:
    void SetUp() override {
        m_tempDir = std::make_unique<QTemporaryDir>();
        ASSERT_TRUE(m_tempDir->isValid());
        m_logPath = m_tempDir->filePath("test.log");
    }

    std::unique_ptr<QTemporaryDir> m_tempDir;
    QString m_logPath;
};

TEST_F(LoggerTest, DefaultLogLevelIsInfo) {
    Logger logger;
    EXPECT_EQ(logger.logLevel(), Logger::Info);
}

TEST_F(LoggerTest, SetLogLevel) {
    Logger logger;
    logger.setLogLevel(Logger::Debug);
    EXPECT_EQ(logger.logLevel(), Logger::Debug);
}

TEST_F(LoggerTest, InfoMessageWrittenToFile) {
    Logger logger;
    logger.setLogFile(m_logPath);
    logger.setLogLevel(Logger::Debug);
    logger.info("hello world");

    QFile file(m_logPath);
    ASSERT_TRUE(file.open(QIODevice::ReadOnly | QIODevice::Text));
    QString content = QTextStream(&file).readAll();
    EXPECT_TRUE(content.contains("INFO"));
    EXPECT_TRUE(content.contains("hello world"));
}

TEST_F(LoggerTest, DebugMessageFilteredWhenLevelIsInfo) {
    Logger logger;
    logger.setLogFile(m_logPath);
    logger.setLogLevel(Logger::Info);
    logger.debug("should not appear");

    QFile file(m_logPath);
    ASSERT_TRUE(file.open(QIODevice::ReadOnly | QIODevice::Text));
    QString content = QTextStream(&file).readAll();
    EXPECT_FALSE(content.contains("should not appear"));
}

TEST_F(LoggerTest, WarningLevel) {
    Logger logger;
    logger.setLogFile(m_logPath);
    logger.setLogLevel(Logger::Warning);
    logger.warning("warn msg");
    logger.info("info msg");

    QFile file(m_logPath);
    ASSERT_TRUE(file.open(QIODevice::ReadOnly | QIODevice::Text));
    QString content = QTextStream(&file).readAll();
    EXPECT_TRUE(content.contains("warn msg"));
    EXPECT_FALSE(content.contains("info msg"));
}

TEST_F(LoggerTest, ErrorLevel) {
    Logger logger;
    logger.setLogFile(m_logPath);
    logger.setLogLevel(Logger::Error);
    logger.error("error msg");
    logger.warning("warn msg");

    QFile file(m_logPath);
    ASSERT_TRUE(file.open(QIODevice::ReadOnly | QIODevice::Text));
    QString content = QTextStream(&file).readAll();
    EXPECT_TRUE(content.contains("error msg"));
    EXPECT_FALSE(content.contains("warn msg"));
}

TEST_F(LoggerTest, CategoryIncludedInOutput) {
    Logger logger;
    logger.setLogFile(m_logPath);
    logger.setLogLevel(Logger::Debug);
    logger.info("test message", "FormatParser");

    QFile file(m_logPath);
    ASSERT_TRUE(file.open(QIODevice::ReadOnly | QIODevice::Text));
    QString content = QTextStream(&file).readAll();
    EXPECT_TRUE(content.contains("FormatParser"));
    EXPECT_TRUE(content.contains("test message"));
}

TEST_F(LoggerTest, LogRotationWhenFileExceedsMaxSize) {
    Logger logger;
    logger.setLogFile(m_logPath);
    logger.setMaxFileSize(512); // very small for testing
    logger.setLogLevel(Logger::Debug);

    // Write enough to trigger rotation
    for (int i = 0; i < 100; ++i) {
        logger.info(QString("message number %1 with padding data").arg(i));
    }

    // Original file should exist and be under max size
    QFile file(m_logPath);
    EXPECT_TRUE(file.exists());
}
```

- [ ] **Step 3: Run tests — verify they fail**

Run:
```bash
cd D:/wxmuma/build/debug
ctest --test-dir . -R test_logger --output-on-failure
```
Expected: Compilation failure — `core/logger.h` not found.

- [ ] **Step 4: Implement Logger header**

```cpp
// src/core/logger.h
#pragma once

#include <QObject>
#include <QString>
#include <QMutex>
#include <memory>

namespace re4 {

class Logger : public QObject {
    Q_OBJECT

public:
    enum Level {
        Debug = 0,
        Info = 1,
        Warning = 2,
        Error = 3,
        Fatal = 4
    };

    explicit Logger(QObject* parent = nullptr);
    ~Logger() override;

    void setLogLevel(Level level);
    Level logLevel() const;

    void setLogFile(const QString& filePath);
    void setMaxFileSize(qint64 bytes);

    void debug(const QString& message, const QString& category = "");
    void info(const QString& message, const QString& category = "");
    void warning(const QString& message, const QString& category = "");
    void error(const QString& message, const QString& category = "");
    void fatal(const QString& message, const QString& category = "");

    static QString levelToString(Level level);

signals:
    void messageLogged(Level level, const QString& message, const QString& category);

private:
    void writeLog(Level level, const QString& message, const QString& category);
    void rotateIfNeeded();

    Level m_level = Info;
    QString m_logFilePath;
    qint64 m_maxFileSize = 10 * 1024 * 1024; // 10 MB default
    QMutex m_mutex;
};

} // namespace re4
```

- [ ] **Step 5: Implement Logger source**

```cpp
// src/core/logger.cpp
#include "core/logger.h"
#include <QFile>
#include <QTextStream>
#include <QDateTime>
#include <QFileInfo>
#include <QDebug>

namespace re4 {

Logger::Logger(QObject* parent) : QObject(parent) {}

Logger::~Logger() = default;

void Logger::setLogLevel(Level level) {
    QMutexLocker locker(&m_mutex);
    m_level = level;
}

Logger::Level Logger::logLevel() const {
    return m_level;
}

void Logger::setLogFile(const QString& filePath) {
    QMutexLocker locker(&m_mutex);
    m_logFilePath = filePath;
}

void Logger::setMaxFileSize(qint64 bytes) {
    QMutexLocker locker(&m_mutex);
    m_maxFileSize = bytes;
}

void Logger::debug(const QString& message, const QString& category) {
    writeLog(Debug, message, category);
}

void Logger::info(const QString& message, const QString& category) {
    writeLog(Info, message, category);
}

void Logger::warning(const QString& message, const QString& category) {
    writeLog(Warning, message, category);
}

void Logger::error(const QString& message, const QString& category) {
    writeLog(Error, message, category);
}

void Logger::fatal(const QString& message, const QString& category) {
    writeLog(Fatal, message, category);
}

QString Logger::levelToString(Level level) {
    switch (level) {
        case Debug:   return "DEBUG";
        case Info:    return "INFO";
        case Warning: return "WARN";
        case Error:   return "ERROR";
        case Fatal:   return "FATAL";
    }
    return "UNKNOWN";
}

void Logger::writeLog(Level level, const QString& message, const QString& category) {
    if (level < m_level) return;

    QMutexLocker locker(&m_mutex);

    const QString timestamp = QDateTime::currentDateTime().toString("yyyy-MM-dd hh:mm:ss.zzz");
    const QString cat = category.isEmpty() ? "" : QString(" [%1]").arg(category);
    const QString line = QString("%1 %2%3 %4")
        .arg(timestamp, levelToString(level), cat, message);

    // Console output
    if (level >= Warning) {
        QTextStream(stderr) << line << "\n";
    } else {
        qDebug().noquote() << line;
    }

    // File output
    if (!m_logFilePath.isEmpty()) {
        rotateIfNeeded();
        QFile file(m_logFilePath);
        if (file.open(QIODevice::Append | QIODevice::Text)) {
            QTextStream stream(&file);
            stream << line << "\n";
        }
    }

    emit messageLogged(level, message, category);
}

void Logger::rotateIfNeeded() {
    QFileInfo info(m_logFilePath);
    if (!info.exists() || info.size() < m_maxFileSize)
        return;

    const QString backupPath = m_logFilePath + ".1";
    QFile::remove(backupPath);
    QFile::rename(m_logFilePath, backupPath);
}

} // namespace re4
```

- [ ] **Step 6: Build and run tests — verify they pass**

Run:
```bash
cd D:/wxmuma
cmake --build build/debug --config Debug --target re4modtool_tests
cd build/debug
ctest --test-dir . -R test_logger --output-on-failure
```
Expected: All 8 logger tests PASS.

- [ ] **Step 7: Commit**

```bash
git add src/core/logger.h src/core/logger.cpp tests/unit/test_logger.cpp tests/CMakeLists.txt
git commit -m "feat: add logger system with file output, level filtering, and rotation"
```

---

### Task 5: Settings System (TDD)

**Files:**
- Create: `tests/unit/test_settings.cpp`
- Create: `src/core/settings.h`
- Create: `src/core/settings.cpp`

- [ ] **Step 1: Write the failing settings tests**

```cpp
// tests/unit/test_settings.cpp
#include <gtest/gtest.h>
#include "core/settings.h"
#include <QTemporaryDir>

using namespace re4;

class SettingsTest : public ::testing::Test {
protected:
    void SetUp() override {
        m_tempDir = std::make_unique<QTemporaryDir>();
        ASSERT_TRUE(m_tempDir->isValid());
    }
    std::unique_ptr<QTemporaryDir> m_tempDir;
};

TEST_F(SettingsTest, DefaultValues) {
    Settings s;
    EXPECT_EQ(s.lastOpenDir(), QString());
    EXPECT_EQ(s.recentFiles().size(), 0);
    EXPECT_EQ(s.maxCacheSizeMB(), 100);
    EXPECT_EQ(s.autoCheckUpdates(), true);
}

TEST_F(SettingsTest, SetAndGetString) {
    Settings s;
    s.setLastOpenDir("C:\\Games\\RE4");
    EXPECT_EQ(s.lastOpenDir(), "C:\\Games\\RE4");
}

TEST_F(SettingsTest, RecentFilesMax10) {
    Settings s;
    for (int i = 0; i < 15; ++i) {
        s.addRecentFile(QString("file%1.tex").arg(i));
    }
    auto files = s.recentFiles();
    EXPECT_LE(files.size(), 10);
    EXPECT_EQ(files.first(), "file14.tex"); // most recent first
}

TEST_F(SettingsTest, AddDuplicateRecentFileMovesToTop) {
    Settings s;
    s.addRecentFile("a.tex");
    s.addRecentFile("b.tex");
    s.addRecentFile("a.tex"); // should move to top
    auto files = s.recentFiles();
    EXPECT_EQ(files.first(), "a.tex");
    EXPECT_EQ(files.size(), 2);
}

TEST_F(SettingsTest, SetMaxCacheSize) {
    Settings s;
    s.setMaxCacheSizeMB(256);
    EXPECT_EQ(s.maxCacheSizeMB(), 256);
}

TEST_F(SettingsTest, SetAutoCheckUpdates) {
    Settings s;
    s.setAutoCheckUpdates(false);
    EXPECT_EQ(s.autoCheckUpdates(), false);
}

TEST_F(SettingsTest, WindowGeometry) {
    Settings s;
    QByteArray geom = QByteArray::fromHex("01020304");
    s.setWindowGeometry(geom);
    EXPECT_EQ(s.windowGeometry(), geom);
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run:
```bash
ctest --test-dir build/debug -R test_settings --output-on-failure
```
Expected: Compilation failure — `core/settings.h` not found.

- [ ] **Step 3: Implement Settings header**

```cpp
// src/core/settings.h
#pragma once

#include <QObject>
#include <QSettings>
#include <QStringList>
#include <QByteArray>
#include <memory>

namespace re4 {

class Settings : public QObject {
    Q_OBJECT

public:
    explicit Settings(QObject* parent = nullptr);
    ~Settings() override;

    // Last used directories
    QString lastOpenDir() const;
    void setLastOpenDir(const QString& dir);
    QString lastSaveDir() const;
    void setLastSaveDir(const QString& dir);

    // Recent files (max 10, most recent first)
    QStringList recentFiles() const;
    void addRecentFile(const QString& filePath);
    void clearRecentFiles();

    // Cache
    int maxCacheSizeMB() const;
    void setMaxCacheSizeMB(int mb);

    // Updates
    bool autoCheckUpdates() const;
    void setAutoCheckUpdates(bool enabled);

    // Window state
    QByteArray windowGeometry() const;
    void setWindowGeometry(const QByteArray& geometry);
    QByteArray windowState() const;
    void setWindowState(const QByteArray& state);

private:
    std::unique_ptr<QSettings> m_settings;

    static constexpr int kMaxRecentFiles = 10;
};

} // namespace re4
```

- [ ] **Step 4: Implement Settings source**

```cpp
// src/core/settings.cpp
#include "core/settings.h"

namespace re4 {

Settings::Settings(QObject* parent)
    : QObject(parent)
    , m_settings(std::make_unique<QSettings>(
          QSettings::IniFormat, QSettings::UserScope,
          "RE4ModTool", "RE4ModTool"))
{}

Settings::~Settings() = default;

QString Settings::lastOpenDir() const {
    return m_settings->value("paths/lastOpenDir").toString();
}

void Settings::setLastOpenDir(const QString& dir) {
    m_settings->setValue("paths/lastOpenDir", dir);
}

QString Settings::lastSaveDir() const {
    return m_settings->value("paths/lastSaveDir").toString();
}

void Settings::setLastSaveDir(const QString& dir) {
    m_settings->setValue("paths/lastSaveDir", dir);
}

QStringList Settings::recentFiles() const {
    return m_settings->value("paths/recentFiles").toStringList();
}

void Settings::addRecentFile(const QString& filePath) {
    QStringList files = recentFiles();
    files.removeAll(filePath); // remove duplicate
    files.prepend(filePath);   // add to front
    while (files.size() > kMaxRecentFiles) {
        files.removeLast();
    }
    m_settings->setValue("paths/recentFiles", files);
}

void Settings::clearRecentFiles() {
    m_settings->remove("paths/recentFiles");
}

int Settings::maxCacheSizeMB() const {
    return m_settings->value("cache/maxSizeMB", 100).toInt();
}

void Settings::setMaxCacheSizeMB(int mb) {
    m_settings->setValue("cache/maxSizeMB", mb);
}

bool Settings::autoCheckUpdates() const {
    return m_settings->value("updates/autoCheck", true).toBool();
}

void Settings::setAutoCheckUpdates(bool enabled) {
    m_settings->setValue("updates/autoCheck", enabled);
}

QByteArray Settings::windowGeometry() const {
    return m_settings->value("window/geometry").toByteArray();
}

void Settings::setWindowGeometry(const QByteArray& geometry) {
    m_settings->setValue("window/geometry", geometry);
}

QByteArray Settings::windowState() const {
    return m_settings->value("window/state").toByteArray();
}

void Settings::setWindowState(const QByteArray& state) {
    m_settings->setValue("window/state", state);
}

} // namespace re4
```

- [ ] **Step 5: Build and run tests — verify they pass**

Run:
```bash
cd D:/wxmuma
cmake --build build/debug --config Debug --target re4modtool_tests
ctest --test-dir build/debug -R test_settings --output-on-failure
```
Expected: All 7 settings tests PASS.

- [ ] **Step 6: Commit**

```bash
git add src/core/settings.h src/core/settings.cpp tests/unit/test_settings.cpp
git commit -m "feat: add settings system with recent files, cache config, and window state"
```

---

### Task 6: File Utilities (TDD)

**Files:**
- Create: `tests/unit/test_fileutils.cpp`
- Create: `src/utils/fileutils.h`
- Create: `src/utils/fileutils.cpp`

- [ ] **Step 1: Write the failing file utilities tests**

```cpp
// tests/unit/test_fileutils.cpp
#include <gtest/gtest.h>
#include "utils/fileutils.h"
#include <QTemporaryDir>
#include <QTemporaryFile>
#include <QDir>

using namespace re4;

class FileUtilsTest : public ::testing::Test {
protected:
    void SetUp() override {
        m_tempDir = std::make_unique<QTemporaryDir>();
        ASSERT_TRUE(m_tempDir->isValid());
    }
    std::unique_ptr<QTemporaryDir> m_tempDir;
};

TEST_F(FileUtilsTest, FileExistsReturnsTrueForExistingFile) {
    QFile file(m_tempDir->filePath("test.txt"));
    ASSERT_TRUE(file.open(QIODevice::WriteOnly));
    file.write("hello");
    file.close();
    EXPECT_TRUE(FileUtils::fileExists(m_tempDir->filePath("test.txt")));
}

TEST_F(FileUtilsTest, FileExistsReturnsFalseForMissingFile) {
    EXPECT_FALSE(FileUtils::fileExists(m_tempDir->filePath("nonexistent.txt")));
}

TEST_F(FileUtilsTest, EnsureDirectoryCreatesNestedDirs) {
    QString path = m_tempDir->filePath("a/b/c");
    EXPECT_TRUE(FileUtils::ensureDirectory(path));
    EXPECT_TRUE(QDir(path).exists());
}

TEST_F(FileUtilsTest, FileExtensionExtraction) {
    EXPECT_EQ(FileUtils::extension("file.tex"), "tex");
    EXPECT_EQ(FileUtils::extension("file.tar.gz"), "gz");
    EXPECT_EQ(FileUtils::extension("noext"), "");
}

TEST_F(FileUtilsTest, FileBaseNameExtraction) {
    EXPECT_EQ(FileUtils::baseName("/path/to/file.tex"), "file");
    EXPECT_EQ(FileUtils::baseName("C:\\path\\to\\file.dds"), "file");
}

TEST_F(FileUtilsTest, ChangeExtension) {
    EXPECT_EQ(FileUtils::changeExtension("file.tex", "png"), "file.png");
    EXPECT_EQ(FileUtils::changeExtension("/path/file.dds", "tga"), "/path/file.tga");
}

TEST_F(FileUtilsTest, ReadWriteBinary) {
    QString path = m_tempDir->filePath("binary.dat");
    QByteArray data = QByteArray::fromHex("deadbeef");
    EXPECT_TRUE(FileUtils::writeBinary(path, data));
    QByteArray loaded = FileUtils::readBinary(path);
    EXPECT_EQ(loaded, data);
}

TEST_F(FileUtilsTest, ReadBinaryReturnsEmptyForMissingFile) {
    QByteArray loaded = FileUtils::readBinary(m_tempDir->filePath("nope.dat"));
    EXPECT_TRUE(loaded.isEmpty());
}

TEST_F(FileUtilsTest, TempFilePath) {
    QString tempPath = FileUtils::tempFilePath("test", ".tex");
    EXPECT_TRUE(tempPath.contains("test"));
    EXPECT_TRUE(tempPath.endsWith(".tex"));
}

TEST_F(FileUtilsTest, CopyFileOverwrite) {
    QString src = m_tempDir->filePath("src.txt");
    QString dst = m_tempDir->filePath("dst.txt");
    FileUtils::writeBinary(src, QByteArray("data"));
    EXPECT_TRUE(FileUtils::copyFile(src, dst, true));
    EXPECT_EQ(FileUtils::readBinary(dst), QByteArray("data"));
}

TEST_F(FileUtilsTest, CopyFileNoOverwrite) {
    QString src = m_tempDir->filePath("src.txt");
    QString dst = m_tempDir->filePath("dst.txt");
    FileUtils::writeBinary(src, QByteArray("data"));
    FileUtils::writeBinary(dst, QByteArray("existing"));
    EXPECT_FALSE(FileUtils::copyFile(src, dst, false));
    EXPECT_EQ(FileUtils::readBinary(dst), QByteArray("existing"));
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run:
```bash
ctest --test-dir build/debug -R test_fileutils --output-on-failure
```
Expected: Compilation failure.

- [ ] **Step 3: Implement FileUtils header**

```cpp
// src/utils/fileutils.h
#pragma once

#include <QString>
#include <QByteArray>

namespace re4 {
namespace FileUtils {

    bool fileExists(const QString& path);
    bool ensureDirectory(const QString& path);

    QString extension(const QString& filePath);
    QString baseName(const QString& filePath);
    QString changeExtension(const QString& filePath, const QString& newExt);

    QByteArray readBinary(const QString& path);
    bool writeBinary(const QString& path, const QByteArray& data);

    bool copyFile(const QString& source, const QString& dest, bool overwrite = true);
    bool removeFile(const QString& path);

    QString tempFilePath(const QString& prefix, const QString& suffix);

} // namespace FileUtils
} // namespace re4
```

- [ ] **Step 4: Implement FileUtils source**

```cpp
// src/utils/fileutils.cpp
#include "utils/fileutils.h"
#include <QFile>
#include <QFileInfo>
#include <QDir>
#include <QTemporaryFile>
#include <QUuid>

namespace re4 {
namespace FileUtils {

bool fileExists(const QString& path) {
    return QFileInfo::exists(path) && QFileInfo(path).isFile();
}

bool ensureDirectory(const QString& path) {
    QDir dir(path);
    if (dir.exists()) return true;
    return dir.mkpath(".");
}

QString extension(const QString& filePath) {
    return QFileInfo(filePath).suffix();
}

QString baseName(const QString& filePath) {
    return QFileInfo(filePath).baseName();
}

QString changeExtension(const QString& filePath, const QString& newExt) {
    QFileInfo info(filePath);
    return info.path() + "/" + info.completeBaseName() + "." + newExt;
}

QByteArray readBinary(const QString& path) {
    QFile file(path);
    if (!file.open(QIODevice::ReadOnly)) return {};
    return file.readAll();
}

bool writeBinary(const QString& path, const QByteArray& data) {
    ensureDirectory(QFileInfo(path).absolutePath());
    QFile file(path);
    if (!file.open(QIODevice::WriteOnly)) return false;
    return file.write(data) == data.size();
}

bool copyFile(const QString& source, const QString& dest, bool overwrite) {
    if (!overwrite && QFileInfo::exists(dest)) return false;
    QFile::remove(dest); // remove if exists for overwrite
    return QFile::copy(source, dest);
}

bool removeFile(const QString& path) {
    if (!QFileInfo::exists(path)) return true;
    return QFile::remove(path);
}

QString tempFilePath(const QString& prefix, const QString& suffix) {
    const QString dir = QDir::tempPath();
    const QString unique = QUuid::createUuid().toString(QUuid::Id128).left(8);
    return dir + "/" + prefix + "_" + unique + suffix;
}

} // namespace FileUtils
} // namespace re4
```

- [ ] **Step 5: Build and run tests — verify they pass**

Run:
```bash
cd D:/wxmuma
cmake --build build/debug --config Debug --target re4modtool_tests
ctest --test-dir build/debug -R test_fileutils --output-on-failure
```
Expected: All 11 file utils tests PASS.

- [ ] **Step 6: Commit**

```bash
git add src/utils/fileutils.h src/utils/fileutils.cpp tests/unit/test_fileutils.cpp
git commit -m "feat: add file utilities — path helpers, binary I/O, temp files"
```

---

### Task 7: Math Utilities (TDD)

**Files:**
- Create: `tests/unit/test_mathutils.cpp`
- Create: `src/utils/mathutils.h`
- Create: `src/utils/mathutils.cpp`

- [ ] **Step 1: Write the failing math utilities tests**

```cpp
// tests/unit/test_mathutils.cpp
#include <gtest/gtest.h>
#include "utils/mathutils.h"
#include <QColor>
#include <cmath>

using namespace re4;

TEST(MathUtilsTest, ClampInt) {
    EXPECT_EQ(MathUtils::clamp(5, 0, 10), 5);
    EXPECT_EQ(MathUtils::clamp(-1, 0, 10), 0);
    EXPECT_EQ(MathUtils::clamp(15, 0, 10), 10);
}

TEST(MathUtilsTest, ClampFloat) {
    EXPECT_FLOAT_EQ(MathUtils::clamp(0.5f, 0.0f, 1.0f), 0.5f);
    EXPECT_FLOAT_EQ(MathUtils::clamp(-1.0f, 0.0f, 1.0f), 0.0f);
    EXPECT_FLOAT_EQ(MathUtils::clamp(2.0f, 0.0f, 1.0f), 1.0f);
}

TEST(MathUtilsTest, LerpFloat) {
    EXPECT_FLOAT_EQ(MathUtils::lerp(0.0f, 10.0f, 0.0f), 0.0f);
    EXPECT_FLOAT_EQ(MathUtils::lerp(0.0f, 10.0f, 0.5f), 5.0f);
    EXPECT_FLOAT_EQ(MathUtils::lerp(0.0f, 10.0f, 1.0f), 10.0f);
}

TEST(MathUtilsTest, LerpColor) {
    QColor a(255, 0, 0, 255);
    QColor b(0, 0, 255, 255);
    QColor mid = MathUtils::lerpColor(a, b, 0.5f);
    EXPECT_NEAR(mid.red(), 127, 1);
    EXPECT_NEAR(mid.green(), 0, 1);
    EXPECT_NEAR(mid.blue(), 127, 1);
    EXPECT_EQ(mid.alpha(), 255);
}

TEST(MathUtilsTest, RgbToHslRoundTrip) {
    QColor original(128, 64, 192, 255);
    auto [h, s, l] = MathUtils::rgbToHsl(original);
    QColor converted = MathUtils::hslToRgb(h, s, l, 255);
    EXPECT_NEAR(converted.red(), original.red(), 2);
    EXPECT_NEAR(converted.green(), original.green(), 2);
    EXPECT_NEAR(converted.blue(), original.blue(), 2);
}

TEST(MathUtilsTest, AdjustBrightness) {
    QColor base(100, 100, 100, 255);
    QColor brighter = MathUtils::adjustBrightness(base, 50);
    EXPECT_GT(brighter.red(), base.red());
    EXPECT_GT(brighter.green(), base.green());
    EXPECT_GT(brighter.blue(), base.blue());
}

TEST(MathUtilsTest, AdjustBrightnessClamped) {
    QColor base(250, 250, 250, 255);
    QColor result = MathUtils::adjustBrightness(base, 100);
    EXPECT_EQ(result.red(), 255);
    EXPECT_EQ(result.green(), 255);
    EXPECT_EQ(result.blue(), 255);
}

TEST(MathUtilsTest, AdjustContrast) {
    QColor base(128, 128, 128, 255);
    QColor highContrast = MathUtils::adjustContrast(base, 2.0f);
    // Contrast of 2.0 on mid-gray should push toward extremes
    EXPECT_NE(highContrast.red(), base.red());
}

TEST(MathUtilsTest, AdjustSaturation) {
    QColor base(128, 100, 100, 255);
    QColor saturated = MathUtils::adjustSaturation(base, 2.0f);
    EXPECT_GT(saturated.saturation(), base.saturation());
}

TEST(MathUtilsTest, DegreesToRadians) {
    EXPECT_FLOAT_EQ(MathUtils::degToRad(180.0f), static_cast<float>(M_PI));
    EXPECT_FLOAT_EQ(MathUtils::degToRad(90.0f), static_cast<float>(M_PI / 2.0));
}

TEST(MathUtilsTest, RadiansToDegrees) {
    EXPECT_FLOAT_EQ(MathUtils::radToDeg(static_cast<float>(M_PI)), 180.0f);
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run:
```bash
ctest --test-dir build/debug -R test_mathutils --output-on-failure
```
Expected: Compilation failure.

- [ ] **Step 3: Implement MathUtils header**

```cpp
// src/utils/mathutils.h
#pragma once

#include <QColor>
#include <tuple>
#include <algorithm>
#include <cmath>

namespace re4 {
namespace MathUtils {

    // Clamping
    int clamp(int value, int min, int max);
    float clamp(float value, float min, float max);

    // Interpolation
    float lerp(float a, float b, float t);
    QColor lerpColor(const QColor& a, const QColor& b, float t);

    // Color space conversion (HSL with H: 0-360, S: 0-1, L: 0-1)
    std::tuple<float, float, float> rgbToHsl(const QColor& rgb);
    QColor hslToRgb(float h, float s, float l, int alpha = 255);

    // Color adjustments
    QColor adjustBrightness(const QColor& color, int amount);
    QColor adjustContrast(const QColor& color, float factor);
    QColor adjustSaturation(const QColor& color, float factor);

    // Angle conversion
    float degToRad(float degrees);
    float radToDeg(float radians);

} // namespace MathUtils
} // namespace re4
```

- [ ] **Step 4: Implement MathUtils source**

```cpp
// src/utils/mathutils.cpp
#include "utils/mathutils.h"
#include <algorithm>
#include <cmath>

namespace re4 {
namespace MathUtils {

int clamp(int value, int minVal, int maxVal) {
    return std::max(minVal, std::min(value, maxVal));
}

float clamp(float value, float minVal, float maxVal) {
    return std::max(minVal, std::min(value, maxVal));
}

float lerp(float a, float b, float t) {
    return a + (b - a) * t;
}

QColor lerpColor(const QColor& a, const QColor& b, float t) {
    return QColor(
        static_cast<int>(lerp(a.red(), b.red(), t)),
        static_cast<int>(lerp(a.green(), b.green(), t)),
        static_cast<int>(lerp(a.blue(), b.blue(), t)),
        static_cast<int>(lerp(a.alpha(), b.alpha(), t))
    );
}

std::tuple<float, float, float> rgbToHsl(const QColor& rgb) {
    qreal h, s, l;
    rgb.getHslF(&h, &s, &l);
    // Qt returns h in [-1, 1] for achromatic; normalize to 0-360
    float hue = (h < 0) ? 0.0f : static_cast<float>(h * 360.0);
    return { hue, static_cast<float>(s), static_cast<float>(l) };
}

QColor hslToRgb(float h, float s, float l, int alpha) {
    QColor color;
    color.setHslF(h / 360.0f, s, l, alpha / 255.0f);
    return color;
}

QColor adjustBrightness(const QColor& color, int amount) {
    return QColor(
        clamp(color.red() + amount, 0, 255),
        clamp(color.green() + amount, 0, 255),
        clamp(color.blue() + amount, 0, 255),
        color.alpha()
    );
}

QColor adjustContrast(const QColor& color, float factor) {
    auto apply = [&](int channel) -> int {
        float normalized = channel / 255.0f;
        float adjusted = (normalized - 0.5f) * factor + 0.5f;
        return clamp(static_cast<int>(adjusted * 255.0f), 0, 255);
    };
    return QColor(apply(color.red()), apply(color.green()), apply(color.blue()), color.alpha());
}

QColor adjustSaturation(const QColor& color, float factor) {
    qreal h, s, l;
    color.getHslF(&h, &s, &l);
    qreal newSat = std::min(1.0, s * factor);
    QColor result;
    result.setHslF(h, newSat, l, color.alphaF());
    return result;
}

float degToRad(float degrees) {
    return degrees * static_cast<float>(M_PI) / 180.0f;
}

float radToDeg(float radians) {
    return radians * 180.0f / static_cast<float>(M_PI);
}

} // namespace MathUtils
} // namespace re4
```

- [ ] **Step 5: Build and run tests — verify they pass**

Run:
```bash
cd D:/wxmuma
cmake --build build/debug --config Debug --target re4modtool_tests
ctest --test-dir build/debug -R test_mathutils --output-on-failure
```
Expected: All 11 math utils tests PASS.

- [ ] **Step 6: Commit**

```bash
git add src/utils/mathutils.h src/utils/mathutils.cpp tests/unit/test_mathutils.cpp
git commit -m "feat: add math utilities — clamp, lerp, color space conversion, adjustments"
```

---

### Task 8: Image Utilities (TDD)

**Files:**
- Create: `tests/unit/test_imageutils.cpp`
- Create: `src/utils/imageutils.h`
- Create: `src/utils/imageutils.cpp`

- [ ] **Step 1: Write the failing image utilities tests**

```cpp
// tests/unit/test_imageutils.cpp
#include <gtest/gtest.h>
#include "utils/imageutils.h"
#include <QImage>
#include <QColor>

using namespace re4;

class ImageUtilsTest : public ::testing::Test {
protected:
    QImage createTestImage(int w = 4, int h = 4) {
        QImage img(w, h, QImage::Format_RGBA8888);
        img.fill(QColor(128, 64, 32, 255));
        return img;
    }
};

TEST_F(ImageUtilsTest, ExtractRedChannel) {
    QImage img = createTestImage();
    img.setPixelColor(0, 0, QColor(200, 100, 50, 255));
    QImage red = ImageUtils::extractChannel(img, ImageUtils::Channel::Red);
    EXPECT_EQ(red.pixelColor(0, 0).red(), 200);
    EXPECT_EQ(red.pixelColor(0, 0).green(), 200); // grayscale
    EXPECT_EQ(red.pixelColor(0, 0).blue(), 200);
}

TEST_F(ImageUtilsTest, ExtractGreenChannel) {
    QImage img = createTestImage();
    img.setPixelColor(0, 0, QColor(200, 100, 50, 255));
    QImage green = ImageUtils::extractChannel(img, ImageUtils::Channel::Green);
    EXPECT_EQ(green.pixelColor(0, 0).red(), 100);
}

TEST_F(ImageUtilsTest, ExtractBlueChannel) {
    QImage img = createTestImage();
    img.setPixelColor(0, 0, QColor(200, 100, 50, 255));
    QImage blue = ImageUtils::extractChannel(img, ImageUtils::Channel::Blue);
    EXPECT_EQ(blue.pixelColor(0, 0).red(), 50);
}

TEST_F(ImageUtilsTest, ExtractAlphaChannel) {
    QImage img(1, 1, QImage::Format_RGBA8888);
    img.setPixelColor(0, 0, QColor(200, 100, 50, 128));
    QImage alpha = ImageUtils::extractChannel(img, ImageUtils::Channel::Alpha);
    EXPECT_EQ(alpha.pixelColor(0, 0).red(), 128);
}

TEST_F(ImageUtilsTest, ConvertToFormat) {
    QImage rgba(1, 1, QImage::Format_RGBA8888);
    rgba.fill(QColor(255, 0, 0, 255));
    QImage argb = ImageUtils::convertFormat(rgba, QImage::Format_ARGB32);
    EXPECT_EQ(argb.format(), QImage::Format_ARGB32);
}

TEST_F(ImageUtilsTest, ResizeImage) {
    QImage img = createTestImage(100, 100);
    QImage resized = ImageUtils::resize(img, 50, 50);
    EXPECT_EQ(resized.width(), 50);
    EXPECT_EQ(resized.height(), 50);
}

TEST_F(ImageUtilsTest, IsValidTextureSize) {
    EXPECT_TRUE(ImageUtils::isValidTextureSize(256, 256));
    EXPECT_TRUE(ImageUtils::isValidTextureSize(1024, 2048));
    EXPECT_FALSE(ImageUtils::isValidTextureSize(0, 256));
    EXPECT_FALSE(ImageUtils::isValidTextureSize(256, -1));
}

TEST_F(ImageUtilsTest, NextPowerOfTwo) {
    EXPECT_EQ(ImageUtils::nextPowerOfTwo(1), 1);
    EXPECT_EQ(ImageUtils::nextPowerOfTwo(3), 4);
    EXPECT_EQ(ImageUtils::nextPowerOfTwo(100), 128);
    EXPECT_EQ(ImageUtils::nextPowerOfTwo(1024), 1024);
}

TEST_F(ImageUtilsTest, DuplicateImage) {
    QImage original = createTestImage();
    original.setPixelColor(0, 0, QColor(1, 2, 3, 4));
    QImage copy = ImageUtils::duplicate(original);
    EXPECT_EQ(copy.pixelColor(0, 0), QColor(1, 2, 3, 4));
    // Modify original, copy should be independent
    original.setPixelColor(0, 0, QColor(255, 255, 255, 255));
    EXPECT_EQ(copy.pixelColor(0, 0), QColor(1, 2, 3, 4));
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run:
```bash
ctest --test-dir build/debug -R test_imageutils --output-on-failure
```
Expected: Compilation failure.

- [ ] **Step 3: Implement ImageUtils header**

```cpp
// src/utils/imageutils.h
#pragma once

#include <QImage>

namespace re4 {
namespace ImageUtils {

    enum class Channel {
        Red,
        Green,
        Blue,
        Alpha
    };

    QImage extractChannel(const QImage& source, Channel channel);
    QImage convertFormat(const QImage& source, QImage::Format format);
    QImage resize(const QImage& source, int width, int height);

    bool isValidTextureSize(int width, int height);
    int nextPowerOfTwo(int value);

    QImage duplicate(const QImage& source);

} // namespace ImageUtils
} // namespace re4
```

- [ ] **Step 4: Implement ImageUtils source**

```cpp
// src/utils/imageutils.cpp
#include "utils/imageutils.h"
#include <QPainter>

namespace re4 {
namespace ImageUtils {

QImage extractChannel(const QImage& source, Channel channel) {
    QImage result = QImage(source.size(), QImage::Format_Grayscale8);
    for (int y = 0; y < source.height(); ++y) {
        for (int x = 0; x < source.width(); ++x) {
            QColor pixel = source.pixelColor(x, y);
            int value = 0;
            switch (channel) {
                case Channel::Red:   value = pixel.red(); break;
                case Channel::Green: value = pixel.green(); break;
                case Channel::Blue:  value = pixel.blue(); break;
                case Channel::Alpha: value = pixel.alpha(); break;
            }
            result.setPixel(x, y, qRgb(value, value, value));
        }
    }
    return result;
}

QImage convertFormat(const QImage& source, QImage::Format format) {
    if (source.format() == format) return source;
    return source.convertToFormat(format);
}

QImage resize(const QImage& source, int width, int height) {
    return source.scaled(width, height, Qt::IgnoreAspectRatio, Qt::SmoothTransformation);
}

bool isValidTextureSize(int width, int height) {
    return width > 0 && height > 0;
}

int nextPowerOfTwo(int value) {
    if (value <= 0) return 1;
    int result = 1;
    while (result < value) result <<= 1;
    return result;
}

QImage duplicate(const QImage& source) {
    return source.copy();
}

} // namespace ImageUtils
} // namespace re4
```

- [ ] **Step 5: Build and run tests — verify they pass**

Run:
```bash
cd D:/wxmuma
cmake --build build/debug --config Debug --target re4modtool_tests
ctest --test-dir build/debug -R test_imageutils --output-on-failure
```
Expected: All 9 image utils tests PASS.

- [ ] **Step 6: Commit**

```bash
git add src/utils/imageutils.h src/utils/imageutils.cpp tests/unit/test_imageutils.cpp
git commit -m "feat: add image utilities — channel extraction, resize, format conversion"
```

---

### Task 9: Build Full Application and Run All Tests

**Files:**
- Modify: `src/core/application.cpp` (minor fixes if needed)

- [ ] **Step 1: Build the main application target**

Run:
```bash
cd D:/wxmuma
cmake --build build/debug --config Debug --target RE4ModTool
```
Expected: Build succeeds with no errors.

- [ ] **Step 2: Run ALL tests**

Run:
```bash
ctest --test-dir build/debug --output-on-failure
```
Expected: All tests pass (8 logger + 7 settings + 11 fileutils + 11 mathutils + 9 imageutils = 46 tests).

- [ ] **Step 3: Run static analysis (optional, requires Clang)**

Run:
```bash
cd D:/wxmuma
cmake --build build/debug --config Debug --target RE4ModTool -- /p:RunCodeAnalysis=true
```
Expected: No critical warnings.

- [ ] **Step 4: Commit any fixes**

```bash
git add -A
git commit -m "chore: fix build issues and verify all tests pass"
```

---

### Task 10: CI/CD Pipeline (GitHub Actions)

**Files:**
- Create: `.github/workflows/build.yml`

- [ ] **Step 1: Create CI workflow**

```yaml
name: Build & Test

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build-windows:
    runs-on: windows-latest

    steps:
    - uses: actions/checkout@v4

    - name: Setup vcpkg
      uses: lukka/run-vcpkg@v11
      with:
        vcpkgGitCommitId: 'a42af01b72c28a8e1d7b48107b33e4f286a55ef6'

    - name: Setup Qt 6.8
      uses: jurplel/install-qt-action@v4
      with:
        version: '6.8.0'
        host: 'windows'
        target: 'desktop'
        arch: 'win64_msvc2022_64'

    - name: Configure CMake
      run: cmake --preset vcpkg-debug

    - name: Build
      run: cmake --build build/debug --config Debug

    - name: Run Tests
      run: ctest --test-dir build/debug --output-on-failure

    - name: Upload Test Results
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: test-results
        path: build/debug/Testing/
```

- [ ] **Step 2: Verify workflow syntax locally**

Run:
```bash
# Validate YAML syntax
python -c "import yaml; yaml.safe_load(open('.github/workflows/build.yml'))" 2>/dev/null || echo "Install pyyaml to validate"
```

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/build.yml
git commit -m "ci: add GitHub Actions build and test workflow for Windows"
```

---

## Phase 1 Complete — Summary

After completing all tasks, you will have:

| Deliverable | Status |
|------------|--------|
| CMake build with vcpkg + Qt 6.8 + DirectXTex | ✅ |
| Qt application skeleton with proper init/shutdown | ✅ |
| Logger: file+console, levels, rotation, categories | ✅ |
| Settings: recent files, cache, window state | ✅ |
| FileUtils: path helpers, binary I/O, temp files | ✅ |
| ImageUtils: channel extraction, resize, format conversion | ✅ |
| MathUtils: clamp, lerp, color space, HSL/RGB | ✅ |
| 46 unit tests (all passing) | ✅ |
| GitHub Actions CI/CD | ✅ |

**Next Phase:** Phase 2 — Format Conversion Module (.tex/.dds ↔ TGA/PNG)
