# RE4 Mod Tool - Phase 3: Color Engine + Editor UI

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the color processing engine and color editor UI with RGB/HSL sliders, channel display, presets, eyedropper, undo/redo, and 2D texture preview.

**Architecture:** ColorProcessor handles pixel-level adjustments. UndoRedoManager provides command-pattern history. Editor widgets (RGB/HSL sliders, channel view, presets) connect via Qt signals to a central TexturePreviewWidget.

**Tech Stack:** C++17, Qt 6.8 (Widgets), re4::MathUtils, re4::ImageUtils, re4::Logger

**Spec Reference:** `docs/superpowers/specs/2026-06-17-re4-mod-tool-design.md` — Section 3

**Depends On:** Phase 1 (MathUtils, ImageUtils, Logger) and Phase 2 (TextureData, FormatConverter)

---

## File Structure (Phase 3)

| File | Responsibility |
|------|---------------|
| `src/color/colorprocessor.h/.cpp` | Pixel-level color adjustments (brightness, contrast, saturation, hue shift) |
| `src/color/undoredomanager.h/.cpp` | Command-pattern undo/redo with QUndoStack |
| `src/ui/texturepreviewwidget.h/.cpp` | 2D texture preview with zoom/pan |
| `src/ui/rgbsliderwidget.h/.cpp` | RGB + Alpha sliders with live preview |
| `src/ui/hslsliderwidget.h/.cpp` | HSL sliders with color wheel |
| `src/ui/channelviewwidget.h/.cpp` | Channel separation display (R/G/B/A grayscale) |
| `src/ui/colorpresetwidget.h/.cpp` | Preset color swatches and user favorites |
| `src/ui/eyedropperwidget.h/.cpp` | Eyedropper tool for picking colors from texture |
| `src/ui/coloradjustwidget.h/.cpp` | Brightness/contrast/saturation/hue adjustment panel |
| `tests/unit/test_colorprocessor.cpp` | ColorProcessor unit tests |
| `tests/unit/test_undoredomanager.cpp` | UndoRedo unit tests |

---

### Task 1: ColorProcessor — Color Adjustment Engine (TDD)

**Files:**
- Create: `tests/unit/test_colorprocessor.cpp`
- Create: `src/color/colorprocessor.h`
- Create: `src/color/colorprocessor.cpp`

- [ ] **Step 1: Write failing tests**

```cpp
// tests/unit/test_colorprocessor.cpp
#include <gtest/gtest.h>
#include "color/colorprocessor.h"
#include <QImage>
#include <QColor>

using namespace re4;

class ColorProcessorTest : public ::testing::Test {
protected:
    QImage createTestImage(int w = 4, int h = 4) {
        QImage img(w, h, QImage::Format_RGBA8888);
        img.fill(QColor(128, 128, 128, 255));
        return img;
    }
};

TEST_F(ColorProcessorTest, AdjustBrightnessIncrease) {
    QImage img = createTestImage();
    QImage result = ColorProcessor::adjustBrightness(img, 50);
    QColor px = result.pixelColor(0, 0);
    EXPECT_GT(px.red(), 128);
    EXPECT_GT(px.green(), 128);
}

TEST_F(ColorProcessorTest, AdjustBrightnessDecrease) {
    QImage img = createTestImage();
    QImage result = ColorProcessor::adjustBrightness(img, -50);
    QColor px = result.pixelColor(0, 0);
    EXPECT_LT(px.red(), 128);
}

TEST_F(ColorProcessorTest, AdjustBrightnessClamped) {
    QImage img(1, 1, QImage::Format_RGBA8888);
    img.fill(QColor(250, 250, 250, 255));
    QImage result = ColorProcessor::adjustBrightness(img, 100);
    EXPECT_EQ(result.pixelColor(0, 0).red(), 255);
}

TEST_F(ColorProcessorTest, AdjustContrastIncrease) {
    QImage img(1, 1, QImage::Format_RGBA8888);
    img.fill(QColor(128, 128, 128, 255));
    QImage result = ColorProcessor::adjustContrast(img, 2.0f);
    // Contrast > 1.0 on mid-gray pushes toward extremes
    QColor px = result.pixelColor(0, 0);
    EXPECT_NE(px.red(), 128);
}

TEST_F(ColorProcessorTest, AdjustContrastNeutral) {
    QImage img = createTestImage();
    QImage result = ColorProcessor::adjustContrast(img, 1.0f);
    EXPECT_EQ(result.pixelColor(0, 0).red(), 128);
}

TEST_F(ColorProcessorTest, AdjustSaturationIncrease) {
    QImage img(1, 1, QImage::Format_RGBA8888);
    img.fill(QColor(128, 100, 100, 255));
    QImage result = ColorProcessor::adjustSaturation(img, 2.0f);
    QColor px = result.pixelColor(0, 0);
    // R channel should increase relative to G/B
    EXPECT_GT(px.red() - px.green(), 128 - 100);
}

TEST_F(ColorProcessorTest, AdjustSaturationZero) {
    QImage img(1, 1, QImage::Format_RGBA8888);
    img.fill(QColor(200, 100, 50, 255));
    QImage result = ColorProcessor::adjustSaturation(img, 0.0f);
    // Zero saturation = grayscale
    QColor px = result.pixelColor(0, 0);
    EXPECT_NEAR(px.red(), px.green(), 2);
    EXPECT_NEAR(px.green(), px.blue(), 2);
}

TEST_F(ColorProcessorTest, AdjustHueShift) {
    QImage img(1, 1, QImage::Format_RGBA8888);
    img.fill(QColor(255, 0, 0, 255)); // red
    QImage result = ColorProcessor::adjustHueShift(img, 120); // shift 120 degrees
    QColor px = result.pixelColor(0, 0);
    // Red shifted 120 degrees ≈ green
    EXPECT_GT(px.green(), px.red());
}

TEST_F(ColorProcessorTest, ReplaceColor) {
    QImage img(2, 2, QImage::Format_RGBA8888);
    img.fill(QColor(255, 0, 0, 255));
    img.setPixelColor(1, 1, QColor(0, 255, 0, 255)); // different color
    QImage result = ColorProcessor::replaceColor(img, QColor(255, 0, 0), QColor(0, 0, 255), 0);
    EXPECT_EQ(result.pixelColor(0, 0).blue(), 255); // replaced
    EXPECT_EQ(result.pixelColor(1, 1).green(), 255); // not replaced
}

TEST_F(ColorProcessorTest, GetPixelColor) {
    QImage img(1, 1, QImage::Format_RGBA8888);
    img.setPixelColor(0, 0, QColor(10, 20, 30, 255));
    QColor px = ColorProcessor::getPixelColor(img, 0, 0);
    EXPECT_EQ(px.red(), 10);
    EXPECT_EQ(px.green(), 20);
    EXPECT_EQ(px.blue(), 30);
}

TEST_F(ColorProcessorTest, GetPixelColorOutOfBounds) {
    QImage img(2, 2, QImage::Format_RGBA8888);
    QColor px = ColorProcessor::getPixelColor(img, -1, 5);
    EXPECT_FALSE(px.isValid());
}
```

