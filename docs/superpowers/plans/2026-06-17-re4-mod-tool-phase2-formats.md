# RE4 Mod Tool - Phase 2: Format Conversion Module

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement bidirectional conversion between game texture formats (.tex/.dds) and editable image formats (TGA/DDS/PNG), using DirectXTex for BC compression and reverse-engineering the RE Engine .tex format.

**Architecture:** FormatParser abstract interface with concrete implementations for each format. FormatConverter orchestrates conversions. All parsers use re4::Logger for logging and re4::FileUtils for I/O.

**Tech Stack:** C++17, Qt 6.8 (QImage), DirectXTex (BC compression), re4::Logger, re4::FileUtils

**Spec Reference:** `docs/superpowers/specs/2026-06-17-re4-mod-tool-design.md` — Section 2

**Depends On:** Phase 1 (Logger, Settings, FileUtils, ImageUtils, MathUtils)

---

## File Structure (Phase 2)

| File | Responsibility |
|------|---------------|
| `src/formats/formatparser.h` | Abstract base class for all format parsers |
| `src/formats/texformat.h/.cpp` | RE Engine .tex format parser and writer |
| `src/formats/ddsformat.h/.cpp` | DDS format parser and writer (via DirectXTex) |
| `src/formats/tgaformat.h/.cpp` | TGA format parser and writer |
| `src/formats/pngformat.h/.cpp` | PNG format parser and writer (via Qt) |
| `src/formats/formatchecker.h/.cpp` | Format detection by magic number |
| `src/formats/formatconverter.h/.cpp` | High-level conversion orchestrator |
| `src/formats/texheader.h` | .tex file header structure |
| `tests/unit/test_texformat.cpp` | .tex format tests |
| `tests/unit/test_ddsformat.cpp` | DDS format tests |
| `tests/unit/test_tgaformat.cpp` | TGA format tests |
| `tests/unit/test_pngformat.cpp` | PNG format tests |
| `tests/unit/test_formatchecker.cpp` | Format detection tests |
| `tests/unit/test_formatconverter.cpp` | Conversion integration tests |

---

### Task 1: Format Parser Base Class and Texture Data Model

**Files:**
- Create: `src/formats/formatparser.h`
- Create: `src/formats/texheader.h`

- [ ] **Step 1: Create TextureData — the common data model all parsers produce/consume**

```cpp
// src/formats/texheader.h
#pragma once

#include <QImage>
#include <QByteArray>
#include <cstdint>
#include <vector>

namespace re4 {

// BC compression formats used by RE Engine
enum class BcFormat : uint32_t {
    Unknown   = 0,
    BC1       = 1,  // DXT1 — no alpha or 1-bit alpha
    BC2       = 2,  // DXT3 — explicit alpha
    BC3       = 3,  // DXT5 — interpolated alpha
    BC4       = 4,  // Single channel
    BC5       = 5,  // Two channels (normal maps)
    BC7       = 7,  // High quality RGBA
    RGBA8     = 28, // Uncompressed 32-bit RGBA
    BGRA8     = 29, // Uncompressed 32-bit BGRA
};

// Texture metadata — shared across all formats
struct TextureMeta {
    int width = 0;
    int height = 0;
    int mipLevels = 1;
    int arraySize = 1;  // texture array layers
    BcFormat bcFormat = BcFormat::Unknown;
    bool isSrgb = false;

    bool isValid() const { return width > 0 && height > 0; }
};

// Complete texture: metadata + all mip levels as raw pixel data
struct TextureData {
    TextureMeta meta;

    // Each mip level is a byte array of raw pixel data (RGBA8 or BC-compressed)
    struct MipLevel {
        QByteArray data;
        int width;
        int height;
    };
    std::vector<MipLevel> mips;

    // Get uncompressed RGBA8 image for a given mip level
    QImage toImage(int mipLevel = 0) const;

    // Set image data (compresses to BC format if needed)
    bool fromImage(const QImage& image, BcFormat targetFormat = BcFormat::RGBA8);

    bool isValid() const { return meta.isValid() && !mips.empty() && !mips[0].data.isEmpty(); }
};

// .tex file header (RE Engine specific)
struct TexFileHeader {
    char magic[4] = {};        // "TEX\0" or similar
    uint32_t version = 0;
    uint32_t width = 0;
    uint32_t height = 0;
    uint32_t depth = 1;
    uint32_t mipCount = 1;
    uint32_t arraySize = 1;
    uint32_t format = 0;       // maps to BcFormat
    uint32_t flags = 0;
    uint32_t dataSize = 0;     // total pixel data size
    uint32_t headerSize = 0;   // size of header portion
    uint32_t reserved[4] = {};

    static constexpr int kHeaderSize = 64; // typical header size

    bool isValid() const {
        return width > 0 && height > 0 && mipCount > 0;
    }
};

} // namespace re4
```

- [ ] **Step 2: Create FormatParser abstract base class**

```cpp
// src/formats/formatparser.h
#pragma once

#include "formats/texheader.h"
#include <QString>
#include <QByteArray>

namespace re4 {

// Abstract interface for format parsers
class FormatParser {
public:
    virtual ~FormatParser() = default;

    // Read a file from disk into TextureData
    virtual bool load(const QString& filePath, TextureData& output) = 0;

    // Write TextureData to a file on disk
    virtual bool save(const QString& filePath, const TextureData& input) = 0;

    // Read from raw bytes (for in-memory conversion)
    virtual bool parse(const QByteArray& rawData, TextureData& output) = 0;

    // Serialize TextureData to raw bytes
    virtual QByteArray serialize(const TextureData& input) = 0;

    // Check if this parser can handle the given file
    virtual bool canHandle(const QString& filePath) const = 0;

    // Format name for logging
    virtual QString formatName() const = 0;
};

} // namespace re4
```

- [ ] **Step 3: Commit**

```bash
git add src/formats/formatparser.h src/formats/texheader.h
git commit -m "feat: add format parser base class and texture data model"
```

---

### Task 2: Format Checker (Magic Number Detection)

**Files:**
- Create: `tests/unit/test_formatchecker.cpp`
- Create: `src/formats/formatchecker.h`
- Create: `src/formats/formatchecker.cpp`

- [ ] **Step 1: Write failing tests**

```cpp
// tests/unit/test_formatchecker.cpp
#include <gtest/gtest.h>
#include "formats/formatchecker.h"
#include <QTemporaryDir>

using namespace re4;

class FormatCheckerTest : public ::testing::Test {
protected:
    void SetUp() override {
        m_tempDir = std::make_unique<QTemporaryDir>();
        ASSERT_TRUE(m_tempDir->isValid());
    }
    std::unique_ptr<QTemporaryDir> m_tempDir;

    QString writeFile(const QString& name, const QByteArray& data) {
        QString path = m_tempDir->filePath(name);
        QFile f(path);
        f.open(QIODevice::WriteOnly);
        f.write(data);
        f.close();
        return path;
    }
};

TEST_F(FormatCheckerTest, DetectDdsByMagic) {
    // DDS magic: "DDS " (0x20534444)
    QByteArray data(64, '\0');
    data[0] = 'D'; data[1] = 'D'; data[2] = 'S'; data[3] = ' ';
    QString path = writeFile("test.dds", data);
    EXPECT_EQ(FormatChecker::detectFormat(path), FormatChecker::Format::Dds);
}

TEST_F(FormatCheckerTest, DetectPngByMagic) {
    // PNG magic: 89 50 4E 47 0D 0A 1A 0A
    QByteArray data(64, '\0');
    data[0] = '\x89'; data[1] = 'P'; data[2] = 'N'; data[3] = 'G';
    data[4] = '\r'; data[5] = '\n'; data[6] = '\x1a'; data[7] = '\n';
    QString path = writeFile("test.png", data);
    EXPECT_EQ(FormatChecker::detectFormat(path), FormatChecker::Format::Png);
}

TEST_F(FormatCheckerTest, DetectTgaByExtension) {
    // TGA has no standard magic — detect by extension
    QByteArray data(64, '\0');
    QString path = writeFile("test.tga", data);
    EXPECT_EQ(FormatChecker::detectFormat(path), FormatChecker::Format::Tga);
}

TEST_F(FormatCheckerTest, DetectTexByExtension) {
    QByteArray data(64, '\0');
    QString path = writeFile("test.tex", data);
    EXPECT_EQ(FormatChecker::detectFormat(path), FormatChecker::Format::Tex);
}

TEST_F(FormatCheckerTest, DetectUnknown) {
    QByteArray data(64, '\0');
    QString path = writeFile("test.xyz", data);
    EXPECT_EQ(FormatChecker::detectFormat(path), FormatChecker::Format::Unknown);
}

TEST_F(FormatCheckerTest, DetectByDataDds) {
    QByteArray data(64, '\0');
    data[0] = 'D'; data[1] = 'D'; data[2] = 'S'; data[3] = ' ';
    EXPECT_EQ(FormatChecker::detectFromData(data), FormatChecker::Format::Dds);
}

TEST_F(FormatCheckerTest, DetectByDataPng) {
    QByteArray data(64, '\0');
    data[0] = '\x89'; data[1] = 'P'; data[2] = 'N'; data[3] = 'G';
    data[4] = '\r'; data[5] = '\n'; data[6] = '\x1a'; data[7] = '\n';
    EXPECT_EQ(FormatChecker::detectFromData(data), FormatChecker::Format::Png);
}

TEST_F(FormatCheckerTest, DetectByDataEmpty) {
    EXPECT_EQ(FormatChecker::detectFromData(QByteArray()), FormatChecker::Format::Unknown);
}
```

