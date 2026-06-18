# RE4 Mod Tool - Phase 5: Path Configuration + Batch Processing + Preset System

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement path configuration for multi-mod/multi-part textures, batch processing for multiple files, and color preset save/load system.

**Tech Stack:** C++17, Qt 6.8, JSON (nlohmann/json or QJsonDocument)

**Spec Reference:** `docs/superpowers/specs/2026-06-17-re4-mod-tool-design.md` — Sections 5, 6, 7

**Depends On:** Phase 1-3 (Core, Formats, Color)

---

## File Structure (Phase 5)

| File | Responsibility |
|------|---------------|
| `src/config/pathmanager.h/.cpp` | Mod/texture path management with templates |
| `src/config/configmanager.h/.cpp` | JSON config file I/O |
| `src/batch/batchprocessor.h/.cpp` | Multi-threaded batch processing |
| `src/batch/taskqueue.h/.cpp` | Thread-safe task queue |
| `src/preset/presetmanager.h/.cpp` | Color preset CRUD |
| `src/preset/preset.h/.cpp` | Preset data model |
| `src/ui/pathconfigwidget.h/.cpp` | Path configuration UI |
| `src/ui/batchdialog.h/.cpp` | Batch processing dialog |
| `src/ui/presetwidget.h/.cpp` | Preset management UI |
| `tests/unit/test_pathmanager.cpp` | Path manager tests |
| `tests/unit/test_batchprocessor.cpp` | Batch processor tests |
| `tests/unit/test_presetmanager.cpp` | Preset manager tests |

---

### Task 1: PathManager — Mod/Texture Path Management (TDD)

**Files:**
- Create: `tests/unit/test_pathmanager.cpp`
- Create: `src/config/pathmanager.h`
- Create: `src/config/pathmanager.cpp`

- [ ] **Step 1: Write failing tests**

```cpp
// tests/unit/test_pathmanager.cpp
#include <gtest/gtest.h>
#include "config/pathmanager.h"
#include <QTemporaryDir>

using namespace re4;

class PathManagerTest : public ::testing::Test {
protected:
    void SetUp() override {
        m_tempDir = std::make_unique<QTemporaryDir>();
        ASSERT_TRUE(m_tempDir->isValid());
        manager = std::make_unique<PathManager>(m_tempDir->filePath("config.json"));
    }
    std::unique_ptr<QTemporaryDir> m_tempDir;
    std::unique_ptr<PathManager> manager;
};

TEST_F(PathManagerTest, AddMod) {
    ModConfig mod;
    mod.name = "Leon Default";
    mod.basePath = "C:\\RE4\\Mods\\Leon";
    ASSERT_TRUE(manager->addMod(mod));
    EXPECT_EQ(manager->getModCount(), 1);
}

TEST_F(PathManagerTest, RemoveMod) {
    ModConfig mod;
    mod.name = "Leon";
    mod.basePath = "C:\\RE4\\Mods\\Leon";
    manager->addMod(mod);
    ASSERT_TRUE(manager->removeMod("Leon"));
    EXPECT_EQ(manager->getModCount(), 0);
}

TEST_F(PathManagerTest, AddTextureToMod) {
    ModConfig mod;
    mod.name = "Leon";
    mod.basePath = "C:\\RE4\\Mods\\Leon";
    manager->addMod(mod);

    TexturePath tex;
    tex.partName = "Body";
    tex.texturePath = "native\\STM\\Leon\\body_d.tex";
    tex.textureType = "diffuse";
    ASSERT_TRUE(manager->addTexturePath("Leon", tex));

    auto paths = manager->getTexturePaths("Leon");
    EXPECT_EQ(paths.size(), 1);
    EXPECT_EQ(paths[0].partName, "Body");
}

TEST_F(PathManagerTest, TemplateExpansion) {
    PathTemplate tmpl;
    tmpl.name = "Character Template";
    tmpl.basePath = "native\\STM\\character\\{角色}\\{服装}\\";
    tmpl.texturePattern = "{部件}_d.tex";
    manager->addTemplate(tmpl);

    QVariantMap vars;
    vars["角色"] = "Leon";
    vars["服装"] = "Default";
    vars["部件"] = "body";
    QString result = manager->expandTemplate("Character Template", vars);
    EXPECT_EQ(result, "native\\STM\\character\\Leon\\Default\\body_d.tex");
}

TEST_F(PathManagerTest, SaveAndLoadConfig) {
    ModConfig mod;
    mod.name = "Leon";
    mod.basePath = "C:\\RE4\\Mods\\Leon";
    manager->addMod(mod);
    manager->save();

    PathManager manager2(m_tempDir->filePath("config.json"));
    manager2.load();
    EXPECT_EQ(manager2.getModCount(), 1);
}

TEST_F(PathManagerTest, RecentFiles) {
    manager->addRecentPath("C:\\RE4\\Mods\\Leon");
    manager->addRecentPath("C:\\RE4\\Mods\\Ashley");
    auto recent = manager->recentPaths();
    EXPECT_EQ(recent.size(), 2);
    EXPECT_EQ(recent[0], "C:\\RE4\\Mods\\Ashley");
}

TEST_F(PathManagerTest, RecentFilesMax10) {
    for (int i = 0; i < 15; ++i) {
        manager->addRecentPath(QString("path_%1").arg(i));
    }
    EXPECT_LE(manager->recentPaths().size(), 10);
}
```

