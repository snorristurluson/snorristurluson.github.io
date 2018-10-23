---
title: Text rendering in Vulkan using FreeType
tags: vulkan freetype c++
---
For my [Vulkan sprite renderer](../BackToBasics) I decided to use
[FreeType](https://www.freetype.org/)
for text rendering.

Text rendering is essentially drawing character bitmaps, laid out one after another
so that they line up, with appropriate spacing.
[TrueType](https://en.wikipedia.org/wiki/TrueType) fonts, however, are defined by straight line
segments and quadratic Bézier curves - this is where FreeType comes in, to convert those lines
and curves to bitmaps.

Here's the main loop for 
[my sample app](https://github.com/snorristurluson/vulkan-sprites/tree/master/samples/text)
for text rendering:
```cpp
void TextApp::Run() {
    Renderer r;
    r.Initialize(m_window, Renderer::ENABLE_VALIDATION);

    auto ta = r.CreateTextureAtlas(512, 512);
    FontManager fm(ta);
    auto font = fm.GetFont("resources/montserrat/Montserrat-Bold.ttf", 36);
    r.SetClearColor({1.0f, 1.0f, 1.0f, 1.0f});
    while (!glfwWindowShouldClose(m_window)) {
        glfwPollEvents();
        if(r.StartFrame()) {
            r.SetColor({0.0f, 0.0f, 0.0f, 1.0f});
            font->Draw(r, 80, 100, "This is a test");

            r.SetColor({1.0f, 0.0f, 0.0f, 1.0f});
            font->Draw(r, 30, 140, "This is a red test");

            r.SetColor({0.0f, 0.0f, 1.0f, 1.0f});
            font->Draw(r, 30, 180, "The quick brown fox");
            font->Draw(r, 60, 220, "jumps over the lazy dog");

            r.EndFrame();
        }
    }
    r.WaitUntilDeviceIdle();
}
```
![TextApp sample](/images/TextAppSample.png)

## The classes
The *FontManager* class has a *GetFont* method that returns a shared pointer to a *Font*.
The *GetFont* method takes two parameters - a path to a font file and the size, in points.

The *Font* class has a *GetGlyph* method, returning a *Glyph* corresponding to a given
Unicode character. The *Glyph* class has a *GetTexture* method, as well as methods for
getting metrics on the glyph, that are used for placing the bitmap when rendering text.

When rendering text on the application level, the *Glyph* class isn't used directly,
though - the *Font* class also has the *Draw* method:

```cpp
void Font::Draw(Renderer &r, float x, float y, const std::string &text) {
    int pos = x;

    for(auto c: text) {
        auto glyph = GetGlyph(c);
        auto texture = glyph->GetTexture();
        if(texture) {
            r.SetTexture(texture);
            float x0 = pos + glyph->GetLeft();
            float y0 = y - glyph->GetTop();
            r.DrawSprite(x0, y0, texture->GetWidth(), texture->GetHeight());
        }
        pos += glyph->GetAdvance();
    }
}
```
Note that the bitmaps for different glyphs in a font are not of a uniform size, but rather
tightly packed around the pixels needed for each character. An upper-case 'A' is much
wider than the lower case 'i', for example. 

A space doesn't get any bitmap, but it
has an advance value that tells us how far to move horizontally. This is why we need the
*if(texture)* check.

The *GetLeft* and *GetTop* methods give us the offsets from the *x* and *y* positions,
to compensate for the different bitmap sizes, and *GetAdvance* tells us how far to move
to position for the next character.

## The implementations
The *GetFont* method on the font manager is a wrapper around *FT_New_Face*:
```cpp
std::shared_ptr<Font> FontManager::GetFont(const std::string& fontname, int pt) {
    FT_Face face;
    auto error = FT_New_Face(m_library, fontname.c_str(), 0, &face);
    if(!error) {
        error = FT_Set_Pixel_Sizes(face, 0, static_cast<FT_UInt>(pt));
        if(error) {
            throw std::runtime_error("couldn't set pixel sizes");
        }
        return std::make_shared<Font>(face, m_textureAtlas);
    }

    throw std::runtime_error("couldn't load font");
}
```
The *Font::GetGlyph* method is a wrapper around *FT_Get_Char_Index*. It also caches the
results, to ensure we're not getting a new bitmap every time we draw a character.
The conversion to a bitmap only needs to happen once per character.
```cpp
std::shared_ptr<Glyph> Font::GetGlyph(uint16_t c) {
    auto foundIt = m_glyphs.find(c);
    if(foundIt != m_glyphs.end()) {
        return foundIt->second;
    }

    auto glyphIndex = FT_Get_Char_Index(m_face, c);
    auto glyph = std::make_shared<Glyph>(m_face, glyphIndex, m_textureAtlas);

    m_glyphs[c] = glyph;
    return glyph;
}
``` 
The constructor for the *Glyph* calls *FT_Load_Glyph* and *FT_Render_Glyph*, then
caches the relevant information about the glyph as well as the bitmap itself:
```cpp
Glyph::Glyph(FT_Face face, FT_UInt ix, std::shared_ptr<TextureAtlas> ta) : m_face(face), m_glyphIndex(ix) {
    auto error = FT_Load_Glyph(m_face, m_glyphIndex, FT_LOAD_DEFAULT);
    if(error) {
        throw std::runtime_error("failed to load glyph");
    }

    FT_GlyphSlot glyphSlot = m_face->glyph;
    error = FT_Render_Glyph(glyphSlot, FT_RENDER_MODE_NORMAL);
    if(error) {
        throw std::runtime_error("failed to render glyph");
    }

    m_left = glyphSlot->bitmap_left;
    m_top = glyphSlot->bitmap_top;
    m_width = static_cast<int>(glyphSlot->metrics.width / 64);
    m_height = static_cast<int>(glyphSlot->metrics.height / 64);
    m_advance = static_cast<int>(glyphSlot->advance.x / 64);

    CreateTextureFromBitmap(ta);
}
```
Finally the *CreateTextureFromBitmap* helper method, that copies the bitmap data from
the FreeType buffer to a texture (I've taken some shortcuts by only supporting the most 
common bitmap format):
```cpp
void Glyph::CreateTextureFromBitmap(std::shared_ptr<TextureAtlas> &ta) {
    FT_GlyphSlot glyphSlot = m_face->glyph;

    if(glyphSlot->bitmap.pixel_mode != FT_PIXEL_MODE_GRAY || glyphSlot->bitmap.num_grays != 256) {
        throw std::runtime_error("unsupported pixel mode");
    }

    auto width = glyphSlot->bitmap.width;
    auto height = glyphSlot->bitmap.rows;
    auto bufferSize = width*height*4;

    if(bufferSize == 0) {
        return;
    }

    std::vector<uint8_t> buffer(bufferSize);

    uint8_t* src = glyphSlot->bitmap.buffer;
    uint8_t* startOfLine = src;
    int dst = 0;

    for(int y = 0; y < height; ++y) {
        src = startOfLine;
        for(int x = 0; x < width; ++x) {
            auto value = *src;
            src++;

            buffer[dst++] = 0xff;
            buffer[dst++] = 0xff;
            buffer[dst++] = 0xff;
            buffer[dst++] = value;
        }
        startOfLine += glyphSlot->bitmap.pitch;
    }
    m_texture = ta->Add(width, height, buffer.data());
}
```
The bitmap provided by FreeType is an 8-bit grayscale format. I'm using a dynamic texture
atlas for all my textures so it is a full 32-bit RGBA format, and this method does the
conversion for each pixel to a full white, with the original grayscale value as the alpha
component.

## Text layout
The *Font* also has a *Measure* method, that iterates over the given text in the same
way as the *Draw*, without actually rendering anything, but it returns the dimensions
of the text if it were to be rendered:
```cpp
TextDimensions Font::Measure(const std::string& text) {
    int width = 0;
    int height = 0;

    for(auto c: text) {
        auto glyph = GetGlyph(c);
        width += glyph->GetAdvance();
        height = std::max(height, glyph->GetTop() + glyph->GetHeight());
    }

    return TextDimensions{width, height};
}
```
This can be used as the basis for centering the text, for example, or right-aligning.

## What's next?
My simple engine can now render text, but it's rather rudimentary. At some point I want
to do proper text layout so I can render text, with some markup, that changes the type
face, font size, with bold and italics, etc.