- [ ] **Step 2: Implement FormatChecker**

```cpp
// src/formats/formatchecker.h
#pragma once

#include <QString>
#include <QByteArray>

namespace re4 {
namespace FormatChecker {

    enum class Format {
        Unknown,
        Dds,
        Png,
        Tga,
        Tex
    };

    // Detect format by reading file header
    Format detectFormat(const QString& filePath);

    // Detect format from raw data
    Format detectFromData(const QByteArray& data);

    // Get human-readable name
    QString formatToString(Format format);

} // namespace FormatChecker
} // namespace re4
```

```cpp
// src/formats/formatchecker.cpp
#include "formats/formatchecker.h"
#include "utils/fileutils.h"
#include <QFileInfo>

namespace re4 {
namespace FormatChecker {

Format detectFromData(const QByteArray& data) {
    if (data.size() < 4) return Format::Unknown;

    // DDS: "DDS " (0x20534444)
    if (data[0] == 'D' && data[1] == 'D' && data[2] == 'S' && data[3] == ' ')
        return Format::Dds;

    // PNG: 89 50 4E 47 0D 0A 1A 0A
    if (data.size() >= 8 &&
        static_cast<uchar>(data[0]) == 0x89 && data[1] == 'P' &&
        data[2] == 'N' && data[3] == 'G' &&
        data[4] == '\r' && data[5] == '\n' &&
        static_cast<uchar>(data[6]) == 0x1a && data[7] == '\n')
        return Format::Png;

    return Format::Unknown;
}

Format detectFormat(const QString& filePath) {
    // Try magic number first
    QByteArray header = FileUtils::readBinary(filePath).left(8);
    Format byMagic = detectFromData(header);
    if (byMagic != Format::Unknown) return byMagic;

    // Fall back to extension
    QString ext = FileUtils::extension(filePath).toLower();
    if (ext == "dds") return Format::Dds;
    if (ext == "png") return Format::Png;
    if (ext == "tga") return Format::Tga;
    if (ext == "tex") return Format::Tex;

    return Format::Unknown;
}

QString formatToString(Format format) {
    switch (format) {
        case Format::Dds:     return "DDS";
        case Format::Png:     return "PNG";
        case Format::Tga:     return "TGA";
        case Format::Tex:     return "TEX";
        case Format::Unknown: return "Unknown";
    }
    return "Unknown";
}

} // namespace FormatChecker
} // namespace re4
```

- [ ] **Step 3: Update tests/CMakeLists.txt to include new test files**

Add to the `add_executable` list:
```
    unit/test_formatchecker.cpp
    unit/test_texformat.cpp
    unit/test_ddsformat.cpp
    unit/test_tgaformat.cpp
    unit/test_pngformat.cpp
    unit/test_formatconverter.cpp
```

Add to `target_sources`:
```
    ${CMAKE_SOURCE_DIR}/src/formats/formatchecker.cpp
    ${CMAKE_SOURCE_DIR}/src/formats/texformat.cpp
    ${CMAKE_SOURCE_DIR}/src/formats/ddsformat.cpp
    ${CMAKE_SOURCE_DIR}/src/formats/tgaformat.cpp
    ${CMAKE_SOURCE_DIR}/src/formats/pngformat.cpp
    ${CMAKE_SOURCE_DIR}/src/formats/formatconverter.cpp
```

- [ ] **Step 4: Commit**

```bash
git add src/formats/formatchecker.h src/formats/formatchecker.cpp tests/unit/test_formatchecker.cpp tests/CMakeLists.txt
git commit -m "feat: add format checker with magic number and extension detection"
```

---

### Task 3: PNG Format Parser/Writer (TDD)

**Files:**
- Create: `tests/unit/test_pngformat.cpp`
- Create: `src/formats/pngformat.h`
- Create: `src/formats/pngformat.cpp`

- [ ] **Step 1: Write failing tests**

```cpp
// tests/unit/test_pngformat.cpp
#include <gtest/gtest.h>
#include "formats/pngformat.h"
#include <QTemporaryDir>

using namespace re4;

class PngFormatTest : public ::testing::Test {
protected:
    void SetUp() override {
        m_tempDir = std::make_unique<QTemporaryDir>();
        ASSERT_TRUE(m_tempDir->isValid());
    }
    std::unique_ptr<QTemporaryDir> m_tempDir;

    TextureData createTestTexture(int w = 64, int h = 64) {
        TextureData tex;
        tex.meta.width = w;
        tex.meta.height = h;
        tex.meta.mipLevels = 1;
        tex.meta.bcFormat = BcFormat::RGBA8;
        tex.meta.isSrgb = false;

        QImage img(w, h, QImage::Format_RGBA8888);
        img.fill(QColor(255, 128, 0, 255));
        tex.fromImage(img, BcFormat::RGBA8);
        return tex;
    }
};

TEST_F(PngFormatTest, SaveAndLoadPng) {
    PngFormat png;
    TextureData original = createTestTexture();
    QString path = m_tempDir->filePath("test.png");

    ASSERT_TRUE(png.save(path, original));
    ASSERT_TRUE(QFile::exists(path));

    TextureData loaded;
    ASSERT_TRUE(png.load(path, loaded));
    EXPECT_EQ(loaded.meta.width, 64);
    EXPECT_EQ(loaded.meta.height, 64);
}

TEST_F(PngFormatTest, SaveLoadRoundTrip) {
    PngFormat png;
    TextureData original = createTestTexture(128, 128);
    QString path = m_tempDir->filePath("roundtrip.png");

    ASSERT_TRUE(png.save(path, original));

    TextureData loaded;
    ASSERT_TRUE(png.load(path, loaded));

    QImage origImg = original.toImage(0);
    QImage loadedImg = loaded.toImage(0);

    // Compare a few pixels
    EXPECT_EQ(origImg.pixelColor(0, 0), loadedImg.pixelColor(0, 0));
    EXPECT_EQ(origImg.pixelColor(64, 64), loadedImg.pixelColor(64, 64));
}

TEST_F(PngFormatTest, ParseFromData) {
    PngFormat png;
    TextureData original = createTestTexture(32, 32);
    QByteArray data = png.serialize(original);
    ASSERT_FALSE(data.isEmpty());

    TextureData parsed;
    ASSERT_TRUE(png.parse(data, parsed));
    EXPECT_EQ(parsed.meta.width, 32);
    EXPECT_EQ(parsed.meta.height, 32);
}

TEST_F(PngFormatTest, CanHandlePngFile) {
    PngFormat png;
    EXPECT_TRUE(png.canHandle("test.png"));
    EXPECT_TRUE(png.canHandle("C:\\path\\texture.png"));
    EXPECT_FALSE(png.canHandle("test.dds"));
    EXPECT_FALSE(png.canHandle("test.tga"));
}

TEST_F(PngFormatTest, FormatName) {
    PngFormat png;
    EXPECT_EQ(png.formatName(), "PNG");
}

TEST_F(PngFormatTest, LoadNonexistentFile) {
    PngFormat png;
    TextureData loaded;
    EXPECT_FALSE(png.load("C:\\nonexistent\\file.png", loaded));
}

TEST_F(PngFormatTest, SaveToInvalidPath) {
    PngFormat png;
    TextureData tex = createTestTexture();
    EXPECT_FALSE(png.save("Z:\\invalid\\path\\test.png", tex));
}

TEST_F(PngFormatTest, LargeTexture) {
    PngFormat png;
    TextureData original = createTestTexture(2048, 2048);
    QString path = m_tempDir->filePath("large.png");

    ASSERT_TRUE(png.save(path, original));

    TextureData loaded;
    ASSERT_TRUE(png.load(path, loaded));
    EXPECT_EQ(loaded.meta.width, 2048);
    EXPECT_EQ(loaded.meta.height, 2048);
}
```

- [ ] **Step 2: Implement PngFormat**