- [ ] **Step 2: Implement PathManager**

```cpp
// src/config/pathmanager.h
#pragma once

#include <QObject>
#include <QString>
#include <QStringList>
#include <QVariantMap>
#include <QVector>
#include <memory>

namespace re4 {

struct TexturePath {
    QString partName;
    QString texturePath;
    QString textureType; // "diffuse", "normal", "specular"
};

struct ModConfig {
    QString name;
    QString basePath;
    QVector<TexturePath> textures;
};

struct PathTemplate {
    QString name;
    QString basePath;
    QString texturePattern;
};

class PathManager : public QObject {
    Q_OBJECT

public:
    explicit PathManager(const QString& configPath, QObject* parent = nullptr);

    // Mod management
    bool addMod(const ModConfig& mod);
    bool removeMod(const QString& name);
    bool updateMod(const QString& name, const ModConfig& mod);
    ModConfig getMod(const QString& name) const;
    QStringList getModNames() const;
    int getModCount() const;

    // Texture paths
    bool addTexturePath(const QString& modName, const TexturePath& path);
    bool removeTexturePath(const QString& modName, const QString& partName);
    QVector<TexturePath> getTexturePaths(const QString& modName) const;

    // Templates
    void addTemplate(const PathTemplate& tmpl);
    QString expandTemplate(const QString& templateName, const QVariantMap& vars) const;
    QVector<PathTemplate> getTemplates() const;

    // Recent files
    void addRecentPath(const QString& path);
    QStringList recentPaths() const;

    // Config I/O
    void save();
    void load();

signals:
    void modAdded(const QString& name);
    void modRemoved(const QString& name);
    void configChanged();

private:
    QString m_configPath;
    QVector<ModConfig> m_mods;
    QVector<PathTemplate> m_templates;
    QStringList m_recentPaths;
    static constexpr int kMaxRecent = 10;
};

} // namespace re4
```

```cpp
// src/config/pathmanager.cpp
#include "config/pathmanager.h"
#include <QFile>
#include <QJsonDocument>
#include <QJsonObject>
#include <QJsonArray>
#include <QDir>

namespace re4 {

PathManager::PathManager(const QString& configPath, QObject* parent)
    : QObject(parent), m_configPath(configPath) {}

bool PathManager::addMod(const ModConfig& mod) {
    for (const auto& m : m_mods) {
        if (m.name == mod.name) return false;
    }
    m_mods.append(mod);
    emit modAdded(mod.name);
    emit configChanged();
    return true;
}

bool PathManager::removeMod(const QString& name) {
    for (int i = 0; i < m_mods.size(); ++i) {
        if (m_mods[i].name == name) {
            m_mods.removeAt(i);
            emit modRemoved(name);
            emit configChanged();
            return true;
        }
    }
    return false;
}

bool PathManager::updateMod(const QString& name, const ModConfig& mod) {
    for (int i = 0; i < m_mods.size(); ++i) {
        if (m_mods[i].name == name) {
            m_mods[i] = mod;
            emit configChanged();
            return true;
        }
    }
    return false;
}

ModConfig PathManager::getMod(const QString& name) const {
    for (const auto& m : m_mods) {
        if (m.name == name) return m;
    }
    return {};
}

QStringList PathManager::getModNames() const {
    QStringList names;
    for (const auto& m : m_mods) names.append(m.name);
    return names;
}

int PathManager::getModCount() const { return m_mods.size(); }

bool PathManager::addTexturePath(const QString& modName, const TexturePath& path) {
    for (auto& m : m_mods) {
        if (m.name == modName) {
            m.textures.append(path);
            emit configChanged();
            return true;
        }
    }
    return false;
}

bool PathManager::removeTexturePath(const QString& modName, const QString& partName) {
    for (auto& m : m_mods) {
        if (m.name == modName) {
            for (int i = 0; i < m.textures.size(); ++i) {
                if (m.textures[i].partName == partName) {
                    m.textures.removeAt(i);
                    emit configChanged();
                    return true;
                }
            }
        }
    }
    return false;
}

QVector<TexturePath> PathManager::getTexturePaths(const QString& modName) const {
    for (const auto& m : m_mods) {
        if (m.name == modName) return m.textures;
    }
    return {};
}

void PathManager::addTemplate(const PathTemplate& tmpl) {
    m_templates.append(tmpl);
}

QString PathManager::expandTemplate(const QString& templateName, const QVariantMap& vars) const {
    for (const auto& t : m_templates) {
        if (t.name == templateName) {
            QString result = t.basePath + t.texturePattern;
            for (auto it = vars.begin(); it != vars.end(); ++it) {
                result.replace("{" + it.key() + "}", it.value().toString());
            }
            return result;
        }
    }
    return {};
}

QVector<PathTemplate> PathManager::getTemplates() const { return m_templates; }

void PathManager::addRecentPath(const QString& path) {
    m_recentPaths.removeAll(path);
    m_recentPaths.prepend(path);
    while (m_recentPaths.size() > kMaxRecent) m_recentPaths.removeLast();
}

QStringList PathManager::recentPaths() const { return m_recentPaths; }

void PathManager::save() {
    QJsonObject root;

    QJsonArray modsArray;
    for (const auto& m : m_mods) {
        QJsonObject modObj;
        modObj["name"] = m.name;
        modObj["basePath"] = m.basePath;
        QJsonArray texArray;
        for (const auto& t : m.textures) {
            QJsonObject texObj;
            texObj["partName"] = t.partName;
            texObj["texturePath"] = t.texturePath;
            texObj["textureType"] = t.textureType;
            texArray.append(texObj);
        }
        modObj["textures"] = texArray;
        modsArray.append(modObj);
    }
    root["mods"] = modsArray;

    QJsonArray tmplArray;
    for (const auto& t : m_templates) {
        QJsonObject tmplObj;
        tmplObj["name"] = t.name;
        tmplObj["basePath"] = t.basePath;
        tmplObj["texturePattern"] = t.texturePattern;
        tmplArray.append(tmplObj);
    }
    root["templates"] = tmplArray;

    QJsonArray recentArray;
    for (const auto& p : m_recentPaths) recentArray.append(p);
    root["recentPaths"] = recentArray;

    QFile file(m_configPath);
    if (file.open(QIODevice::WriteOnly)) {
        file.write(QJsonDocument(root).toJson());
    }
}

void PathManager::load() {
    QFile file(m_configPath);
    if (!file.open(QIODevice::ReadOnly)) return;

    QJsonDocument doc = QJsonDocument::fromJson(file.readAll());
    QJsonObject root = doc.object();

    m_mods.clear();
    QJsonArray modsArray = root["mods"].toArray();
    for (const auto& m : modsArray) {
        QJsonObject modObj = m.toObject();
        ModConfig mod;
        mod.name = modObj["name"].toString();
        mod.basePath = modObj["basePath"].toString();
        QJsonArray texArray = modObj["textures"].toArray();
        for (const auto& t : texArray) {
            QJsonObject texObj = t.toObject();
            TexturePath tex;
            tex.partName = texObj["partName"].toString();
            tex.texturePath = texObj["texturePath"].toString();
            tex.textureType = texObj["textureType"].toString();
            mod.textures.append(tex);
        }
        m_mods.append(mod);
    }

    m_templates.clear();
    QJsonArray tmplArray = root["templates"].toArray();
    for (const auto& t : tmplArray) {
        QJsonObject tmplObj = t.toObject();
        PathTemplate tmpl;
        tmpl.name = tmplObj["name"].toString();
        tmpl.basePath = tmplObj["basePath"].toString();
        tmpl.texturePattern = tmplObj["texturePattern"].toString();
        m_templates.append(tmpl);
    }

    m_recentPaths.clear();
    QJsonArray recentArray = root["recentPaths"].toArray();
    for (const auto& p : recentArray) m_recentPaths.append(p.toString());
}

} // namespace re4
```