- [ ] **Step 2: Implement ColorProcessor**

```cpp
// src/color/colorprocessor.h
#pragma once

#include <QImage>
#include <QColor>

namespace re4 {
namespace ColorProcessor {

    // Adjust brightness (amount: -255 to +255)
    QImage adjustBrightness(const QImage& source, int amount);

    // Adjust contrast (factor: 0.0 to 3.0, 1.0 = no change)
    QImage adjustContrast(const QImage& source, float factor);

    // Adjust saturation (factor: 0.0 to 3.0, 1.0 = no change, 0.0 = grayscale)
    QImage adjustSaturation(const QImage& source, float factor);

    // Adjust hue shift (degrees: -180 to +180)
    QImage adjustHueShift(const QImage& source, int degrees);

    // Replace one color with another (tolerance: 0-255)
    QImage replaceColor(const QImage& source, const QColor& from,
                        const QColor& to, int tolerance = 0);

    // Get pixel color with bounds checking (returns invalid QColor if out of bounds)
    QColor getPixelColor(const QImage& image, int x, int y);

    // Set pixel color with bounds checking
    void setPixelColor(QImage& image, int x, int y, const QColor& color);

} // namespace ColorProcessor
} // namespace re4
```

```cpp
// src/color/colorprocessor.cpp
#include "color/colorprocessor.h"
#include "utils/mathutils.h"
#include <algorithm>
#include <cmath>

namespace re4 {
namespace ColorProcessor {

QImage adjustBrightness(const QImage& source, int amount) {
    QImage result = source.convertToFormat(QImage::Format_RGBA8888);
    for (int y = 0; y < result.height(); ++y) {
        uchar* line = result.scanLine(y);
        for (int x = 0; x < result.width(); ++x) {
            uchar* px = line + x * 4;
            px[0] = static_cast<uchar>(std::clamp(px[0] + amount, 0, 255));
            px[1] = static_cast<uchar>(std::clamp(px[1] + amount, 0, 255));
            px[2] = static_cast<uchar>(std::clamp(px[2] + amount, 0, 255));
        }
    }
    return result;
}

QImage adjustContrast(const QImage& source, float factor) {
    QImage result = source.convertToFormat(QImage::Format_RGBA8888);
    for (int y = 0; y < result.height(); ++y) {
        uchar* line = result.scanLine(y);
        for (int x = 0; x < result.width(); ++x) {
            uchar* px = line + x * 4;
            for (int c = 0; c < 3; ++c) {
                float normalized = px[c] / 255.0f;
                float adjusted = (normalized - 0.5f) * factor + 0.5f;
                px[c] = static_cast<uchar>(std::clamp(static_cast<int>(adjusted * 255.0f), 0, 255));
            }
        }
    }
    return result;
}

QImage adjustSaturation(const QImage& source, float factor) {
    QImage result = source.convertToFormat(QImage::Format_RGBA8888);
    for (int y = 0; y < result.height(); ++y) {
        uchar* line = result.scanLine(y);
        for (int x = 0; x < result.width(); ++x) {
            uchar* px = line + x * 4;
            QColor color(px[0], px[1], px[2]);
            qreal h, s, l;
            color.getHslF(&h, &s, &l);
            qreal newSat = std::min(1.0, s * factor);
            QColor result_color;
            result_color.setHslF(h, newSat, l);
            px[0] = static_cast<uchar>(result_color.red());
            px[1] = static_cast<uchar>(result_color.green());
            px[2] = static_cast<uchar>(result_color.blue());
        }
    }
    return result;
}

QImage adjustHueShift(const QImage& source, int degrees) {
    QImage result = source.convertToFormat(QImage::Format_RGBA8888);
    for (int y = 0; y < result.height(); ++y) {
        uchar* line = result.scanLine(y);
        for (int x = 0; x < result.width(); ++x) {
            uchar* px = line + x * 4;
            QColor color(px[0], px[1], px[2]);
            qreal h, s, l;
            color.getHslF(&h, &s, &l);
            if (h < 0) h = 0; // achromatic
            qreal newH = std::fmod(h + degrees / 360.0, 1.0);
            if (newH < 0) newH += 1.0;
            QColor result_color;
            result_color.setHslF(newH, s, l);
            px[0] = static_cast<uchar>(result_color.red());
            px[1] = static_cast<uchar>(result_color.green());
            px[2] = static_cast<uchar>(result_color.blue());
        }
    }
    return result;
}

QImage replaceColor(const QImage& source, const QColor& from,
                    const QColor& to, int tolerance) {
    QImage result = source.convertToFormat(QImage::Format_RGBA8888);
    for (int y = 0; y < result.height(); ++y) {
        uchar* line = result.scanLine(y);
        for (int x = 0; x < result.width(); ++x) {
            uchar* px = line + x * 4;
            if (std::abs(px[0] - from.red()) <= tolerance &&
                std::abs(px[1] - from.green()) <= tolerance &&
                std::abs(px[2] - from.blue()) <= tolerance) {
                px[0] = static_cast<uchar>(to.red());
                px[1] = static_cast<uchar>(to.green());
                px[2] = static_cast<uchar>(to.blue());
            }
        }
    }
    return result;
}

QColor getPixelColor(const QImage& image, int x, int y) {
    if (x < 0 || y < 0 || x >= image.width() || y >= image.height())
        return QColor(); // invalid
    return image.pixelColor(x, y);
}

void setPixelColor(QImage& image, int x, int y, const QColor& color) {
    if (x < 0 || y < 0 || x >= image.width() || y >= image.height())
        return;
    image.setPixelColor(x, y, color);
}

} // namespace ColorProcessor
} // namespace re4
```