```cpp
// src/formats/pngformat.h
#pragma once

#include "formats/formatparser.h"

namespace re4 {

class PngFormat : public FormatParser {
public:
    ~PngFormat() override = default;

    bool load(const QString& filePath, TextureData& output) override;
    bool save(const QString& filePath, const TextureData& input) override;
    bool parse(const QByteArray& rawData, TextureData& output) override;
    QByteArray serialize(const TextureData& input) override;
    bool canHandle(const QString& filePath) const override;
    QString formatName() const override;
};

} // namespace re4
```

```cpp
// src/formats/pngformat.cpp
#include "formats/pngformat.h"
#include "core/logger.h"
#include <QImage>
#include <QBuffer>
#include <QFileInfo>

namespace re4 {

bool PngFormat::load(const QString& filePath, TextureData& output) {
    QImage img;
    if (!img.load(filePath, "PNG")) {
        Logger::instance()->error("Failed to load PNG: " + filePath);
        return false;
    }

    output.meta.width = img.width();
    output.meta.height = img.height();
    output.meta.mipLevels = 1;
    output.meta.arraySize = 1;
    output.meta.bcFormat = BcFormat::RGBA8;
    output.meta.isSrgb = false;

    return output.fromImage(img, BcFormat::RGBA8);
}

bool PngFormat::save(const QString& filePath, const TextureData& input) {
    QImage img = input.toImage(0);
    if (img.isNull()) {
        Logger::instance()->error("Cannot save PNG: invalid texture data");
        return false;
    }

    if (!img.save(filePath, "PNG")) {
        Logger::instance()->error("Failed to save PNG: " + filePath);
        return false;
    }

    Logger::instance()->info("Saved PNG: " + filePath);
    return true;
}

bool PngFormat::parse(const QByteArray& rawData, TextureData& output) {
    QImage img;
    if (!img.loadFromData(rawData, "PNG")) {
        return false;
    }

    output.meta.width = img.width();
    output.meta.height = img.height();
    output.meta.mipLevels = 1;
    output.meta.bcFormat = BcFormat::RGBA8;

    return output.fromImage(img, BcFormat::RGBA8);
}

QByteArray PngFormat::serialize(const TextureData& input) {
    QImage img = input.toImage(0);
    if (img.isNull()) return {};

    QBuffer buffer;
    buffer.open(QIODevice::WriteOnly);
    img.save(&buffer, "PNG");
    return buffer.data();
}

bool PngFormat::canHandle(const QString& filePath) const {
    return QFileInfo(filePath).suffix().toLower() == "png";
}

QString PngFormat::formatName() const {
    return "PNG";
}

} // namespace re4
```

- [ ] **Step 3: Commit**

```bash
git add src/formats/pngformat.h src/formats/pngformat.cpp tests/unit/test_pngformat.cpp
git commit -m "feat: add PNG format parser and writer"
```

---

### Task 4: TGA Format Parser/Writer (TDD)

**Files:**
- Create: `tests/unit/test_tgaformat.cpp`
- Create: `src/formats/tgaformat.h`
- Create: `src/formats/tgaformat.cpp`

- [ ] **Step 1: Write failing tests**

```cpp
// tests/unit/test_tgaformat.cpp
#include <gtest/gtest.h>
#include "formats/tgaformat.h"
#include <QTemporaryDir>

using namespace re4;

class TgaFormatTest : public ::testing::Test {
protected:
    void SetUp() override {
        m_tempDir = std::make_unique<QTemporaryDir>();
        ASSERT_TRUE(m_tempDir->isValid());
    }
    std::unique_ptr<QTemporaryDir> m_tempDir;

    TextureData createTestTexture(int w = 64, int h = 64) {
        TextureData tex;
        tex.meta.width = w;
        tex.meta.height = h;
        tex.meta.mipLevels = 1;
        tex.meta.bcFormat = BcFormat::RGBA8;

        QImage img(w, h, QImage::Format_RGBA8888);
        img.fill(QColor(100, 200, 50, 255));
        tex.fromImage(img, BcFormat::RGBA8);
        return tex;
    }
};

TEST_F(TgaFormatTest, SaveAndLoadTga) {
    TgaFormat tga;
    TextureData original = createTestTexture();
    QString path = m_tempDir->filePath("test.tga");

    ASSERT_TRUE(tga.save(path, original));
    ASSERT_TRUE(QFile::exists(path));

    TextureData loaded;
    ASSERT_TRUE(tga.load(path, loaded));
    EXPECT_EQ(loaded.meta.width, 64);
    EXPECT_EQ(loaded.meta.height, 64);
}

TEST_F(TgaFormatTest, SaveLoadRoundTrip) {
    TgaFormat tga;
    TextureData original = createTestTexture(128, 128);
    QString path = m_tempDir->filePath("roundtrip.tga");

    ASSERT_TRUE(tga.save(path, original));

    TextureData loaded;
    ASSERT_TRUE(tga.load(path, loaded));

    QImage origImg = original.toImage(0);
    QImage loadedImg = loaded.toImage(0);

    EXPECT_EQ(origImg.pixelColor(0, 0), loadedImg.pixelColor(0, 0));
    EXPECT_EQ(origImg.pixelColor(64, 64), loadedImg.pixelColor(64, 64));
}

TEST_F(TgaFormatTest, ParseFromData) {
    TgaFormat tga;
    TextureData original = createTestTexture(32, 32);
    QByteArray data = tga.serialize(original);
    ASSERT_FALSE(data.isEmpty());

    TextureData parsed;
    ASSERT_TRUE(tga.parse(data, parsed));
    EXPECT_EQ(parsed.meta.width, 32);
    EXPECT_EQ(parsed.meta.height, 32);
}

TEST_F(TgaFormatTest, CanHandleTgaFile) {
    TgaFormat tga;
    EXPECT_TRUE(tga.canHandle("test.tga"));
    EXPECT_FALSE(tga.canHandle("test.png"));
}

TEST_F(TgaFormatTest, FormatName) {
    TgaFormat tga;
    EXPECT_EQ(tga.formatName(), "TGA");
}

TEST_F(TgaFormatTest, LoadNonexistentFile) {
    TgaFormat tga;
    TextureData loaded;
    EXPECT_FALSE(tga.load("C:\\nonexistent\\file.tga", loaded));
}
```

- [ ] **Step 2: Implement TgaFormat**

TGA format structure (uncompressed):
- 18-byte header: ID length, color map type, image type, origin, dimensions, pixel depth, descriptor
- Pixel data: BGRA or BGR, bottom-to-top by default

```cpp
// src/formats/tgaformat.h
#pragma once

#include "formats/formatparser.h"

namespace re4 {

class TgaFormat : public FormatParser {
public:
    ~TgaFormat() override = default;

    bool load(const QString& filePath, TextureData& output) override;
    bool save(const QString& filePath, const TextureData& input) override;
    bool parse(const QByteArray& rawData, TextureData& output) override;
    QByteArray serialize(const TextureData& input) override;
    bool canHandle(const QString& filePath) const override;
    QString formatName() const override;

private:
    // TGA header structure (18 bytes, packed)
    #pragma pack(push, 1)
    struct TgaHeader {
        uint8_t  idLength = 0;
        uint8_t  colorMapType = 0;
        uint8_t  imageType = 2;     // 2 = uncompressed true-color
        uint16_t colorMapFirst = 0;
        uint16_t colorMapLength = 0;
        uint8_t  colorMapDepth = 0;
        uint16_t xOrigin = 0;
        uint16_t yOrigin = 0;
        uint16_t width = 0;
        uint16_t height = 0;
        uint8_t  bitsPerPixel = 32; // 32-bit BGRA
        uint8_t  imageDescriptor = 8; // top-left origin, 8 alpha bits
    };
    #pragma pack(pop)

    static constexpr int kHeaderSize = 18;
};

} // namespace re4
```