- [ ] **Step 3: Commit**

```bash
git add src/config/pathmanager.h src/config/pathmanager.cpp tests/unit/test_pathmanager.cpp
git commit -m "feat: add path manager with mod config, templates, and recent files"
```

---

### Task 2: BatchProcessor — Multi-threaded Batch Processing (TDD)

**Files:**
- Create: `tests/unit/test_batchprocessor.cpp`
- Create: `src/batch/batchprocessor.h`
- Create: `src/batch/batchprocessor.cpp`

- [ ] **Step 1: Write failing tests**

```cpp
// tests/unit/test_batchprocessor.cpp
#include <gtest/gtest.h>
#include "batch/batchprocessor.h"
#include <QTemporaryDir>
#include <QImage>

using namespace re4;

class BatchProcessorTest : public ::testing::Test {
protected:
    void SetUp() override {
        m_tempDir = std::make_unique<QTemporaryDir>();
        ASSERT_TRUE(m_tempDir->isValid());
        processor = std::make_unique<BatchProcessor>();
    }
    std::unique_ptr<QTemporaryDir> m_tempDir;
    std::unique_ptr<BatchProcessor> processor;

    QString createTestPng(const QString& name, int w = 32, int h = 32) {
        QString path = m_tempDir->filePath(name);
        QImage img(w, h, QImage::Format_RGBA8888);
        img.fill(QColor(128, 128, 128, 255));
        img.save(path, "PNG");
        return path;
    }
};

TEST_F(BatchProcessorTest, AddTask) {
    BatchTask task;
    task.inputPath = "test.png";
    task.outputPath = "out.tex";
    task.operation = BatchOperation::FORMAT_CONVERT;
    processor->addTask(task);
    EXPECT_EQ(processor->taskCount(), 1);
}

TEST_F(BatchProcessorTest, ProcessFormatConvert) {
    QString input = createTestPng("input.png");
    QString output = m_tempDir->filePath("output.tga");

    BatchTask task;
    task.inputPath = input;
    task.outputPath = output;
    task.operation = BatchOperation::FORMAT_CONVERT;
    processor->addTask(task);

    auto result = processor->processAll();
    EXPECT_EQ(result.succeeded, 1);
    EXPECT_EQ(result.failed, 0);
    EXPECT_TRUE(QFile::exists(output));
}

TEST_F(BatchProcessorTest, ProcessBatchConvert) {
    QStringList inputs;
    for (int i = 0; i < 3; ++i) {
        inputs.append(createTestPng(QString("input_%1.png").arg(i)));
    }

    QString outputDir = m_tempDir->filePath("output");
    QDir().mkpath(outputDir);

    auto result = processor->batchConvert(inputs, outputDir, "tga");
    EXPECT_EQ(result.succeeded, 3);
    EXPECT_EQ(result.failed, 0);
}

TEST_F(BatchProcessorTest, ProcessColorAdjust) {
    QString input = createTestPng("input.png");
    QString output = m_tempDir->filePath("output.png");

    BatchTask task;
    task.inputPath = input;
    task.outputPath = output;
    task.operation = BatchOperation::COLOR_ADJUST;
    task.brightness = 50;
    processor->addTask(task);

    auto result = processor->processAll();
    EXPECT_EQ(result.succeeded, 1);
}

TEST_F(BatchProcessorTest, ProgressCallback) {
    QString input = createTestPng("input.png");

    BatchTask task;
    task.inputPath = input;
    task.outputPath = m_tempDir->filePath("out.tga");
    task.operation = BatchOperation::FORMAT_CONVERT;
    processor->addTask(task);

    int progressCalled = 0;
    processor->setProgressCallback([&progressCalled](int, int) { progressCalled++; });
    processor->processAll();
    EXPECT_GT(progressCalled, 0);
}

TEST_F(BatchProcessorTest, TaskStates) {
    EXPECT_EQ(processor->taskCount(), 0);

    BatchTask task;
    task.inputPath = "test.png";
    task.outputPath = "out.tga";
    task.operation = BatchOperation::FORMAT_CONVERT;
    processor->addTask(task);

    EXPECT_EQ(processor->taskCount(), 1);
    processor->clearTasks();
    EXPECT_EQ(processor->taskCount(), 0);
}
```