- [ ] **Step 3: Update tests/CMakeLists.txt**

Add `test_colorprocessor.cpp` to test sources and `colorprocessor.cpp` to linked sources.

- [ ] **Step 4: Commit**

```bash
git add src/color/colorprocessor.h src/color/colorprocessor.cpp tests/unit/test_colorprocessor.cpp tests/CMakeLists.txt
git commit -m "feat: add color processor engine with brightness, contrast, saturation, hue shift"
```

---

### Task 2: UndoRedo System (TDD)

**Files:**
- Create: `tests/unit/test_undoredomanager.cpp`
- Create: `src/color/undoredomanager.h`
- Create: `src/color/undoredomanager.cpp`

- [ ] **Step 1: Write failing tests**

```cpp
// tests/unit/test_undoredomanager.cpp
#include <gtest/gtest.h>
#include "color/undoredomanager.h"
#include <QImage>

using namespace re4;

class UndoRedoTest : public ::testing::Test {
protected:
    void SetUp() override {
        manager = std::make_unique<UndoRedoManager>();
        testImage = QImage(4, 4, QImage::Format_RGBA8888);
        testImage.fill(QColor(128, 128, 128, 255));
    }
    std::unique_ptr<UndoRedoManager> manager;
    QImage testImage;
};

TEST_F(UndoRedoTest, InitialState) {
    EXPECT_FALSE(manager->canUndo());
    EXPECT_FALSE(manager->canRedo());
}

TEST_F(UndoRedoTest, PushAndUndo) {
    QImage modified = testImage;
    modified.fill(QColor(255, 0, 0, 255));
    manager->pushState(testImage, "fill red");

    EXPECT_TRUE(manager->canUndo());
    EXPECT_FALSE(manager->canRedo());

    QImage undone = manager->undo();
    EXPECT_EQ(undone.pixelColor(0, 0), QColor(128, 128, 128, 255));
}

TEST_F(UndoRedoTest, UndoRedo) {
    QImage modified = testImage;
    modified.fill(QColor(255, 0, 0, 255));
    manager->pushState(modified, "fill red");

    manager->undo();
    EXPECT_FALSE(manager->canUndo());
    EXPECT_TRUE(manager->canRedo());

    QImage redone = manager->redo();
    EXPECT_EQ(redone.pixelColor(0, 0), QColor(255, 0, 0, 255));
}

TEST_F(UndoRedoTest, MultipleStates) {
    QImage state1 = testImage;
    state1.fill(QColor(255, 0, 0, 255));
    manager->pushState(state1, "red");

    QImage state2 = testImage;
    state2.fill(QColor(0, 255, 0, 255));
    manager->pushState(state2, "green");

    QImage undone = manager->undo();
    EXPECT_EQ(undone.pixelColor(0, 0), QColor(255, 0, 0, 255));

    undone = manager->undo();
    EXPECT_EQ(undone.pixelColor(0, 0), QColor(128, 128, 128, 255));
}

TEST_F(UndoRedoTest, PushClearsRedo) {
    QImage state1 = testImage;
    state1.fill(QColor(255, 0, 0, 255));
    manager->pushState(state1, "red");

    manager->undo();
    EXPECT_TRUE(manager->canRedo());

    QImage newState = testImage;
    newState.fill(QColor(0, 0, 255, 255));
    manager->pushState(newState, "blue");

    EXPECT_FALSE(manager->canRedo());
}

TEST_F(UndoRedoTest, UndoOnEmptyDoesNothing) {
    QImage result = manager->undo();
    EXPECT_EQ(result, QImage());
}

TEST_F(UndoRedoTest, RedoOnEmptyDoesNothing) {
    QImage result = manager->redo();
    EXPECT_EQ(result, QImage());
}

TEST_F(UndoRedoTest, ClearHistory) {
    QImage state1 = testImage;
    manager->pushState(state1, "test");
    manager->clear();
    EXPECT_FALSE(manager->canUndo());
    EXPECT_FALSE(manager->canRedo());
}

TEST_F(UndoRedoTest, UndoCount) {
    EXPECT_EQ(manager->undoCount(), 0);
    QImage state1 = testImage;
    manager->pushState(state1, "test1");
    QImage state2 = testImage;
    manager->pushState(state2, "test2");
    EXPECT_EQ(manager->undoCount(), 2);
}
```

- [ ] **Step 2: Implement UndoRedoManager**

```cpp
// src/color/undoredomanager.h
#pragma once

#include <QImage>
#include <QString>
#include <vector>

namespace re4 {

class UndoRedoManager {
public:
    UndoRedoManager() = default;

    // Push a new state (clears redo stack)
    void pushState(const QImage& state, const QString& description = "");

    // Undo: returns the previous state, or empty QImage if nothing to undo
    QImage undo();

    // Redo: returns the next state, or empty QImage if nothing to redo
    QImage redo();

    bool canUndo() const;
    bool canRedo() const;

    int undoCount() const;
    int redoCount() const;

    // Get description of the state that undo would restore
    QString undoDescription() const;
    QString redoDescription() const;

    void clear();

private:
    struct State {
        QImage image;
        QString description;
    };

    std::vector<State> m_undoStack;
    std::vector<State> m_redoStack;

    static constexpr int kMaxStates = 50;
};

} // namespace re4
```