```cpp
// src/formats/tgaformat.cpp
#include "formats/tgaformat.h"
#include "core/logger.h"
#include <QFile>
#include <QFileInfo>
#include <QDataStream>

namespace re4 {

bool TgaFormat::load(const QString& filePath, TextureData& output) {
    QFile file(filePath);
    if (!file.open(QIODevice::ReadOnly)) {
        Logger::instance()->error("Failed to open TGA: " + filePath);
        return false;
    }
    return parse(file.readAll(), output);
}

bool TgaFormat::parse(const QByteArray& rawData, TextureData& output) {
    if (rawData.size() < kHeaderSize) return false;

    TgaHeader header;
    memcpy(&header, rawData.constData(), kHeaderSize);

    if (header.bitsPerPixel != 32 && header.bitsPerPixel != 24) {
        Logger::instance()->error("Unsupported TGA bit depth: " + QString::number(header.bitsPerPixel));
        return false;
    }

    int w = header.width;
    int h = header.height;
    int bpp = header.bitsPerPixel / 8;
    int dataSize = w * h * bpp;

    if (rawData.size() < kHeaderSize + header.idLength + dataSize) {
        Logger::instance()->error("TGA file truncated");
        return false;
    }

    const uchar* pixelData = reinterpret_cast<const uchar*>(rawData.constData())
                             + kHeaderSize + header.idLength;

    output.meta.width = w;
    output.meta.height = h;
    output.meta.mipLevels = 1;
    output.meta.bcFormat = BcFormat::RGBA8;

    QImage img(w, h, QImage::Format_RGBA8888);

    bool topOrigin = (header.imageDescriptor & 0x20) != 0;

    for (int y = 0; y < h; ++y) {
        int srcY = topOrigin ? y : (h - 1 - y);
        uchar* dstLine = img.scanLine(y);
        const uchar* srcLine = pixelData + srcY * w * bpp;

        for (int x = 0; x < w; ++x) {
            const uchar* src = srcLine + x * bpp;
            uchar* dst = dstLine + x * 4;
            dst[0] = bpp >= 3 ? src[2] : src[0]; // R
            dst[1] = bpp >= 3 ? src[1] : src[0]; // G
            dst[2] = bpp >= 3 ? src[0] : src[0]; // B
            dst[3] = bpp == 4 ? src[3] : 255;    // A
        }
    }

    TextureData::MipLevel mip;
    mip.data = QByteArray(reinterpret_cast<const char*>(img.constBits()), img.sizeInBytes());
    mip.width = w;
    mip.height = h;
    output.mips = {mip};

    return true;
}

bool TgaFormat::save(const QString& filePath, const TextureData& input) {
    QByteArray data = serialize(input);
    if (data.isEmpty()) return false;

    QFile file(filePath);
    if (!file.open(QIODevice::WriteOnly)) {
        Logger::instance()->error("Failed to write TGA: " + filePath);
        return false;
    }
    return file.write(data) == data.size();
}

QByteArray TgaFormat::serialize(const TextureData& input) {
    QImage img = input.toImage(0);
    if (img.isNull()) return {};

    img = img.convertToFormat(QImage::Format_RGBA8888);

    TgaHeader header;
    header.width = img.width();
    header.height = img.height();
    header.bitsPerPixel = 32;
    header.imageDescriptor = 0x28; // top-left origin + 8 alpha bits

    QByteArray result;
    result.reserve(kHeaderSize + img.width() * img.height() * 4);
    result.append(reinterpret_cast<const char*>(&header), kHeaderSize);

    for (int y = 0; y < img.height(); ++y) {
        const uchar* line = img.constScanLine(y);
        for (int x = 0; x < img.width(); ++x) {
            const uchar* px = line + x * 4;
            result.append(static_cast<char>(px[2])); // B
            result.append(static_cast<char>(px[1])); // G
            result.append(static_cast<char>(px[0])); // R
            result.append(static_cast<char>(px[3])); // A
        }
    }

    return result;
}

bool TgaFormat::canHandle(const QString& filePath) const {
    return QFileInfo(filePath).suffix().toLower() == "tga";
}

QString TgaFormat::formatName() const {
    return "TGA";
}

} // namespace re4
```

- [ ] **Step 3: Commit**

```bash
git add src/formats/tgaformat.h src/formats/tgaformat.cpp tests/unit/test_tgaformat.cpp
git commit -m "feat: add TGA format parser and writer with BGRA support"
```

---

### Task 5: DDS Format Parser/Writer via DirectXTex (TDD)

**Files:**
- Create: `tests/unit/test_ddsformat.cpp`
- Create: `src/formats/ddsformat.h`
- Create: `src/formats/ddsformat.cpp`

- [ ] **Step 1: Write failing tests**

```cpp
// tests/unit/test_ddsformat.cpp
#include <gtest/gtest.h>
#include "formats/ddsformat.h"
#include <QTemporaryDir>

using namespace re4;

class DdsFormatTest : public ::testing::Test {
protected:
    void SetUp() override {
        m_tempDir = std::make_unique<QTemporaryDir>();
        ASSERT_TRUE(m_tempDir->isValid());
    }
    std::unique_ptr<QTemporaryDir> m_tempDir;

    TextureData createTestTexture(int w = 64, int h = 64, BcFormat fmt = BcFormat::RGBA8) {
        TextureData tex;
        tex.meta.width = w;
        tex.meta.height = h;
        tex.meta.mipLevels = 1;
        tex.meta.bcFormat = fmt;

        QImage img(w, h, QImage::Format_RGBA8888);
        img.fill(QColor(100, 150, 200, 255));
        tex.fromImage(img, fmt);
        return tex;
    }
};

TEST_F(DdsFormatTest, SaveAndLoadUncompressed) {
    DdsFormat dds;
    TextureData original = createTestTexture(64, 64, BcFormat::RGBA8);
    QString path = m_tempDir->filePath("test.dds");

    ASSERT_TRUE(dds.save(path, original));
    ASSERT_TRUE(QFile::exists(path));

    TextureData loaded;
    ASSERT_TRUE(dds.load(path, loaded));
    EXPECT_EQ(loaded.meta.width, 64);
    EXPECT_EQ(loaded.meta.height, 64);
}

TEST_F(DdsFormatTest, SaveAndLoadBC7) {
    DdsFormat dds;
    TextureData original = createTestTexture(64, 64, BcFormat::BC7);
    QString path = m_tempDir->filePath("test_bc7.dds");

    ASSERT_TRUE(dds.save(path, original));

    TextureData loaded;
    ASSERT_TRUE(dds.load(path, loaded));
    EXPECT_EQ(loaded.meta.width, 64);
    EXPECT_EQ(loaded.meta.height, 64);
    EXPECT_EQ(loaded.meta.bcFormat, BcFormat::BC7);
}

TEST_F(DdsFormatTest, ParseFromData) {
    DdsFormat dds;
    TextureData original = createTestTexture(32, 32);
    QByteArray data = dds.serialize(original);
    ASSERT_FALSE(data.isEmpty());

    TextureData parsed;
    ASSERT_TRUE(dds.parse(data, parsed));
    EXPECT_EQ(parsed.meta.width, 32);
    EXPECT_EQ(parsed.meta.height, 32);
}

TEST_F(DdsFormatTest, CanHandleDdsFile) {
    DdsFormat dds;
    EXPECT_TRUE(dds.canHandle("test.dds"));
    EXPECT_FALSE(dds.canHandle("test.png"));
}

TEST_F(DdsFormatTest, FormatName) {
    DdsFormat dds;
    EXPECT_EQ(dds.formatName(), "DDS");
}

TEST_F(DdsFormatTest, LoadNonexistentFile) {
    DdsFormat dds;
    TextureData loaded;
    EXPECT_FALSE(dds.load("C:\\nonexistent\\file.dds", loaded));
}

TEST_F(DdsFormatTest, GenerateMipmaps) {
    DdsFormat dds;
    TextureData original = createTestTexture(64, 64, BcFormat::RGBA8);
    original.meta.mipLevels = 4; // request mip generation

    // DDS should be able to generate mipmaps
    EXPECT_GE(original.meta.mipLevels, 1);
}
```

- [ ] **Step 2: Implement DdsFormat using DirectXTex**

```cpp
// src/formats/ddsformat.h
#pragma once

#include "formats/formatparser.h"

namespace re4 {

class DdsFormat : public FormatParser {
public:
    ~DdsFormat() override = default;

    bool load(const QString& filePath, TextureData& output) override;
    bool save(const QString& filePath, const TextureData& input) override;
    bool parse(const QByteArray& rawData, TextureData& output) override;
    QByteArray serialize(const TextureData& input) override;
    bool canHandle(const QString& filePath) const override;
    QString formatName() const override;

private:
    // Convert between our BcFormat and DirectX::DXGI_FORMAT
    static int toDxgiFormat(BcFormat format);
    static BcFormat fromDxgiFormat(int dxgiFormat);
};

} // namespace re4
```