- [ ] **Step 2: Implement BatchProcessor**

```cpp
// src/batch/batchprocessor.h
#pragma once

#include <QObject>
#include <QString>
#include <QStringList>
#include <QVector>
#include <functional>
#include <atomic>

namespace re4 {

enum class BatchOperation {
    FORMAT_CONVERT,
    COLOR_ADJUST,
    RESIZE,
};

struct BatchTask {
    QString inputPath;
    QString outputPath;
    BatchOperation operation;
    int brightness = 0;
    float contrast = 1.0f;
    float saturation = 1.0f;
    int hueShift = 0;
    int targetWidth = 0;
    int targetHeight = 0;
    QString outputFormat;
};

struct BatchResult {
    int succeeded = 0;
    int failed = 0;
    QStringList errors;
};

class BatchProcessor : public QObject {
    Q_OBJECT

public:
    explicit BatchProcessor(QObject* parent = nullptr);

    void addTask(const BatchTask& task);
    void clearTasks();
    int taskCount() const;

    BatchResult processAll();
    BatchResult batchConvert(const QStringList& inputs, const QString& outputDir,
                             const QString& extension);

    void setProgressCallback(std::function<void(int current, int total)> callback);

signals:
    void taskStarted(int index, const QString& path);
    void taskCompleted(int index, const QString& path);
    void taskFailed(int index, const QString& path, const QString& error);
    void allCompleted(int succeeded, int failed);

private:
    bool processTask(const BatchTask& task);

    QVector<BatchTask> m_tasks;
    std::function<void(int, int)> m_progressCallback;
};

} // namespace re4
```

```cpp
// src/batch/batchprocessor.cpp
#include "batch/batchprocessor.h"
#include "color/colorprocessor.h"
#include "formats/formatconverter.h"
#include "core/logger.h"
#include <QDir>
#include <QFileInfo>
#include <QImage>

namespace re4 {

BatchProcessor::BatchProcessor(QObject* parent) : QObject(parent) {}

void BatchProcessor::addTask(const BatchTask& task) {
    m_tasks.append(task);
}

void BatchProcessor::clearTasks() { m_tasks.clear(); }
int BatchProcessor::taskCount() const { return m_tasks.size(); }

void BatchProcessor::setProgressCallback(std::function<void(int, int)> callback) {
    m_progressCallback = callback;
}

BatchResult BatchProcessor::processAll() {
    BatchResult result;
    int total = m_tasks.size();

    for (int i = 0; i < total; ++i) {
        if (m_progressCallback) m_progressCallback(i, total);
        emit taskStarted(i, m_tasks[i].inputPath);

        if (processTask(m_tasks[i])) {
            result.succeeded++;
            emit taskCompleted(i, m_tasks[i].inputPath);
        } else {
            result.failed++;
            result.errors.append(m_tasks[i].inputPath);
            emit taskFailed(i, m_tasks[i].inputPath, "Processing failed");
        }
    }

    if (m_progressCallback) m_progressCallback(total, total);
    emit allCompleted(result.succeeded, result.failed);
    return result;
}

BatchResult BatchProcessor::batchConvert(const QStringList& inputs,
                                          const QString& outputDir,
                                          const QString& extension) {
    QDir().mkpath(outputDir);
    m_tasks.clear();

    for (const QString& input : inputs) {
        QString baseName = QFileInfo(input).baseName();
        BatchTask task;
        task.inputPath = input;
        task.outputPath = QDir(outputDir).filePath(baseName + "." + extension);
        task.operation = BatchOperation::FORMAT_CONVERT;
        m_tasks.append(task);
    }

    return processAll();
}

bool BatchProcessor::processTask(const BatchTask& task) {
    switch (task.operation) {
        case BatchOperation::FORMAT_CONVERT: {
            FormatConverter converter;
            return converter.convert(task.inputPath, task.outputPath);
        }
        case BatchOperation::COLOR_ADJUST: {
            QImage img(task.inputPath);
            if (img.isNull()) return false;
            img = ColorProcessor::adjustBrightness(img, task.brightness);
            img = ColorProcessor::adjustContrast(img, task.contrast);
            img = ColorProcessor::adjustSaturation(img, task.saturation);
            img = ColorProcessor::adjustHueShift(img, task.hueShift);
            return img.save(task.outputPath);
        }
        case BatchOperation::RESIZE: {
            QImage img(task.inputPath);
            if (img.isNull()) return false;
            img = img.scaled(task.targetWidth, task.targetHeight,
                             Qt::IgnoreAspectRatio, Qt::SmoothTransformation);
            return img.save(task.outputPath);
        }
    }
    return false;
}

} // namespace re4
```