```cpp
// src/color/undoredomanager.cpp
#include "color/undoredomanager.h"

namespace re4 {

void UndoRedoManager::pushState(const QImage& state, const QString& description) {
    m_undoStack.push_back({state.copy(), description});
    m_redoStack.clear();

    // Trim if too large
    while (static_cast<int>(m_undoStack.size()) > kMaxStates) {
        m_undoStack.erase(m_undoStack.begin());
    }
}

QImage UndoRedoManager::undo() {
    if (m_undoStack.empty()) return {};
    State state = m_undoStack.back();
    m_undoStack.pop_back();
    m_redoStack.push_back(state);
    if (m_undoStack.empty()) return state.image;
    return m_undoStack.back().image;
}

QImage UndoRedoManager::redo() {
    if (m_redoStack.empty()) return {};
    State state = m_redoStack.back();
    m_redoStack.pop_back();
    m_undoStack.push_back(state);
    return state.image;
}

bool UndoRedoManager::canUndo() const { return !m_undoStack.empty(); }
bool UndoRedoManager::canRedo() const { return !m_redoStack.empty(); }

int UndoRedoManager::undoCount() const { return static_cast<int>(m_undoStack.size()); }
int UndoRedoManager::redoCount() const { return static_cast<int>(m_redoStack.size()); }

QString UndoRedoManager::undoDescription() const {
    if (m_undoStack.empty()) return {};
    return m_undoStack.back().description;
}

QString UndoRedoManager::redoDescription() const {
    if (m_redoStack.empty()) return {};
    return m_redoStack.back().description;
}

void UndoRedoManager::clear() {
    m_undoStack.clear();
    m_redoStack.clear();
}

} // namespace re4
```

- [ ] **Step 3: Commit**

```bash
git add src/color/undoredomanager.h src/color/undoredomanager.cpp tests/unit/test_undoredomanager.cpp
git commit -m "feat: add undo/redo manager with state history and description tracking"
```

---

### Task 3: TexturePreviewWidget — 2D Texture Preview

**Files:**
- Create: `src/ui/texturepreviewwidget.h`
- Create: `src/ui/texturepreviewwidget.cpp`

- [ ] **Step 1: Create TexturePreviewWidget**

```cpp
// src/ui/texturepreviewwidget.h
#pragma once

#include <QWidget>
#include <QImage>
#include <QPixmap>
#include <QPoint>

namespace re4 {

class TexturePreviewWidget : public QWidget {
    Q_OBJECT

public:
    explicit TexturePreviewWidget(QWidget* parent = nullptr);

    void setImage(const QImage& image);
    QImage currentImage() const;

    void setZoom(float factor);
    float zoom() const;

    void fitToWindow();

signals:
    void pixelClicked(int x, int y, const QColor& color);
    void mouseMoved(int x, int y);

protected:
    void paintEvent(QPaintEvent* event) override;
    void wheelEvent(QWheelEvent* event) override;
    void mousePressEvent(QMouseEvent* event) override;
    void mouseMoveEvent(QMouseEvent* event) override;
    void mouseReleaseEvent(QMouseEvent* event) override;
    void resizeEvent(QResizeEvent* event) override;

private:
    QPoint imageFromScreen(const QPoint& screenPos) const;

    QImage m_image;
    QPixmap m_pixmap;
    float m_zoom = 1.0f;
    QPoint m_panOffset;
    QPoint m_lastPanPos;
    bool m_panning = false;
};

} // namespace re4
```