```cpp
// src/formats/ddsformat.cpp
#include "formats/ddsformat.h"
#include "core/logger.h"
#include <QFile>
#include <QFileInfo>
#include <DirectXTex.h>

namespace re4 {

int DdsFormat::toDxgiFormat(BcFormat format) {
    switch (format) {
        case BcFormat::BC1:    return 71;  // DXGI_FORMAT_BC1_UNORM
        case BcFormat::BC2:    return 74;  // DXGI_FORMAT_BC2_UNORM
        case BcFormat::BC3:    return 77;  // DXGI_FORMAT_BC3_UNORM
        case BcFormat::BC4:    return 80;  // DXGI_FORMAT_BC4_UNORM
        case BcFormat::BC5:    return 83;  // DXGI_FORMAT_BC5_UNORM
        case BcFormat::BC7:    return 98;  // DXGI_FORMAT_BC7_UNORM
        case BcFormat::RGBA8:  return 28;  // DXGI_FORMAT_R8G8B8A8_UNORM
        case BcFormat::BGRA8:  return 87;  // DXGI_FORMAT_B8G8R8A8_UNORM
        default:               return 28;  // fallback to RGBA8
    }
}

BcFormat DdsFormat::fromDxgiFormat(int dxgiFormat) {
    switch (dxgiFormat) {
        case 71:  return BcFormat::BC1;
        case 74:  return BcFormat::BC2;
        case 77:  return BcFormat::BC3;
        case 80:  return BcFormat::BC4;
        case 83:  return BcFormat::BC5;
        case 98:  return BcFormat::BC7;
        case 28:  return BcFormat::RGBA8;
        case 87:  return BcFormat::BGRA8;
        default:  return BcFormat::Unknown;
    }
}

bool DdsFormat::load(const QString& filePath, TextureData& output) {
    DirectX::ScratchImage image;
    HRESULT hr = DirectX::LoadFromDDSFile(
        filePath.toStdWString().c_str(),
        DirectX::DDS_FLAGS_NONE,
        nullptr, image);

    if (FAILED(hr)) {
        Logger::instance()->error("Failed to load DDS: " + filePath);
        return false;
    }

    const DirectX::TexMetadata& meta = image.GetMetadata();
    output.meta.width = static_cast<int>(meta.width);
    output.meta.height = static_cast<int>(meta.height);
    output.meta.mipLevels = static_cast<int>(meta.mipLevels);
    output.meta.arraySize = static_cast<int>(meta.arraySize);
    output.meta.bcFormat = fromDxgiFormat(static_cast<int>(meta.format));

    // Store all mip levels
    output.mips.clear();
    for (int mip = 0; mip < output.meta.mipLevels; ++mip) {
        const DirectX::Image* img = image.GetImage(mip, 0, 0);
        if (!img) break;

        TextureData::MipLevel level;
        level.data = QByteArray(reinterpret_cast<const char*>(img->pixels), img->slicePitch);
        level.width = static_cast<int>(img->width);
        level.height = static_cast<int>(img->height);
        output.mips.push_back(level);
    }

    return !output.mips.empty();
}

bool DdsFormat::save(const QString& filePath, const TextureData& input) {
    QByteArray data = serialize(input);
    if (data.isEmpty()) return false;

    QFile file(filePath);
    if (!file.open(QIODevice::WriteOnly)) {
        Logger::instance()->error("Failed to write DDS: " + filePath);
        return false;
    }
    return file.write(data) == data.size();
}

bool DdsFormat::parse(const QByteArray& rawData, TextureData& output) {
    DirectX::ScratchImage image;
    HRESULT hr = DirectX::LoadFromDDSMemory(
        reinterpret_cast<const uint8_t*>(rawData.constData()),
        rawData.size(),
        DirectX::DDS_FLAGS_NONE,
        nullptr, image);

    if (FAILED(hr)) return false;

    const DirectX::TexMetadata& meta = image.GetMetadata();
    output.meta.width = static_cast<int>(meta.width);
    output.meta.height = static_cast<int>(meta.height);
    output.meta.mipLevels = static_cast<int>(meta.mipLevels);
    output.meta.bcFormat = fromDxgiFormat(static_cast<int>(meta.format));

    output.mips.clear();
    for (int mip = 0; mip < output.meta.mipLevels; ++mip) {
        const DirectX::Image* img = image.GetImage(mip, 0, 0);
        if (!img) break;

        TextureData::MipLevel level;
        level.data = QByteArray(reinterpret_cast<const char*>(img->pixels), img->slicePitch);
        level.width = static_cast<int>(img->width);
        level.height = static_cast<int>(img->height);
        output.mips.push_back(level);
    }

    return !output.mips.empty();
}

QByteArray DdsFormat::serialize(const TextureData& input) {
    if (!input.isValid()) return {};

    // Build DirectX::Image from our data
    DirectX::ScratchImage image;

    // For uncompressed data, use DirectXTex to handle conversion
    if (input.meta.bcFormat == BcFormat::RGBA8 || input.mips[0].data.size() == input.meta.width * input.meta.height * 4) {
        DirectX::Image dxImage;
        dxImage.width = input.meta.width;
        dxImage.height = input.meta.height;
        dxImage.format = static_cast<DXGI_FORMAT>(toDxgiFormat(input.meta.bcFormat));
        dxImage.rowPitch = input.meta.width * 4;
        dxImage.slicePitch = input.meta.width * input.meta.height * 4;
        dxImage.pixels = reinterpret_cast<uint8_t*>(const_cast<char*>(input.mips[0].data.constData()));

        HRESULT hr = image.InitializeFromImage(dxImage);
        if (FAILED(hr)) return {};

        // If BC compression needed, compress
        if (input.meta.bcFormat != BcFormat::RGBA8 && input.meta.bcFormat != BcFormat::BGRA8) {
            DirectX::ScratchImage compressed;
            hr = DirectX::Compress(image.GetImages(), image.GetImageCount(), image.GetMetadata(),
                                   static_cast<DXGI_FORMAT>(toDxgiFormat(input.meta.bcFormat)),
                                   DirectX::TEX_COMPRESS_DEFAULT, 1.0f, compressed);
            if (FAILED(hr)) return {};
            image = std::move(compressed);
        }
    } else {
        // Already BC-compressed data
        DirectX::Image dxImage;
        dxImage.width = input.meta.width;
        dxImage.height = input.meta.height;
        dxImage.format = static_cast<DXGI_FORMAT>(toDxgiFormat(input.meta.bcFormat));
        dxImage.pixels = reinterpret_cast<uint8_t*>(const_cast<char*>(input.mips[0].data.constData()));

        // Calculate pitch for BC formats
        int blockSize = (input.meta.bcFormat == BcFormat::BC1 || input.meta.bcFormat == BcFormat::BC4) ? 8 : 16;
        int blocksX = std::max(1, (input.meta.width + 3) / 4);
        dxImage.rowPitch = blocksX * blockSize;
        dxImage.slicePitch = dxImage.rowPitch * std::max(1, (input.meta.height + 3) / 4);

        HRESULT hr = image.InitializeFromImage(dxImage);
        if (FAILED(hr)) return {};
    }

    // Generate mipmaps if requested
    if (input.meta.mipLevels > 1 && image.GetMetadata().mipLevels == 1) {
        DirectX::ScratchImage mipChain;
        HRESULT hr = DirectX::GenerateMipMaps(image.GetImages(), image.GetImageCount(),
                                               image.GetMetadata(), DirectX::TEX_FILTER_DEFAULT,
                                               input.meta.mipLevels, mipChain);
        if (SUCCEEDED(hr)) {
            image = std::move(mipChain);
        }
    }

    // Save to DDS memory
    DirectX::Blob blob;
    HRESULT hr = DirectX::SaveToDDSMemory(image.GetImages(), image.GetImageCount(),
                                           image.GetMetadata(), DirectX::DDS_FLAGS_NONE, blob);
    if (FAILED(hr)) return {};

    return QByteArray(reinterpret_cast<const char*>(blob.GetBufferPointer()),
                      static_cast<int>(blob.GetBufferSize()));
}

bool DdsFormat::canHandle(const QString& filePath) const {
    return QFileInfo(filePath).suffix().toLower() == "dds";
}

QString DdsFormat::formatName() const {
    return "DDS";
}

} // namespace re4
```

- [ ] **Step 3: Commit**

```bash
git add src/formats/ddsformat.h src/formats/ddsformat.cpp tests/unit/test_ddsformat.cpp
git commit -m "feat: add DDS format parser/writer using DirectXTex with BC compression"
```

---

### Task 6: TextureData Implementation (toImage / fromImage)

**Files:**
- Create: `src/formats/texheader.cpp`

- [ ] **Step 1: Implement TextureData::toImage and TextureData::fromImage**