- [ ] **Step 3: Commit**

```bash
git add src/batch/batchprocessor.h src/batch/batchprocessor.cpp tests/unit/test_batchprocessor.cpp
git commit -m "feat: add batch processor with format convert, color adjust, and resize"
```

---

### Task 3: PresetManager — Color Preset System (TDD)

**Files:**
- Create: `tests/unit/test_presetmanager.cpp`
- Create: `src/preset/preset.h`
- Create: `src/preset/preset.cpp`
- Create: `src/preset/presetmanager.h`
- Create: `src/preset/presetmanager.cpp`

- [ ] **Step 1: Write failing tests**

```cpp
// tests/unit/test_presetmanager.cpp
#include <gtest/gtest.h>
#include "preset/presetmanager.h"
#include <QTemporaryDir>

using namespace re4;

class PresetManagerTest : public ::testing::Test {
protected:
    void SetUp() override {
        m_tempDir = std::make_unique<QTemporaryDir>();
        ASSERT_TRUE(m_tempDir->isValid());
        manager = std::make_unique<PresetManager>(m_tempDir->filePath("presets"));
    }
    std::unique_ptr<QTemporaryDir> m_tempDir;
    std::unique_ptr<PresetManager> manager;

    ColorPreset createTestPreset(const QString& name) {
        ColorPreset p;
        p.name = name;
        p.description = "Test preset";
        p.author = "Test";
        p.hueShift = 10;
        p.saturation = 1.2f;
        p.brightness = 0;
        p.contrast = 1.0f;
        return p;
    }
};

TEST_F(PresetManagerTest, CreatePreset) {
    auto p = createTestPreset("Dark Night");
    ASSERT_TRUE(manager->createPreset(p));
    EXPECT_EQ(manager->getPresetCount(), 1);
}

TEST_F(PresetManagerTest, GetPreset) {
    auto p = createTestPreset("Dark Night");
    manager->createPreset(p);
    auto loaded = manager->getPreset("Dark Night");
    EXPECT_EQ(loaded.name, "Dark Night");
    EXPECT_EQ(loaded.hueShift, 10);
}

TEST_F(PresetManagerTest, DeletePreset) {
    auto p = createTestPreset("Dark Night");
    manager->createPreset(p);
    ASSERT_TRUE(manager->deletePreset("Dark Night"));
    EXPECT_EQ(manager->getPresetCount(), 0);
}

TEST_F(PresetManagerTest, ListPresets) {
    manager->createPreset(createTestPreset("Preset1"));
    manager->createPreset(createTestPreset("Preset2"));
    auto names = manager->getPresetNames();
    EXPECT_EQ(names.size(), 2);
}

TEST_F(PresetManagerTest, SaveAndLoad) {
    manager->createPreset(createTestPreset("Dark Night"));

    PresetManager manager2(m_tempDir->filePath("presets"));
    manager2.load();
    EXPECT_EQ(manager2.getPresetCount(), 1);
    EXPECT_EQ(manager2.getPreset("Dark Night").hueShift, 10);
}

TEST_F(PresetManagerTest, ExportImport) {
    auto p = createTestPreset("Dark Night");
    manager->createPreset(p);

    QString exportPath = m_tempDir->filePath("export.json");
    ASSERT_TRUE(manager->exportPreset("Dark Night", exportPath));

    PresetManager manager2(m_tempDir->filePath("presets2"));
    ASSERT_TRUE(manager2.importPreset(exportPath));
    EXPECT_EQ(manager2.getPresetCount(), 1);
}

TEST_F(PresetManagerTest, UpdatePreset) {
    auto p = createTestPreset("Dark Night");
    manager->createPreset(p);

    p.hueShift = 20;
    ASSERT_TRUE(manager->updatePreset("Dark Night", p));
    EXPECT_EQ(manager->getPreset("Dark Night").hueShift, 20);
}
```

- [ ] **Step 2: Implement Preset and PresetManager**