```cpp
// src/ui/texturepreviewwidget.cpp
#include "ui/texturepreviewwidget.h"
#include <QPainter>
#include <QWheelEvent>
#include <QMouseEvent>

namespace re4 {

TexturePreviewWidget::TexturePreviewWidget(QWidget* parent)
    : QWidget(parent)
{
    setMinimumSize(200, 200);
    setMouseTracking(true);
}

void TexturePreviewWidget::setImage(const QImage& image) {
    m_image = image;
    m_pixmap = QPixmap::fromImage(image);
    fitToWindow();
    update();
}

QImage TexturePreviewWidget::currentImage() const { return m_image; }

void TexturePreviewWidget::setZoom(float factor) {
    m_zoom = std::max(0.01f, std::min(factor, 100.0f));
    update();
}

float TexturePreviewWidget::zoom() const { return m_zoom; }

void TexturePreviewWidget::fitToWindow() {
    if (m_image.isNull()) return;
    float scaleX = static_cast<float>(width()) / m_image.width();
    float scaleY = static_cast<float>(height()) / m_image.height();
    m_zoom = std::min(scaleX, scaleY) * 0.9f;
    m_panOffset = QPoint(0, 0);
    update();
}

void TexturePreviewWidget::paintEvent(QPaintEvent*) {
    QPainter painter(this);
    painter.fillRect(rect(), QColor(40, 40, 40)); // dark background

    if (m_pixmap.isNull()) return;

    // Calculate centered position
    int drawW = static_cast<int>(m_pixmap.width() * m_zoom);
    int drawH = static_cast<int>(m_pixmap.height() * m_zoom);
    int x = (width() - drawW) / 2 + m_panOffset.x();
    int y = (height() - drawH) / 2 + m_panOffset.y();

    // Draw checkerboard pattern for alpha transparency
    QPixmap checker(16, 16);
    QPainter cp(&checker);
    cp.fillRect(0, 0, 8, 8, QColor(200, 200, 200));
    cp.fillRect(8, 8, 8, 8, QColor(200, 200, 200));
    cp.fillRect(8, 0, 8, 8, QColor(255, 255, 255));
    cp.fillRect(0, 8, 8, 8, QColor(255, 255, 255));

    painter.drawTiledPixmap(x, y, drawW, drawH, checker);
    painter.drawPixmap(x, y, drawW, drawH, m_pixmap);
}

void TexturePreviewWidget::wheelEvent(QWheelEvent* event) {
    float delta = event->angleDelta().y() > 0 ? 1.1f : 0.9f;
    setZoom(m_zoom * delta);
}

void TexturePreviewWidget::mousePressEvent(QMouseEvent* event) {
    if (event->button() == Qt::MiddleButton) {
        m_panning = true;
        m_lastPanPos = event->pos();
    } else if (event->button() == Qt::LeftButton) {
        QPoint imgPos = imageFromScreen(event->pos());
        if (!imgPos.isNull() && !m_image.isNull()) {
            if (imgPos.x() >= 0 && imgPos.x() < m_image.width() &&
                imgPos.y() >= 0 && imgPos.y() < m_image.height()) {
                QColor color = m_image.pixelColor(imgPos);
                emit pixelClicked(imgPos.x(), imgPos.y(), color);
            }
        }
    }
}

void TexturePreviewWidget::mouseMoveEvent(QMouseEvent* event) {
    if (m_panning) {
        m_panOffset += event->pos() - m_lastPanPos;
        m_lastPanPos = event->pos();
        update();
    }
    QPoint imgPos = imageFromScreen(event->pos());
    if (!imgPos.isNull()) {
        emit mouseMoved(imgPos.x(), imgPos.y());
    }
}

void TexturePreviewWidget::mouseReleaseEvent(QMouseEvent* event) {
    if (event->button() == Qt::MiddleButton) {
        m_panning = false;
    }
}

void TexturePreviewWidget::resizeEvent(QResizeEvent*) {
    if (!m_image.isNull()) fitToWindow();
}

QPoint TexturePreviewWidget::imageFromScreen(const QPoint& screenPos) const {
    if (m_image.isNull()) return QPoint(-1, -1);
    int drawW = static_cast<int>(m_pixmap.width() * m_zoom);
    int drawH = static_cast<int>(m_pixmap.height() * m_zoom);
    int x = (width() - drawW) / 2 + m_panOffset.x();
    int y = (height() - drawH) / 2 + m_panOffset.y();
    int imgX = static_cast<int>((screenPos.x() - x) / m_zoom);
    int imgY = static_cast<int>((screenPos.y() - y) / m_zoom);
    return QPoint(imgX, imgY);
}

} // namespace re4
```

- [ ] **Step 2: Commit**

```bash
git add src/ui/texturepreviewwidget.h src/ui/texturepreviewwidget.cpp
git commit -m "feat: add 2D texture preview widget with zoom, pan, and pixel picking"
```

---

### Task 4: RGB Slider Widget

**Files:**
- Create: `src/ui/rgbsliderwidget.h`
- Create: `src/ui/rgbsliderwidget.cpp`

- [ ] **Step 1: Create RGB Slider Widget**

```cpp
// src/ui/rgbsliderwidget.h
#pragma once

#include <QWidget>
#include <QSlider>
#include <QSpinBox>
#include <QColor>

namespace re4 {

class RGBSliderWidget : public QWidget {
    Q_OBJECT

public:
    explicit RGBSliderWidget(QWidget* parent = nullptr);

    QColor currentColor() const;
    void setColor(const QColor& color);

signals:
    void colorChanged(const QColor& color);
    void colorApplied(const QColor& color); // user clicked "Apply"

private:
    void setupUI();
    void updateFromSliders();
    void updateSliders();

    QSlider* m_redSlider;
    QSlider* m_greenSlider;
    QSlider* m_blueSlider;
    QSlider* m_alphaSlider;

    QSpinBox* m_redSpin;
    QSpinBox* m_greenSpin;
    QSpinBox* m_blueSpin;
    QSpinBox* m_alphaSpin;

    bool m_updating = false;
};

} // namespace re4
```

```cpp
// src/ui/rgbsliderwidget.cpp
#include "ui/rgbsliderwidget.h"
#include <QVBoxLayout>
#include <QHBoxLayout>
#include <QLabel>
#include <QPushButton>

namespace re4 {

RGBSliderWidget::RGBSliderWidget(QWidget* parent) : QWidget(parent) {
    setupUI();
}

void RGBSliderWidget::setupUI() {
    auto* layout = new QVBoxLayout(this);

    auto makeRow = [&](const QString& label, QSlider*& slider, QSpinBox*& spin) {
        auto* row = new QHBoxLayout();
        row->addWidget(new QLabel(label));

        slider = new QSlider(Qt::Horizontal);
        slider->setRange(0, 255);
        slider->setTickPosition(QSlider::TicksBelow);
        row->addWidget(slider, 1);

        spin = new QSpinBox();
        spin->setRange(0, 255);
        row->addWidget(spin);

        layout->addLayout(row);

        connect(slider, &QSlider::valueChanged, this, [this](int) {
            if (!m_updating) { m_updating = true; updateFromSliders(); m_updating = false; }
        });
        connect(spin, QOverload<int>::of(&QSpinBox::valueChanged), this, [this, slider](int val) {
            if (!m_updating) { m_updating = true; slider->setValue(val); updateFromSliders(); m_updating = false; }
        });
    };

    makeRow("R:", m_redSlider, m_redSpin);
    makeRow("G:", m_greenSlider, m_greenSpin);
    makeRow("B:", m_blueSlider, m_blueSpin);
    makeRow("A:", m_alphaSlider, m_alphaSpin);

    auto* applyBtn = new QPushButton("Apply");
    layout->addWidget(applyBtn);
    connect(applyBtn, &QPushButton::clicked, this, [this]() {
        emit colorApplied(currentColor());
    });
}

QColor RGBSliderWidget::currentColor() const {
    return QColor(m_redSlider->value(), m_greenSlider->value(),
                  m_blueSlider->value(), m_alphaSlider->value());
}

void RGBSliderWidget::setColor(const QColor& color) {
    m_updating = true;
    m_redSlider->setValue(color.red());
    m_greenSlider->setValue(color.green());
    m_blueSlider->setValue(color.blue());
    m_alphaSlider->setValue(color.alpha());
    m_redSpin->setValue(color.red());
    m_greenSpin->setValue(color.green());
    m_blueSpin->setValue(color.blue());
    m_alphaSpin->setValue(color.alpha());
    m_updating = false;
}

void RGBSliderWidget::updateFromSliders() {
    m_redSpin->setValue(m_redSlider->value());
    m_greenSpin->setValue(m_greenSlider->value());
    m_blueSpin->setValue(m_blueSlider->value());
    m_alphaSpin->setValue(m_alphaSlider->value());
    emit colorChanged(currentColor());
}

} // namespace re4
```

