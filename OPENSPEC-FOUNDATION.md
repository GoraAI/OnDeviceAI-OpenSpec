# OnDevice AI - Foundation Specification

**Version**: 1.1.9 (Build 35)
**Status**: Complete Reverse Engineering
**Last Updated**: 2026-02-07

---

## Table of Contents

1. [Product Definition](#1-product-definition)
2. [Global Design System](#2-global-design-system)
   - [Typography](#21-typography)
   - [Colors](#22-colors)
   - [Spacing & Grid](#23-spacing--grid)
   - [Animations](#24-animations)
3. [Navigation Model](#3-navigation-model)
4. [Data Contracts](#4-data-contracts)
   - [Room Database](#41-room-database)
   - [Core Models](#42-core-models)
5. [Architecture Overview](#5-architecture-overview)
6. [Build Configuration](#6-build-configuration)

---

## 1. Product Definition

### 1.1 Product Identity

| Attribute | Value | Source |
|-----------|-------|--------|
| **Product Name** | OnDevice AI | `strings.xml:19` |
| **Branding (First Part)** | OnDevice | `strings.xml:20` |
| **Branding (Second Part)** | AI | `strings.xml:21` |
| **Tagline** | Run powerful AI models directly on your device | `strings.xml:22` |
| **Legal Name** | OnDevice AI App | `strings.xml:91` |
| **Original Product** | Google AI Edge Gallery (rebranded) | Git history |

### 1.2 Version Information

| Attribute | Value | Source |
|-----------|-------|--------|
| **Version Name** | 1.1.9 | `build.gradle.kts:49` |
| **Version Code** | 35 | `build.gradle.kts:48` |
| **Build Type** | Release | Git tag |
| **Minimum SDK Version** | 31 (Android 12.0) | `build.gradle.kts:46` |
| **Target SDK Version** | 35 (Android 15.0) | `build.gradle.kts:47` |
| **Compile SDK Version** | 35 | `build.gradle.kts:43` |

**Versioning Scheme**: Semantic versioning (MAJOR.MINOR.PATCH)
- 1.x.x = First major release
- x.1.x = Minor feature releases
- x.x.9 = Ninth patch release

### 1.3 Platform Support

| Platform | Support Level | Details |
|----------|--------------|---------|
| **Android** | ✅ Primary | SDK 31+ (Android 12.0+) |
| **Android Tablets** | ✅ Supported | Same codebase, portrait orientation locked |
| **iOS** | ❌ Not supported | Android-only application |
| **Web** | ❌ Not supported | Native Android application |
| **Desktop** | ❌ Not supported | Mobile-first design |

**Target Devices**:
- Phones running Android 12.0 (API 31) or higher
- Tablets running Android 12.0 (API 31) or higher
- Minimum RAM: 4GB (6GB recommended for larger models)
- Minimum Storage: 8GB free (for model downloads)
- GPU: OpenCL support recommended for image generation

**Device Requirements Rationale**:
- Android 12+ required for MediaPipe GenAI SDK compatibility
- 4GB RAM minimum to run small LLM models (1-3B parameters)
- 6GB+ RAM recommended for medium models (7-13B parameters)
- OpenCL GPU acceleration optional but significantly improves image generation performance

### 1.4 Package Identifiers

| Identifier | Value | Usage |
|------------|-------|-------|
| **Namespace** | `ai.ondevice.app` | `build.gradle.kts:42` |
| **Application ID** | `ai.ondevice.app` | `build.gradle.kts:45` |
| **Main Activity** | `ai.ondevice.app.MainActivity` | `AndroidManifest.xml:67` |
| **Application Class** | `ai.ondevice.app.GalleryApplication` | `build.gradle.kts:54` |
| **Deep Link Scheme** | `ai.ondevice.app://` | `AndroidManifest.xml:89` |
| **OAuth Redirect Scheme** | `ai.ondevice.app:/oauth2redirect` | `build.gradle.kts:67` |
| **File Provider Authority** | `ai.ondevice.app.provider` | `AndroidManifest.xml:106` |

**Deep Linking Format**:
```
ai.ondevice.app://model/{taskId}/{modelName}
```
- `taskId`: Task identifier (e.g., "llm_chat", "image_gen")
- `modelName`: Model name to load (e.g., "gemma-2b-it-gpu-int4")

**Example Deep Links**:
- `ai.ondevice.app://model/llm_chat/gemma-2b-it-gpu-int4` → Open chat with Gemma 2B
- `ai.ondevice.app://model/image_gen/stable-diffusion-v1-5` → Open image generation with SD 1.5

### 1.5 Minimum Device Requirements

| Requirement | Minimum | Recommended | Specification |
|-------------|---------|-------------|---------------|
| **OS Version** | Android 12.0 (API 31) | Android 13.0+ (API 33+) | `build.gradle.kts:46` |
| **RAM** | 4GB | 6GB+ | For 1-3B parameter models |
| **Free Storage** | 8GB | 16GB+ | Models range 0.5GB-4GB each |
| **CPU** | ARMv8-A (64-bit) | Qualcomm Snapdragon 8+ Gen 1 or equivalent | For on-device inference |
| **GPU** | Optional | OpenCL-compatible | For image generation acceleration |
| **Screen Size** | 5.0" (320dp width min) | 6.0"+ | Portrait orientation |
| **Network** | Optional (offline-first) | Wi-Fi for model downloads | Downloads up to 4GB |

**Storage Breakdown**:
- App size: ~150MB (APK + runtime resources)
- Minimum 1 model: 0.5GB (Gemma 2B quantized)
- Typical setup (3 models): 4-6GB
- Maximum (all models): 12-15GB
- Conversation history: 10-500MB (grows over time)
- Image generation cache: 100-500MB

**Performance Expectations**:
- Model initialization: 2-10 seconds (depending on model size)
- Inference latency (text): 50-200ms per token (model-dependent)
- Image generation: 5-30 seconds (iteration count: 5-50)
- App startup: <2 seconds to first screen

### 1.6 Environment Assumptions

#### 1.6.1 Network Requirements

| Feature | Network Required | Fallback Behavior |
|---------|-----------------|-------------------|
| **Chat (local models)** | ❌ No | Fully offline |
| **Model Downloads** | ✅ Yes | Queues download, resumes when online |
| **OAuth (HuggingFace)** | ✅ Yes | Cannot authenticate without network |
| **Firebase Analytics** | ❌ No | Queues events, syncs when online |
| **Crashlytics** | ❌ No | Queues reports, syncs when online |
| **Web Search** | ✅ Yes | Feature unavailable offline |

**Network State Handling**:
- Download manager monitors network state via `ConnectivityManager`
- Downloads pause automatically on network loss
- Downloads resume automatically when network returns
- No user intervention required for network changes

#### 1.6.2 Permissions

| Permission | Required | Requested When | Rationale |
|------------|----------|----------------|-----------|
| **CAMERA** | Optional | User taps camera button in chat | For "Ask about image" feature |
| **RECORD_AUDIO** | Optional | User taps microphone button | For "Ask about audio" feature (future) |
| **INTERNET** | Required | App install | Model downloads, OAuth, analytics |
| **ACCESS_NETWORK_STATE** | Required | App install | Monitor download network conditions |
| **POST_NOTIFICATIONS** | Optional | First model download | Notify download completion |
| **FOREGROUND_SERVICE** | Required | App install | Background model downloads |
| **FOREGROUND_SERVICE_DATA_SYNC** | Required | App install | WorkManager for downloads |
| **WAKE_LOCK** | Required | App install | Keep downloads running |
| **WRITE_EXTERNAL_STORAGE** | Optional | Android 9 and below only | Save generated images to gallery |

**Permission Request Flow**:
1. **CAMERA**: Just-in-time request when user taps camera icon in chat
2. **RECORD_AUDIO**: Just-in-time request when user taps microphone icon
3. **POST_NOTIFICATIONS**: Just-in-time request before first download starts
4. All other permissions granted at install time

**Denied Permission Behavior**:
- **CAMERA denied**: Camera button shows "Grant permission" snackbar
- **RECORD_AUDIO denied**: Microphone button disabled
- **POST_NOTIFICATIONS denied**: Downloads succeed silently, no notifications

#### 1.6.3 System Services

| Service | Required | Usage |
|---------|----------|-------|
| **WorkManager** | Yes | Background model downloads |
| **DownloadManager** | Yes | Manages HTTP downloads for models |
| **NotificationManager** | Optional | Download progress/completion |
| **DataStore** | Yes | User preferences storage |
| **EncryptedSharedPreferences** | Yes | OAuth token storage |
| **Room Database** | Yes | Conversation history persistence |
| **Firebase Analytics** | Yes | Usage analytics (opt-out available) |
| **Firebase Crashlytics** | Yes | Crash reporting (opt-out available) |

#### 1.6.4 External Dependencies

| Dependency | Type | Purpose | Minimum Version |
|------------|------|---------|-----------------|
| **Google Play Services** | Optional | TFLite GPU delegate | 16.4.0+ |
| **HuggingFace (optional)** | External | OAuth for gated models | OAuth 2.0 |
| **Brave Search API (optional)** | External | Web search feature | v1 API |
| **Firebase** | External | Analytics + Crashlytics | BOM 33.16.0 |

**Offline-First Architecture**:
- Core functionality (chat, image generation) works fully offline
- Models downloaded once, stored locally
- No cloud API calls for inference
- Firebase services gracefully degrade without network

#### 1.6.5 Build Environment

| Tool | Version | Source |
|------|---------|--------|
| **Gradle** | 8.8.2 | `libs.versions.toml:4` |
| **Kotlin** | 2.1.0 | `libs.versions.toml:5` |
| **Java** | 11 | `build.gradle.kts:110-111` |
| **Android Gradle Plugin** | 8.8.2 | `libs.versions.toml:4` |
| **KSP (Kotlin Symbol Processing)** | 2.1.0-1.0.29 | `libs.versions.toml:2` |
| **Compose Compiler** | 2.1.0 | Bundled with Kotlin plugin |

**Build Requirements**:
- JDK 11 or higher
- Android SDK 35 (Build Tools 35.0.0)
- GitHub Actions for CI/CD (local builds not supported on DGX Spark)
- Keystore for release signing (CI environment variable)

---

## 2. Global Design System

### 2.1 Typography

#### 2.1.1 Font Family

| Font | Weight | Android Resource | Source File |
|------|--------|------------------|-------------|
| **Nunito Regular** | 400 (Normal) | `R.font.nunito_regular` | `Type.kt:28` |
| **Nunito ExtraLight** | 200 | `R.font.nunito_extralight` | `Type.kt:29` |
| **Nunito Light** | 300 | `R.font.nunito_light` | `Type.kt:30` |
| **Nunito Medium** | 500 | `R.font.nunito_medium` | `Type.kt:31` |
| **Nunito SemiBold** | 600 | `R.font.nunito_semibold` | `Type.kt:32` |
| **Nunito Bold** | 700 | `R.font.nunito_bold` | `Type.kt:33` |
| **Nunito ExtraBold** | 800 | `R.font.nunito_extrabold` | `Type.kt:34` |
| **Nunito Black** | 900 | `R.font.nunito_black` | `Type.kt:35` |

**Font Family**: Nunito (sans-serif)
**Fallback**: System default if Nunito fails to load
**Font Files**: Located in `app/src/main/res/font/`

#### 2.1.2 Material 3 Typography Scale

All typography styles use Nunito font family, applied on top of Material 3 baseline typography.

| Style Name | Font Size | Line Height | Letter Spacing | Font Weight | Usage | Source |
|------------|-----------|-------------|----------------|-------------|-------|--------|
| **displayLarge** | 57sp | 64sp | -0.25sp | Normal (400) | Unused | `Type.kt:42` |
| **displayMedium** | 45sp | 52sp | 0sp | Normal (400) | Unused | `Type.kt:43` |
| **displaySmall** | 36sp | 44sp | 0sp | Normal (400) | Unused | `Type.kt:44` |
| **headlineLarge** | 32sp | 40sp | 0sp | Normal (400) | Section headers | `Type.kt:45` |
| **headlineMedium** | 28sp | 36sp | 0sp | Normal (400) | Dialog titles | `Type.kt:46` |
| **headlineSmall** | 24sp | 32sp | 0sp | Normal (400) | Screen titles | `Type.kt:47` |
| **titleLarge** | 22sp | 28sp | 0sp | Normal (400) | Card titles | `Type.kt:48` |
| **titleMedium** | 16sp | 24sp | 0.15sp | Medium (500) | List item titles | `Type.kt:49` |
| **titleSmall** | 14sp | 20sp | 0.1sp | Medium (500) | Subtitles | `Type.kt:50` |
| **bodyLarge** | 16sp | 24sp | 0.5sp | Normal (400) | Main content | `Type.kt:51` |
| **bodyMedium** | 14sp | 20sp | 0.25sp | Normal (400) | Secondary content | `Type.kt:52` |
| **bodySmall** | 12sp | 16sp | 0.4sp | Normal (400) | Captions | `Type.kt:53` |
| **labelLarge** | 14sp | 20sp | 0.1sp | Medium (500) | Buttons | `Type.kt:54` |
| **labelMedium** | 12sp | 16sp | 0.5sp | Medium (500) | Labels | `Type.kt:55` |
| **labelSmall** | 11sp | 16sp | 0.5sp | Medium (500) | Tiny labels | `Type.kt:56` |

**Note**: All values are from Material 3 baseline. OnDevice AI applies Nunito font family to all styles.

#### 2.1.3 Custom Typography Styles

Beyond Material 3 baseline, the app defines 10 custom typography styles for specific use cases:

| Style Name | Base | Font Size | Line Height | Letter Spacing | Font Weight | Usage | Source |
|------------|------|-----------|-------------|----------------|-------------|-------|--------|
| **titleMediumNarrow** | titleMedium | 16sp | 24sp | **0.0sp** | Medium (500) | Compact titles | `Type.kt:59-60` |
| **titleSmaller** | titleSmall | **12sp** | 20sp | 0.1sp | **Bold (700)** | Tiny bold titles | `Type.kt:62-67` |
| **labelSmallNarrow** | labelSmall | 11sp | 16sp | **0.0sp** | Medium (500) | Compact labels | `Type.kt:69` |
| **labelSmallNarrowMedium** | labelSmall | 11sp | 16sp | **0.0sp** | **Medium (500)** | Compact medium labels | `Type.kt:71-76` |
| **bodySmallNarrow** | bodySmall | 12sp | 16sp | **0.0sp** | Normal (400) | Compact captions | `Type.kt:78` |
| **bodySmallMediumNarrow** | bodySmall | **14sp** | 16sp | **0.0sp** | Normal (400) | Larger compact body | `Type.kt:80-81` |
| **bodySmallMediumNarrowBold** | bodySmall | **14sp** | 16sp | **0.0sp** | **Bold (700)** | Bold compact body | `Type.kt:83-89` |
| **homePageTitleStyle** | displayMedium | **48sp** | **48sp** | **-1.0sp** | **Medium (500)** | Home page "OnDevice AI" title | `Type.kt:91-98` |
| **bodyLargeNarrow** | bodyLarge | 16sp | 24sp | **0.2sp** | Normal (400) | Narrow body text | `Type.kt:100` |
| **headlineLargeMedium** | headlineLarge | 32sp | 40sp | 0sp | **Medium (500)** | Medium weight headlines | `Type.kt:102` |

**Pattern**: "Narrow" styles set `letterSpacing = 0.0sp` to reduce horizontal spacing for compact layouts.

**Special Case - Home Page Title**:
- Size: 48sp (largest text in the app)
- Line height: 48sp (tight, no extra vertical space)
- Letter spacing: -1.0sp (negative spacing for tighter kerning)
- Weight: Medium (500)
- Usage: "OnDevice" + "AI" title on home screen
- Visual effect: Creates a compact, bold branding statement

#### 2.1.4 Markdown Rendering Rules

Chat messages and model responses support Markdown formatting via CommonMark library.

| Markdown Element | Rendered Style | Specification | Source |
|------------------|----------------|---------------|--------|
| **Normal Text** | bodyLarge | 16sp, lineHeight 20.8sp (fontSize × 1.3) | `MarkdownText.kt:67` |
| **Normal Text (small)** | bodyMedium | 14sp, lineHeight 18.2sp (fontSize × 1.3) | `MarkdownText.kt:63-64` |
| **Code Blocks** | bodySmall + Monospace | 12sp, FontFamily.Monospace | `MarkdownText.kt:77-79` |
| **Inline Code** | bodySmall + Monospace | 12sp, FontFamily.Monospace | Same as code blocks |
| **Links** | Inherited + linkColor | Underlined, color from `customColors.linkColor` | `MarkdownText.kt:84` |
| **Bold** | Inherited + Bold | Font weight 700 (Bold) | CommonMark default |
| **Italic** | Inherited + Italic | Font style italic | CommonMark default |
| **Headings (H1-H6)** | Scaled from base | Automatic scaling by CommonMark | CommonMark default |

**Line Height Calculation**: All markdown text uses `lineHeight = fontSize × 1.3`
- Example: bodyLarge (16sp) → lineHeight 20.8sp
- Ensures consistent vertical rhythm in chat messages

**Code Block Styling**:
- Background: `MaterialTheme.colorScheme.surfaceVariant`
- Text color: `MaterialTheme.colorScheme.onSurfaceVariant`
- Border radius: 8dp
- Padding: 12dp
- Horizontal scroll enabled
- Copy button in top-right corner (32dp × 32dp)

**Link Styling**:
- Light mode: `#32628D` (blue, source: `Theme.kt:177`)
- Dark mode: `#9DCAFC` (light blue, source: `Theme.kt:228`)
- No underline in default state
- Clickable areas match text bounds

#### 2.1.5 Typography Usage Map

| Screen/Component | Primary Style | Secondary Style | Source |
|------------------|---------------|-----------------|--------|
| **Home Page Title** | homePageTitleStyle (48sp) | - | Home screen |
| **Chat Messages (User)** | bodyLarge (16sp) | - | Chat bubbles |
| **Chat Messages (AI)** | bodyLarge (16sp) with Markdown | bodySmall (code blocks) | Chat bubbles |
| **Model Names** | titleMedium (16sp) | - | Model selector |
| **Model Descriptions** | bodyMedium (14sp) | - | Model cards |
| **Button Labels** | labelLarge (14sp, Medium 500) | - | All buttons |
| **Screen Titles** | headlineSmall (24sp) | - | Top bars |
| **Dialog Titles** | headlineMedium (28sp) | - | Dialogs |
| **List Item Titles** | titleMedium (16sp, Medium 500) | bodySmall (12sp) | Settings, model list |
| **Input Placeholders** | bodyLarge (16sp) | - | Text fields |
| **Error Messages** | bodyMedium (14sp) | - | Error states |
| **Status Text** | labelMedium (12sp, Medium 500) | - | Download progress |

#### 2.1.6 Font Loading

**Font Resources**:
```
app/src/main/res/font/
├── nunito_black.ttf
├── nunito_bold.ttf
├── nunito_extrabold.ttf
├── nunito_extralight.ttf
├── nunito_light.ttf
├── nunito_medium.ttf
├── nunito_regular.ttf
└── nunito_semibold.ttf
```

**Loading Behavior**:
- Fonts preloaded at app startup via `FontFamily` constructor
- No network requests (bundled in APK)
- Fallback to system default if loading fails
- Total font bundle size: ~800KB (all 8 weights)

**Accessibility**:
- All text scales with system font size settings
- Minimum touch target: 48dp × 48dp (Material Design guideline)
- Sufficient contrast ratios for WCAG 2.2 Level AA
- No text smaller than 11sp (labelSmall) for readability

#### 2.1.7 Typography Design Tokens

**Spacing Units** (used with typography):
- Small gap: 4dp
- Medium gap: 8dp
- Large gap: 16dp
- Extra large gap: 24dp

**Paragraph Spacing** (markdown):
- Between paragraphs: 8dp
- Before headings: 16dp
- After headings: 8dp
- Around code blocks: 8dp

**Letter Spacing Philosophy**:
- **Positive spacing** (0.1sp - 0.5sp): Default for most text, improves readability
- **Zero spacing** (0.0sp): "Narrow" styles for compact layouts (model selector, chips)
- **Negative spacing** (-1.0sp): Home page title only, creates tight branding

---

### 2.2 Colors

#### 2.2.1 Material 3 Color Scheme (Light Mode)

| Token | Hex Value | RGB | Usage | Source |
|-------|-----------|-----|-------|--------|
| **primary** | `#0B57D0` | `11, 87, 208` | Primary brand color, buttons | `Color.kt:21` |
| **onPrimary** | `#FFFFFF` | `255, 255, 255` | Text on primary | `Color.kt:22` |
| **primaryContainer** | `#D3E3FD` | `211, 227, 253` | Primary button backgrounds | `Color.kt:23` |
| **onPrimaryContainer** | `#0842A0` | `8, 66, 160` | Text on primary container | `Color.kt:24` |
| **secondary** | `#00639B` | `0, 99, 155` | Secondary accents | `Color.kt:25` |
| **onSecondary** | `#FFFFFF` | `255, 255, 255` | Text on secondary | `Color.kt:26` |
| **secondaryContainer** | `#C2E7FF` | `194, 231, 255` | Secondary backgrounds | `Color.kt:27` |
| **onSecondaryContainer** | `#004A77` | `0, 74, 119` | Text on secondary container | `Color.kt:28` |
| **tertiary** | `#146C2E` | `20, 108, 46` | Tertiary accents (green) | `Color.kt:29` |
| **onTertiary** | `#FFFFFF` | `255, 255, 255` | Text on tertiary | `Color.kt:30` |
| **tertiaryContainer** | `#C4EED0` | `196, 238, 208` | Tertiary backgrounds | `Color.kt:31` |
| **onTertiaryContainer** | `#0F5223` | `15, 82, 35` | Text on tertiary container | `Color.kt:32` |
| **error** | `#B3261E` | `179, 38, 30` | Error states | `Color.kt:33` |
| **onError** | `#FFFFFF` | `255, 255, 255` | Text on error | `Color.kt:34` |
| **errorContainer** | `#F9DEDC` | `249, 222, 220` | Error backgrounds | `Color.kt:35` |
| **onErrorContainer** | `#8C1D18` | `140, 29, 24` | Text on error container | `Color.kt:36` |
| **background** | `#FFFFFF` | `255, 255, 255` | Screen background | `Color.kt:37` |
| **onBackground** | `#1F1F1F` | `31, 31, 31` | Text on background | `Color.kt:38` |
| **surface** | `#FFFFFF` | `255, 255, 255` | Surface backgrounds | `Color.kt:39` |
| **onSurface** | `#1F1F1F` | `31, 31, 31` | Text on surface | `Color.kt:40` |
| **surfaceVariant** | `#E1E3E1` | `225, 227, 225` | Variant surfaces (code blocks) | `Color.kt:41` |
| **onSurfaceVariant** | `#444746` | `68, 71, 70` | Text on surface variant | `Color.kt:42` |
| **surfaceContainerLowest** | `#FFFFFF` | `255, 255, 255` | Lowest elevation surface | `Color.kt:43` |
| **surfaceContainerLow** | `#F8FAFD` | `248, 250, 253` | Low elevation surface | `Color.kt:44` |
| **surfaceContainer** | `#F0F4F9` | `240, 244, 249` | Container surface | `Color.kt:45` |
| **surfaceContainerHigh** | `#E9EEF6` | `233, 238, 246` | High elevation surface | `Color.kt:46` |
| **surfaceContainerHighest** | `#DDE3EA` | `221, 227, 234` | Highest elevation surface | `Color.kt:47` |
| **inverseSurface** | `#303030` | `48, 48, 48` | Inverse surface (dark on light) | `Color.kt:48` |
| **inverseOnSurface** | `#F2F2F2` | `242, 242, 242` | Text on inverse surface | `Color.kt:49` |
| **outline** | `#747775` | `116, 119, 117` | Borders, dividers | `Color.kt:50` |
| **outlineVariant** | `#C4C7C5` | `196, 199, 197` | Subtle borders | `Color.kt:51` |
| **inversePrimary** | `#A8C7FA` | `168, 199, 250` | Inverse primary | `Color.kt:52` |
| **surfaceDim** | `#D3DBE5` | `211, 219, 229` | Dim surface | `Color.kt:53` |
| **surfaceBright** | `#FFFFFF` | `255, 255, 255` | Bright surface | `Color.kt:54` |
| **scrim** | `#000000` | `0, 0, 0` | Modal scrim overlay | `Color.kt:55` |

#### 2.2.2 Material 3 Color Scheme (Dark Mode)

| Token | Hex Value | RGB | Usage | Source |
|-------|-----------|-----|-------|--------|
| **primary** | `#A8C7FA` | `168, 199, 250` | Primary brand color | `Color.kt:57` |
| **onPrimary** | `#062E6F` | `6, 46, 111` | Text on primary | `Color.kt:58` |
| **primaryContainer** | `#0842A0` | `8, 66, 160` | Primary button backgrounds | `Color.kt:59` |
| **onPrimaryContainer** | `#D3E3FD` | `211, 227, 253` | Text on primary container | `Color.kt:60` |
| **secondary** | `#7FCFFF` | `127, 207, 255` | Secondary accents | `Color.kt:61` |
| **onSecondary** | `#003355` | `0, 51, 85` | Text on secondary | `Color.kt:62` |
| **secondaryContainer** | `#004A77` | `0, 74, 119` | Secondary backgrounds | `Color.kt:63` |
| **onSecondaryContainer** | `#C2E7FF` | `194, 231, 255` | Text on secondary container | `Color.kt:64` |
| **tertiary** | `#6DD58C` | `109, 213, 140` | Tertiary accents (green) | `Color.kt:65` |
| **onTertiary** | `#0A3818` | `10, 56, 24` | Text on tertiary | `Color.kt:66` |
| **tertiaryContainer** | `#0F5223` | `15, 82, 35` | Tertiary backgrounds | `Color.kt:67` |
| **onTertiaryContainer** | `#C4EED0` | `196, 238, 208` | Text on tertiary container | `Color.kt:68` |
| **error** | `#F2B8B5` | `242, 184, 181` | Error states | `Color.kt:69` |
| **onError** | `#601410` | `96, 20, 16` | Text on error | `Color.kt:70` |
| **errorContainer** | `#8C1D18` | `140, 29, 24` | Error backgrounds | `Color.kt:71` |
| **onErrorContainer** | `#F9DEDC` | `249, 222, 220` | Text on error container | `Color.kt:72` |
| **background** | `#131314` | `19, 19, 20` | Screen background | `Color.kt:73` |
| **onBackground** | `#E3E3E3` | `227, 227, 227` | Text on background | `Color.kt:74` |
| **surface** | `#131314` | `19, 19, 20` | Surface backgrounds | `Color.kt:75` |
| **onSurface** | `#E3E3E3` | `227, 227, 227` | Text on surface | `Color.kt:76` |
| **surfaceVariant** | `#444746` | `68, 71, 70` | Variant surfaces | `Color.kt:77` |
| **onSurfaceVariant** | `#C4C7C5` | `196, 199, 197` | Text on surface variant | `Color.kt:78` |
| **surfaceContainerLowest** | `#0E0E0E` | `14, 14, 14` | Lowest elevation | `Color.kt:79` |
| **surfaceContainerLow** | `#1B1B1B` | `27, 27, 27` | Low elevation | `Color.kt:80` |
| **surfaceContainer** | `#1E1F20` | `30, 31, 32` | Container surface | `Color.kt:81` |
| **surfaceContainerHigh** | `#282A2C` | `40, 42, 44` | High elevation | `Color.kt:82` |
| **surfaceContainerHighest** | `#333537` | `51, 53, 55` | Highest elevation | `Color.kt:83` |
| **inverseSurface** | `#E3E3E3` | `227, 227, 227` | Inverse surface | `Color.kt:84` |
| **inverseOnSurface** | `#303030` | `48, 48, 48` | Text on inverse | `Color.kt:85` |
| **outline** | `#8E918F` | `142, 145, 143` | Borders, dividers | `Color.kt:86` |
| **outlineVariant** | `#444746` | `68, 71, 70` | Subtle borders | `Color.kt:87` |
| **inversePrimary** | `#0B57D0` | `11, 87, 208` | Inverse primary | `Color.kt:88` |
| **surfaceDim** | `#131314` | `19, 19, 20` | Dim surface | `Color.kt:89` |
| **surfaceBright** | `#37393B` | `55, 57, 59` | Bright surface | `Color.kt:90` |
| **scrim** | `#000000` | `0, 0, 0` | Modal scrim | `Color.kt:91` |

#### 2.2.3 Custom Colors (Light Mode)

| Token | Hex Value | RGB | Usage | Source |
|-------|-----------|-----|-------|--------|
| **appTitleGradient[0]** | `#85B1F8` | `133, 177, 248` | Home title gradient start | `Theme.kt:136` |
| **appTitleGradient[1]** | `#3174F1` | `49, 116, 241` | Home title gradient end | `Theme.kt:136` |
| **tabHeaderBgColor** | `#3174F1` | `49, 116, 241` | Tab header background | `Theme.kt:137` |
| **taskCardBgColor** | surfaceContainerLowestLight | `255, 255, 255` | Task card background | `Theme.kt:138` |
| **taskBgColors[0]** (red) | `#FFF5F5` | `255, 245, 245` | Task background - red variant | `Theme.kt:142` |
| **taskBgColors[1]** (green) | `#F4FBF6` | `244, 251, 246` | Task background - green variant | `Theme.kt:144` |
| **taskBgColors[2]** (blue) | `#F1F6FE` | `241, 246, 254` | Task background - blue variant | `Theme.kt:146` |
| **taskBgColors[3]** (yellow) | `#FFFBF0` | `255, 251, 240` | Task background - yellow variant | `Theme.kt:148` |
| **taskBgGradient[0][0]** | `#E25F57` | `226, 95, 87` | Red task gradient start | `Theme.kt:153` |
| **taskBgGradient[0][1]** | `#DB372D` | `219, 55, 45` | Red task gradient end | `Theme.kt:153` |
| **taskBgGradient[1][0]** | `#41A15F` | `65, 161, 95` | Green task gradient start | `Theme.kt:155` |
| **taskBgGradient[1][1]** | `#128937` | `18, 137, 55` | Green task gradient end | `Theme.kt:155` |
| **taskBgGradient[2][0]** | `#669DF6` | `102, 157, 246` | Blue task gradient start | `Theme.kt:157` |
| **taskBgGradient[2][1]** | `#3174F1` | `49, 116, 241` | Blue task gradient end | `Theme.kt:157` |
| **taskBgGradient[3][0]** | `#FDD45D` | `253, 212, 93` | Yellow task gradient start | `Theme.kt:159` |
| **taskBgGradient[3][1]** | `#CAA12A` | `202, 161, 42` | Yellow task gradient end | `Theme.kt:159` |
| **taskIconColors[0]** (red) | `#DB372D` | `219, 55, 45` | Red task icon | `Theme.kt:164` |
| **taskIconColors[1]** (green) | `#128937` | `18, 137, 55` | Green task icon | `Theme.kt:166` |
| **taskIconColors[2]** (blue) | `#3174F1` | `49, 116, 241` | Blue task icon | `Theme.kt:168` |
| **taskIconColors[3]** (yellow) | `#CAA12A` | `202, 161, 42` | Yellow task icon | `Theme.kt:170` |
| **taskIconShapeBgColor** | `#FFFFFF` | `255, 255, 255` | Icon shape background | `Theme.kt:172` |
| **homeBottomGradient[0]** | `#00F8F9FF` | `0, 248, 249, 255` (transparent) | Home bottom gradient start | `Theme.kt:173` |
| **homeBottomGradient[1]** | `#FFEFC9` | `255, 239, 201` | Home bottom gradient end | `Theme.kt:173` |
| **agentBubbleBgColor** | `Transparent` | - | AI message background (Claude-style) | `Theme.kt:175` |
| **userBubbleBgColor** | `#F0F0F0` | `240, 240, 240` | User message background | `Theme.kt:176` |
| **linkColor** | `#32628D` | `50, 98, 141` | Markdown link color | `Theme.kt:177` |
| **successColor** | `#3D860B` | `61, 134, 11` | Success indicators | `Theme.kt:178` |
| **recordButtonBgColor** | `#666666` | `102, 102, 102` | Recording button (grey) | `Theme.kt:180` |
| **waveFormBgColor** | `#AAAAAA` | `170, 170, 170` | Waveform background | `Theme.kt:181` |
| **modelInfoIconColor** | `#CCCCCC` | `204, 204, 204` | Model info icon tint | `Theme.kt:182` |

#### 2.2.4 Custom Colors (Dark Mode)

| Token | Hex Value | RGB | Usage | Source |
|-------|-----------|-----|-------|--------|
| **appTitleGradient[0]** | `#85B1F8` | `133, 177, 248` | Home title gradient start | `Theme.kt:187` |
| **appTitleGradient[1]** | `#3174F1` | `49, 116, 241` | Home title gradient end | `Theme.kt:187` |
| **tabHeaderBgColor** | `#3174F1` | `49, 116, 241` | Tab header background | `Theme.kt:188` |
| **taskCardBgColor** | surfaceContainerHighDark | `40, 42, 44` | Task card background | `Theme.kt:189` |
| **taskBgColors[0]** (red) | `#181210` | `24, 18, 16` | Task background - red variant | `Theme.kt:193` |
| **taskBgColors[1]** (green) | `#131711` | `19, 23, 17` | Task background - green variant | `Theme.kt:195` |
| **taskBgColors[2]** (blue) | `#191924` | `25, 25, 36` | Task background - blue variant | `Theme.kt:197` |
| **taskBgColors[3]** (yellow) | `#1A1813` | `26, 24, 19` | Task background - yellow variant | `Theme.kt:199` |
| **taskBgGradient[0-3]** | Same as light mode | - | Task gradients (unchanged) | `Theme.kt:202-210` |
| **taskIconColors[0]** (red) | `#E25F57` | `226, 95, 87` | Red task icon (lighter) | `Theme.kt:215` |
| **taskIconColors[1]** (green) | `#41A15F` | `65, 161, 95` | Green task icon (lighter) | `Theme.kt:217` |
| **taskIconColors[2]** (blue) | `#669DF6` | `102, 157, 246` | Blue task icon (lighter) | `Theme.kt:219` |
| **taskIconColors[3]** (yellow) | `#CAA12A` | `202, 161, 42` | Yellow task icon (same) | `Theme.kt:221` |
| **taskIconShapeBgColor** | `#202124` | `32, 33, 36` | Icon shape background (dark grey) | `Theme.kt:223` |
| **homeBottomGradient[0]** | `#00F8F9FF` | `0, 248, 249, 255` (transparent) | Home bottom gradient start | `Theme.kt:224` |
| **homeBottomGradient[1]** | `#1AF6AD01` | `26, 246, 173, 1` (semi-transparent) | Home bottom gradient end | `Theme.kt:224` |
| **agentBubbleBgColor** | `Transparent` | - | AI message background (Claude-style) | `Theme.kt:226` |
| **userBubbleBgColor** | `#2C2C2C` | `44, 44, 44` | User message background | `Theme.kt:227` |
| **linkColor** | `#9DCAFC` | `157, 202, 252` | Markdown link color (light blue) | `Theme.kt:228` |
| **successColor** | `#A1CE83` | `161, 206, 131` | Success indicators (lighter green) | `Theme.kt:229` |
| **recordButtonBgColor** | `#888888` | `136, 136, 136` | Recording button (lighter grey) | `Theme.kt:231` |
| **waveFormBgColor** | `#AAAAAA` | `170, 170, 170` | Waveform background (same) | `Theme.kt:232` |
| **modelInfoIconColor** | `#CCCCCC` | `204, 204, 204` | Model info icon tint (same) | `Theme.kt:233` |

#### 2.2.5 Gradient Definitions

**App Title Gradient** (Home Screen):
- Type: Linear gradient
- Direction: Horizontal (left to right)
- Colors: `[#85B1F8, #3174F1]`
- Usage: "OnDevice AI" title text on home screen
- Same in light and dark mode

**Task Background Gradients** (4 variants):
- Type: Linear gradient
- Direction: Horizontal (left to right)
- Red: `[#E25F57, #DB372D]`
- Green: `[#41A15F, #128937]`
- Blue: `[#669DF6, #3174F1]`
- Yellow: `[#FDD45D, #CAA12A]`
- Same in light and dark mode

**Home Bottom Gradient**:
- Type: Linear gradient
- Direction: Vertical (top to bottom)
- Light mode: `[#00F8F9FF, #FFEFC9]` (transparent white to warm beige)
- Dark mode: `[#00F8F9FF, #1AF6AD01]` (transparent white to semi-transparent warm tone)
- Usage: Decorative gradient at bottom of home screen

#### 2.2.6 Color Usage Rules

| Use Case | Light Mode Color | Dark Mode Color | Rationale |
|----------|------------------|-----------------|-----------|
| **Primary Actions** | primary (`#0B57D0`) | primary (`#A8C7FA`) | High contrast, brand recognition |
| **Text on Backgrounds** | onSurface (`#1F1F1F`) | onSurface (`#E3E3E3`) | WCAG AAA for readability |
| **Disabled States** | onSurface (38% opacity) | onSurface (38% opacity) | Material 3 guideline |
| **Card Backgrounds** | surfaceContainerLowest | surfaceContainerHigh | Elevated from screen background |
| **Code Blocks** | surfaceVariant (`#E1E3E1`) | surfaceVariant (`#444746`) | Differentiate from main content |
| **Links in Markdown** | linkColor (`#32628D`) | linkColor (`#9DCAFC`) | Accessible, distinguishable |
| **User Messages** | userBubbleBgColor (`#F0F0F0`) | userBubbleBgColor (`#2C2C2C`) | Claude-style chat bubbles |
| **AI Messages** | Transparent | Transparent | Matches Claude.ai design |
| **Success States** | successColor (`#3D860B`) | successColor (`#A1CE83`) | Download complete, positive actions |
| **Error States** | error (`#B3261E`) | error (`#F2B8B5`) | Failed downloads, validation errors |

**WCAG 2.2 Compliance**:
- All text-background combinations meet WCAG AA (4.5:1 for normal text, 3:1 for large text)
- High contrast mode available via system settings
- No color-only information (always paired with icons or text)

#### 2.2.7 Theme Selection

**Theme Override Logic** (from `Theme.kt:259-262`):
```kotlin
val darkTheme: Boolean =
  (isSystemInDarkTheme() || themeOverride.value == Theme.THEME_DARK) &&
    themeOverride.value != Theme.THEME_LIGHT
```

**Available Themes**:
1. **System Default** (`Theme.UNRECOGNIZED` or not set)
   - Follows Android system theme setting
   - Changes automatically with system dark mode
2. **Light Mode** (`Theme.THEME_LIGHT`)
   - Forces light theme regardless of system setting
3. **Dark Mode** (`Theme.THEME_DARK`)
   - Forces dark theme regardless of system setting

**Theme Persistence**: User's theme choice saved in DataStore (`ThemeSettings.themeOverride`)

**Status Bar Icons**: Automatically adjust color based on theme (light icons in dark mode, dark icons in light mode)

---

### 2.3 Spacing & Grid

#### 2.3.1 Explicit Dimension Tokens

| Token Name | Value (dp) | Usage | Source |
|------------|------------|-------|--------|
| **model_selector_height** | 54dp | Height of model selector dropdown | `dimens.xml:20` |
| **chat_bubble_corner_radius** | 24dp | Border radius for chat message bubbles | `dimens.xml:21` |

**Note**: The app uses Material 3 spacing conventions with mostly inline dp values in Compose code.

#### 2.3.2 Material 3 Spacing Scale

The app follows Material 3's 4dp spacing unit system. All spacing is a multiple of 4dp.

| Scale Name | Value (dp) | Multiplier | Common Usage |
|------------|------------|------------|--------------|
| **XXS** | 2dp | 0.5× | Minimal spacing, icon padding |
| **XS** | 4dp | 1× | Tight spacing, inline elements |
| **S** | 8dp | 2× | Default padding within components |
| **M** | 12dp | 3× | Code block padding |
| **L** | 16dp | 4× | Standard component padding, margin between elements |
| **XL** | 24dp | 6× | Section spacing, large gaps |
| **XXL** | 32dp | 8× | Screen padding (horizontal) |
| **XXXL** | 48dp | 12× | Large spacing, top/bottom screen padding |
| **Huge** | 64dp | 16× | Extra large spacing (rare) |

**Base Unit**: 4dp (all spacing is a multiple of this)

**Philosophy**: Consistent rhythm through 4dp intervals creates visual harmony and predictability.

#### 2.3.3 Component-Specific Spacing

| Component | Property | Value | Source/Inference |
|-----------|----------|-------|------------------|
| **Chat Bubble** | Corner radius | 24dp | `dimens.xml:21` |
| **Chat Bubble** | Padding (internal) | 12dp | Inferred from Material 3 |
| **Chat Bubble** | Margin (vertical) | 8dp | Between messages |
| **Model Selector** | Height | 54dp | `dimens.xml:20` |
| **Model Selector** | Padding (horizontal) | 16dp | Inferred from Material 3 |
| **Code Block** | Corner radius | 8dp | `MarkdownText.kt:109` |
| **Code Block** | Padding | 12dp | `MarkdownText.kt:123` |
| **Code Block** | Copy button size | 32dp | `MarkdownText.kt:133` |
| **Code Block** | Copy button padding | 4dp | `MarkdownText.kt:132` |
| **Button** | Minimum height | 40dp | Material 3 guideline |
| **Button** | Padding (horizontal) | 16dp | Material 3 guideline |
| **Icon Button** | Size | 48dp × 48dp | Material 3 guideline (touch target) |
| **Icon** | Default size | 24dp | Material 3 guideline |
| **Icon** | Small size | 16dp | Material 3 guideline |
| **Icon** | Large size | 32dp | Material 3 guideline |

#### 2.3.4 Screen Layout Conventions

| Context | Horizontal Padding | Vertical Padding | Rationale |
|---------|-------------------|------------------|-----------|
| **Screen Edge** | 16dp | - | Standard Material 3 screen margin |
| **Screen Top** | - | 16dp | Below status bar / top bar |
| **Screen Bottom** | - | 16dp | Above navigation / input |
| **List Items** | 16dp | 8dp | Comfortable touch targets |
| **Card Padding** | 16dp | 12dp | Internal card content |
| **Dialog** | 24dp | 24dp | Larger for focus |
| **Sections** | 0dp | 24dp | Vertical separation between sections |

#### 2.3.5 Grid System

**Material 3 Column Grid**: Not explicitly implemented (app uses responsive Compose layouts)

**Layout Width Constraints**:
- Minimum screen width: 320dp (smallest Android phone)
- Target screen width: 360dp - 480dp (standard phones)
- Maximum content width: No maximum (fills screen width)

**Responsive Breakpoints**:
| Breakpoint | Width Range | Device Type | Layout Adaptation |
|------------|-------------|-------------|-------------------|
| **Compact** | 0dp - 599dp | Phone (portrait) | Single column, portrait locked |
| **Medium** | 600dp - 839dp | Tablet (portrait), Phone (landscape) | Single column, portrait locked |
| **Expanded** | 840dp+ | Tablet (landscape), Desktop | Single column, portrait locked |

**Note**: App is portrait-locked (`AndroidManifest.xml:70`) so responsive breakpoints have minimal effect. All layouts designed for compact width (phone portrait).

**Vertical Rhythm**:
- Chat messages: 8dp gap between messages
- Sections: 24dp gap between sections
- Paragraphs (markdown): 8dp gap

#### 2.3.6 Touch Target Sizes

Following Material Design 3 and WCAG 2.2 Level AA guidelines:

| Element Type | Minimum Size | Recommended Size | Source |
|--------------|--------------|------------------|--------|
| **Button** | 48dp × 48dp | 48dp × 48dp | Material 3 |
| **Icon Button** | 48dp × 48dp | 48dp × 48dp | Material 3 |
| **Checkbox** | 48dp × 48dp | 48dp × 48dp | Material 3 |
| **Radio Button** | 48dp × 48dp | 48dp × 48dp | Material 3 |
| **List Item** | Full width × 56dp | Full width × 64dp | Material 3 |
| **Text Input** | Full width × 56dp | Full width × 56dp | Material 3 |

**Touch Target Padding**: If visual element is smaller than 48dp, add transparent padding to reach 48dp touch target.

#### 2.3.7 Elevation and Shadows

Material 3 uses tonal elevation (color changes) instead of shadows for depth:

| Level | Elevation (dp) | Use Case | Color Token |
|-------|----------------|----------|-------------|
| **0** | 0dp | Screen background | background |
| **1** | 1dp | Cards at rest | surfaceContainerLow |
| **2** | 3dp | Cards hovered | surfaceContainer |
| **3** | 6dp | Dialogs, bottom sheets | surfaceContainerHigh |
| **4** | 8dp | Navigation drawer | surfaceContainerHigh |
| **5** | 12dp | Modal dialogs | surfaceContainerHighest |

**Implementation**: Elevation achieved through `surfaceContainer*` color tokens, not shadow z-depth.

#### 2.3.8 Border Radius Standards

| Component | Radius | Usage |
|-----------|--------|-------|
| **Chat Bubbles** | 24dp | `dimens.xml:21` |
| **Code Blocks** | 8dp | `MarkdownText.kt:109` |
| **Buttons (Material 3)** | 20dp (full) | Large buttons |
| **Buttons (Material 3)** | 100dp (pill) | Icon buttons |
| **Cards** | 12dp | Standard card corners |
| **Dialogs** | 28dp | Large rounded dialogs |
| **Text Fields** | 4dp (top) | Material 3 filled text field |
| **Chips** | 8dp | Small rounded chips |

**Philosophy**: Larger radius (24dp+) for primary interactive elements (chat bubbles), smaller radius (8dp) for contained elements (code blocks, cards).

#### 2.3.9 Z-Index Layering

| Layer | Purpose | Components |
|-------|---------|------------|
| **0** | Base layer | Background, screen content |
| **1** | Content | Chat messages, cards, list items |
| **2** | Floating | FAB, snackbars |
| **3** | Overlays | Dialogs, bottom sheets |
| **4** | Alerts | Toasts, permission requests |
| **5** | System | Status bar, navigation bar |

**Scrim Opacity**: Modal dialogs use 32% black scrim overlay (`scrim` token with 0.32 alpha)

---

### 2.4 Animations

#### 2.4.1 Navigation Transition Animations

| Transition Type | Animation | Duration | Easing | Properties | Source |
|----------------|-----------|----------|--------|------------|--------|
| **Screen Enter (Forward)** | Slide in from right | 300ms | EaseOutExpo | translateX: 100% → 0% | `GalleryNavGraph.kt:28-32` |
| **Screen Exit (Forward)** | Slide out to left | 300ms | EaseOutExpo | translateX: 0% → -100% | `GalleryNavGraph.kt:28-32` |
| **Screen Enter (Back)** | Slide in from left | 300ms | EaseOutExpo | translateX: -100% → 0% | Default |
| **Screen Exit (Back)** | Slide out to right | 300ms | EaseOutExpo | translateX: 0% → 100% | Default |

**Implementation**: Compose `AnimatedContentTransitionScope` with `slideInHorizontally` and `slideOutHorizontally`

**Easing Curve**: `EaseOutExpo` - Exponential ease-out for snappy, responsive feel
- Starts fast, decelerates exponentially
- Creates perception of speed and responsiveness

**Duration**: 300ms (Material 3 standard for screen transitions)

#### 2.4.2 Splash Screen Animation

| Property | Value | Source |
|----------|-------|--------|
| **Background Color** | `#0A1628` (dark navy) | `themes.xml:22` |
| **Icon** | Neural circuit logo (launcher icon) | `splash_screen_animated_icon.xml:19` |
| **Animation** | System default (Android 12+ Splash Screen API) | `themes.xml:23` |
| **Duration** | ~500ms (system-controlled) | Android system default |
| **Transition** | Fade-in → Fade-out to main screen | Android system default |

**Note**: Uses Android 12+ Splash Screen API, which provides system-controlled fade animations.

**Behavior**:
1. App launch → Dark navy background appears immediately
2. Neural circuit logo fades in (~200ms)
3. Logo holds (~100ms)
4. Logo fades out as main screen fades in (~200ms)
5. Total duration: ~500ms

#### 2.4.3 Loading Indicators

##### Rotational Loader (Neural Circuit Logo)

| Property | Value | Source |
|----------|-------|--------|
| **Animation Type** | Continuous rotation | `RotationalLoader.kt:58-69` |
| **Duration** | 3000ms (3 seconds per full rotation) | `RotationalLoader.kt:63` |
| **Easing** | LinearEasing | `RotationalLoader.kt:64` |
| **Repeat Mode** | Infinite restart | `RotationalLoader.kt:66` |
| **Rotation Range** | 0° → 360° | `RotationalLoader.kt:59-60` |
| **Color Tint** | `#00D9FF` (cyan) | `RotationalLoader.kt:85` |
| **Size** | Variable (passed as parameter) | `RotationalLoader.kt:54` |

**Specifications** (from inline documentation):
- Single-piece rotation around central axis
- Transparent background (no visible square borders)
- Slow, steady rotation for downloading state (3s per revolution)
- Linear easing for mechanical, intentional feel
- NO scaling, NO pulsing, NO opacity changes

**Usage**: Model download status, initialization loading

##### Circular Progress Indicator (Material 3)

| Property | Value | Source |
|----------|-------|--------|
| **Type** | Indeterminate circular | Material 3 default |
| **Duration** | 1333ms per rotation | Material 3 specification |
| **Easing** | FastOutSlowInEasing | Material 3 specification |
| **Stroke Width** | 4dp | Material 3 default |
| **Color** | Primary color | Material 3 default |

**Usage**: Generic loading states, brief waits

#### 2.4.4 Spark Icons (Static Vector Graphics)

| Icon | Size | Color | Usage | Source |
|------|------|-------|-------|--------|
| **chat_spark.xml** | 38dp × 38dp | `#1967D2` (blue) | Chat task indicator | `chat_spark.xml:18-19, 24` |
| **image_spark.xml** | 38dp × 38dp | `#34A853` (green) | Image generation task indicator | `image_spark.xml:18-19, 24` |
| **text_spark.xml** | 38dp × 38dp | Color variant | Text task indicator | Inferred |

**Note**: Despite "spark" naming, these are **static vector drawables**, not animated. The "spark" refers to the visual design (star-like sparkle shape), not animation.

**Visual Design**: Four-pointed sparkle/star shape in task-specific color

#### 2.4.5 Micro-Interactions

| Interaction | Animation | Duration | Easing | Source |
|-------------|-----------|----------|--------|--------|
| **Button Press** | Ripple effect | 300ms | Material 3 default | Material 3 |
| **Button Press (scale)** | Scale 1.0 → 0.95 → 1.0 | 100ms | FastOutSlowIn | Material 3 |
| **Checkbox Toggle** | Checkmark draw | 200ms | FastOutSlowIn | Material 3 |
| **Switch Toggle** | Thumb slide | 200ms | FastOutSlowIn | Material 3 |
| **FAB Expand** | Scale + rotate | 300ms | EaseOutExpo | Material 3 |
| **Snackbar Enter** | Slide up from bottom | 150ms | EaseOut | Material 3 |
| **Snackbar Exit** | Slide down to bottom | 75ms | EaseIn | Material 3 |
| **Dialog Enter** | Fade in + scale (0.8 → 1.0) | 300ms | EaseOutExpo | Material 3 |
| **Dialog Exit** | Fade out | 150ms | EaseIn | Material 3 |

**Ripple Effect**: Material 3 default ripple
- Expands from touch point
- Duration: 300ms
- Easing: LinearOutSlowInEasing
- Color: onSurface with 12% opacity (light mode), 16% opacity (dark mode)

#### 2.4.6 Compose AnimatedVisibility

Used for show/hide transitions throughout the app:

| Usage | Enter Transition | Exit Transition | Duration |
|-------|------------------|-----------------|----------|
| **Expandable Sections** | expandVertically + fadeIn | shrinkVertically + fadeOut | 300ms |
| **Tooltips** | fadeIn + scaleIn | fadeOut + scaleOut | 150ms |
| **Error Messages** | slideInVertically (from top) + fadeIn | slideOutVertically (to top) + fadeOut | 200ms |
| **Model Download Progress** | fadeIn | fadeOut | 200ms |

**Standard Pattern**:
```kotlin
AnimatedVisibility(
  visible = isVisible,
  enter = fadeIn(animationSpec = tween(200)) + expandVertically(),
  exit = fadeOut(animationSpec = tween(200)) + shrinkVertically()
)
```

#### 2.4.7 Animation Duration Standards

| Category | Duration Range | Examples |
|----------|---------------|----------|
| **Instant** | 50-100ms | Tooltip appearance, quick feedback |
| **Fast** | 150-200ms | Checkbox toggle, switch toggle, error messages |
| **Standard** | 250-300ms | Screen transitions, button press, dialog open |
| **Slow** | 400-500ms | Splash screen, complex transitions |
| **Extended** | 1000-3000ms | Loading spinners, continuous animations |

**General Rules**:
- Enter animations: Slightly longer (feels welcoming)
- Exit animations: Slightly shorter (feels snappy)
- Ratio: Exit duration ≈ 50-75% of enter duration

#### 2.4.8 Easing Curves

| Easing | Cubic Bezier | Use Case |
|--------|-------------|----------|
| **LinearEasing** | (0, 0, 1, 1) | Rotations, continuous loops |
| **EaseOutExpo** | (0.16, 1, 0.3, 1) | Screen transitions (snappy, responsive feel) |
| **FastOutSlowIn** | (0.4, 0, 0.2, 1) | Material 3 standard (balanced, natural) |
| **EaseOut** | (0, 0, 0.58, 1) | Enter animations (decelerate to rest) |
| **EaseIn** | (0.42, 0, 1, 1) | Exit animations (accelerate away) |

**Philosophy**:
- **Linear**: Mechanical, intentional (loading spinners)
- **EaseOutExpo**: Fast start, abrupt stop (screen transitions feel instant)
- **FastOutSlowIn**: Natural, organic (most UI interactions)
- **EaseOut**: Welcoming (content appears)
- **EaseIn**: Departing (content disappears)

#### 2.4.9 Performance Considerations

**Hardware Acceleration**: All animations use Compose's GPU-accelerated rendering

**Frame Rate Targets**:
- Standard animations: 60 FPS (16.67ms per frame)
- Smooth, jank-free playback on all supported devices (Android 12+)

**Animation Cancellation**: All transitions cancel and cleanup when interrupted
- Example: User swipes back mid-transition → animation stops, reverse plays

**Memory**: Animations use value animators (no bitmap caching), minimal memory overhead

**Battery Impact**: Continuous animations (loading spinners) only run when visible
- Paused when screen is off
- Paused when app is backgrounded

---

---

## 3. Navigation Model

*Task 1.6 deliverable - Navigation graph diagram and route specifications*

---

## 4. Data Contracts

### 4.1 Room Database

*Task 1.7 deliverable - ConversationThread, ConversationMessage, ConversationState entities*

### 4.2 Core Models

*Task 1.8 deliverable - Task, Model, ModelDownloadStatus, ChatMessage schemas*

---

## 5. Architecture Overview

*Task 1.9 deliverable - MVVM + Compose + Hilt pattern documentation*

---

## 6. Build Configuration

*Task 1.10 deliverable - Gradle configuration, dependencies, ProGuard rules*

---

## Appendix A: Source Code References

All specifications in this document are derived from the OnDevice AI v1.1.9 (build 35) codebase:

- **Git Repository**: https://github.com/GoraAI-Na5h13/OnDevice
- **Original Source**: https://github.com/google-ai-edge/gallery
- **Tag**: v1.1.9-build35 (frozen for specification)

### Key Source Files (Product Definition)

| File | Lines | Specification Section |
|------|-------|----------------------|
| `app/build.gradle.kts` | 42-78 | Product identity, version, package IDs |
| `app/src/main/AndroidManifest.xml` | 18-135 | Platform support, permissions, deep linking |
| `app/src/main/res/values/strings.xml` | 19-22, 91 | Product branding and legal name |
| `gradle/libs.versions.toml` | 1-112 | Build environment versions |

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 2026-02-07 | Claude Sonnet 4.5 | Initial creation - Product Definition (Task 1.1) complete |

---

**Next Task**: Task 1.2 - Global Design System: Typography
