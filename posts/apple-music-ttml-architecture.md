# How Apple Music for Android Parses TTML Lyrics: The Complete Inside Story

> This blog is AI-helped blog, information may could be missed

> **Fun Fact:** Even though Apple Music for Android runs on an operating system built around Java and Kotlin—and even bundles Google's ExoPlayer (which already includes a built-in `TtmlDecoder`)—Apple **completely ignores Android's native TTML parser**. Instead, Apple ships a massive **C++ engine (`libandroidappmusic.so`)** internally codenamed **`FuseSV`** alongside Apple's own **CoreFoundation (`CFTree` / `CFString`)** ported straight from iOS and macOS, bridging it to the Android UI using **JavaCPP (JNI)**.

This guide explores **every single piece of data** hidden inside an Apple Music **TTML (Timed Text Markup Language)** file and walks through how Apple Music for Android parses, validates, transforms, and renders it. Each section starts with a clear, everyday explanation, followed by **"For Engineers"** deep dives backed by exact C++ class names, memory structures, and disassembly findings from `libandroidappmusic.so`.

---

## 1. The Big Picture: What Lives Inside an Apple Music TTML File?

Modern Apple Music lyrics pack an entire theatrical script and relational metadata database into a single XML document:

* **Syllable-by-syllable karaoke wipes** where individual words light up and gently float upward as they are sung.
* **True multi-singer duets** where Singer A appears on the left, Singer B appears on the right, and groups appear centered—even when multiple vocalists sing overlapping lines at the exact same millisecond.
* **Independent background vocals** that appear in smaller text underneath the main line (like `(Ooh, yeah)`), complete with their own word-by-word karaoke timing and optional secondary singer attribution.
* **Three distinct localization modes**: Subtitles below the line (`type="subtitle"`), pronunciation/romanization guides floating above each syllable (`<transliterations>`), and seamless full-script replacements (`type="replacement"`).
* **Rich catalog & audio metadata**: Singer types (person, character, group, organization), multiple artist name variants (full name, given name, alias), localized songwriter credits linked to Apple Music catalog IDs, and even a **Dolby Atmos-specific timing offset**.