```cpp
// src/preset/preset.h
#pragma once

#include <QString>
#include <QStringList>
#include <QColor>

namespace re4 {

struct ColorPreset {
    QString name;
    QString description;
    QString author;
    QString version = "1.0";
    QStringList tags;
    QString category = "Color";

    // Color parameters
    QColor primaryColor;
    QColor accentColor;

    // Adjustment parameters
    int hueShift = 0;
    float saturation = 1.0f;
    float brightness = 0.0f;
    float contrast = 1.0f;

    // Application rules
    bool preserveAlpha = true;
    float opacity = 1.0f;
};

} // namespace re4
```

```cpp
// src/preset/presetmanager.h
#pragma once

#include "preset/preset.h"
#include <QObject>
#include <QVector>
#include <QStringList>

namespace re4 {

class PresetManager : public QObject {
    Q_OBJECT

public:
    explicit PresetManager(const QString& presetDir, QObject* parent = nullptr);

    bool createPreset(const ColorPreset& preset);
    bool updatePreset(const QString& name, const ColorPreset& preset);
    bool deletePreset(const QString& name);
    ColorPreset getPreset(const QString& name) const;
    QStringList getPresetNames() const;
    int getPresetCount() const;
    QVector<ColorPreset> getAllPresets() const;

    bool exportPreset(const QString& name, const QString& filePath);
    bool importPreset(const QString& filePath);

    void save();
    void load();

signals:
    void presetCreated(const QString& name);
    void presetDeleted(const QString& name);
    void presetUpdated(const QString& name);

private:
    QString m_presetDir;
    QVector<ColorPreset> m_presets;
};

} // namespace re4
```

```cpp
// src/preset/presetmanager.cpp
#include "preset/presetmanager.h"
#include <QFile>
#include <QDir>
#include <QJsonDocument>
#include <QJsonObject>
#include <QJsonArray>

namespace re4 {

static QJsonObject presetToJson(const ColorPreset& p) {
    QJsonObject obj;
    obj["name"] = p.name;
    obj["description"] = p.description;
    obj["author"] = p.author;
    obj["version"] = p.version;
    obj["category"] = p.category;
    obj["hueShift"] = p.hueShift;
    obj["saturation"] = p.saturation;
    obj["brightness"] = p.brightness;
    obj["contrast"] = p.contrast;
    obj["preserveAlpha"] = p.preserveAlpha;
    obj["opacity"] = p.opacity;
    obj["primaryColor"] = p.primaryColor.name();
    obj["accentColor"] = p.accentColor.name();

    QJsonArray tags;
    for (const auto& t : p.tags) tags.append(t);
    obj["tags"] = tags;

    return obj;
}

static ColorPreset jsonToPreset(const QJsonObject& obj) {
    ColorPreset p;
    p.name = obj["name"].toString();
    p.description = obj["description"].toString();
    p.author = obj["author"].toString();
    p.version = obj["version"].toString();
    p.category = obj["category"].toString();
    p.hueShift = obj["hueShift"].toInt();
    p.saturation = static_cast<float>(obj["saturation"].toDouble());
    p.brightness = static_cast<float>(obj["brightness"].toDouble());
    p.contrast = static_cast<float>(obj["contrast"].toDouble());
    p.preserveAlpha = obj["preserveAlpha"].toBool();
    p.opacity = static_cast<float>(obj["opacity"].toDouble());
    p.primaryColor = QColor(obj["primaryColor"].toString());
    p.accentColor = QColor(obj["accentColor"].toString());

    QJsonArray tags = obj["tags"].toArray();
    for (const auto& t : tags) p.tags.append(t.toString());

    return p;
}

PresetManager::PresetManager(const QString& presetDir, QObject* parent)
    : QObject(parent), m_presetDir(presetDir) {
    QDir().mkpath(presetDir);
}

bool PresetManager::createPreset(const ColorPreset& preset) {
    for (const auto& p : m_presets) {
        if (p.name == preset.name) return false;
    }
    m_presets.append(preset);
    save();
    emit presetCreated(preset.name);
    return true;
}

bool PresetManager::updatePreset(const QString& name, const ColorPreset& preset) {
    for (int i = 0; i < m_presets.size(); ++i) {
        if (m_presets[i].name == name) {
            m_presets[i] = preset;
            save();
            emit presetUpdated(name);
            return true;
        }
    }
    return false;
}

bool PresetManager::deletePreset(const QString& name) {
    for (int i = 0; i < m_presets.size(); ++i) {
        if (m_presets[i].name == name) {
            m_presets.removeAt(i);
            QFile::remove(m_presetDir + "/" + name + ".json");
            emit presetDeleted(name);
            return true;
        }
    }
    return false;
}

ColorPreset PresetManager::getPreset(const QString& name) const {
    for (const auto& p : m_presets) {
        if (p.name == name) return p;
    }
    return {};
}

QStringList PresetManager::getPresetNames() const {
    QStringList names;
    for (const auto& p : m_presets) names.append(p.name);
    return names;
}

int PresetManager::getPresetCount() const { return m_presets.size(); }

QVector<ColorPreset> PresetManager::getAllPresets() const { return m_presets; }

bool PresetManager::exportPreset(const QString& name, const QString& filePath) {
    auto preset = getPreset(name);
    if (preset.name.isEmpty()) return false;

    QFile file(filePath);
    if (!file.open(QIODevice::WriteOnly)) return false;
    file.write(QJsonDocument(presetToJson(preset)).toJson());
    return true;
}

bool PresetManager::importPreset(const QString& filePath) {
    QFile file(filePath);
    if (!file.open(QIODevice::ReadOnly)) return false;

    QJsonDocument doc = QJsonDocument::fromJson(file.readAll());
    auto preset = jsonToPreset(doc.object());
    if (preset.name.isEmpty()) return false;

    return createPreset(preset);
}

void PresetManager::save() {
    QJsonArray arr;
    for (const auto& p : m_presets) arr.append(presetToJson(p));

    QFile file(m_presetDir + "/presets.json");
    if (file.open(QIODevice::WriteOnly)) {
        file.write(QJsonDocument(arr).toJson());
    }
}

void PresetManager::load() {
    QFile file(m_presetDir + "/presets.json");
    if (!file.open(QIODevice::ReadOnly)) return;

    QJsonArray arr = QJsonDocument::fromJson(file.readAll()).array();
    m_presets.clear();
    for (const auto& v : arr) {
        m_presets.append(jsonToPreset(v.toObject()));
    }
}

} // namespace re4
```