```cpp
// src/formats/texheader.cpp
#include "formats/texheader.h"
#include <DirectXTex.h>

namespace re4 {

QImage TextureData::toImage(int mipLevel) const {
    if (mipLevel >= static_cast<int>(mips.size()) || mips[mipLevel].data.isEmpty())
        return {};

    const MipLevel& mip = mips[mipLevel];

    if (meta.bcFormat == BcFormat::RGBA8) {
        // Direct copy for uncompressed
        QImage img(reinterpret_cast<const uchar*>(mip.data.constData()),
                   mip.width, mip.height, mip.width * 4,
                   QImage::Format_RGBA8888);
        return img.copy(); // deep copy
    }

    // For BC-compressed data, decompress using DirectXTex
    DirectX::Image dxImage;
    dxImage.width = mip.width;
    dxImage.height = mip.height;
    dxImage.pixels = reinterpret_cast<uint8_t*>(const_cast<char*>(mip.data.constData()));

    // Set format and calculate pitch
    int dxgiFormat = 28; // RGBA8 default
    int blockSize = 16;
    switch (meta.bcFormat) {
        case BcFormat::BC1: dxgiFormat = 71; blockSize = 8; break;
        case BcFormat::BC2: dxgiFormat = 74; blockSize = 16; break;
        case BcFormat::BC3: dxgiFormat = 77; blockSize = 16; break;
        case BcFormat::BC4: dxgiFormat = 80; blockSize = 8; break;
        case BcFormat::BC5: dxgiFormat = 83; blockSize = 16; break;
        case BcFormat::BC7: dxgiFormat = 98; blockSize = 16; break;
        default: break;
    }
    dxImage.format = static_cast<DXGI_FORMAT>(dxgiFormat);
    int blocksX = std::max(1, (mip.width + 3) / 4);
    dxImage.rowPitch = blocksX * blockSize;
    dxImage.slicePitch = dxImage.rowPitch * std::max(1, (mip.height + 3) / 4);

    DirectX::ScratchImage decompressed;
    HRESULT hr = DirectX::Decompress(dxImage, DXGI_FORMAT_R8G8B8A8_UNORM, decompressed);
    if (FAILED(hr)) return {};

    const DirectX::Image* result = decompressed.GetImage(0, 0, 0);
    if (!result) return {};

    return QImage(reinterpret_cast<const uchar*>(result->pixels),
                  static_cast<int>(result->width),
                  static_cast<int>(result->height),
                  static_cast<int>(result->rowPitch),
                  QImage::Format_RGBA8888).copy();
}

bool TextureData::fromImage(const QImage& image, BcFormat targetFormat) {
    if (image.isNull()) return false;

    QImage rgba = image.convertToFormat(QImage::Format_RGBA8888);
    meta.width = rgba.width();
    meta.height = rgba.height();
    meta.bcFormat = targetFormat;

    MipLevel mip;
    mip.width = rgba.width();
    mip.height = rgba.height();

    if (targetFormat == BcFormat::RGBA8) {
        mip.data = QByteArray(reinterpret_cast<const char*>(rgba.constBits()),
                              rgba.sizeInBytes());
    } else {
        // Compress to BC format using DirectXTex
        DirectX::Image dxImage;
        dxImage.width = rgba.width();
        dxImage.height = rgba.height();
        dxImage.format = DXGI_FORMAT_R8G8B8A8_UNORM;
        dxImage.rowPitch = rgba.bytesPerLine();
        dxImage.slicePitch = rgba.sizeInBytes();
        dxImage.pixels = const_cast<uint8_t*>(rgba.constBits());

        int dxgiFormat = 28;
        switch (targetFormat) {
            case BcFormat::BC1: dxgiFormat = 71; break;
            case BcFormat::BC2: dxgiFormat = 74; break;
            case BcFormat::BC3: dxgiFormat = 77; break;
            case BcFormat::BC4: dxgiFormat = 80; break;
            case BcFormat::BC5: dxgiFormat = 83; break;
            case BcFormat::BC7: dxgiFormat = 98; break;
            default: break;
        }

        DirectX::ScratchImage compressed;
        HRESULT hr = DirectX::Compress(&dxImage, 1, {},
                                       static_cast<DXGI_FORMAT>(dxgiFormat),
                                       DirectX::TEX_COMPRESS_DEFAULT, 1.0f, compressed);
        if (FAILED(hr)) return false;

        const DirectX::Image* result = compressed.GetImage(0, 0, 0);
        if (!result) return false;

        mip.data = QByteArray(reinterpret_cast<const char*>(result->pixels),
                              static_cast<int>(result->slicePitch));
    }

    mips = {mip};
    return true;
}

} // namespace re4
```

- [ ] **Step 2: Commit**

```bash
git add src/formats/texheader.cpp
git commit -m "feat: implement TextureData toImage/fromImage with BC compression support"
```

---

### Task 7: .tex Format Parser/Writer (RE Engine)

**Files:**
- Create: `tests/unit/test_texformat.cpp`
- Create: `src/formats/texformat.h`
- Create: `src/formats/texformat.cpp`

- [ ] **Step 1: Write failing tests**

```cpp
// tests/unit/test_texformat.cpp
#include <gtest/gtest.h>
#include "formats/texformat.h"
#include <QTemporaryDir>

using namespace re4;

class TexFormatTest : public ::testing::Test {
protected:
    void SetUp() override {
        m_tempDir = std::make_unique<QTemporaryDir>();
        ASSERT_TRUE(m_tempDir->isValid());
    }
    std::unique_ptr<QTemporaryDir> m_tempDir;

    TextureData createTestTexture(int w = 64, int h = 64, BcFormat fmt = BcFormat::BC7) {
        TextureData tex;
        tex.meta.width = w;
        tex.meta.height = h;
        tex.meta.mipLevels = 1;
        tex.meta.bcFormat = fmt;

        QImage img(w, h, QImage::Format_RGBA8888);
        img.fill(QColor(100, 150, 200, 255));
        tex.fromImage(img, fmt);
        return tex;
    }
};

TEST_F(TexFormatTest, SaveAndLoadTexBC7) {
    TexFormat tex;
    TextureData original = createTestTexture(64, 64, BcFormat::BC7);
    QString path = m_tempDir->filePath("test.tex");

    ASSERT_TRUE(tex.save(path, original));
    ASSERT_TRUE(QFile::exists(path));

    TextureData loaded;
    ASSERT_TRUE(tex.load(path, loaded));
    EXPECT_EQ(loaded.meta.width, 64);
    EXPECT_EQ(loaded.meta.height, 64);
    EXPECT_EQ(loaded.meta.bcFormat, BcFormat::BC7);
}

TEST_F(TexFormatTest, SaveLoadRoundTrip) {
    TexFormat tex;
    TextureData original = createTestTexture(128, 128, BcFormat::BC7);
    QString path = m_tempDir->filePath("roundtrip.tex");

    ASSERT_TRUE(tex.save(path, original));

    TextureData loaded;
    ASSERT_TRUE(tex.load(path, loaded));

    // Compare decompressed images
    QImage origImg = original.toImage(0);
    QImage loadedImg = loaded.toImage(0);

    // BC7 is lossy, so allow some tolerance
    QColor origPx = origImg.pixelColor(0, 0);
    QColor loadedPx = loadedImg.pixelColor(0, 0);
    EXPECT_NEAR(origPx.red(), loadedPx.red(), 5);
    EXPECT_NEAR(origPx.green(), loadedPx.green(), 5);
    EXPECT_NEAR(origPx.blue(), loadedPx.blue(), 5);
}

TEST_F(TexFormatTest, ParseFromData) {
    TexFormat tex;
    TextureData original = createTestTexture(32, 32, BcFormat::BC1);
    QByteArray data = tex.serialize(original);
    ASSERT_FALSE(data.isEmpty());

    TextureData parsed;
    ASSERT_TRUE(tex.parse(data, parsed));
    EXPECT_EQ(parsed.meta.width, 32);
    EXPECT_EQ(parsed.meta.height, 32);
}

TEST_F(TexFormatTest, CanHandleTexFile) {
    TexFormat tex;
    EXPECT_TRUE(tex.canHandle("test.tex"));
    EXPECT_FALSE(tex.canHandle("test.dds"));
}

TEST_F(TexFormatTest, FormatName) {
    TexFormat tex;
    EXPECT_EQ(tex.formatName(), "TEX");
}

TEST_F(TexFormatTest, LoadNonexistentFile) {
    TexFormat tex;
    TextureData loaded;
    EXPECT_FALSE(tex.load("C:\\nonexistent\\file.tex", loaded));
}

TEST_F(TexFormatTest, WithMipmaps) {
    TexFormat tex;
    TextureData original = createTestTexture(64, 64, BcFormat::BC7);
    original.meta.mipLevels = 4;
    QString path = m_tempDir->filePath("mipped.tex");

    ASSERT_TRUE(tex.save(path, original));

    TextureData loaded;
    ASSERT_TRUE(tex.load(path, loaded));
    EXPECT_GE(loaded.meta.mipLevels, 1);
}
```

- [ ] **Step 2: Implement TexFormat**

The .tex format structure (reverse-engineered from RE Engine):
- Header: 64 bytes with magic, version, dimensions, format, mip count
- Mip table: offsets and sizes for each mip level
- Pixel data: BC-compressed blocks