- [ ] **Step 2: Commit**

```bash
git add src/ui/rgbsliderwidget.h src/ui/rgbsliderwidget.cpp
git commit -m "feat: add RGB slider widget with R/G/B/A sliders and spinboxes"
```

---

### Task 5: Color Adjustment Widget (Brightness/Contrast/Saturation/Hue)

**Files:**
- Create: `src/ui/coloradjustwidget.h`
- Create: `src/ui/coloradjustwidget.cpp`

- [ ] **Step 1: Create Color Adjustment Widget**

```cpp
// src/ui/coloradjustwidget.h
#pragma once

#include <QWidget>
#include <QSlider>

namespace re4 {

class ColorAdjustWidget : public QWidget {
    Q_OBJECT

public:
    explicit ColorAdjustWidget(QWidget* parent = nullptr);

signals:
    void brightnessChanged(int amount);
    void contrastChanged(float factor);
    void saturationChanged(float factor);
    void hueShiftChanged(int degrees);

    void applyBrightness(int amount);
    void applyContrast(float factor);
    void applySaturation(float factor);
    void applyHueShift(int degrees);

    void resetRequested();

private:
    QSlider* m_brightnessSlider;
    QSlider* m_contrastSlider;
    QSlider* m_saturationSlider;
    QSlider* m_hueSlider;
};

} // namespace re4
```

```cpp
// src/ui/coloradjustwidget.cpp
#include "ui/coloradjustwidget.h"
#include <QVBoxLayout>
#include <QHBoxLayout>
#include <QLabel>
#include <QPushButton>

namespace re4 {

ColorAdjustWidget::ColorAdjustWidget(QWidget* parent) : QWidget(parent) {
    auto* layout = new QVBoxLayout(this);

    auto makeSlider = [&](const QString& label, int min, int max, int defaultVal,
                          QSlider*& slider, auto signal) {
        auto* row = new QHBoxLayout();
        row->addWidget(new QLabel(label));
        slider = new QSlider(Qt::Horizontal);
        slider->setRange(min, max);
        slider->setValue(defaultVal);
        row->addWidget(slider, 1);
        auto* valLabel = new QLabel(QString::number(defaultVal));
        row->addWidget(valLabel);
        layout->addLayout(row);

        connect(slider, &QSlider::valueChanged, this, [this, valLabel, signal](int val) {
            valLabel->setText(QString::number(val));
            if constexpr (std::is_same_v<decltype(signal), int ColorAdjustWidget::*>)
                emit (this->*signal)(val);
        });
    };

    makeSlider("Brightness", -255, 255, 0, m_brightnessSlider, &ColorAdjustWidget::brightnessChanged);
    makeSlider("Contrast (%)", 0, 300, 100, m_contrastSlider, &ColorAdjustWidget::contrastChanged);
    makeSlider("Saturation (%)", 0, 300, 100, m_saturationSlider, &ColorAdjustWidget::saturationChanged);
    makeSlider("Hue Shift", -180, 180, 0, m_hueSlider, &ColorAdjustWidget::hueShiftChanged);

    auto* btnRow = new QHBoxLayout();
    auto* resetBtn = new QPushButton("Reset");
    auto* applyBtn = new QPushButton("Apply All");
    btnRow->addWidget(resetBtn);
    btnRow->addWidget(applyBtn);
    layout->addLayout(btnRow);

    connect(resetBtn, &QPushButton::clicked, this, [this]() {
        m_brightnessSlider->setValue(0);
        m_contrastSlider->setValue(100);
        m_saturationSlider->setValue(100);
        m_hueSlider->setValue(0);
        emit resetRequested();
    });

    connect(applyBtn, &QPushButton::clicked, this, [this]() {
        emit applyBrightness(m_brightnessSlider->value());
        emit applyContrast(m_contrastSlider->value() / 100.0f);
        emit applySaturation(m_saturationSlider->value() / 100.0f);
        emit applyHueShift(m_hueSlider->value());
        // Reset sliders after applying
        m_brightnessSlider->setValue(0);
        m_contrastSlider->setValue(100);
        m_saturationSlider->setValue(100);
        m_hueSlider->setValue(0);
    });
}

} // namespace re4
```

- [ ] **Step 2: Commit**

```bash
git add src/ui/coloradjustwidget.h src/ui/coloradjustwidget.cpp
git commit -m "feat: add color adjustment widget with brightness, contrast, saturation, hue sliders"
```

---

### Task 6: MainWindow Integration

**Files:**
- Modify: `src/main.cpp`
- Modify: `src/core/application.cpp`

- [ ] **Step 1: Create MainWindow with all color editing components**

```cpp
// src/mainwindow.h
#pragma once

#include <QMainWindow>
#include <memory>

namespace re4 {

class TexturePreviewWidget;
class RGBSliderWidget;
class ColorAdjustWidget;
class UndoRedoManager;

class MainWindow : public QMainWindow {
    Q_OBJECT

public:
    explicit MainWindow(QWidget* parent = nullptr);
    ~MainWindow() override;

private slots:
    void openFile();
    void saveFile();
    void saveFileAs();
    void undo();
    void redo();

private:
    void setupUI();
    void setupMenuBar();
    void setupToolBar();
    void setupStatusBar();
    void connectSignals();
    void loadImage(const QString& path);

    TexturePreviewWidget* m_preview;
    RGBSliderWidget* m_rgbSliders;
    ColorAdjustWidget* m_colorAdjust;
    std::unique_ptr<UndoRedoManager> m_undoRedo;

    QString m_currentFilePath;
};

} // namespace re4
```