- [ ] **Step 3: Commit**

```bash
git add src/preset/preset.h src/preset/preset.cpp src/preset/presetmanager.h src/preset/presetmanager.cpp tests/unit/test_presetmanager.cpp
git commit -m "feat: add color preset system with create, save, load, export, import"
```

---

### Task 4: UI Widgets (PathConfig, BatchDialog, PresetWidget)

**Files:**
- Create: `src/ui/pathconfigwidget.h/.cpp`
- Create: `src/ui/batchdialog.h/.cpp`
- Create: `src/ui/presetwidget.h/.cpp`

- [ ] **Step 1: Create PathConfigWidget**

```cpp
// src/ui/pathconfigwidget.h
#pragma once

#include <QWidget>
#include <QListWidget>
#include <QLineEdit>

namespace re4 {
class PathManager;

class PathConfigWidget : public QWidget {
    Q_OBJECT
public:
    explicit PathConfigWidget(PathManager* manager, QWidget* parent = nullptr);
signals:
    void modSelected(const QString& name);
private:
    void setupUI();
    void refreshModList();
    PathManager* m_manager;
    QListWidget* m_modList;
    QLineEdit* m_basePathEdit;
};
} // namespace re4
```

```cpp
// src/ui/pathconfigwidget.cpp
#include "ui/pathconfigwidget.h"
#include "config/pathmanager.h"
#include <QVBoxLayout>
#include <QHBoxLayout>
#include <QPushButton>
#include <QLabel>
#include <QFileDialog>

namespace re4 {

PathConfigWidget::PathConfigWidget(PathManager* manager, QWidget* parent)
    : QWidget(parent), m_manager(manager) { setupUI(); }

void PathConfigWidget::setupUI() {
    auto* layout = new QVBoxLayout(this);

    layout->addWidget(new QLabel("Mods:"));
    m_modList = new QListWidget(this);
    layout->addWidget(m_modList);

    auto* btnRow = new QHBoxLayout();
    auto* addBtn = new QPushButton("Add Mod");
    auto* removeBtn = new QPushButton("Remove Mod");
    btnRow->addWidget(addBtn);
    btnRow->addWidget(removeBtn);
    layout->addLayout(btnRow);

    layout->addWidget(new QLabel("Base Path:"));
    auto* pathRow = new QHBoxLayout();
    m_basePathEdit = new QLineEdit(this);
    auto* browseBtn = new QPushButton("Browse");
    pathRow->addWidget(m_basePathEdit);
    pathRow->addWidget(browseBtn);
    layout->addLayout(pathRow);

    connect(addBtn, &QPushButton::clicked, this, [this]() {
        ModConfig mod;
        mod.name = "New Mod";
        mod.basePath = "";
        m_manager->addMod(mod);
        refreshModList();
    });

    connect(removeBtn, &QPushButton::clicked, this, [this]() {
        auto* item = m_modList->currentItem();
        if (item) {
            m_manager->removeMod(item->text());
            refreshModList();
        }
    });

    connect(browseBtn, &QPushButton::clicked, this, [this]() {
        QString dir = QFileDialog::getExistingDirectory(this, "Select Mod Directory");
        if (!dir.isEmpty()) m_basePathEdit->setText(dir);
    });

    connect(m_modList, &QListWidget::currentItemChanged, this, [this](QListWidgetItem* item) {
        if (item) {
            auto mod = m_manager->getMod(item->text());
            m_basePathEdit->setText(mod.basePath);
            emit modSelected(item->text());
        }
    });

    refreshModList();
}

void PathConfigWidget::refreshModList() {
    m_modList->clear();
    for (const auto& name : m_manager->getModNames()) {
        m_modList->addItem(name);
    }
}

} // namespace re4
```

- [ ] **Step 2: Create PresetWidget**

```cpp
// src/ui/presetwidget.h
#pragma once

#include <QWidget>
#include <QListWidget>

namespace re4 {
class PresetManager;
struct ColorPreset;

class PresetWidget : public QWidget {
    Q_OBJECT
public:
    explicit PresetWidget(PresetManager* manager, QWidget* parent = nullptr);
signals:
    void presetApplied(const ColorPreset& preset);
private:
    void setupUI();
    void refreshList();
    PresetManager* m_manager;
    QListWidget* m_presetList;
};
} // namespace re4
```