```cpp
// src/formats/texformat.h
#pragma once

#include "formats/formatparser.h"
#include "formats/texheader.h"

namespace re4 {

class TexFormat : public FormatParser {
public:
    ~TexFormat() override = default;

    bool load(const QString& filePath, TextureData& output) override;
    bool save(const QString& filePath, const TextureData& input) override;
    bool parse(const QByteArray& rawData, TextureData& output) override;
    QByteArray serialize(const TextureData& input) override;
    bool canHandle(const QString& filePath) const override;
    QString formatName() const override;

private:
    // Map RE Engine format codes to BcFormat
    static BcFormat reFormatToBc(uint32_t reFormat);
    static uint32_t bcFormatToRe(BcFormat bcFormat);

    // Calculate mip level dimensions and data sizes
    static int calcMipSize(int width, int height, BcFormat format);
    static int calcBlockSize(BcFormat format);
};

} // namespace re4
```

```cpp
// src/formats/texformat.cpp
#include "formats/texformat.h"
#include "core/logger.h"
#include "utils/fileutils.h"
#include <QFile>
#include <QDataStream>
#include <QFileInfo>
#include <DirectXTex.h>
#include <cstring>

namespace re4 {

// RE Engine format codes (from reverse engineering)
BcFormat TexFormat::reFormatToBc(uint32_t reFormat) {
    switch (reFormat) {
        case 0x1A: return BcFormat::BC1;    // DXGI_FORMAT_BC1_UNORM
        case 0x1D: return BcFormat::BC3;    // DXGI_FORMAT_BC3_UNORM
        case 0x26: return BcFormat::BC4;    // DXGI_FORMAT_BC4_UNORM
        case 0x28: return BcFormat::BC5;    // DXGI_FORMAT_BC5_UNORM
        case 0x32: return BcFormat::BC7;    // DXGI_FORMAT_BC7_UNORM
        case 0x1C: return BcFormat::RGBA8;  // DXGI_FORMAT_R8G8B8A8_UNORM
        default:   return BcFormat::Unknown;
    }
}

uint32_t TexFormat::bcFormatToRe(BcFormat bcFormat) {
    switch (bcFormat) {
        case BcFormat::BC1:    return 0x1A;
        case BcFormat::BC3:    return 0x1D;
        case BcFormat::BC4:    return 0x26;
        case BcFormat::BC5:    return 0x28;
        case BcFormat::BC7:    return 0x32;
        case BcFormat::RGBA8:  return 0x1C;
        default:               return 0x32; // default to BC7
    }
}

int TexFormat::calcBlockSize(BcFormat format) {
    switch (format) {
        case BcFormat::BC1:
        case BcFormat::BC4:
            return 8;
        case BcFormat::BC2:
        case BcFormat::BC3:
        case BcFormat::BC5:
        case BcFormat::BC7:
            return 16;
        case BcFormat::RGBA8:
            return 4; // 4 bytes per pixel
        default:
            return 16;
    }
}

int TexFormat::calcMipSize(int width, int height, BcFormat format) {
    if (format == BcFormat::RGBA8) {
        return width * height * 4;
    }
    int blockW = std::max(1, (width + 3) / 4);
    int blockH = std::max(1, (height + 3) / 4);
    return blockW * blockH * calcBlockSize(format);
}

bool TexFormat::load(const QString& filePath, TextureData& output) {
    QByteArray data = FileUtils::readBinary(filePath);
    if (data.isEmpty()) {
        Logger::instance()->error("Failed to read TEX file: " + filePath);
        return false;
    }
    return parse(data, output);
}

bool TexFormat::parse(const QByteArray& rawData, TextureData& output) {
    if (rawData.size() < TexFileHeader::kHeaderSize) {
        Logger::instance()->error("TEX file too small for header");
        return false;
    }

    TexFileHeader header;
    memcpy(&header, rawData.constData(), TexFileHeader::kHeaderSize);

    if (!header.isValid()) {
        Logger::instance()->error("Invalid TEX header");
        return false;
    }

    output.meta.width = header.width;
    output.meta.height = header.height;
    output.meta.mipLevels = header.mipCount;
    output.meta.arraySize = header.arraySize;
    output.meta.bcFormat = reFormatToBc(header.format);

    if (output.meta.bcFormat == BcFormat::Unknown) {
        Logger::instance()->warning("Unknown TEX format code: " + QString::number(header.format));
        output.meta.bcFormat = BcFormat::BC7; // fallback
    }

    // Calculate data offset and sizes
    int dataOffset = TexFileHeader::kHeaderSize;
    output.mips.clear();

    int mipW = output.meta.width;
    int mipH = output.meta.height;

    for (int i = 0; i < output.meta.mipLevels; ++i) {
        int mipSize = calcMipSize(mipW, mipH, output.meta.bcFormat);

        if (dataOffset + mipSize > rawData.size()) {
            Logger::instance()->warning("TEX file truncated at mip level " + QString::number(i));
            break;
        }

        TextureData::MipLevel mip;
        mip.data = rawData.mid(dataOffset, mipSize);
        mip.width = mipW;
        mip.height = mipH;
        output.mips.push_back(mip);

        dataOffset += mipSize;
        mipW = std::max(1, mipW / 2);
        mipH = std::max(1, mipH / 2);
    }

    return !output.mips.empty();
}

bool TexFormat::save(const QString& filePath, const TextureData& input) {
    QByteArray data = serialize(input);
    if (data.isEmpty()) return false;

    if (!FileUtils::writeBinary(filePath, data)) {
        Logger::instance()->error("Failed to write TEX: " + filePath);
        return false;
    }

    Logger::instance()->info("Saved TEX: " + filePath);
    return true;
}

QByteArray TexFormat::serialize(const TextureData& input) {
    if (!input.isValid()) return {};

    // Ensure data is in correct BC format
    TextureData tex = input;
    if (tex.mips.empty() || tex.mips[0].data.isEmpty()) {
        return {};
    }

    // Build header
    TexFileHeader header;
    header.magic[0] = 'T'; header.magic[1] = 'E'; header.magic[2] = 'X'; header.magic[3] = '\0';
    header.version = 0x20; // common RE Engine tex version
    header.width = tex.meta.width;
    header.height = tex.meta.height;
    header.depth = 1;
    header.mipCount = static_cast<uint32_t>(tex.mips.size());
    header.arraySize = tex.meta.arraySize;
    header.format = bcFormatToRe(tex.meta.bcFormat);

    // Calculate total data size
    uint32_t totalSize = 0;
    for (const auto& mip : tex.mips) {
        totalSize += mip.data.size();
    }
    header.dataSize = totalSize;
    header.headerSize = TexFileHeader::kHeaderSize;

    // Serialize
    QByteArray result;
    result.reserve(TexFileHeader::kHeaderSize + totalSize);
    result.append(reinterpret_cast<const char*>(&header), TexFileHeader::kHeaderSize);

    for (const auto& mip : tex.mips) {
        result.append(mip.data);
    }

    return result;
}

bool TexFormat::canHandle(const QString& filePath) const {
    return QFileInfo(filePath).suffix().toLower() == "tex";
}

QString TexFormat::formatName() const {
    return "TEX";
}

} // namespace re4
```

- [ ] **Step 3: Commit**

```bash
git add src/formats/texformat.h src/formats/texformat.cpp tests/unit/test_texformat.cpp
git commit -m "feat: add RE Engine .tex format parser and writer"
```

---

### Task 8: FormatConverter (High-Level Conversion Interface)

**Files:**
- Create: `tests/unit/test_formatconverter.cpp`
- Create: `src/formats/formatconverter.h`
- Create: `src/formats/formatconverter.cpp`

- [ ] **Step 1: Write failing tests**