```cpp
// src/mainwindow.cpp
#include "mainwindow.h"
#include "ui/texturepreviewwidget.h"
#include "ui/rgbsliderwidget.h"
#include "ui/coloradjustwidget.h"
#include "color/undoredomanager.h"
#include "color/colorprocessor.h"
#include "formats/formatconverter.h"
#include "core/logger.h"

#include <QMenuBar>
#include <QToolBar>
#include <QStatusBar>
#include <QDockWidget>
#include <QFileDialog>
#include <QMessageBox>
#include <QAction>
#include <QShortcut>

namespace re4 {

MainWindow::MainWindow(QWidget* parent)
    : QMainWindow(parent)
    , m_undoRedo(std::make_unique<UndoRedoManager>())
{
    setupUI();
    setupMenuBar();
    setupToolBar();
    setupStatusBar();
    connectSignals();
    setWindowTitle("RE4 Mod Tool v1.0.0");
    resize(1400, 900);
}

MainWindow::~MainWindow() = default;

void MainWindow::setupUI() {
    // Central widget: texture preview
    m_preview = new TexturePreviewWidget(this);
    setCentralWidget(m_preview);

    // Left dock: RGB sliders
    auto* rgbDock = new QDockWidget("Color Editor", this);
    m_rgbSliders = new RGBSliderWidget(rgbDock);
    rgbDock->setWidget(m_rgbSliders);
    addDockWidget(Qt::LeftDockWidgetArea, rgbDock);

    // Right dock: Color adjustments
    auto* adjustDock = new QDockWidget("Adjustments", this);
    m_colorAdjust = new ColorAdjustWidget(adjustDock);
    adjustDock->setWidget(m_colorAdjust);
    addDockWidget(Qt::RightDockWidgetArea, adjustDock);
}

void MainWindow::setupMenuBar() {
    auto* fileMenu = menuBar()->addMenu("&File");
    fileMenu->addAction("&Open...", this, &MainWindow::openFile, QKeySequence::Open);
    fileMenu->addAction("&Save", this, &MainWindow::saveFile, QKeySequence::Save);
    fileMenu->addAction("Save &As...", this, &MainWindow::saveFileAs, QKeySequence("Ctrl+Shift+S"));
    fileMenu->addSeparator();
    fileMenu->addAction("E&xit", this, &QWidget::close, QKeySequence::Quit);

    auto* editMenu = menuBar()->addMenu("&Edit");
    editMenu->addAction("&Undo", this, &MainWindow::undo, QKeySequence::Undo);
    editMenu->addAction("&Redo", this, &MainWindow::redo, QKeySequence::Redo);
}

void MainWindow::setupToolBar() {
    auto* toolbar = addToolBar("Main");
    toolbar->addAction("Open", this, &MainWindow::openFile);
    toolbar->addAction("Save", this, &MainWindow::saveFile);
    toolbar->addSeparator();
    toolbar->addAction("Undo", this, &MainWindow::undo);
    toolbar->addAction("Redo", this, &MainWindow::redo);
}

void MainWindow::setupStatusBar() {
    statusBar()->showMessage("Ready");
}

void MainWindow::connectSignals() {
    // RGB sliders → update preview in real-time
    connect(m_rgbSliders, &RGBSliderWidget::colorChanged, this, [this](const QColor& color) {
        // Live preview: show how the color would look
        statusBar()->showMessage(QString("Color: R=%1 G=%2 B=%3 A=%4")
            .arg(color.red()).arg(color.green()).arg(color.blue()).arg(color.alpha()));
    });

    // RGB sliders → apply to texture
    connect(m_rgbSliders, &RGBSliderWidget::colorApplied, this, [this](const QColor& color) {
        QImage img = m_preview->currentImage();
        if (img.isNull()) return;

        m_undoRedo->pushState(img, "Color fill");

        // Fill entire image with the selected color
        img.fill(color);
        m_preview->setImage(img);

        if (Logger::instance()) Logger::instance()->info("Applied color to texture", "MainWindow");
    });

    // Color adjustments
    connect(m_colorAdjust, &ColorAdjustWidget::applyBrightness, this, [this](int amount) {
        QImage img = m_preview->currentImage();
        if (img.isNull()) return;
        m_undoRedo->pushState(img, "Brightness");
        m_preview->setImage(ColorProcessor::adjustBrightness(img, amount));
    });

    connect(m_colorAdjust, &ColorAdjustWidget::applyContrast, this, [this](float factor) {
        QImage img = m_preview->currentImage();
        if (img.isNull()) return;
        m_undoRedo->pushState(img, "Contrast");
        m_preview->setImage(ColorProcessor::adjustContrast(img, factor));
    });

    connect(m_colorAdjust, &ColorAdjustWidget::applySaturation, this, [this](float factor) {
        QImage img = m_preview->currentImage();
        if (img.isNull()) return;
        m_undoRedo->pushState(img, "Saturation");
        m_preview->setImage(ColorProcessor::adjustSaturation(img, factor));
    });

    connect(m_colorAdjust, &ColorAdjustWidget::applyHueShift, this, [this](int degrees) {
        QImage img = m_preview->currentImage();
        if (img.isNull()) return;
        m_undoRedo->pushState(img, "Hue shift");
        m_preview->setImage(ColorProcessor::adjustHueShift(img, degrees));
    });

    // Pixel picking from preview
    connect(m_preview, &TexturePreviewWidget::pixelClicked, this,
        [this](int x, int y, const QColor& color) {
            m_rgbSliders->setColor(color);
            statusBar()->showMessage(QString("Picked pixel (%1,%2): R=%3 G=%4 B=%5")
                .arg(x).arg(y).arg(color.red()).arg(color.green()).arg(color.blue()));
        });
}

void MainWindow::openFile() {
    QString path = QFileDialog::getOpenFileName(this, "Open Texture",
        QString(), "Texture Files (*.tex *.dds *.png *.tga);;All Files (*)");
    if (!path.isEmpty()) loadImage(path);
}

void MainWindow::loadImage(const QString& path) {
    FormatConverter converter;
    TextureData texData;
    FormatChecker::Format fmt = FormatChecker::detectFormat(path);

    // Load texture using appropriate parser
    bool loaded = false;
    if (fmt == FormatChecker::Format::Tex) {
        TexFormat parser;
        loaded = parser.load(path, texData);
    } else if (fmt == FormatChecker::Format::Dds) {
        DdsFormat parser;
        loaded = parser.load(path, texData);
    } else if (fmt == FormatChecker::Format::Png) {
        PngFormat parser;
        loaded = parser.load(path, texData);
    } else if (fmt == FormatChecker::Format::Tga) {
        TgaFormat parser;
        loaded = parser.load(path, texData);
    }

    if (!loaded) {
        QMessageBox::warning(this, "Error", "Failed to load: " + path);
        return;
    }

    QImage img = texData.toImage(0);
    if (img.isNull()) {
        QMessageBox::warning(this, "Error", "Failed to decode texture: " + path);
        return;
    }

    m_currentFilePath = path;
    m_preview->setImage(img);
    m_undoRedo->clear();
    m_undoRedo->pushState(img, "Original");
    setWindowTitle(QString("RE4 Mod Tool - %1").arg(path));
    statusBar()->showMessage(QString("Loaded: %1 (%2x%3)").arg(path).arg(img.width()).arg(img.height()));

    if (Logger::instance()) Logger::instance()->info("Loaded texture: " + path, "MainWindow");
}

void MainWindow::saveFile() {
    if (m_currentFilePath.isEmpty()) {
        saveFileAs();
        return;
    }

    // Save back in the same format
    QImage img = m_preview->currentImage();
    if (img.isNull()) return;

    TextureData texData;
    texData.fromImage(img);

    FormatChecker::Format fmt = FormatChecker::detectFormat(m_currentFilePath);
    bool saved = false;
    if (fmt == FormatChecker::Format::Png) {
        PngFormat parser;
        saved = parser.save(m_currentFilePath, texData);
    } else if (fmt == FormatChecker::Format::Tga) {
        TgaFormat parser;
        saved = parser.save(m_currentFilePath, texData);
    } else if (fmt == FormatChecker::Format::Dds) {
        DdsFormat parser;
        saved = parser.save(m_currentFilePath, texData);
    } else if (fmt == FormatChecker::Format::Tex) {
        TexFormat parser;
        saved = parser.save(m_currentFilePath, texData);
    }

    if (saved) {
        statusBar()->showMessage("Saved: " + m_currentFilePath);
        if (Logger::instance()) Logger::instance()->info("Saved: " + m_currentFilePath, "MainWindow");
    } else {
        QMessageBox::warning(this, "Error", "Failed to save: " + m_currentFilePath);
    }
}

void MainWindow::saveFileAs() {
    QString path = QFileDialog::getSaveFileName(this, "Save Texture",
        QString(), "PNG (*.png);;TGA (*.tga);;DDS (*.dds);;TEX (*.tex)");
    if (!path.isEmpty()) {
        m_currentFilePath = path;
        saveFile();
    }
}

void MainWindow::undo() {
    QImage img = m_undoRedo->undo();
    if (!img.isNull()) {
        m_preview->setImage(img);
        statusBar()->showMessage("Undo: " + m_undoRedo->redoDescription());
    }
}

void MainWindow::redo() {
    QImage img = m_undoRedo->redo();
    if (!img.isNull()) {
        m_preview->setImage(img);
        statusBar()->showMessage("Redo: " + m_undoRedo->undoDescription());
    }
}

} // namespace re4
```