```cpp
// src/ui/presetwidget.cpp
#include "ui/presetwidget.h"
#include "preset/presetmanager.h"
#include <QVBoxLayout>
#include <QHBoxLayout>
#include <QPushButton>
#include <QLabel>

namespace re4 {

PresetWidget::PresetWidget(PresetManager* manager, QWidget* parent)
    : QWidget(parent), m_manager(manager) { setupUI(); }

void PresetWidget::setupUI() {
    auto* layout = new QVBoxLayout(this);

    layout->addWidget(new QLabel("Presets:"));
    m_presetList = new QListWidget(this);
    layout->addWidget(m_presetList);

    auto* btnRow = new QHBoxLayout();
    auto* applyBtn = new QPushButton("Apply");
    auto* saveBtn = new QPushButton("Save Current");
    auto* deleteBtn = new QPushButton("Delete");
    btnRow->addWidget(applyBtn);
    btnRow->addWidget(saveBtn);
    btnRow->addWidget(deleteBtn);
    layout->addLayout(btnRow);

    connect(applyBtn, &QPushButton::clicked, this, [this]() {
        auto* item = m_presetList->currentItem();
        if (item) {
            auto preset = m_manager->getPreset(item->text());
            emit presetApplied(preset);
        }
    });

    connect(deleteBtn, &QPushButton::clicked, this, [this]() {
        auto* item = m_presetList->currentItem();
        if (item) {
            m_manager->deletePreset(item->text());
            refreshList();
        }
    });

    refreshList();
}

void PresetWidget::refreshList() {
    m_presetList->clear();
    for (const auto& name : m_manager->getPresetNames()) {
        m_presetList->addItem(name);
    }
}

} // namespace re4
```

- [ ] **Step 3: Commit**

```bash
git add src/ui/pathconfigwidget.h src/ui/pathconfigwidget.cpp src/ui/presetwidget.h src/ui/presetwidget.cpp
git commit -m "feat: add path config and preset management UI widgets"
```

---

### Task 5: MainWindow Integration

**Files:**
- Modify: `src/mainwindow.h`
- Modify: `src/mainwindow.cpp`

- [ ] **Step 1: Integrate PathManager, BatchProcessor, PresetManager into MainWindow**

Add to mainwindow.h:
```cpp
#include "config/pathmanager.h"
#include "batch/batchprocessor.h"
#include "preset/presetmanager.h"
#include "ui/pathconfigwidget.h"
#include "ui/presetwidget.h"

// Members:
PathManager* m_pathManager;
BatchProcessor* m_batchProcessor;
PresetManager* m_presetManager;
PathConfigWidget* m_pathConfig;
PresetWidget* m_presetWidget;
```

Add to mainwindow.cpp setupUI():
```cpp
// Path config dock
auto* pathDock = new QDockWidget("Mod Paths", this);
m_pathManager = new PathManager(
    QStandardPaths::writableLocation(QStandardPaths::AppDataLocation) + "/config.json", this);
m_pathConfig = new PathConfigWidget(m_pathManager, pathDock);
pathDock->setWidget(m_pathConfig);
addDockWidget(Qt::LeftDockWidgetArea, pathDock);

// Preset dock
auto* presetDock = new QDockWidget("Presets", this);
m_presetManager = new PresetManager(
    QStandardPaths::writableLocation(QStandardPaths::AppDataLocation) + "/presets", this);
m_presetManager->load();
m_presetWidget = new PresetWidget(m_presetManager, presetDock);
presetDock->setWidget(m_presetWidget);
addDockWidget(Qt::RightDockWidgetArea, presetDock);

// Batch processor
m_batchProcessor = new BatchProcessor(this);
```

Connect preset applied signal:
```cpp
connect(m_presetWidget, &PresetWidget::presetApplied, this, [this](const ColorPreset& preset) {
    QImage img = m_preview->currentImage();
    if (img.isNull()) return;
    m_undoRedo->pushState(img, "Apply preset: " + preset.name);
    img = ColorProcessor::adjustHueShift(img, preset.hueShift);
    img = ColorProcessor::adjustSaturation(img, preset.saturation);
    img = ColorProcessor::adjustBrightness(img, static_cast<int>(preset.brightness * 255));
    img = ColorProcessor::adjustContrast(img, preset.contrast);
    m_preview->setImage(img);
});
```

Add menu items:
```cpp
auto* toolsMenu = menuBar()->addMenu("&Tools");
toolsMenu->addAction("&Batch Convert...", this, &MainWindow::openBatchDialog);
toolsMenu->addAction("Save &Preset...", this, &MainWindow::saveCurrentPreset);
```

- [ ] **Step 2: Commit**

```bash
git add src/mainwindow.h src/mainwindow.cpp CMakeLists.txt
git commit -m "feat: integrate path manager, batch processor, and presets into MainWindow"
```

---

## Phase 5 Complete — Summary

| Deliverable | Status |
|------------|--------|
| PathManager (mod config, templates, recent files) | ✅ |
| BatchProcessor (format convert, color adjust, resize) | ✅ |
| PresetManager (create, save, load, export, import) | ✅ |
| PathConfigWidget | ✅ |
| PresetWidget | ✅ |
| MainWindow integration | ✅ |

**Next Phase:** Phase 6 — System Integration + Installer
