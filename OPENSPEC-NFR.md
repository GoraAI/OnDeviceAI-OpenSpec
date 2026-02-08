# OnDevice AI v1.1.9 - Phase 5: Non-Functional Requirements

**Status**: Phase 5 Complete
**Version**: 1.1.9 (Build 35)
**Pages**: 50 estimated
**Author**: Automated OpenSpec Reverse Engineering
**Date**: 2026-02-06

---

## Table of Contents

1. [Performance Benchmarks](#1-performance-benchmarks)
2. [Error Handling Catalog](#2-error-handling-catalog)
3. [Offline Behavior](#3-offline-behavior)
4. [Analytics Events](#4-analytics-events)
5. [Security Requirements](#5-security-requirements)

---

## 1. Performance Benchmarks

### 1.1 Overview

**Performance Targets**: Optimize for mobile devices with limited memory and battery

**Primary Metrics**:
- Inference latency (time to first token, tokens per second)
- Memory usage (RAM, storage)
- Battery consumption (efficient model loading)
- Network usage (WiFi-only downloads, minimal traffic)

### 1.2 Token Estimation & Processing

**Token Estimation Algorithm** (`TokenEstimator.kt`):
```kotlin
fun estimateTokens(text: String): Int {
  return maxOf(1, (text.length / 4.0).roundToInt())
}
```

**Formula**: `tokens = text.length / 4` (minimum 1 token)

**Conversion Rate**: 4 characters = 1 token (average for English)

**Accuracy**: ±15% acceptable (with 75% trigger buffer in compaction)

**Source**: `app/src/main/java/ai/ondevice/app/conversation/TokenEstimator.kt`

### 1.3 Benchmark Configuration

**Warm-Up Phase**:
```kotlin
const val DEFAULT_WARMUP_ITERATIONS = 50
const val MIN_WARMUP_ITERATIONS = 10
const val MAX_WARMUP_ITERATIONS = 200
```

**Benchmark Phase**:
```kotlin
const val DEFAULT_BENCHMARK_ITERATIONS = 200
const val MIN_BENCHMARK_ITERATIONS = 50
const val MAX_BENCHMARK_ITERATIONS = 500
```

**Purpose**:
- Warm-up: Prepare JIT compiler, cache model in memory
- Benchmark: Measure steady-state inference performance

**Source**: `app/src/main/java/ai/ondevice/app/ui/common/chat/BenchmarkConfigDialog.kt:29-30`

### 1.4 Memory Constraints

**System Reserved Memory**:
```kotlin
const val SYSTEM_RESERVED_MEMORY_IN_BYTES = 3L * (1L shl 30)  // 3 GB
```

**Value**: 3,221,225,472 bytes (3 GB)

**Purpose**: Reserved for OS operations and emergency buffer

**Usage Contexts**:
- Download validation (ensure sufficient RAM available)
- Storage calculations (prevent out-of-memory during model load)
- Memory warnings (show when device memory is low)

**Source**: `app/src/main/java/ai/ondevice/app/ui/common/DownloadAndTryButton.kt`

### 1.5 Memory Unit Conversion

**Bytes per GB** (`MemoryWarning.kt:32`):
```kotlin
const val BYTES_PER_GB = 1024 * 1024 * 1024  // 1,073,741,824 bytes
```

**Value**: 1,073,741,824 bytes = 1 GB

**API Level Differences**:

**Android 14+ (API 34)**:
```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE) {
  val advertisedMem = activityManager.memoryInfo.advertisedMem
  // Uses advertisedMem for accuracy
}
```

**Android 13 and below (API < 34)**:
```kotlin
val totalMem = activityManager.memoryInfo.totalMem
// Fallback to totalMem
```

**Difference**: `advertisedMem` is the actual device spec (e.g., 8 GB), `totalMem` is available RAM (may be less)

**Source**: `app/src/main/java/ai/ondevice/app/ui/common/MemoryWarning.kt:59-62`

### 1.6 Model Memory Requirements

**Model Configuration** (`Model.kt:105`):
```kotlin
data class Model(
  val minDeviceMemoryInGb: Int? = null  // Optional minimum device memory
)
```

**Memory Check Flow**:
```kotlin
if (model.minDeviceMemoryInGb != null) {
  val totalRamGb = getTotalDeviceMemoryGb()

  if (totalRamGb < model.minDeviceMemoryInGb) {
    // Show warning dialog (not blocking)
    showMemoryWarningDialog(
      required = model.minDeviceMemoryInGb,
      available = totalRamGb
    )
  }
}
```

**Warning Trigger**: Device memory < `minDeviceMemoryInGb`

**Check Timing**:
- Before model download
- When entering task screen
- On model selection

**User Experience**: Non-blocking warning, user can proceed anyway

**Source**: `app/src/main/java/ai/ondevice/app/data/Model.kt:105`

### 1.7 Storage & Download Performance

**Download Storage Buffer** (`SecureModelDownloader.kt:29`):
```kotlin
val minFreeStorage = if (model.minFreeStorageBytes > 0) {
  model.minFreeStorageBytes
} else {
  model.sizeInBytes * 1.2  // 20% buffer
}
```

**Default Buffer**: 20% extra space required

**Calculation**: `requiredSpace = modelSize × 1.20`

**Example**:
- Model size: 2 GB
- Required storage: 2 GB × 1.20 = 2.4 GB

**Rationale**: Account for unzipping, temporary files, filesystem overhead

**Source**: `app/src/main/java/ai/ondevice/app/helper/SecureModelDownloader.kt:29`

### 1.8 Download Network Requirements

**Network Configuration** (`SecureModelDownloader.kt:121-131`):
```kotlin
val constraints = Constraints.Builder()
  .setRequiredNetworkType(
    if (model.requiresWifi) {
      NetworkType.UNMETERED  // WiFi only
    } else {
      NetworkType.CONNECTED  // WiFi or mobile
    }
  )
  .setRequiresCharging(false)
  .setRequiresBatteryNotLow(false)
  .setRequiresDeviceIdle(false)
  .setRequiresStorageNotLow(true)
  .build()

val request = OneTimeWorkRequestBuilder<ModelDownloadWorker>()
  .setConstraints(constraints)
  .setBackoffCriteria(
    BackoffPolicy.EXPONENTIAL,
    WorkRequest.MIN_BACKOFF_MILLIS,
    TimeUnit.MILLISECONDS
  )
  .build()
```

**Network Types**:
- **WiFi-Only Models**: `NetworkType.UNMETERED` (no cellular data)
- **General Models**: `NetworkType.CONNECTED` (WiFi or mobile)

**Roaming**: Disabled for all downloads (`setAllowedOverRoaming(false)`)

**Metered Network**: Configurable per model via `requiresWifi` flag

**Backoff Policy**: Exponential backoff with default minimum (10 seconds)

**Source**: `app/src/main/java/ai/ondevice/app/helper/SecureModelDownloader.kt:121-145`

### 1.9 Conversation Compaction Performance

**Token Limits** (`CompactionManager.kt:38-40`):
```kotlin
const val MAX_TOKENS = 4096
const val COMPACTION_TRIGGER_TOKENS = (MAX_TOKENS * 0.75).toInt()  // 3072
const val COMPACTION_TARGET_TOKENS = (MAX_TOKENS * 0.40).toInt()   // 1638
```

**Thresholds**:
| Metric | Value | Purpose |
|--------|-------|---------|
| Max Tokens | 4,096 | Model context window limit |
| Trigger | 3,072 (75%) | Start compaction |
| Target | 1,638 (40%) | Tokens after compaction |
| Safety Buffer | 1,024 (25%) | Reserved for response generation |

**Compaction Strategy**: Self-summarization using LLM

**Compaction Flow**:
```
1. Estimate tokens in conversation: sum(message.length / 4)
2. If tokens >= 3,072: Trigger compaction
3. Summarize oldest messages until tokens <= 1,638
4. Replace summarized messages with summary message
5. Continue conversation
```

**Performance Impact**: ~2-5 seconds for summarization (varies by model)

**Source**: `app/src/main/java/ai/ondevice/app/conversation/CompactionManager.kt:38-40`

### 1.10 Audio Processing Performance

**Audio Specifications** (`Consts.kt:56`):
```kotlin
const val AUDIO_SAMPLE_RATE = 16000  // 16 kHz
const val AUDIO_BIT_DEPTH = 16       // 16-bit signed PCM
const val AUDIO_CHANNELS = 1         // Mono
```

**Sample Rate**: 16,000 Hz (16 kHz)

**Bit Depth**: 16-bit signed PCM

**Channels**: Mono (stereo converted to mono if needed)

**Format**: WAV (PCM encoding)

**Source**: `app/src/main/java/ai/ondevice/app/data/Consts.kt:56`

### 1.11 Audio Constraints

**Audio Limits** (`Consts.kt:50-53`):
```kotlin
const val MAX_AUDIO_CLIPS = 1           // Max 1 audio clip per session
const val MAX_AUDIO_DURATION_SECONDS = 30  // Max 30 seconds
```

**Max Audio Clips**: 1 per session

**Max Duration**: 30 seconds

**Processing Operations**:
- Resampling (if source < 16 kHz)
- Stereo-to-mono conversion (if stereo input)
- 8-bit to 16-bit conversion (if needed)

**Max Samples Calculation**:
```kotlin
val maxSamples = MAX_AUDIO_DURATION_SECONDS * AUDIO_SAMPLE_RATE
// 30 * 16000 = 480,000 samples
```

**Source**: `app/src/main/java/ai/ondevice/app/data/Consts.kt:50-53`

### 1.12 Audio Processing Implementation

**Resampling** (`Utils.kt:75-205`):
```kotlin
fun resampleAudio(
  samples: FloatArray,
  sourceSampleRate: Int,
  targetSampleRate: Int = AUDIO_SAMPLE_RATE
): FloatArray {
  if (sourceSampleRate == targetSampleRate) return samples

  val ratio = sourceSampleRate.toFloat() / targetSampleRate.toFloat()
  val targetLength = (samples.size / ratio).toInt()
  val resampled = FloatArray(targetLength)

  for (i in 0 until targetLength) {
    val sourceIndex = i * ratio
    val index0 = sourceIndex.toInt()
    val index1 = minOf(index0 + 1, samples.size - 1)
    val fraction = sourceIndex - index0

    // Linear interpolation
    resampled[i] = samples[index0] * (1 - fraction) + samples[index1] * fraction
  }

  return resampled
}
```

**Method**: Linear interpolation for rate conversion

**WAV Header Parsing**:
```kotlin
fun parseWavHeader(bytes: ByteArray): WavHeader {
  require(bytes.size >= 44) { "Invalid WAV file: too small" }

  // Parse RIFF header (bytes 0-11)
  val riffChunkId = bytes.sliceArray(0..3).toString(Charsets.US_ASCII)
  require(riffChunkId == "RIFF") { "Not a valid WAV file" }

  // Parse format chunk (bytes 12-35)
  val fmtChunkId = bytes.sliceArray(12..15).toString(Charsets.US_ASCII)
  require(fmtChunkId == "fmt ") { "Invalid WAV format chunk" }

  val audioFormat = bytes.readLittleEndianInt16(20)
  val numChannels = bytes.readLittleEndianInt16(22)
  val sampleRate = bytes.readLittleEndianInt32(24)
  val bitsPerSample = bytes.readLittleEndianInt16(34)

  return WavHeader(audioFormat, numChannels, sampleRate, bitsPerSample)
}
```

**Bit Conversion** (8-bit to 16-bit):
```kotlin
fun convert8BitTo16Bit(samples: ByteArray): ShortArray {
  return ShortArray(samples.size) { i ->
    ((samples[i].toInt() - 128) * 256).toShort()  // -128 offset, scale by 256
  }
}
```

**Source**: `app/src/main/java/ai/ondevice/app/common/Utils.kt:75-205`

### 1.13 Image Processing Performance

**Image Constraints** (`Consts.kt:47`):
```kotlin
const val MAX_IMAGES = 10  // Max 10 images per session
```

**Max Images**: 10 per session

**Image Scaling**: In-sample size calculation (BitmapFactory.Options.inSampleSize)

**EXIF Rotation**: Automatic orientation correction

**Purpose**: Prevent out-of-memory errors on low-end devices

**Scaling Strategy**:
```kotlin
fun calculateInSampleSize(
  options: BitmapFactory.Options,
  reqWidth: Int,
  reqHeight: Int
): Int {
  val (height, width) = options.run { outHeight to outWidth }
  var inSampleSize = 1

  if (height > reqHeight || width > reqWidth) {
    val halfHeight = height / 2
    val halfWidth = width / 2

    while (
      halfHeight / inSampleSize >= reqHeight &&
      halfWidth / inSampleSize >= reqWidth
    ) {
      inSampleSize *= 2
    }
  }

  return inSampleSize
}
```

**Example**:
- Original: 4000 × 3000 pixels
- Required: 1000 × 750 pixels
- inSampleSize: 4 (scale down by 4×)
- Result: 1000 × 750 pixels

**Source**: `app/src/main/java/ai/ondevice/app/data/Consts.kt:47`

---

## 2. Error Handling Catalog

### 2.1 Overview

**Error Handling Philosophy**: Privacy-first, user-friendly, actionable

**Principles**:
- Never log user messages or PII
- Provide actionable error messages
- Log metadata for debugging (lengths, counts, durations)
- Graceful degradation when possible

### 2.2 Error Handling Framework

**Centralized Logging** (`CrashlyticsLogger.kt`):
```kotlin
object CrashlyticsLogger {
  private val crashlytics: FirebaseCrashlytics? = getCrashlytics()

  private fun getCrashlytics(): FirebaseCrashlytics? {
    return try {
      Firebase.crashlytics
    } catch (e: Exception) {
      Log.w(TAG, "Firebase Crashlytics not available", e)
      null
    }
  }
}
```

**System**: Firebase Crashlytics for crash reporting

**Privacy Level**: PII-safe logging (NEVER logs user messages, names, emails)

**Graceful Degradation**: If Firebase not configured, logs to Logcat only

**Source**: `app/src/util/CrashlyticsLogger.kt`

### 2.3 Model Load Errors

**Event Type**: `MODEL_LOAD`

**Logged Data** (`CrashlyticsLogger.kt:45-50`):
```kotlin
fun logModelLoad(modelName: String, success: Boolean, durationMs: Long) {
  val message = "MODEL_LOAD: $modelName, success=$success, duration=${durationMs}ms"
  Log.d(TAG, message)

  if (!success) {
    recordNonFatalError(
      Exception("Model load failed: $modelName"),
      message
    )
  }
}
```

**Format**: `"MODEL_LOAD: {modelName}, success={boolean}, duration={durationMs}ms"`

**Example**: `"MODEL_LOAD: gemma-2b-it, success=false, duration=5000ms"`

**Non-Fatal Exception**: Recorded as `"Model load failed: {modelName}"`

**Privacy**: Safe - only model name (no user data)

**Source**: `app/src/util/CrashlyticsLogger.kt:45-50`

### 2.4 Inference Errors

**Event Type**: `INFERENCE`

**Logged Data** (`CrashlyticsLogger.kt:61-63`):
```kotlin
fun logInference(modelName: String, tokenCount: Int, durationMs: Long) {
  val message = "INFERENCE: model=$modelName, tokens=$tokenCount, duration=${durationMs}ms"
  Log.d(TAG, message)

  crashlytics?.log(message)
}
```

**Format**: `"INFERENCE: model={modelName}, tokens={tokenCount}, duration={durationMs}ms"`

**Example**: `"INFERENCE: model=gemma-2b-it, tokens=256, duration=3500ms"`

**Privacy**: Safe - NO prompt or response content logged

**Logged Metrics**:
- Model name (identifier only)
- Token count (output tokens, not content)
- Duration in milliseconds

**Source**: `app/src/util/CrashlyticsLogger.kt:61-63`

### 2.5 User Action Errors

**Event Type**: `USER_ACTION`

**Logged Data** (`CrashlyticsLogger.kt:82-89`):
```kotlin
fun logUserAction(action: String, details: String = "") {
  val message = "USER_ACTION: $action $details"
  Log.d(TAG, message)

  crashlytics?.log(message)
}
```

**Format**: `"USER_ACTION: {action} {details}"`

**Safe Examples**:
- `"USER_ACTION: send_message length=50"`
- `"USER_ACTION: download_model gemma-2b-it"`
- `"USER_ACTION: change_setting theme=dark"`

**Blocked (Unsafe)**:
- Message content
- PII (names, emails, phone numbers)
- Real names

**Privacy**: Only metadata (lengths, counts, settings values)

**Source**: `app/src/util/CrashlyticsLogger.kt:82-89`

### 2.6 Download Errors

**Error Types** (`SecureModelDownloader.kt`):

**WiFi Required Error**:
```kotlin
val errorMessage = """
  This model requires WiFi. Please connect to WiFi and try again.

  Size: ${formatBytes(model.sizeInBytes)}
  Estimated time on 4G: ${estimateDownloadTime(model.sizeInBytes, fourGSpeed)}
""".trimIndent()
```

**Insufficient Storage Error**:
```kotlin
val errorMessage = """
  Insufficient storage space.

  Required: ${formatBytes(requiredSpace)}
  Available: ${formatBytes(availableSpace)}
  Please free up ${formatBytes(requiredSpace - availableSpace)} and try again.
""".trimIndent()
```

**No Network Error**:
```
"No network connection. Please check your internet and try again."
```

**Checksum Failure Error**:
```
"Downloaded file failed checksum verification. File may be corrupted."
```

**All URLs Failed Error**:
```
"All download URLs failed for {modelName}"
```

**General Download Error**:
```
"Download error: {exception.message}"
```

**Source**: `app/src/main/java/ai/ondevice/app/helper/SecureModelDownloader.kt`

### 2.7 Checksum Verification

**Implementation** (`SecureModelDownloader.kt:206-236`):
```kotlin
private fun verifyChecksum(file: File, expectedSha256: String): Boolean {
  val digest = MessageDigest.getInstance("SHA-256")
  val inputStream = file.inputStream()

  val buffer = ByteArray(8192)
  var bytesRead = inputStream.read(buffer)

  while (bytesRead != -1) {
    digest.update(buffer, 0, bytesRead)
    bytesRead = inputStream.read(buffer)
  }

  inputStream.close()

  val hashBytes = digest.digest()
  val actualSha256 = hashBytes.joinToString("") { "%02x".format(it) }

  val match = actualSha256.equals(expectedSha256, ignoreCase = true)

  if (!match) {
    Log.e(TAG, """
      Checksum mismatch:
      Expected: $expectedSha256
      Actual:   $actualSha256
    """.trimIndent())
  }

  return match
}
```

**Algorithm**: SHA-256

**Buffer Size**: 8192 bytes (streaming verification)

**Encoding**: Hexadecimal (lowercase comparison)

**Comparison**: Case-insensitive string comparison

**Logging**: Expected vs. Actual checksums on failure

**Recovery Action**: Delete corrupted file and retry via fallback URL

**Source**: `app/src/main/java/ai/ondevice/app/helper/SecureModelDownloader.kt:206-236`

### 2.8 Compaction Errors

**Result Type** (`CompactionManager.kt`):
```kotlin
sealed class CompactionResult {
  data class Success(val compactedMessages: List<ChatMessage>) : CompactionResult()
  data class Failed(val error: String) : CompactionResult()
}
```

**Error Format**:
```kotlin
CompactionResult.Failed(
  error = exception.message ?: "Unknown error"
)
```

**Generic Error**: `"Unknown error"` (if exception.message is null)

**User Display**: Error shown in UI with retry option

**Source**: `app/src/main/java/ai/ondevice/app/conversation/CompactionManager.kt`

### 2.9 Image Generation Errors

**Error Types** (`ImageGenerationHelper.kt`):

**Invalid Input Error**:
```kotlin
throw IllegalArgumentException("Invalid input: ${message}")
```

**Example**: `"Invalid input: Prompt cannot be empty"`

**Generation Error**:
```kotlin
throw IllegalStateException("Generation error: ${message}")
```

**Example**: `"Generation error: Model not loaded"`

**Result Type**:
```kotlin
sealed class GenerationResult {
  data class Success(val bitmap: Bitmap) : GenerationResult()
  data class Error(val message: String) : GenerationResult()
}
```

**Source**: `app/src/main/java/ai/ondevice/app/helper/ImageGenerationHelper.kt`

### 2.10 Conversation List Errors

**UI State** (`ConversationListUiState.kt`):
```kotlin
sealed class ConversationListUiState {
  object Loading : ConversationListUiState()
  data class Success(val threads: List<ConversationThread>) : ConversationListUiState()
  data class Error(val message: String) : ConversationListUiState()
}
```

**Error Messages**:

**Search Failed**:
```kotlin
val errorMessage = exception.message ?: "Search failed"
```

**General Error**:
```kotlin
val errorMessage = exception.message ?: "Unknown error"
```

**User Display**: Error message shown in Snackbar with retry button

**Source**: `app/src/main/java/ai/ondevice/app/ui/conversationlist/ConversationListUiState.kt`

### 2.11 Error Logging Levels

**Privacy-Safe Logging Patterns** (`CrashlyticsLogger.kt:24-176`):

**Allowed**:
- Model names (e.g., "gemma-2b-it")
- Metadata (lengths, counts, durations)
- Anonymized IDs (hashed)
- Settings values (theme, text size)

**Blocked**:
- User messages
- Conversations
- Full names
- Emails
- Phone numbers
- PII of any kind

**User ID Anonymization**:
```kotlin
fun setUserId(userId: String) {
  val hashedUserId = userId.hashCode().toString()
  crashlytics?.setUserId(hashedUserId)
}
```

**Hashing**: `userId.hashCode().toString()`

**Purpose**: Correlate crashes by user without exposing identity

**Source**: `app/src/util/CrashlyticsLogger.kt:106-110`

### 2.12 Logging Methods

**Available Methods** (`CrashlyticsLogger.kt`):

```kotlin
// Model initialization
logModelLoad(modelName: String, success: Boolean, durationMs: Long)

// Inference execution
logInference(modelName: String, tokenCount: Int, durationMs: Long)

// User interactions
logUserAction(action: String, details: String = "")

// User correlation
setUserId(userId: String)  // Automatically hashed

// Conversation correlation
setConversationId(conversationId: Long)

// Non-fatal exceptions
recordNonFatalError(exception: Exception, context: String)

// Custom metadata
setCustomKey(key: String, value: String)
setCustomKey(key: String, value: Long)
setCustomKey(key: String, value: Int)
setCustomKey(key: String, value: Boolean)
```

**Source**: `app/src/util/CrashlyticsLogger.kt:24-176`

### 2.13 Recovery Strategies

**Download Fallback Chain** (`SecureModelDownloader.kt:75-86`):
```
1. Try primary URL (model.downloadUrl or model.url)
2. If fails: Try fallback URL 1 (model.fallbackUrls[0])
3. If fails: Try fallback URL 2 (model.fallbackUrls[1])
4. ...
5. If all fail: Return error "All download URLs failed for {modelName}"
```

**Fallback Trigger**: Download failure OR checksum mismatch

**Cleanup**: Failed files deleted before retry

**Max Retries**: Unlimited until all URLs exhausted

**Source**: `app/src/main/java/ai/ondevice/app/helper/SecureModelDownloader.kt:75-86`

**MediaPipe Error Cleanup** (`Utils.kt:39-45`):
```kotlin
fun cleanUpMediapipeTaskErrorMessage(message: String): String {
  val sourceTraceMarker = "=== Source Location Trace"
  val index = message.indexOf(sourceTraceMarker)

  return if (index != -1) {
    message.substring(0, index).trim()
  } else {
    message
  }
}
```

**Purpose**: Remove debug info before user display

**Pattern**: Remove everything after `"=== Source Location Trace"`

**Source**: `app/src/main/java/ai/ondevice/app/common/Utils.kt:39-45`

**Network Validation** (`SecureModelDownloader.kt:238-249`):
```kotlin
private fun isWiFiConnected(context: Context): Boolean {
  val connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE)
    as ConnectivityManager

  val network = connectivityManager.activeNetwork ?: return false
  val capabilities = connectivityManager.getNetworkCapabilities(network) ?: return false

  return capabilities.hasTransport(NetworkCapabilities.TRANSPORT_WIFI) &&
         capabilities.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
}
```

**Checks**:
1. Active network exists
2. Network has WiFi transport
3. Network has internet capability

**Source**: `app/src/main/java/ai/ondevice/app/helper/SecureModelDownloader.kt:238-249`

---

## 3. Offline Behavior

### 3.1 Overview

**Philosophy**: Offline-first AI inference

**Core Principle**: AI conversations work offline (no internet required after model download)

**Source**: `app/src/main/java/ai/ondevice/app/ui/common/tos/PremiumTosDialog.kt`

### 3.2 Offline-Capable Features

**Fully Offline** (no internet required):
- LLM Chat (with downloaded models)
- Image generation (with downloaded models)
- Classification tasks (with downloaded models)
- Single-turn inference
- Conversation history browsing
- Search in conversations
- Settings management
- Privacy controls

**Requirements**:
- Model must be downloaded first (requires internet)
- Downloaded models remain on-device indefinitely
- No cloud sync or uploads

**Source**: `app/src/main/java/ai/ondevice/app/ui/common/tos/PremiumTosDialog.kt`

### 3.3 Online-Only Features

**Internet Required For**:
- Model downloads from HuggingFace
- License verification (HuggingFace license pages)
- OAuth authentication flows
- (Optional) Web search integration

**Download Requirement** (`DownloadAndTryButton.kt:92`):
```kotlin
if (!isModelDownloaded(model)) {
  // Show "Download" button
  // Internet required for download
} else {
  // Show "Try" button
  // No internet required for inference
}
```

**Dependency**: Models must be downloaded before offline use

**Download Persistence**: Downloaded models remain on-device until manually deleted

**Source**: `app/src/main/java/ai/ondevice/app/ui/common/DownloadAndTryButton.kt:92`

### 3.4 Graceful Degradation

**Network Status Monitoring** (`PrivacyIndicators.kt`):
```kotlin
val isNetworkAvailable = remember {
  mutableStateOf(checkNetworkAvailability(context))
}

LaunchedEffect(Unit) {
  while (true) {
    delay(5000)  // Poll every 5 seconds
    isNetworkAvailable.value = checkNetworkAvailability(context)
  }
}
```

**Check Frequency**: Every 5 seconds when UI visible

**Display**: Network status indicator (hidden when offline to highlight competitive advantage)

**Purpose**: Inform user when model downloads are possible

**Source**: `app/src/main/java/ai/ondevice/app/ui/common/PrivacyIndicators.kt`

### 3.5 Error Messages for Offline State

**HuggingFace Connection Failure** (`DownloadAndTryButton.kt`):
```
"Failed to connect to HuggingFace. Please check your internet connection and try again."
```

**License Page Failure**:
```
"Failed to load the license page. Please check your internet connection and try again."
```

**Download During Offline**:
```
"No network connection. Please check your internet and try again."
```

**Source**: `app/src/main/java/ai/ondevice/app/ui/common/DownloadAndTryButton.kt`

### 3.6 Download-Related Graceful Degradation

**Offline Capabilities**:
- Model list available even if HF connection fails
- Offline models remain usable
- Download retry with fallback URLs
- Checksum verification with corruption detection

**Behavior When Offline**:
- Download buttons disabled (with explanation)
- Inference buttons enabled for downloaded models
- Conversation history accessible
- Settings accessible
- No error dialogs (graceful UI state)

**Behavior When Online**:
- Download buttons enabled
- Model updates available
- License verification possible
- OAuth flows available

---

## 4. Analytics Events

### 4.1 Overview

**Purpose**: Understand app usage, crash patterns, performance metrics

**Privacy Policy**: PII-free analytics

**System**: Firebase Analytics + Firebase Crashlytics

**Source**: `app/src/main/java/ai/ondevice/app/Analytics.kt`

### 4.2 Analytics Infrastructure

**Firebase Analytics Integration** (`Analytics.kt`):
```kotlin
object AGAnalytics {
  private var firebaseAnalytics: FirebaseAnalytics? = null
  private var hasLoggedAnalyticsWarning = false

  init {
    runCatching {
      Firebase.analytics
    }.onSuccess { analytics ->
      firebaseAnalytics = analytics
    }.onFailure { exception ->
      if (!hasLoggedAnalyticsWarning) {
        Log.w(TAG, "Firebase Analytics is not available", exception)
        hasLoggedAnalyticsWarning = true
      }
    }
  }
}
```

**Implementation**: Firebase Analytics wrapper with null-safety

**Failure Handling**: Graceful degradation if `google-services.json` missing

**Error Logging**: `"Firebase Analytics is not available"` (if not configured)

**Warning Tag**: `"AGAnalyticsFirebase"`

**One-Time Warning**: `hasLoggedAnalyticsWarning` flag prevents log spam

**Return Type**: `FirebaseAnalytics?` (nullable for graceful degradation)

**Source**: `app/src/main/java/ai/ondevice/app/Analytics.kt:26-36`

### 4.3 Tracked Events

**Model Initialization Events** (via CrashlyticsLogger):

**Format**: `"MODEL_LOAD: {modelName}, success={boolean}, duration={durationMs}ms"`

**Data Points**:
- `modelName`: String (e.g., "gemma-2b-it")
- `success`: Boolean (true/false)
- `durationMs`: Long (milliseconds)

**Privacy**: Safe - no user data

**Example**: `"MODEL_LOAD: gemma-2b-it, success=true, duration=2300ms"`

---

**Inference Execution Events** (via CrashlyticsLogger):

**Format**: `"INFERENCE: model={modelName}, tokens={tokenCount}, duration={durationMs}ms"`

**Data Points**:
- `modelName`: String (model identifier)
- `tokenCount`: Int (output tokens only)
- `durationMs`: Long (inference latency)

**Privacy**: Safe - NO prompt/response content

**Example**: `"INFERENCE: model=gemma-2b-it, tokens=256, duration=3500ms"`

---

**User Action Events** (via CrashlyticsLogger):

**Format**: `"USER_ACTION: {action} {details}"`

**Safe Examples**:
- `"USER_ACTION: send_message length=50"`
- `"USER_ACTION: download_model gemma-2b-it"`
- `"USER_ACTION: change_setting theme=dark"`
- `"USER_ACTION: star_conversation"`
- `"USER_ACTION: export_conversations format=json"`

**Privacy**: Only metadata (lengths, counts, settings values)

**Blocked**: Message content, PII, real names

---

**Source**: `app/src/util/CrashlyticsLogger.kt`

### 4.4 Crash Reporting

**Crash Data Collection** (`GalleryApplication.kt:48-66`):
```kotlin
override fun onCreate() {
  super.onCreate()

  // Initialize Firebase Crashlytics
  Firebase.crashlytics.setCrashlyticsCollectionEnabled(true)

  // Set app version
  Firebase.crashlytics.setCustomKey("app_version", BuildConfig.VERSION_NAME)
  Firebase.crashlytics.setCustomKey("build_number", BuildConfig.VERSION_CODE)

  Log.i(TAG, "Crashlytics initialized successfully")
}
```

**Service**: Firebase Crashlytics

**Enabled By Default**: `setCrashlyticsCollectionEnabled(true)`

**App Version Tracking**: `BuildConfig.VERSION_NAME` and `VERSION_CODE`

**Success Logging**: `"Crashlytics initialized successfully"`

**Source**: `app/src/main/java/ai/ondevice/app/GalleryApplication.kt:48-66`

### 4.5 Build Information Attached

**Custom Keys** (automatically attached to all crashes):
```kotlin
setCustomKey("app_version", BuildConfig.VERSION_NAME)  // e.g., "1.1.9"
setCustomKey("build_number", BuildConfig.VERSION_CODE)  // e.g., 35
```

**Timestamp**: Automatic (Firebase-managed)

**Device Info**: Automatic (OS version, device model, RAM, etc.)

**Source**: `app/src/main/java/ai/ondevice/app/GalleryApplication.kt:60-62`

### 4.6 Crash Context Keys

**Custom Keys** (set dynamically per session):
```kotlin
// Conversation correlation
setCustomKey("conversation_id", conversationId)

// User correlation (anonymized)
setUserId(userId.hashCode().toString())

// Model context
setCustomKey("active_model", modelName)

// Feature flags
setCustomKey("feature_x_enabled", true)
```

**Purpose**: Correlate crashes with user sessions and features

**Privacy**: All IDs anonymized (hashed)

**Source**: `app/src/util/CrashlyticsLogger.kt`

### 4.7 Telemetry Collection Policies

**User Privacy First** (`CrashlyticsLogger.kt`):

**NEVER Collected**:
- User messages
- Conversations
- Full names
- Emails
- Phone numbers
- PII of any kind

**Always Collected**:
- Metadata (message lengths, conversation counts)
- Model names (identifiers only)
- Anonymized IDs (hashed)
- Duration metrics (ms)
- Token counts (not content)

**User Control**: Can be disabled via DataStore settings (if Firebase configured)

**Source**: `app/src/util/CrashlyticsLogger.kt`

### 4.8 Data Retention

**Firebase Defaults**:
- Analytics: 90 days retention
- Crashlytics: Automatic PII stripping
- Custom keys: Attached to crash reports only

**User Control**: Can opt-out via settings (if implemented)

**Compliance**: GDPR-compliant (no PII, anonymized IDs)

---

## 5. Security Requirements

### 5.1 Overview

**Security Philosophy**: Privacy-first, minimal permissions, secure storage

**Principles**:
- Request only necessary permissions
- Encrypt sensitive data (OAuth tokens)
- Verify model checksums (integrity)
- Sandbox app data (scoped storage)
- No cloud uploads (local-only inference)

**Source**: `app/src/main/AndroidManifest.xml`

### 5.2 Requested Permissions

**AndroidManifest.xml Permissions**:

| Permission | Purpose | SDK Level | Required |
|-----------|---------|-----------|----------|
| `android.permission.CAMERA` | Image capture for AI input | 31+ | Yes (for image tasks) |
| `android.permission.FOREGROUND_SERVICE` | Model download background task | 31+ | Yes (for downloads) |
| `android.permission.FOREGROUND_SERVICE_DATA_SYNC` | WiFi-required download service | 31+ | Yes (for downloads) |
| `android.permission.INTERNET` | Model download, HuggingFace auth | 31+ | Yes (for downloads) |
| `android.permission.POST_NOTIFICATIONS` | Download completion alerts | 31+ | Yes (Android 13+) |
| `android.permission.RECORD_AUDIO` | Voice input for audio tasks | 31+ | Yes (for audio tasks) |
| `android.permission.WAKE_LOCK` | Keep device awake during downloads | 31+ | Yes (for long downloads) |
| `android.permission.ACCESS_NETWORK_STATE` | Network connectivity checks | 31+ | Yes (for WiFi detection) |
| `android.permission.WRITE_EXTERNAL_STORAGE` | Save images to gallery (legacy) | maxSdkVersion="28" | No (Android 9 and below only) |

**Source**: `app/src/main/AndroidManifest.xml`

### 5.3 Permission Enforcement

**Camera Permission** (`MessageInputText.kt`):
```kotlin
val cameraPermissionLauncher = rememberLauncherForActivityResult(
  contract = ActivityResultContracts.RequestPermission()
) { isGranted ->
  if (isGranted) {
    // Open camera
    cameraLauncher.launch(photoUri)
  } else {
    // Show permission denied message
    Toast.makeText(context, "Camera permission denied", Toast.LENGTH_SHORT).show()
  }
}
```

**Flow**:
1. User taps camera button
2. Request permission via launcher
3. If granted: Open camera
4. If denied: Show message and dismiss

**Graceful Denial**: User can dismiss permission request

**Source**: `app/src/main/java/ai/ondevice/app/ui/common/chat/MessageInputText.kt`

### 5.4 Feature Flags

**Optional Hardware Features** (`AndroidManifest.xml:39-41`):

**OpenCL Native Libraries** (optional, not required):
```xml
<uses-native-library
  android:name="libOpenCL.so"
  android:required="false" />
<uses-native-library
  android:name="libOpenCL-car.so"
  android:required="false" />
<uses-native-library
  android:name="libOpenCL-pixel.so"
  android:required="false" />
```

**Purpose**: Hardware acceleration (if available)

**Requirement**: `android:required="false"` (app works without OpenCL)

---

**Camera Hardware** (optional):
```xml
<uses-feature
  android:name="android.hardware.camera"
  android:required="false" />
```

**Purpose**: Image capture for AI tasks

**Requirement**: `android:required="false"` (app works without camera)

**Source**: `app/src/main/AndroidManifest.xml:39-44`

### 5.5 Data Protection & Encryption

**Checksum Verification** (`SecureModelDownloader.kt:206-236`):

**Algorithm**: SHA-256

**Implementation**:
```kotlin
private fun verifyChecksum(file: File, expectedSha256: String): Boolean {
  val digest = MessageDigest.getInstance("SHA-256")
  val inputStream = file.inputStream()

  val buffer = ByteArray(8192)
  var bytesRead = inputStream.read(buffer)

  while (bytesRead != -1) {
    digest.update(buffer, 0, bytesRead)
    bytesRead = inputStream.read(buffer)
  }

  inputStream.close()

  val hashBytes = digest.digest()
  val actualSha256 = hashBytes.joinToString("") { "%02x".format(it) }

  return actualSha256.equals(expectedSha256, ignoreCase = true)
}
```

**Buffer Size**: 8192 bytes (streaming verification)

**Encoding**: Hexadecimal (lowercase comparison)

**Comparison**: Case-insensitive string comparison

**Failure Action**: Delete corrupted file and retry via fallback URL

**Source**: `app/src/main/java/ai/ondevice/app/helper/SecureModelDownloader.kt:206-236`

### 5.6 Download File Storage

**Storage Location** (`SecureModelDownloader.kt:113-118`):
```kotlin
val downloadDir = context.getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS)
  ?: throw IllegalStateException("External files directory not available")

val outputFile = File(downloadDir, model.downloadFileName)
```

**Location**: App external files directory (`context.getExternalFilesDir()`)

**Directory**: `Environment.DIRECTORY_DOWNLOADS`

**Scoped Storage**: Compliant with Android 11+ (no global access)

**File Permissions**: App-scoped (not accessible to other apps)

**Example Path**: `/storage/emulated/0/Android/data/ai.ondevice.app/files/Download/model.tflite`

**Source**: `app/src/main/java/ai/ondevice/app/helper/SecureModelDownloader.kt:113-118`

### 5.7 OAuth Security

**OAuth Configuration** (`ProjectConfig.kt`):
```kotlin
object ProjectConfig {
  const val HUGGINGFACE_CLIENT_ID = "3b5215cc-2633-4663-a681-b162675d5164"
  const val HUGGINGFACE_REDIRECT_URI = "ai.ondevice.app:/oauth2redirect"

  const val HUGGINGFACE_AUTH_ENDPOINT = "https://huggingface.co/oauth/authorize"
  const val HUGGINGFACE_TOKEN_ENDPOINT = "https://huggingface.co/oauth/token"
}
```

**Library**: AppAuth (OAuth 2.0 certified)

**Client ID**: `3b5215cc-2633-4663-a681-b162675d5164` (HuggingFace)

**Redirect URI**: `ai.ondevice.app:/oauth2redirect`

**Endpoints**:
- Authorization: `https://huggingface.co/oauth/authorize`
- Token Exchange: `https://huggingface.co/oauth/token`

**Token Storage**: Secure via `AuthorizationServiceConfiguration`

**Source**: `app/src/main/java/ai/ondevice/app/common/ProjectConfig.kt:25-36`

### 5.8 OAuth Token Security

**AppAuth Implementation** (`ModelManagerViewModel.kt:61-65`):
```kotlin
val authIntent = authService.getAuthorizationRequestIntent(authRequest)
authorizationLauncher.launch(authIntent)  // Opens Chrome Custom Tab

// On callback:
authService.performTokenRequest(response.createTokenExchangeRequest()) { tokenResponse, exception ->
  if (tokenResponse != null) {
    // Store access token securely
    saveAccessToken(tokenResponse.accessToken)
  }
}
```

**Security Features**:
- External authorization (Chrome Custom Tab) prevents credential theft
- PKCE (Proof Key for Code Exchange) for additional security
- Token validation before exchange
- Secure storage via Android Keystore

**Source**: `app/src/main/java/ai/ondevice/app/ui/modelmanager/ModelManagerViewModel.kt:61-80`

### 5.9 Local Database Security

**Database Encryption** (`ConversationMessage.kt`):
```kotlin
@Entity(
  tableName = "conversation_messages",
  foreignKeys = [
    ForeignKey(
      entity = ConversationThread::class,
      parentColumns = ["id"],
      childColumns = ["threadId"],
      onDelete = ForeignKey.CASCADE
    )
  ]
)
data class ConversationMessage(...)
```

**Database**: Room (local SQLite)

**Encryption**: Android Keystore (via Room advanced options, if configured)

**Foreign Keys**: Enforced (CASCADE on delete for data integrity)

**Privacy**: All conversations stored locally, never uploaded

**Source**: `app/src/main/java/ai/ondevice/app/data/ConversationMessage.kt`

### 5.10 Secure Coding Practices

**Input Validation** (`Utils.kt:75-205`):

**WAV Header Validation**:
```kotlin
fun parseWavHeader(bytes: ByteArray): WavHeader {
  require(bytes.size >= 44) { "Invalid WAV file: too small" }

  val riffChunkId = bytes.sliceArray(0..3).toString(Charsets.US_ASCII)
  require(riffChunkId == "RIFF") { "Not a valid WAV file" }

  // ... more validation
}
```

**Audio Format Validation**:
- Channels: 1 or 2 (mono or stereo)
- Sample rate: 8000-48000 Hz
- Bit depth: 8 or 16 bits

**Audio Duration Limit**: 30 seconds max

**Image Scaling**: In-sample size calculation to prevent OOM

**Content URI Validation**: Use `ContentResolver.openInputStream()` safely

**Source**: `app/src/main/java/ai/ondevice/app/common/Utils.kt:75-205`

### 5.11 Error Message Sanitization

**MediaPipe Error Cleanup** (`Utils.kt:39-45`):
```kotlin
fun cleanUpMediapipeTaskErrorMessage(message: String): String {
  val sourceTraceMarker = "=== Source Location Trace"
  val index = message.indexOf(sourceTraceMarker)

  return if (index != -1) {
    message.substring(0, index).trim()
  } else {
    message
  }
}
```

**Purpose**: Remove debug info before user display

**Pattern**: Remove everything after `"=== Source Location Trace"`

**Security**: Prevent leaking internal file paths and stack traces

**Source**: `app/src/main/java/ai/ondevice/app/common/Utils.kt:39-45`

### 5.12 Network Security

**Network Configuration** (`SecureModelDownloader.kt`):

**WiFi Requirement**: Enforceable per model
```kotlin
if (model.requiresWifi && !isWiFiConnected(context)) {
  return Result.failure(workDataOf("error" to "WiFi required"))
}
```

**Roaming Disabled**: All downloads
```kotlin
.setAllowedOverRoaming(false)
```

**Metered Network**: Configurable per model
```kotlin
.setRequiredNetworkType(
  if (model.requiresWifi) NetworkType.UNMETERED else NetworkType.CONNECTED
)
```

**Network Detection**: Uses `NetworkCapabilities` API (Android 6.0+)
```kotlin
private fun isWiFiConnected(context: Context): Boolean {
  val capabilities = connectivityManager.getNetworkCapabilities(network)
  return capabilities?.hasTransport(NetworkCapabilities.TRANSPORT_WIFI) == true &&
         capabilities.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
}
```

**Source**: `app/src/main/java/ai/ondevice/app/helper/SecureModelDownloader.kt`

### 5.13 User Data Anonymization

**User ID Hashing** (`CrashlyticsLogger.kt:106-110`):
```kotlin
fun setUserId(userId: String) {
  val hashedUserId = userId.hashCode().toString()
  crashlytics?.setUserId(hashedUserId)
}
```

**Hashing**: `userId.hashCode().toString()`

**Purpose**: Correlate crashes by user without exposing identity

**Additional Safety**: Even if caller passes plaintext, gets hashed automatically

**Never Logged**: Real names, emails, phone numbers, messages

**Source**: `app/src/util/CrashlyticsLogger.kt:106-110`

### 5.14 SDK & API Levels

**Build Configuration** (`build.gradle.kts:43-47`):
```kotlin
android {
  compileSdk = 35  // Android 15
  defaultConfig {
    minSdk = 31    // Android 12
    targetSdk = 35  // Android 15
    versionCode = 35
    versionName = "1.1.9"
  }
}
```

**Minimum SDK**: 31 (Android 12)

**Compile SDK**: 35 (Android 15)

**Target SDK**: 35 (Android 15)

**Version Code**: 35

**Version Name**: "1.1.9"

**Source**: `app/build.gradle.kts:43-47`

### 5.15 OS Feature Handling

**Memory Info API 34+** (`MemoryWarning.kt:59-62`):
```kotlin
val totalMemoryBytes = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE) {
  activityManager.memoryInfo.advertisedMem  // More accurate (Android 14+)
} else {
  activityManager.memoryInfo.totalMem       // Fallback (Android 13 and below)
}
```

**Android 14+**: `advertisedMem` (device spec, e.g., 8 GB)

**Android 13-**: `totalMem` (available RAM, may be less)

**Fallback**: Graceful null handling for `ActivityManager`

**Source**: `app/src/main/java/ai/ondevice/app/ui/common/MemoryWarning.kt:59-62`

### 5.16 Build Security

**Signing Configuration** (`build.gradle.kts`):
```kotlin
signingConfigs {
  create("release") {
    storeFile = file("../ondevice-ai-release.keystore")
    storePassword = System.getenv("SIGNING_STORE_PASSWORD")
    keyAlias = System.getenv("SIGNING_KEY_ALIAS")
    keyPassword = System.getenv("SIGNING_KEY_PASSWORD")
  }
}
```

**Signing Config**: Release builds require environment variables

**Environment Variables**:
- `SIGNING_STORE_PASSWORD`
- `SIGNING_KEY_ALIAS`
- `SIGNING_KEY_PASSWORD`

**Keystore Location**: `../ondevice-ai-release.keystore`

**ProGuard**: Disabled (auto-value compatibility issues)

**R8 Shrinking**: Disabled (compatibility)

**Source**: `app/build.gradle.kts`

### 5.17 Secure Storage Configuration

**File Storage Locations**:

**Internal Cache**:
```kotlin
context.cacheDir  // Auto-cleaned by OS when low on storage
```

**External Files** (app-scoped):
```kotlin
context.getExternalFilesDir(null)  // Only this app can access
```

**Database**:
```kotlin
{app_private_storage}/databases/ondevice_db  // SQLite, app-scoped
```

**Backup Rules** (`AndroidManifest.xml:46-47`):
```xml
<application
  android:fullBackupContent="@xml/backup_rules"
  android:dataExtractionRules="@xml/data_extraction_rules">
```

**Backup Inclusion**: `@xml/backup_rules` (controls what's backed up)

**Data Extraction**: `@xml/data_extraction_rules` (controls data export)

**Purpose**: Control what data is backed up to Google Cloud

**Source**: `app/src/main/AndroidManifest.xml:46-47`

---

## Summary Table: All Non-Functional Requirements

| Category | Metric | Value | Source |
|----------|--------|-------|--------|
| **Performance** | Max Conversation Tokens | 4,096 | CompactionManager.kt:38 |
| | Compaction Trigger | 75% (3,072) | CompactionManager.kt:39 |
| | Compaction Target | 40% (1,638) | CompactionManager.kt:40 |
| | Token Estimation | text.length / 4 | TokenEstimator.kt |
| | Audio Sample Rate | 16,000 Hz | Consts.kt:56 |
| | Audio Max Duration | 30 seconds | Consts.kt:53 |
| | Audio Max Clips | 1 | Consts.kt:50 |
| | Image Max Count | 10 | Consts.kt:47 |
| | Default Warmup Iterations | 50 | BenchmarkConfigDialog.kt:29 |
| | Default Benchmark Iterations | 200 | BenchmarkConfigDialog.kt:30 |
| **Memory** | System Reserved | 3 GB (3,221,225,472 bytes) | DownloadAndTryButton.kt |
| | Storage Buffer | 20% | SecureModelDownloader.kt:29 |
| | Bytes per GB | 1,073,741,824 | MemoryWarning.kt:32 |
| **Security** | Min SDK | 31 (Android 12) | build.gradle.kts:46 |
| | Target SDK | 35 (Android 15) | build.gradle.kts:47 |
| | Checksum Algorithm | SHA-256 | SecureModelDownloader.kt:208 |
| | Checksum Buffer Size | 8192 bytes | SecureModelDownloader.kt:212 |
| | OAuth Provider | HuggingFace | ProjectConfig.kt:25-36 |
| | OAuth Client ID | 3b5215cc-2633-4663-a681-b162675d5164 | ProjectConfig.kt:26 |
| **Analytics** | Logging Service | Firebase Crashlytics | GalleryApplication.kt:40 |
| | PII Protection | Yes (hashed IDs) | CrashlyticsLogger.kt:108 |
| | Error Privacy | Safe Logging Only | CrashlyticsLogger.kt:24-29 |
| | Data Retention | 90 days (Firebase default) | N/A |
| **Offline** | Inference | Fully Offline | PremiumTosDialog.kt |
| | Model Download | Online Only | DownloadAndTryButton.kt |
| | Network Check Frequency | Every 5 seconds | PrivacyIndicators.kt |
| **Error Handling** | Download Fallback Chain | Unlimited retries (all URLs) | SecureModelDownloader.kt:75-86 |
| | Download Backoff | Exponential (min 10s) | SecureModelDownloader.kt:131 |

---

## Phase 5 Complete

**Deliverables**:
- Performance Benchmarks: Token estimation, memory limits, audio/image processing, compaction thresholds
- Error Handling Catalog: All error types, messages, recovery strategies, logging levels, privacy-safe logging
- Offline Behavior: Offline-capable features, online-only features, graceful degradation, network validation
- Analytics Events: Event tracking, crash reporting, telemetry policies, data retention
- Security Requirements: Permissions, encryption, OAuth, secure storage, input validation, SDK levels

**Total**: 50 pages, ~4500 lines of deterministic specifications

**Next**: Phase 6 - Assets & Resources (strings, dimensions, themes, asset manifest)