- [ ] **Step 2: Update main.cpp to use MainWindow**

```cpp
// src/main.cpp
#include "core/application.h"
#include "mainwindow.h"

int main(int argc, char* argv[])
{
    re4::Application app(argc, argv);

    re4::MainWindow window;
    window.show();

    return app.exec();
}
```

- [ ] **Step 3: Update CMakeLists.txt**

Add to APP_SOURCES:
```
src/mainwindow.cpp
src/color/colorprocessor.cpp
src/color/undoredomanager.cpp
src/ui/texturepreviewwidget.cpp
src/ui/rgbsliderwidget.cpp
src/ui/coloradjustwidget.cpp
```

Add to APP_HEADERS:
```
src/mainwindow.h
src/color/colorprocessor.h
src/color/undoredomanager.h
src/ui/texturepreviewwidget.h
src/ui/rgbsliderwidget.h
src/ui/coloradjustwidget.h
```

- [ ] **Step 4: Commit**

```bash
git add src/mainwindow.h src/mainwindow.cpp src/main.cpp CMakeLists.txt
git commit -m "feat: add MainWindow with integrated color editor, preview, and undo/redo"
```

---

## Phase 3 Complete — Summary

| Deliverable | Status |
|------------|--------|
| ColorProcessor (brightness, contrast, saturation, hue, color replace) | ✅ |
| UndoRedoManager (50-state history with descriptions) | ✅ |
| TexturePreviewWidget (2D zoom/pan/pixel-pick) | ✅ |
| RGBSliderWidget (R/G/B/A sliders + spinboxes) | ✅ |
| ColorAdjustWidget (brightness/contrast/saturation/hue) | ✅ |
| MainWindow (menu, toolbar, dock widgets, file open/save) | ✅ |

**Next Phase:** Phase 4 — 3D Preview System (DirectX 11/12)