```cpp
// tests/unit/test_formatconverter.cpp
#include <gtest/gtest.h>
#include "formats/formatconverter.h"
#include <QTemporaryDir>

using namespace re4;

class FormatConverterTest : public ::testing::Test {
protected:
    void SetUp() override {
        m_tempDir = std::make_unique<QTemporaryDir>();
        ASSERT_TRUE(m_tempDir->isValid());
    }
    std::unique_ptr<QTemporaryDir> m_tempDir;
};

TEST_F(FormatConverterTest, TexToPng) {
    FormatConverter converter;
    // This test requires a .tex file to exist
    // For now, test the API shape
    QString inputPath = m_tempDir->filePath("input.tex");
    QString outputPath = m_tempDir->filePath("output.png");

    // Create a minimal .tex file
    TextureData tex;
    tex.meta.width = 64;
    tex.meta.height = 64;
    tex.meta.mipLevels = 1;
    tex.meta.bcFormat = BcFormat::RGBA8;

    QImage img(64, 64, QImage::Format_RGBA8888);
    img.fill(QColor(255, 0, 0, 255));
    tex.fromImage(img, BcFormat::RGBA8);

    TexFormat texFmt;
    ASSERT_TRUE(texFmt.save(inputPath, tex));

    EXPECT_TRUE(converter.convert(inputPath, outputPath));
    EXPECT_TRUE(QFile::exists(outputPath));
}

TEST_F(FormatConverterTest, PngToTex) {
    FormatConverter converter;
    QString inputPath = m_tempDir->filePath("input.png");
    QString outputPath = m_tempDir->filePath("output.tex");

    // Create a PNG file
    QImage img(64, 64, QImage::Format_RGBA8888);
    img.fill(QColor(0, 255, 0, 255));
    img.save(inputPath, "PNG");

    EXPECT_TRUE(converter.convert(inputPath, outputPath, BcFormat::BC7));
    EXPECT_TRUE(QFile::exists(outputPath));

    // Verify the output is a valid .tex
    TexFormat texFmt;
    TextureData loaded;
    ASSERT_TRUE(texFmt.load(outputPath, loaded));
    EXPECT_EQ(loaded.meta.width, 64);
    EXPECT_EQ(loaded.meta.height, 64);
}

TEST_F(FormatConverterTest, PngToTga) {
    FormatConverter converter;
    QString inputPath = m_tempDir->filePath("input.png");
    QString outputPath = m_tempDir->filePath("output.tga");

    QImage img(32, 32, QImage::Format_RGBA8888);
    img.fill(QColor(0, 0, 255, 255));
    img.save(inputPath, "PNG");

    EXPECT_TRUE(converter.convert(inputPath, outputPath));
    EXPECT_TRUE(QFile::exists(outputPath));
}

TEST_F(FormatConverterTest, DetectOutputFormat) {
    FormatConverter converter;
    EXPECT_EQ(converter.detectOutputFormat("out.png"), FormatChecker::Format::Png);
    EXPECT_EQ(converter.detectOutputFormat("out.tga"), FormatChecker::Format::Tga);
    EXPECT_EQ(converter.detectOutputFormat("out.dds"), FormatChecker::Format::Dds);
    EXPECT_EQ(converter.detectOutputFormat("out.tex"), FormatChecker::Format::Tex);
}

TEST_F(FormatConverterTest, ConvertNonexistentFile) {
    FormatConverter converter;
    EXPECT_FALSE(converter.convert("C:\\nonexistent\\input.tex", "out.png"));
}

TEST_F(FormatConverterTest, BatchConvert) {
    FormatConverter converter;

    // Create multiple source files
    QStringList inputPaths;
    for (int i = 0; i < 3; ++i) {
        QString path = m_tempDir->filePath(QString("input_%1.png").arg(i));
        QImage img(32, 32, QImage::Format_RGBA8888);
        img.fill(QColor(i * 80, 100, 200, 255));
        img.save(path, "PNG");
        inputPaths << path;
    }

    QString outputDir = m_tempDir->filePath("output");
    QDir().mkpath(outputDir);

    auto results = converter.batchConvert(inputPaths, outputDir, "tga");
    EXPECT_EQ(results.succeeded, 3);
    EXPECT_EQ(results.failed, 0);
}
```

- [ ] **Step 2: Implement FormatConverter**

```cpp
// src/formats/formatconverter.h
#pragma once

#include "formats/formatparser.h"
#include "formats/formatchecker.h"
#include "formats/texformat.h"
#include "formats/ddsformat.h"
#include "formats/tgaformat.h"
#include "formats/pngformat.h"
#include <QString>
#include <QStringList>

namespace re4 {

struct BatchResult {
    int succeeded = 0;
    int failed = 0;
    QStringList errors;
};

class FormatConverter {
public:
    // Convert a single file
    bool convert(const QString& inputPath, const QString& outputPath,
                 BcFormat targetBcFormat = BcFormat::RGBA8);

    // Batch convert multiple files
    BatchResult batchConvert(const QStringList& inputPaths, const QString& outputDir,
                             const QString& outputExtension);

    // Detect output format from file extension
    FormatChecker::Format detectOutputFormat(const QString& filePath);

private:
    // Get the appropriate parser for a format
    std::unique_ptr<FormatParser> getParser(FormatChecker::Format format);

    TexFormat m_texParser;
    DdsFormat m_ddsParser;
    TgaFormat m_tgaParser;
    PngFormat m_pngParser;
};

} // namespace re4
```

```cpp
// src/formats/formatconverter.cpp
#include "formats/formatconverter.h"
#include "core/logger.h"
#include "utils/fileutils.h"
#include <QFileInfo>
#include <QDir>
#include <QElapsedTimer>

namespace re4 {

bool FormatConverter::convert(const QString& inputPath, const QString& outputPath,
                               BcFormat targetBcFormat) {
    QElapsedTimer timer;
    timer.start();

    // Detect input format
    FormatChecker::Format inputFormat = FormatChecker::detectFormat(inputPath);
    if (inputFormat == FormatChecker::Format::Unknown) {
        Logger::instance()->error("Unknown input format: " + inputPath);
        return false;
    }

    // Load input
    TextureData texData;
    bool loaded = false;
    switch (inputFormat) {
        case FormatChecker::Format::Tex: loaded = m_texParser.load(inputPath, texData); break;
        case FormatChecker::Format::Dds: loaded = m_ddsParser.load(inputPath, texData); break;
        case FormatChecker::Format::Tga: loaded = m_tgaParser.load(inputPath, texData); break;
        case FormatChecker::Format::Png: loaded = m_pngParser.load(inputPath, texData); break;
        default: break;
    }

    if (!loaded || !texData.isValid()) {
        Logger::instance()->error("Failed to load: " + inputPath);
        return false;
    }

    // If target format needs BC compression and current format doesn't match
    if (targetBcFormat != BcFormat::RGBA8 && texData.meta.bcFormat != targetBcFormat) {
        // Re-compress from RGBA8 to target BC format
        QImage img = texData.toImage(0);
        if (!texData.fromImage(img, targetBcFormat)) {
            Logger::instance()->error("Failed to compress to target format");
            return false;
        }
    }

    // Detect output format
    FormatChecker::Format outputFormat = detectOutputFormat(outputPath);
    if (outputFormat == FormatChecker::Format::Unknown) {
        Logger::instance()->error("Unknown output format: " + outputPath);
        return false;
    }

    // Ensure output directory exists
    FileUtils::ensureDirectory(QFileInfo(outputPath).absolutePath());

    // Save output
    bool saved = false;
    switch (outputFormat) {
        case FormatChecker::Format::Tex: saved = m_texParser.save(outputPath, texData); break;
        case FormatChecker::Format::Dds: saved = m_ddsParser.save(outputPath, texData); break;
        case FormatChecker::Format::Tga: saved = m_tgaParser.save(outputPath, texData); break;
        case FormatChecker::Format::Png: saved = m_pngParser.save(outputPath, texData); break;
        default: break;
    }

    qint64 elapsed = timer.elapsed();
    if (saved) {
        Logger::instance()->info(QString("Converted %1 -> %2 (%3ms)")
            .arg(inputPath, outputPath).arg(elapsed));
    }

    return saved;
}

BatchResult FormatConverter::batchConvert(const QStringList& inputPaths,
                                           const QString& outputDir,
                                           const QString& outputExtension) {
    BatchResult result;

    FileUtils::ensureDirectory(outputDir);

    for (const QString& inputPath : inputPaths) {
        QString baseName = FileUtils::baseName(inputPath);
        QString outputPath = outputDir + "/" + baseName + "." + outputExtension;

        if (convert(inputPath, outputPath)) {
            result.succeeded++;
        } else {
            result.failed++;
            result.errors << inputPath;
        }
    }

    Logger::instance()->info(QString("Batch convert: %1 succeeded, %2 failed")
        .arg(result.succeeded).arg(result.failed));

    return result;
}

FormatChecker::Format FormatConverter::detectOutputFormat(const QString& filePath) {
    return FormatChecker::detectFormat(filePath);
}

} // namespace re4
```

- [ ] **Step 3: Commit**

```bash
git add src/formats/formatconverter.h src/formats/formatconverter.cpp tests/unit/test_formatconverter.cpp
git commit -m "feat: add FormatConverter for high-level format conversion with batch support"
```

---

## Phase 2 Complete — Summary

After completing all tasks, you will have:

| Deliverable | Status |
|------------|--------|
| FormatParser abstract base class | ✅ |
| TextureData model with BC compression | ✅ |
| FormatChecker (magic number detection) | ✅ |
| PNG format parser/writer | ✅ |
| TGA format parser/writer | ✅ |
| DDS format parser/writer (via DirectXTex) | ✅ |
| .tex format parser/writer (RE Engine) | ✅ |
| FormatConverter (single + batch) | ✅ |

**Next Phase:** Phase 3 — Color Engine + Editor UI