To store all of this inside a single file, Apple Music uses a customized dialect of **TTML** (an XML-based timed text standard created by the [W3C](https://www.w3.org/TR/ttml1/)).

| TTML Subsystem | XML Location | What Apple Music's `FuseSV` Engine Extracts |
| :--- | :--- | :--- |
| **Sync Granularity** | `<tt itunes:timing="...">` | `Word` (`2`, syllable-synced), `Line` (`1`, line-synced), or `None` (`0`, static) |
| **Performers (`<ttm:agent>`)** | `<head>` -> `<metadata>` | 6 Agent Types (`Person`, `Character`, `Group`, `Organization`, `Other`, `None`), catalog `artistId`, & 6 `<ttm:name>` types |
| **Background Vocals** | `<body>` & `<head>` | Dedicated `<span ttm:role="x-bg">` stream (supported in main lines, translations, *and* transliterations!) |
| **Translations & Scripts** | `<head>` -> `<translations>` | Subtitle lines (`type="subtitle"`) vs. in-place script replacements (`type="replacement"`) + `automaticallyCreated` flags |
| **Pronunciation / Romanization** | `<head>` -> `<transliterations>` | Word-synced `<transliteration>` streams with main and background vocal words + `automaticallyCreated` flags |
| **Songwriter Credits** | `<head>` -> `<songwriters>` | Localized `<songwriters xml:lang="...">` blocks with per-writer Apple Music catalog `artistId` |
| **Audio Codec Compensation** | `<head>` -> `<audio>` | Conditional `<audio lyricOffset="..."/>` activated **exclusively for Dolby Atmos (`EAC3_JOC`)** |
| **Song Structure** | `<body>` -> `<div>` | Structural sections (`Intro`, `Verse`, `PreChorus`, `Chorus`, `Bridge`, `Outro`) with time bounds and agent links |

---

## 2. Before Parsing Begins: How Apple Music Fetches TTML

Before the C++ parser ever sees a character of XML, the Android app fetches the lyrics payload from Apple's Media API (`amp-api.music.apple.com`).

When requesting a song's lyrics, Apple Music for Android doesn't just ask for `lyrics`—it appends two special query parameters based on your phone's language and script settings (`LocaleUtil.getSystemLyricsScripts()`):

* `extend=ttmlLocalizations`
* `l[script]=en-Latn,ja-Latn,...`

When the JSON response arrives (`com.apple.android.music.ttml.f.java`), the app checks two fields inside `attributes` in strict priority order:

1. **`attributes.getTtmlLocalizations()` (Primary)**: A localized TTML payload that already bundles the `<translations>` and `<transliterations>` blocks matching the user's device scripts.
2. **`attributes.getTtml()` (Fallback)**: The base TTML file without extra regional localization bundles.
3. **Static Lyrics Fallback (`StaticLyricsParser`)**: If a song only has untimed plain-text lyrics, Apple Music still calls into `libandroidappmusic.so` via `StaticLyricsParser::songInfoFromStaticLyricsString`, wrapping the plain text into the exact same C++ `SongInfo` model with `Timing = None (0)`.

---

## 3. Step-by-Step: How the C++ Engine Parses TTML

When you open the lyrics view on Android, the raw TTML string crosses the JNI bridge into `libandroidappmusic.so` via `TTMLParser::songInfoFromTTML`. Here is what happens inside the native binary:

### Step 1: Building Apple's CoreFoundation XML Tree (`CFXMLTree`)

Rather than using Android's `XmlPullParser` or `javax.xml`, `TTMLParser::songInfoFromTTML` wraps the UTF-8 string in a `CFData` buffer and calls **`CFXMLTreeCreateFromDataWithError`**—an XML parser from Apple's own **CoreFoundation** framework compiled directly for ARM64 Android.

Every XML tag, attribute map, and text node is stored as a `CFTree` node containing `CFString` objects.

### Step 2: Namespace-Agnostic Tag & Attribute Matching

In XML, tags and attributes often carry namespace prefixes like `itunes:key="L1"`, `ttm:agent="v1"`, or `xml:lang="ja"`. A strict XML parser will break if a producer changes a prefix or omits a namespace declaration.

Apple's C++ parser avoids this by stripping prefixes during traversal:

* Whenever `getAttribute` or `enumerateChildren` inspects a name, it calls `removeNamespace`, which **splits the string on `:` (`0x3a`) and keeps only the local name after the colon** (for example, `itunes:songPart` becomes `songPart`, and `xml:lang` becomes `lang`), comparing it case-insensitively (`CFStringCompare` with option `1`).
* Because of this, `itunes:timing="Word"`, `ttm:timing="Word"`, and unprefixed `timing="Word"` all behave identically.

> **For Engineers — Two Unprefixed Exceptions in the Binary:**
>
> While `getAttribute` and `enumerateChildren` strip namespace prefixes, two lookups in `libandroidappmusic.so` compare raw node names directly without calling `removeNamespace`:
> 1. `findChildNodeWithName` when locating the root `<tt>` element (`0x304ff0`).
> 2. `LyricsSectionParser::make` when checking if a child of `<div>` is `<p>` (`0x3058d0`). If a child inside `<div>` is not literally named `p` (case-insensitive), the parser logs `"Encountered an non-P element under the lines! Ignore it. {0}"` and skips it.

### Step 3: Reading the `<tt>` Root Attributes

Once `<tt>` is found, `SongInfoParser::make` reads two root attributes:

* **`xml:lang` (`lang`)**: Stored as the song's base language via `SongInfo::setLanguage`.
* **`itunes:timing` (`timing`)**: Parsed by `SongInfoParser::getTiming` (`0x3050a4`) into the `SongInfo::Timing` enum:
  * **`2 = Word`**: Matches `"Word"` (case-insensitive). Enables syllable-by-syllable karaoke parsing.
  * **`1 = Line`**: Matches `"Line"` (or serves as the **default fallback** if the `timing` attribute is omitted entirely!).
  * **`0 = None`**: Matches `"None"`. Treats all lines as static, unsynced lyrics even if timestamps exist on `<p>` tags.

### Step 4: Converting Time Strings into Milliseconds (`timestampFromCFString`)

Every time attribute in TTML (`begin`, `end`, `dur`) is parsed by `timestampFromCFString` (`0x308b8c`) into an integer millisecond value:

1. **Colon Splitting**: Splits the string by `:`. It accepts 1 part (`SS.mmm`), 2 parts (`MM:SS.mmm`), or 3 parts (`HH:MM:SS.mmm`). If more than 3 parts exist, it logs `"Invalid time string: {0} contains more than 3 colons"` and returns `0`.
2. **Hours & Minutes**: Parsed as integers (`intFromCFString`) and multiplied by `3,600,000` and `60,000`.
3. **Seconds to Milliseconds**: Parsed as a 32-bit float (`std::stof`), multiplied by `1000.0f` (`0x447a0000`), and **truncated toward zero** into a signed integer using the ARM64 instruction `fcvtzs`.
4. **Zero-Duration Fix (`begin == end`)**: Inside `LyricsLineParser`, whenever a line or word span has `begin > 0` and `begin == end`, the parser automatically adds **`+1 ms`** to `end` (`end = begin + 1`) so zero-length spans never divide by zero during rendering.

> **For Engineers — The `"1ms"` Trap & Signed Integer Overflow:**
>
> * **No Unit Suffix Support**: `timestampFromCFString` never checks for W3C TTML unit suffixes like `ms`, `s`, `m`, `h`, or `f` (frames). Because `std::stof("1ms")` parses the leading `1` and stops at `'m'`, writing `begin="1ms"` is multiplied by `1000.0f` and becomes **`1000 ms` (1 full second)**! Always write decimal seconds (`0.001`).
> * **32-Bit Signed Truncation**: Although `timestampFromCFString` returns a 64-bit `uint64_t` in `x0`, the float-to-int conversion uses 32-bit `fcvtzs w8, s0` followed by sign-extension `sxtw x19, w8`. Timestamps with a seconds component exceeding `2,147,483.647` seconds overflow into negative numbers.

---

## 4. Deep Dive: Every Piece of Data Inside `<head>`

Many developers assume `<head>` only holds basic `v1`/`v2` duet IDs and simple translation strings. In reality, `SongInfoParser::parseHead` (`0x305264`) runs a **dual-pass discovery** across both `<iTunesMetadata>` and `<metadata>` to extract a rich relational database of performers, catalog IDs, localized songwriters, audio codec offsets, and multi-layered localization trees.

```xml
<head>
  <metadata>
    <!-- 1. Performers / Vocal Agents (Supports 6 Agent Types, Catalog artistId, & 6 Name Types!) -->
    <ttm:agent type="person" xml:id="v1" artistId="400592831">
      <ttm:name type="full">Amala Ratna Zandile Dlamini</ttm:name>
      <ttm:name type="alias">Doja Cat</ttm:name>
    </ttm:agent>
    <ttm:agent type="character" xml:id="v2">
      <ttm:name type="given">Alter Ego</ttm:name>
    </ttm:agent>
    <ttm:agent type="group" xml:id="v1000"/>

    <iTunesMetadata xmlns="http://music.apple.com/lyric-ttml-internal" leadingSilence="0.020">
      <!-- 2. Dolby Atmos (EAC3_JOC) Conditional Timing Offset -->
      <audio lyricOffset="0.150"/>

      <!-- 3. Localized Songwriter Credits with Apple Music Catalog IDs -->
      <songwriters xml:lang="en">
        <songwriter artistId="400592831">Doja Cat</songwriter>
        <songwriter artistId="987654321">Sky Adams</songwriter>
      </songwriters>
      <songwriters xml:lang="ja">
        <songwriter artistId="400592831">ドージャ・キャット</songwriter>
      </songwriters>

      <!-- 4. Translations: Subtitles vs. Script Replacements (+ AI/Auto Flag) -->
      <translations>
        <translation type="subtitle" xml:lang="id" automaticallyCreated="true">
          <text for="L1">
            <span begin="0:05.100" end="0:06.200">Kita berlari ke malam hari</span>
            <span ttm:role="x-bg"><span begin="0:06.300" end="0:07.500">(Iya)</span></span>
          </text>
        </translation>
        <translation type="replacement" xml:lang="ja-Kana">
          <text for="L1">
            <span begin="0:05.100" end="0:05.500">ぼくら</span> <span begin="0:05.500" end="0:06.200">は</span>
          </text>
        </translation>
      </translations>

      <!-- 5. Pronunciations / Romanization (+ AI/Auto Flag) -->
      <transliterations>
        <transliteration xml:lang="ja-Latn" automaticallyCreated="false">
          <text for="L1">
            <span begin="0:05.100" end="0:05.500">Bokura</span> <span begin="0:05.500" end="0:06.200">wa</span>
            <span ttm:role="x-bg"><span begin="0:06.300" end="0:07.500">(Yeah)</span></span>
          </text>
        </transliteration>
      </transliterations>
    </iTunesMetadata>
  </metadata>
</head>
```

### 4.1 Dual-Pass `<head>` Discovery (`parseHead` @ `0x305264`)

Why do some TTML files put `<translations>` inside `<iTunesMetadata>`, while others put them directly inside `<metadata>`? Because `SongInfoParser::parseHead` runs **two separate search passes**:

1. **Pass 1 (`<iTunesMetadata>` via lambda `0x309730`)**: Searches `<head>` for `<iTunesMetadata>` and extracts `<translations>`, `<transliterations>`, `<songwriters>`, and `<audio>`.
2. **Pass 2 (`<metadata>` via lambda `0x3126c4`)**: Searches `<head>` for `<metadata>` and extracts `<agent>` (`<ttm:agent>`), `<translations>`, and `<transliterations>`.

### 4.2 The Complete `<ttm:agent>` Schema (`LyricsAgentParser` @ `0x312990`)

In casual discussions, `<ttm:agent>` is usually described as just `v1` (left singer), `v2` (right singer), and `v1000` (group). However, inspecting `LyricsAgentParser::make` (`0x312990`) and `LyricsAgent.java` reveals a much richer data model:

* **`xml:id` (`id`)**: The unique identifier (such as `v1`, `v2`, `v1000`) referenced by `<div ttm:agent="...">`, `<p ttm:agent="...">`, or `<span ttm:role="x-bg" ttm:agent="...">`.
* **`type`**: Parsed by `LyricsAgentParser::typeFromStringType` (`0x312c38`) into **6 enum values** (`LyricsAgent::Type`):

| `type` Attribute String | C++ Enum (`LyricsAgent::Type`) | Meaning & Usage |
| :--- | :--- | :--- |
| *(omitted or unrecognized)* | `0 = None` | Default fallback if `type` is missing or unknown |
| `"person"` | `1 = Person` | Individual human vocalist (`v1`, `v2`, etc.) |
| `"character"` | `2 = Character` | Fictional persona, musical theater role, or virtual singer |
| `"group"` | `3 = Group` | Multiple singers or ensemble singing together (`v1000`) |
| `"organization"` | `4 = Organization` | Choir, orchestra, or institutional ensemble |
| `"other"` | `5 = Other` | Non-standard vocal source (e.g., synthesized voice or sample) |

* **`artistId`**: Parsed as a **64-bit integer** (`int64FromCFString` @ `0x312a9c`) and saved via `LyricsAgent::setArtistId(unsigned long)`. This links a vocal agent directly to their Apple Music catalog artist page!
* **Child `<ttm:name type="...">` Elements**: `LyricsAgentParser::make` enumerates all `<name>` children (`0x312fe0`) and maps their `type` attribute via `LyricsAgentParser::nameTypeFromStringType` (`0x313158`) into **6 name types** (`LyricsAgent::NameType`):
  * `0 = None` (untyped or unrecognized)
  * `1 = Full` (`type="full"`)
  * `2 = Family` (`type="family"`)
  * `3 = Given` (`type="given"`)
  * `4 = Alias` (`type="alias"`)
  * `5 = Other` (`type="other"`)
  A single `<ttm:agent>` can store **multiple `<ttm:name>` children** simultaneously (for example, both a performer's `full` legal name and their stage `alias`), keyed by `NameType` inside `LyricsAgent`.

### 4.3 Localized Songwriters (`parseSongwriters` @ `0x309a00`)

Inside `<iTunesMetadata>`, `<songwriters>` is not restricted to a single list:

* **Per-Language Songwriter Credits (`xml:lang`)**: `SongInfoParser::parseSongwriters` (`0x309a34`) checks for a `lang` (`xml:lang`) attribute on the `<songwriters>` tag itself. If omitted, it falls back to the song's base `SongInfo::getLanguage()`. This allows a single TTML file to ship multiple `<songwriters xml:lang="...">` blocks so songwriter names can be displayed in native scripts (e.g., English vs. Japanese katakana/kanji).
* **Catalog Artist Linking (`artistId`)**: For each `<songwriter artistId="...">Name</songwriter>` child (`0x312424`), the parser reads `artistId` (converted via `atoi`) and calls `SongInfo::addSongwriter(lang, artistId, name)`.

### 4.4 The Secret Behind `<audio lyricOffset="..."/>`: It Is Exclusively for Dolby Atmos!

If you inspect `<iTunesMetadata>`, you will sometimes spot `<audio lyricOffset="0.150"/>` alongside `leadingSilence="0.000"`. What is the difference between the two, and when is `lyricOffset` actually used?

* **`leadingSilence="0.000"`**: Indicates how many seconds of digital silence exist at the start of the master audio track before the first audible waveform begins. It is audio mastering metadata and **is never read by `SongInfoParser` to shift lyric timestamps**.
* **`<audio lyricOffset="..."/>`**: Parsed as a floating-point number of seconds by `SongInfoParser` (`0x3097f0`) and stored in `SongInfo::setLyricsOffset`.

Here is the secret uncovered from `PlayerLyricsViewFragment.java` (line 1766) and `SongInfoTimeProcessor.cpp`:

```java
// From com/apple/android/music/player/ui/PlayerLyricsViewFragment.java:1766
fVar.a().applyLyricsOffset(str.equalsIgnoreCase(AudioCodec.AUDIO_CODEC_EAC3_JOC));
```

And inside `libandroidappmusic.so` (`SongInfoTimeProcessor.cpp` log string at `0x181000`):

```text
processEvents for position {0} {1} (line offset {2} word offset {3}) (Apply Atmos offset {4} value = {5})
```

**Why this exists:** Apple Music stores **one** TTML lyric file per song, timed against the standard stereo master (AAC / ALAC). However, when a song is mixed in **Dolby Atmos (`AUDIO_CODEC_EAC3_JOC` / `ec+3`)**, the spatial audio encoder or Atmos master often introduces a slight pre-roll latency compared to the stereo file. Rather than maintaining two separate TTML files, Apple embeds `<audio lyricOffset="..."/>` inside `<iTunesMetadata>`, and the Android player **only turns on `applyLyricsOffset(true)` when the active audio codec is `AUDIO_CODEC_EAC3_JOC` (Dolby Atmos)**! When listening in standard Stereo or Lossless, `applyLyricsOffset` is `false` and `lyricOffset` is ignored (`0 ms`).

---

## 5. Deep Dive: Translations, Replacements, and Pronunciations

Apple Music's `<head>` supports three localization mechanisms under `<translations>` and `<transliterations>`. While they look similar in XML, the C++ parser handles them through a sophisticated **dual-storage architecture** inside `ParseContext`.

| TTML Element in `<head>` | Internal C++ System | How It Renders in Apple Music UI | How It Activates |
| :--- | :--- | :--- | :--- |
| **`<translation type="subtitle">`** | `Translation` (`setTranslation`) | **Small subtitle line underneath the main lyric** (`22sp`, dimmed opacity). Lights up as a full line without syllable wipe. | **Manual Toggle** (*"Show Translation"*) |
| **`<translation type="replacement">`** *(or omitted `type`!)* | `Script` (`setScript` / `applyScript`) | **Overwrites the primary vocal line (`34sp`) in place!** Full word-by-word karaoke wipe, `-2dp` spring lift, and glow run on the replaced words. | **100% Automatic** on load when device locale/script matches `xml:lang` |
| **`<transliteration>`** | `Pronunciation` (`setPronunciation`) | **Small ruby/interlinear text floating directly above each corresponding word** (`22sp`), grouped syllable-by-syllable with the main line. | **Manual Toggle** (*"Show Pronunciation"*) |

### 5.1 Hidden Discovery #1: Dual Storage in `ParseContext` (Every Subtitle & Transliteration Is Also Parsed as a Word-Timed Line!)

When `SongInfoParser::parseTranslations` (`0x30a110`) processes a `<text for="L1">` node inside `<translation type="subtitle">`, or when `SongInfoParser::parseTransliterations` (`0x30f584`) processes a `<text for="L1">` node inside `<transliteration>`, it doesn't just extract plain text. It stores **two representations** in `ParseContext` for every single key:

1. **Plain-Text Map**: Calls `getInnerText` and pushes the concatenated string into `ParseContext::pushTranslationString(lang, key, text)` (or `pushPronunciationString`).
2. **Word-Timed `LyricsLine` + Background Vocals Map**: Calls `LyricsLineParser::makeTranslation` (`0x30abbc`) or `LyricsLineParser::makePronunciation` (`0x310504`), which runs `processTranslationSpanElements` / `processPronunciationSpanElements`. This extracts:
   * Timed main words -> stored via `ParseContext::addTranslationLine` / `addPronunciationLine`.
   * Timed **background vocal words (`<span ttm:role="x-bg">`)** inside the translation or transliteration -> stored via `ParseContext::addTranslationBackgroundLine` / `addPronunciationBackgroundLine`!

When `SongInfoParser::parseBody` later builds each `<p itunes:key="L1">` line in `<body>`, it looks up `L1` in `ParseContext` and attaches all four maps (plain string, main timed words, and background timed words for both translations and pronunciations) directly onto the C++ `LyricsLine` object!

### 5.2 Hidden Discovery #2: What Happens If `<translation>` Omits the `type` Attribute?

In `SongInfoParser::parseTranslations` (at address `0x309de0`), the parser checks the `type` attribute of `<translation>`:

* If `type == "subtitle"` (8 characters, `0x656c746974627573`), it routes to the **Subtitle** handler (`0x30a110`).
* If `type == "replacement"` (11 characters) **OR if the `type` attribute is empty/omitted (`cbz x9, 0x309eb4`)**, it routes to the **Script Replacement** handler (`0x30e128`)!
* Inside the Script Replacement handler (`0x30e128`):
  1. It always stores the inner text via `SongInfo::addScript(lang, key, innerText)`.
  2. It calls `LyricsLineParser::make` (`0x30e26c`)—the **exact same line parser used for `<p>` tags in `<body>`**!
  3. It checks if `line->getWords()` is non-empty (`0x30e3c4`). If the `<text for="L1">` block contains timed `<span>` words, it stores the full `shared_ptr<LyricsLine>` via `SongInfo::addScript(lang, key, line)` (`0x30e44c`).
  4. When `SongInfo::setScript(lang)` is called by Android's `PlayerLyricsViewModel`, `LyricsLine::applyScript` swaps the `<p>` line's main words and background words in-place with the replacement `LyricsLine`.

### 5.3 AI / Machine-Generated Attribution (`automaticallyCreated="true"`)

Both `<translation>` (`0x309da8`) and `<transliteration>` (`0x30f2c4`) support an optional boolean attribute:

* **`automaticallyCreated="true"`** (parsed via `boolFromCFString`):
  * On `<translation>`, calls `SongInfo::setTranslationAutomaticallyCreated(lang, true)`.
  * On `<transliteration>`, calls `SongInfo::setPronunciationAutomaticallyCreated(lang, true)`.
  * In the Android UI (`SongInfo.isSelectedTranslationAutomaticallyCreated()` / `isSelectedPronunciationAutomaticallyCreated()`), this flag tells Apple Music whether to display a machine-translation attribution notice to the listener.

### 5.4 Smart Language Fallback (`findFallbackTranslationLanguage` @ `0x2f6c90`)

What happens if a user's phone locale is `en-GB` or `zh-Hant-HK`, but the TTML file only lists `xml:lang="en"` or `xml:lang="zh-Hant"`?

When `setTranslation`, `setScript`, or `setPronunciation` is called, if an exact locale match isn't found in the map, `SongInfo` calls `getPrimaryLanguageCode` (`0x2f95f4`) to strip regional subtags and searches for a matching base language, logging:
`"Exact translation {0} was not found. Falling back to {1}."`

---

## 6. Deep Dive: Every Piece of Data Inside `<body>`, `<div>`, `<p>`, and `<span>`

Once `<head>` is stored in `ParseContext`, `SongInfoParser::parseBody` (`0x3053bc`) parses `<body>` and constructs the hierarchy of sections (`LyricsSection`), lines (`LyricsLine`), and words (`LyricsWord`).

```xml
<body dur="3:42.500">
  <!-- Section (<div>) -->
  <div begin="0:05.100" end="0:25.000" itunes:songPart="Chorus" ttm:agent="v1">
    <!-- Line (<p>) -->
    <p begin="0:05.100" end="0:10.000" itunes:key="L1" ttm:agent="v1">
      <!-- Main Vocal Words (<span>) -->
      <span begin="0:05.100" end="0:05.500">僕ら</span> <span begin="0:05.500" end="0:06.200">は</span>
      <!-- Background Vocal Container (<span ttm:role="x-bg">) -->
      <span ttm:role="x-bg" itunes:parenthesis="keep" ttm:agent="v2">
        <span begin="0:06.300" end="0:07.500">(Yeah)</span>
      </span>
    </p>
  </div>
</body>
```

### 6.1 `<body dur="...">`: The Mandatory Duration Attribute

`SongInfoParser::getDuration` (`0x3054d8`) reads the `dur` attribute on `<body>`. If `dur` is missing on `<body>`, the native parser logs an error:
`"<BODY> must always have a \"duration\" attribute!"`

### 6.2 Song Sections (`<div>` & `LyricsSectionParser::make` @ `0x30562c`)

Every `<div>` inside `<body>` becomes a `LyricsSection` containing:

* **`begin` and `end`**: Section start and end timestamps in milliseconds.
* **`itunes:songPart` (`songPart`)**: Structural label such as `"Intro"`, `"Verse"`, `"PreChorus"`, `"Chorus"`, `"Bridge"`, `"Instrumental"`, or `"Outro"`.
* **`ttm:agent` (`agent`)**: If a `<div>` specifies an agent ID (e.g., `ttm:agent="v1"`), it is looked up in `ParseContext::getAgent` and linked to the section.

### 6.3 Lyric Lines (`<p>` & `LyricsLineParser::make` @ `0x306194`)

Every `<p>` inside `<div>` becomes a `LyricsLine` containing:

* **`begin` and `end`**: Line start and end timestamps in milliseconds (with `+1 ms` protection if `begin == end`).
* **`itunes:key` (`key`)**: The primary lookup key (e.g., `"L1"`, `"L2"`) used to bind translations, replacement scripts, and pronunciations from `ParseContext` onto this `LyricsLine`.
* **`ttm:agent` (`agent`)**: Resolves the singer (`LyricsAgent`) for this specific line (`getLyricsAgent()`).
* **Line-Timed vs. Word-Timed Routing (`0x306480`)**:
  * If `SongInfo::getTiming() == Word (2)`, `LyricsLineParser::make` calls `processSpanElements` to parse child `<span>` tags.
  * If a `<p>` line in a `Word`-timed song has **no child `<span>` elements** (or if `SongInfo::getTiming() != Word`), it extracts the plain inner text of `<p>` (`getInnerText`) and stores it as a single line string (`setOriginalLineText`).

### 6.4 The 4 Hidden Rules of `<span>` Parsing (`processSpanElements` @ `0x30788c`)

How does `LyricsLineParser::processSpanElements` handle text nodes, spaces between words, background vocals, and timing errors? Disassembling `0x30788c` through `0x307290` reveals four strict rules:

#### Rule 1: Any Text or Whitespace *Before* the First `<span>` Is Discarded!
When iterating through the child nodes of a word-timed `<p>` (or a `<text>` in translations/transliterations), if the parser encounters a plain text node *before* seeing the very first `<span>` (`mainWords.empty()`), it logs:
`"Ignore whitespace/text before first <span>"` (`0x3079fc`) and **throws that text away**. Never put untimed words before the first `<span>` in a word-timed line!

#### Rule 2: Spaces *Between* `<span>` Tags Become `LyricsWord` Objects with `-1` Timestamps
Once at least one `<span>` has been parsed, any plain text or space between `<span>` elements (such as the space in `<span>Hello</span> <span>world</span>`) is **not** merged into the neighboring word string. Instead, the C++ parser creates a dedicated `LyricsWord` object with:
* `begin = -1` (`0xffffffffffffffff`)
* `end = -1` (`0xffffffffffffffff`)
* `LyricsWord::isWhitespace()` (`0x2eaf08`) returning `true` when the text consists of whitespace characters.

#### Rule 3: Single-Depth Background Vocals (`ttm:role="x-bg"`) & Automatic Parentheses
When `processSpanElements` sees a `<span>` with `role="x-bg"` (`0x307c3c`):
* **Single-Depth Guard**: It checks a boolean flag (`isBackground`). If `isBackground` is already `true` (meaning someone nested an `x-bg` span inside another `x-bg` span), it logs `"Encountered a background vocals inside a background vocals! Ignore it."` and skips it.
* **Separate Word Stream**: Otherwise, it recursively calls `processSpanElements` with `isBackground = true`, routing all inner `<span>` words into the line's `backgroundWords` vector instead of `mainWords`.
* **Automatic Parenthesis Wrapping (`_getHtmlBackgroundVocalsLineTextInternal` @ `0x2ee454`)**: When rendering background vocals, Apple Music automatically wraps the background line in parentheses `(...)`. To prevent double parentheses like `((Yeah))`, the C++ helper inspects the very first character of the first background word (if `'('`, `0x28`, it strips it) and the very last character of the last background word (if `')'`, `0x29`, it strips it) before adding its own outer parentheses—**unless** `itunes:parenthesis="keep"` is set on the `x-bg` span!

#### Rule 4: Strict Word Invariant Validation (`LyricsLineParser::validate` @ `0x307290`)
After parsing all words in a line, `LyricsLineParser::validate` checks three timing invariants on every non-whitespace `LyricsWord` (`word.begin != -1 && word.end != -1`):
1. **Minimum Word Duration (`> 9 ms`)**: If `word.getDuration() <= 9` (`cmp x8, #9; b.le`), it logs `"Invalid word with duration of {0}: {1}"`.
2. **Lower Bound Check**: If `word.begin < line.begin`, it logs `"Invalid word with begin {0} and line being {1}: {2}"`.
3. **Upper Bound Check**: If `word.end > line.end`, it logs `"Invalid word with end {0} and line end {1}: {2}"`.

---

## 7. How the 5-Stream Playback Clock & Karaoke Physics Work

Once the C++ `SongInfo` object is constructed, how does Apple Music synchronize 60fps/120fps animations with the audio track without draining your battery?

### 7.1 Event-Driven Boundary Scheduling (`SongInfoTimeProcessor::processEvents`)

A naive lyric player runs a loop every 16ms polling all lines and words. Apple Music's **`SongInfoTimeProcessor::processEvents`** uses **event-driven boundary scheduling**:

1. Given the current audio playback position (plus any anticipation offsets and, if playing Dolby Atmos, the `<audio lyricOffset>`), it evaluates **5 parallel streams** using the half-open interval `begin + offset <= position < end + offset`:
   * **Stream 1**: Active `LyricsLine` vector (`getLines`)
   * **Stream 2**: Active Main `LyricsWord` vector (`getWords`)
   * **Stream 3**: Active Background `LyricsWord` vector (`getBackgroundWords`)
   * **Stream 4**: Active Pronunciation `LyricsWord` vector (`getPronunciationWords`)
   * **Stream 5**: Active Pronunciation Background `LyricsWord` vector (`getPronunciationBackgroundWords`)
2. Because all 5 streams return **vectors** (not single pointers), two singers in a duet—or overlapping main and background vocals—can be active at the exact same millisecond.
3. While scanning, `processEvents` calculates the **nearest upcoming timestamp (`nextBoundary`)** across all 5 streams and returns it to Java.
4. `PlayerLyricsViewFragment` schedules a single Android `Handler.postDelayed` callback for `nextBoundary - currentPosition` ms in the future. Between boundaries, the CPU does zero lyric searching while hardware-accelerated View animators smoothly interpolate the syllable wipes!

### 7.2 Built-In Anticipation Offsets

To make lyrics feel responsive to human perception, `PlayerLyricsViewFragment` feeds negative anticipation offsets into `processEvents`:

* **Line Anticipation Offset (`-500 ms` base, dynamic `-480 ms` to `-750 ms` in Karaoke mode)**: Lines activate and scroll into focus roughly **half a second before** the vocalist begins singing.
* **Word Anticipation Offset (`-100 ms` in Karaoke mode)**: Each syllable's wipe begins **100 ms early** so the visual highlight matches the perceived vocal attack.

### 7.3 The 3 Signature Karaoke Animations (`com.apple.android.music.player.A.java`)

Inside `FullWidthAlphaGradientFlexboxLayout` (`player/A.java`), three distinct physics and shader animations bring word-timed TTML to life:

1. **The 30dp Feathered Gradient Wipe (`A.q0`)**:
   Instead of hard-clipping letters, Apple Music draws a horizontal `LinearGradient` mask (`PorterDuff.Mode.DST_IN`) across each wrapped row of words. The soft transition edge (*feather*) is `30 dp` wide (`stops = [0, clamp(progress - feather), clamp(progress), 1]`, transitioning from alpha `255` to inactive alpha `89`). If the row ends with CJK, Hiragana, or Katakana characters (`Character.UnicodeBlock`), it appends an extra **`+500 ms` tail sweep** so dense logographic characters finish wiping gracefully.
2. **The `-2dp` Syllable Spring Lift (`A.j0`)**:
   As each word becomes active, it floats upward by **`-2 dp`** on the Y-axis (`R.dimen.lyrics_karaoke_syllable_lift = -2dp`, loaded into `this.f38953J` at `A.java:1004`), driven by an AndroidX `SpringAnimation` (`M2.d` / `M2.e`) with `dampingRatio = 0.93f`, `stiffness = 25.0f`, and `initialVelocity = 0.0f`:

```java
// From res/values/dimens.xml:
// <dimen name="lyrics_karaoke_syllable_lift">-2dp</dimen>

// From com/apple/android/music/player/A.java (lines 1004 & 1175-1197):
this.f38953J = context.getResources().getDimension(R.dimen.lyrics_karaoke_syllable_lift);

public static p419w6.b j0(float startValue, float finalPosition, final View... viewArr) {
    if (viewArr == null || viewArr.length == 0) {
        return null;
    }
    final M2.c valueHolder = new M2.c(startValue); // FloatValueHolder
    M2.e springForce = new M2.e(finalPosition);    // SpringForce(finalPosition = -2dp)
    springForce.a(0.93f);                          // setDampingRatio(0.93f)
    springForce.b(25.0f);                          // setStiffness(25.0f) -> Math.sqrt(25.0d)
    M2.d springAnim = new M2.d(valueHolder);       // SpringAnimation
    springAnim.f9770a = 0.0f;                      // mVelocity = 0.0f
    springAnim.f9786s = springForce;
    springAnim.b((anim, value, velocity) -> {
        for (View view : viewArr) {
            if (view != null) {
                view.setTranslationY(valueHolder.f9785a);
            }
        }
    });
    return new p419w6.b(springAnim, "KARAOKE_WORD_LIFT_TAG");
}
```

3. **Sustained-Word Emphasis / Letter-by-Letter Glow (`A.a0` & `C3439h.java`)**:
   When a singer holds a note, Apple Music makes the word expand and glow—but **only when all 3 conditions pass**:
   * It is a **main vocal word** (background vocals never trigger emphasis).
   * Its cumulative duration is **`>= 1000 ms` (1.0 second or longer)**.
   * Its text length is **`<= 7` characters**.
   When triggered, each character scales up to **`1.14x`** (`1.07x` on low-RAM devices) around its baseline with a **`5 dp` white glow shadow**, staggered letter-by-letter along a `cubic-bezier(0.25, 0.1, 0.25, 1)` curve while dynamically translating neighboring characters outward so letters never collide!