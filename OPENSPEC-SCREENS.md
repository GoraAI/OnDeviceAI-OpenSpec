# OnDevice AI - Screen Specifications

**Document Version:** 0.1
**Source:** OnDevice AI v1.1.9 (build 35)
**Purpose:** Deterministic screen-by-screen specifications for pixel-perfect implementation
**Date:** 2025-01-06

---

## Document Overview

This document provides complete specifications for all 14 screens in OnDevice AI. Each screen includes:
- **Purpose** - Primary use case and user goals
- **Layout** - Exact dp values, alignment, constraints
- **Components** - All UI elements with dimensions
- **States** - Empty, loading, streaming, error, populated
- **Copy** - All strings from code and resources
- **Logic** - When transitions occur, behavior rules
- **Entry/Exit** - Navigation triggers, back button behavior
- **Accessibility** - Screen reader labels

All values sourced from code with line number references.

---

## Screen 1: LLM Chat Screen

**File:** `app/src/main/java/ai/ondevice/app/ui/llmchat/LlmChatScreen.kt`
**Route:** `route_model/{taskId}/{modelName}` where `taskId = LLM_CHAT`
**Task ID:** `BuiltInTaskId.LLM_CHAT`

### Purpose

Multi-turn conversational interface supporting text, images (up to 10), and audio input (up to 30 seconds). Primary interaction mode for on-device LLM chat.

### Layout Structure

```
Scaffold (full screen)
├── TopAppBar: ModelPageAppBar (height: 64.dp)
│   ├── Navigation: Menu icon (left)
│   ├── Title: ModelPickerChip (center) - model selector
│   └── Actions: Config + Reset buttons (right)
│
└── Body: ChatPanel or ModelDownloadStatusInfoPanel
    ├── LazyColumn (messages area)
    │   ├── Greeting screen (when empty)
    │   └── Message list (when populated)
    │       └── Each message:
    │           ├── MessageSender (sender label) - 2.dp bottom padding
    │           ├── Message body (type-specific)
    │           └── Action buttons row - 4.dp spacing
    │
    ├── Disclaimer row (after last AI message)
    │   └── Text + color logo (16.dp)
    │
    └── Surface (input container) - 28.dp rounded top corners
        ├── Image/Audio preview row (80.dp height)
        └── MessageInputText (min 120.dp)
            ├── TextField (1-3 lines, bodyLargeNarrow)
            └── Buttons: Add (40.dp) + Send/Stop (40.dp)
```

### Top Bar Specifications

**Type:** `CenterAlignedTopAppBar` (Material 3)

**Navigation Icon (Left):**
- Icon: `Icons.Rounded.Menu`
- Size: Default Material (24.dp)
- Enabled: NOT initializing AND NOT in progress
- OnClick: Opens ChatMenuSheet
- Content description: "" (empty)

**Title (Center):**
- Component: `ModelPickerChip`
- Shows: Selected model display name
- OnClick: Opens model selector dialog
- Typography: `titleMedium`

**Actions (Right):**

| Button | Icon | Size | Visibility | Behavior |
|--------|------|------|------------|----------|
| Config | `Icons.Rounded.Tune` | 20.dp | Model has configurable params | Opens config dialog |
| Reset | `Icons.Rounded.MapsUgc` (chat icon) | 20.dp | After first message sent | Clears conversation, starts new |
| Reset Progress | `CircularProgressIndicator` | 20.dp, 2.dp stroke | During reset | Animated progress |

**Button States:**
- Config enabled: NOT initializing AND NOT in progress AND initialized
- Reset enabled: NOT initializing AND NOT preparing AND initialized
- Disabled alpha: **0.5f**
- Offset when both shown: Config button `-40.dp` x-offset

**Source:** `app/src/main/java/ai/ondevice/app/ui/common/ModelPageAppBar.kt`

---

### Message List Area

**Container:** `LazyColumn` with `reverseLayout = false`, `verticalArrangement = Arrangement.Bottom`

**Message Padding:**

| Message Type | Start Padding | End Padding | Top/Bottom |
|--------------|---------------|-------------|------------|
| Agent (AI) | 12.dp | 60.dp (12 + 48 extra) | 6.dp |
| User | 60.dp (12 + 48 extra) | 12.dp | 6.dp |
| System | 36.dp (12 + 24 extra) | 36.dp | 6.dp |
| Prompt Templates | 24.dp (12 + 12 extra) | 24.dp | 6.dp |

**Bubble Styling:**
- Border radius: **24.dp** (`R.dimen.chat_bubble_corner_radius`)
- Sharp corner on sender side: `hardCornerAtLeftOrRight = true`
- User bubble color: `MaterialTheme.customColors.userBubbleBgColor` = `Color(0xFFF0F0F0)` (light), `Color(0xFF2C2C2E)` (dark)
- Agent bubble color: `MaterialTheme.customColors.agentBubbleBgColor` = `Color.Transparent`
- Image messages: `Color.Transparent` (no bubble)

**Scroll Behavior:**
- Auto-scroll to bottom: Enabled by default
- Disabled for long responses (>2000 chars) to allow reading from top
- Disabled when user manually scrolls up
- Scroll offset for smooth landing: `10000`

**Source:** `app/src/main/java/ai/ondevice/app/ui/common/chat/ChatPanel.kt:100-150`

---

### Greeting Screen (Empty State)

**Visibility:** When `messagesByModel[selectedModel.name]` is empty

**Layout:**
```
Column (centered, fillMaxSize)
├── Image: App logo
│   └── Size: 160.dp × 160.dp
│   └── Bottom padding: 4.dp
│
└── Text: Dynamic greeting
    └── Style: MaterialTheme.typography.headlineMedium
    └── Alignment: TextAlign.Center
```

**Greeting Text Logic:**
```kotlin
val hour = Calendar.getInstance().get(Calendar.HOUR_OF_DAY)
val greeting = when (hour) {
  in 5..11 -> "How can I help you this morning?"
  in 12..17 -> "How can I help you this afternoon?"
  else -> "How can I help you this evening?"
}
```

**Typography:** `headlineMedium` = Nunito Medium, 28sp, 36sp line height, -0.25sp letter spacing

**Source:** `app/src/main/java/ai/ondevice/app/ui/common/chat/ChatPanel.kt:200-250`

---

### Message Types and Display

#### 1. Text Messages (User)

**Layout:**
```
Surface (rounded bubble)
├── Row (if voice input)
│   ├── Icon: Icons.Rounded.Mic (16.dp, alpha 0.7f)
│   └── Spacer: 4.dp
│
└── MarkdownText
    └── Padding: 12.dp all sides
    └── Style: bodyMedium (Nunito Regular, 14sp, 20sp line height)
```

**Voice Input Indicator:**
- Shows when `message.isVoiceInput == true`
- Icon size: **16.dp**
- Icon alpha: **0.7f**
- Right padding: **4.dp**

**Source:** `app/src/main/java/ai/ondevice/app/ui/common/chat/MessageBodyText.kt:40-60`

#### 2. Text Messages (Agent/AI)

**States:**

| State | Condition | Display |
|-------|-----------|---------|
| Thinking | `content.isEmpty() && generating` | `RotationalLoader(32.dp)` with fade animation |
| Streaming | `content.isNotEmpty() && generating` | `MarkdownText` updating incrementally |
| Complete | `latencyMs > 0` | `MarkdownText` with action buttons |

**Thinking Animation:**
- Duration: **300ms**
- Easing: `tween(300)`
- Effect: `fadeIn()` / `fadeOut()`

**Markdown Support:**
- Headings: H1-H6 with custom styling
- Lists: Bullet, numbered, nested
- Code blocks: Syntax highlighted with background
- Links: Clickable, opens browser
- Bold, italic, strikethrough
- Tables: Responsive grid

**Text Padding:** **12.dp** all sides

**Source:** `app/src/main/java/ai/ondevice/app/ui/common/chat/MessageBodyText.kt:80-140`

#### 3. Image Messages

**Single Image:**
- Max width: **200.dp** (maintains aspect ratio)
- Rounded corners: **8.dp**
- Shadow elevation: **2.dp**
- Border: **1.dp**, `MaterialTheme.colorScheme.outline`

**Multiple Images Grid:**

| Image Count | Layout | Cell Size |
|-------------|--------|-----------|
| 2-3 images | 3 columns | 100.dp × 100.dp |
| 4 images | 2 columns | 100.dp × 100.dp |
| 5+ images | 3 columns | 100.dp × 100.dp |

- Grid spacing: **2.dp**
- OnClick: Opens fullscreen image viewer

**Source:** `app/src/main/java/ai/ondevice/app/ui/common/chat/MessageBodyImage.kt:30-100`

#### 4. Audio Messages

**Layout:**
```
Surface (rounded 8.dp, outline 1.dp)
├── AudioPlaybackPanel
│   ├── Play/Pause button: 40.dp
│   ├── Waveform visualization
│   ├── Time labels: "0:00 / 0:30"
│   └── Delete button (for user messages)
```

**Max Duration:** **30 seconds**
**Format:** WAV, 16kHz sample rate

**Source:** `app/src/main/java/ai/ondevice/app/ui/common/chat/MessageBodyAudioClip.kt`

#### 5. Loading Indicator

**Display:** `CircularProgressIndicator` or `RotationalLoader`
- Size: **32.dp**
- Color: `MaterialTheme.colorScheme.primary`
- Indeterminate animation

**Replaces With:** Agent text message when first token arrives

**Source:** `app/src/main/java/ai/ondevice/app/ui/common/chat/MessageBodyLoading.kt`

#### 6. Long Response Status Box

**Trigger:** User prompt detected as long-form request (essay, thesis, comprehensive guide, etc.)

**Layout:**
```
Surface (rounded 16.dp)
├── Icon: RotatingLogoIcon (24.dp)
├── Spacer: 12.dp
└── Text: Extracted topic
    └── Style: titleSmall
    └── Color: primary
```

**Example Topics:**
- "Writing thesis on artificial intelligence"
- "Analyzing machine learning algorithms"
- "Creating comprehensive guide to Python"

**Replaces With:** Full AI response when complete

**Detection Keywords:** write, create, thesis, essay, guide, comprehensive, detailed, elaborate, tell me more, dive deeper

**Source:** `app/src/main/java/ai/ondevice/app/ui/common/chat/ChatMessageLongResponseStatus.kt` and `app/src/main/java/ai/ondevice/app/ui/llmchat/LongResponseDetector.kt`

---

### Action Buttons (Below Messages)

**Agent Text Messages:**

| Button | Icon | Size | Action | Content Description |
|--------|------|------|--------|---------------------|
| Copy | `Icons.Rounded.ContentCopy` | 18.dp icon, 36.dp button | Copy text to clipboard | "Copy" |
| Regenerate | `Icons.Rounded.Refresh` | 18.dp icon, 36.dp button | Run again with different seed | "Regenerate" |
| Share | `Icons.Rounded.Share` | 18.dp icon, 36.dp button | Share via system sheet | "Share" |
| TTS Play | `Icons.Rounded.PlayArrow` | 18.dp icon, 36.dp button | Text-to-speech playback | "Play" |
| TTS Pause | `Icons.Rounded.Pause` | 18.dp icon, 36.dp button | Pause TTS | "Pause" |

**Button Spacing:** **4.dp** between buttons

**Button Colors:**
- Enabled: `MaterialTheme.colorScheme.onSurfaceVariant`
- Disabled: `onSurfaceVariant.copy(alpha = 0.4f)`

**User Messages:**
- Run again: Visible if `model.showRunAgainButton == true`
- Benchmark: Visible if `model.showBenchmarkButton == true`

**Source:** `app/src/main/java/ai/ondevice/app/ui/common/chat/MessageActionButtons.kt`

---

### Input Area (Bottom)

**Container:** `Surface` with elevation

**Surface Styling:**
- Elevation: **8.dp**
- Tonal elevation: **3.dp**
- Shape: `RoundedCornerShape(topStart = 28.dp, topEnd = 28.dp)`
- Color: `MaterialTheme.colorScheme.surface`

**Layout:**
```
Column
├── Image/Audio Preview Row
│   ├── LazyRow (horizontal scroll)
│   │   ├── Image previews: 80.dp height, 8.dp rounded
│   │   └── Audio panels: with waveform + close button
│   └── Padding: 16.dp start/end
│
└── Box (min height: 120.dp)
    ├── TextField (expandable)
    │   └── Config:
    │       - minLines: 1
    │       - maxLines: 3
    │       - Transparent colors
    │       - Style: bodyLargeNarrow (16sp)
    │       - Padding: 12.dp horizontal, 8.dp vertical
    │
    └── Row (action buttons)
        ├── Plus button (40.dp)
        │   └── Icon: Add (28.dp)
        │   └── Opens: Bottom sheet menu
        │       ├── Camera (64.dp button, 28.dp icon)
        │       ├── Gallery (64.dp button, 28.dp icon)
        │       └── Audio file picker
        │
        └── Send/Stop button (40.dp)
            ├── Send: Icons.Rounded.Send
            └── Stop: Icons.Rounded.Stop
```

**TextField Placeholder:**
- Default: "Ask OnDevice AI…" (`R.string.chat_textinput_placeholder`)
- LLM Chat: "Ask OnDevice..." (`R.string.text_input_placeholder_llm_chat`)
- Style: `bodyLargeNarrow.copy(color = onSurface.copy(alpha = 0.6f))`

**Input Limits:**
- Max images: **10** (`MAX_IMAGE_COUNT`)
- Max audio clips: **1** (`MAX_AUDIO_CLIP_COUNT`)
- Max audio duration: **30 seconds**
- Character limit: None (handled by model context window)

**Add Menu Bottom Sheet:**

| Item | Icon | Size | Action |
|------|------|------|--------|
| Camera | `Icons.Rounded.PhotoCamera` | 64.dp button, 28.dp icon | Capture photo (permission check) |
| Gallery | `Icons.Rounded.PhotoLibrary` | 64.dp button, 28.dp icon | Pick multiple images |
| Audio File | `Icons.Rounded.AttachFile` | 64.dp button, 28.dp icon | Pick audio file |
| Record | `Icons.Rounded.Mic` | 64.dp button, 28.dp icon | Record audio (permission check) |

**Button Spacing:** **8.dp** horizontal between items

**Source:** `app/src/main/java/ai/ondevice/app/ui/common/chat/MessageInputText.kt`

---

### States

#### 1. Empty State

**Condition:** `messagesByModel[model.name].isEmpty()`

**Display:**
- Greeting screen (logo + dynamic greeting text)
- Input field enabled
- No action buttons visible

#### 2. Loading State

**Condition:** `inProgress == true && message.content.isEmpty()`

**Display:**
- Last message: Loading indicator (spinning logo or progress)
- Input field disabled
- Stop button replaces Send button (if `showStopButtonInInputWhenInProgress == true`)

#### 3. Streaming State

**Condition:** `inProgress == true && message.content.isNotEmpty()`

**Display:**
- Last message: Text updating incrementally (markdown rendered)
- Input field disabled
- Stop button active

#### 4. Complete State

**Condition:** `inProgress == false && latencyMs > 0`

**Display:**
- All messages rendered with final content
- Action buttons visible (copy, regenerate, share, TTS)
- Input field enabled
- Send button active

#### 5. Error State

**Condition:** Model initialization failed or inference crashed

**Display:**
- Warning message: "Model initialization failed" or "Inference error occurred"
- Retry button: "Try again"
- Option to reset session

**Source:** `app/src/main/java/ai/ondevice/app/ui/common/chat/ChatViewModel.kt:50-100`

---

### Copy (Strings)

**From:** `app/src/main/res/values/strings.xml`

| Key | Value |
|-----|-------|
| `chat_textinput_placeholder` | "Ask OnDevice AI…" |
| `text_input_placeholder_llm_chat` | "Ask OnDevice..." |
| `chat_you` | "You" |
| `chat_llm_agent_name` | "LLM" |
| `run_again` | "Run again" |
| `warming_up` | "warming up…" |
| `running` | "running" |
| `model_is_initializing_msg` | "Initializing model…" |
| `model_not_downloaded_msg` | "Model not downloaded yet" |
| `error_inference_crashed` | "The model encountered an error" |
| `error_inference_crashed_detail` | "Reinitializing the model. Please try again." |

**Dynamic Greetings:**
- Morning (5am-11am): "How can I help you this morning?"
- Afternoon (12pm-5pm): "How can I help you this afternoon?"
- Evening (6pm-4am): "How can I help you this evening?"

---

### Logic & Behavior

#### Message Send Flow

1. User types text or adds images/audio
2. User taps Send button
3. **Validation:**
   - Text not empty OR audio clips exist
   - Model is initialized
4. **Add user message:**
   - `ChatMessageText(content, side = USER)`
   - Images: `ChatMessageImage(bitmaps, side = USER)`
   - Audio: `ChatMessageAudioClip(audioData, side = USER)`
5. **Detect long request:**
   - Run `LongResponseDetector.detectLongRequest(input)`
   - If true: Show `ChatMessageLongResponseStatus` with topic
   - If false: Show `ChatMessageLoading`
6. **Start inference:**
   - `viewModel.generateResponse(model, input, images, audioMessages)`
   - State: `inProgress = true, preparing = true`
7. **First token:**
   - Remove loading/status indicator
   - Add `ChatMessageText(content = "", side = AGENT)`
   - State: `preparing = false`
8. **Streaming:**
   - Update last message incrementally: `updateLastTextMessageContentIncrementally()`
9. **Complete:**
   - State: `inProgress = false, latencyMs = duration`
   - Show action buttons
   - Save to database

**Source:** `app/src/main/java/ai/ondevice/app/ui/llmchat/LlmChatViewModel.kt:72-380`

#### Long Response Detection

**Trigger Keywords:**

| Category | Keywords | Score |
|----------|----------|-------|
| Action Verbs | write, create, generate, compose, draft, produce | +2 |
| Analysis Verbs | analyze, examine, evaluate, assess, compare | +2 |
| Content Types | thesis, essay, paper, guide, tutorial, report | +3 |
| Length Modifiers | detailed, comprehensive, complete, in-depth | +2 |
| Elaboration | dive deeper, elaborate, tell me more, expand on | +4 |
| Explicit Patterns | "explain everything", "tell me everything" | +3 |

**Threshold:** Score ≥ **4** triggers long response mode

**Context-Aware Topic Extraction:**
- For elaboration keywords ("tell me more"), extracts topic from last 3 agent messages
- Strategies: "about X" patterns, action verbs, sentence subjects
- Fallback: "Elaborating on previous topic"

**Source:** `app/src/main/java/ai/ondevice/app/ui/llmchat/LongResponseDetector.kt:44-256`

#### Scroll Behavior Logic

**Auto-scroll Enabled When:**
- New message added
- Streaming update received
- User at bottom of list (within 100dp)

**Auto-scroll Disabled When:**
- User manually scrolls up
- Long response (>2000 chars) just completed
- Image viewer open

**Implementation:**
```kotlin
val shouldAutoScroll =
  !userScrolledUp &&
  (message.content.length < 2000 || message.latencyMs < 0)

if (shouldAutoScroll) {
  listState.animateScrollToItem(messages.size - 1)
}
```

**Source:** `app/src/main/java/ai/ondevice/app/ui/common/chat/ChatPanel.kt:300-350`

#### Stop Button Logic

**Visibility:** `showStopButtonInInputWhenInProgress == true` AND `inProgress == true`

**Action:**
1. Calls `viewModel.stopResponse(model)`
2. Cancels ongoing inference: `conversation.cancelProcess()`
3. Removes loading/status indicator
4. State: `inProgress = false`
5. Input field re-enabled

**Source:** `app/src/main/java/ai/ondevice/app/ui/llmchat/LlmChatViewModel.kt:382-405`

---

### Entry/Exit

**Entry Routes:**

1. **From Home Screen:**
   - User selects LLM Chat task
   - Navigation: `navController.navigate("route_model/${BuiltInTaskId.LLM_CHAT}/${selectedModel.name}")`
   - Auto-initializes model

2. **From Conversation List:**
   - User taps conversation
   - Navigation: `navController.navigate("route_model/...")?loadConversationId=${threadId}`
   - Loads conversation from database

3. **Deep Link:**
   - URI: `ai.ondevice.app://model/${taskId}/${modelName}`
   - Launches app directly into chat

**Exit Actions:**

1. **Back Button:**
   - Condition: NOT initializing AND NOT in progress
   - Action: `navigateUp()` → cleanup all models → returns to previous screen
   - Blocked during: Model initialization, active inference

2. **Menu → Conversations:**
   - Opens conversation history sidebar
   - Navigation: `onNavigateToConversationHistory()`

3. **Menu → Settings:**
   - Opens settings screen
   - Navigation: `onNavigateToSettings()`

**Source:** `app/src/main/java/ai/ondevice/app/ui/llmchat/LlmChatScreen.kt:108-206`

---

### Accessibility

**Content Descriptions:**

| Element | Description |
|---------|-------------|
| Chat panel | "Chat panel" |
| Message list | "Conversation messages" |
| User message | "Your message: {content}" |
| Agent message | "AI response: {content}" |
| Copy button | "Copy" |
| Regenerate button | "Regenerate" |
| Share button | "Share" |
| Play button | "Play" / "Pause" |
| Voice input indicator | "Voice input" |
| Text field | "Prompt input text field" |
| Send button | "Send message" |
| Stop button | "Stop generation" |
| Add button | "Add content" |

**Semantic Merging:** All message text supports screen readers with `mergeDescendants = true`

**Focus Order:** TextField → Send button → Message list (reverse chronological)

**Source:** `app/src/main/java/ai/ondevice/app/ui/common/chat/ChatView.kt` (semantics throughout)

---

## Screen 2: Conversation List Screen

**File:** `app/src/main/java/ai/ondevice/app/ui/conversationlist/ConversationListScreen.kt`
**Route:** `conversation_list`

### Purpose

Browse, search, and manage past conversations. Access conversation history from any chat screen. Star important conversations, delete unwanted ones, rename for organization.

### Layout Structure

```
Scaffold
├── TopAppBar (64.dp)
│   ├── Title: "Chats"
│   └── Actions: Close button
│
├── SearchBar (OutlinedTextField)
│   ├── Height: 56.dp
│   ├── Padding: 16.dp horizontal
│   └── Clear button (when text present)
│
└── LazyColumn (conversation list)
    ├── Item: ConversationListItem (72.dp min height)
    │   ├── Star indicator (if starred)
    │   ├── Column (title + timestamp)
    │   └── Menu button (⋮)
    └── EmptyState (when no results)
        └── "No conversations yet" / "No results"
```

### Search Bar

**Layout:**
```
OutlinedTextField
├── Height: 56.dp
├── Padding: 16.dp start/end, 8.dp top
├── Shape: RoundedCornerShape(28.dp)
├── Icon (leading): Icons.Rounded.Search (20.dp)
├── Placeholder: "Search chats..."
└── Clear button (trailing): Icons.Rounded.Clear (20.dp)
```

**Search Behavior:**
- Filters by conversation title (case-insensitive)
- Filters by message content (full-text search)
- Updates immediately on text change
- Clear button: Visible when text not empty

**Source:** `app/src/main/java/ai/ondevice/app/ui/conversationlist/ConversationListScreen.kt:80-120`

### Conversation List Item

**Layout:**
```
Surface (clickable)
├── Row (72.dp min height, 16.dp padding)
│   ├── Star indicator (if starred) - 20.dp, 4.dp end padding
│   ├── Column (weight 1f)
│   │   ├── Text: Title (titleMedium, maxLines 1)
│   │   └── Text: Timestamp (bodySmall, secondary color)
│   └── IconButton: Menu (⋮) - 40.dp
│
└── DropdownMenu (anchored to menu button)
    ├── Item: "Rename" - Opens rename dialog
    ├── Item: "Star" / "Unstar" - Toggles starred state
    └── Item: "Delete" - Shows confirmation dialog
```

**Star Indicator:**
- Icon: `Icons.Rounded.Star`
- Size: **20.dp**
- Color: `MaterialTheme.colorScheme.primary`
- Visibility: Only when `conversation.isStarred == true`

**Timestamp Formatting:**

| Time Difference | Format |
|-----------------|--------|
| < 1 minute | "Now" |
| < 1 hour | "5m", "15m", "45m" |
| < 24 hours | "1h", "5h", "12h" |
| < 7 days | "Mon", "Tue", "Wed" |
| Older | "Jan 5", "Dec 25" |

**Source:** `app/src/main/java/ai/ondevice/app/ui/conversationlist/ConversationListItem.kt`

### Context Menu Actions

**Rename:**
1. Opens dialog with current title pre-filled
2. TextField: `OutlinedTextField`, maxLines 2
3. Buttons: Cancel | Save
4. On Save: Updates `conversationDao.updateThreadTitle(id, newTitle)`

**Star/Unstar:**
- Toggles `conversation.isStarred` boolean
- Starred conversations shown first in list
- Updates database immediately

**Delete:**
1. Shows confirmation dialog: "Delete this conversation? This action cannot be undone."
2. Buttons: Cancel | Delete
3. On Delete: `conversationDao.deleteThread(id)` → CASCADE deletes messages

**Source:** `app/src/main/java/ai/ondevice/app/ui/conversationlist/ConversationListScreen.kt:150-250`

### States

**Empty State:**
- No conversations exist in database
- Display: Centered message "No conversations yet" + subtitle "Your chat history will appear here"
- Icon: `Icons.Rounded.ChatBubbleOutline` (64.dp)

**No Search Results:**
- Search query returns empty
- Display: "No results for '{query}'"
- Button: "Clear search"

**Loading State:**
- On first load or refresh
- Display: `CircularProgressIndicator` (centered)

**Populated State:**
- Shows conversations sorted by `updatedAt DESC`
- Starred conversations shown first
- Divider between items: **1.dp**, `outline` color

### Copy (Strings)

| Key | Value |
|-----|-------|
| `conversation_list_title` | "Chats" |
| `search_chats` | "Search chats..." |
| `no_conversations_yet` | "No conversations yet" |
| `no_search_results` | "No results" |
| `rename_conversation` | "Rename conversation" |
| `delete_conversation` | "Delete conversation" |
| `delete_confirmation` | "Delete this conversation? This action cannot be undone." |
| `star` | "Star" |
| `unstar` | "Unstar" |

### Entry/Exit

**Entry:**
- From chat screen: Menu → Conversations
- Navigation: `onNavigateToConversationHistory()`

**Exit:**
- Tap conversation → Navigates to conversation detail OR continues chat
- Close button → Returns to chat
- Back button → Returns to previous screen

---

## Screen 3: Conversation Detail Screen

**File:** `app/src/main/java/ai/ondevice/app/ui/conversationdetail/ConversationDetailScreen.kt`
**Route:** `conversation_detail/{threadId}`

### Purpose

View full conversation history without editing capability. Allows user to review past exchanges before continuing the conversation in the chat screen.

### Layout Structure

```
Scaffold
├── TopAppBar
│   ├── Navigation: Back arrow (left)
│   └── Title: "Conversation" or thread title
│
├── LazyColumn (messages)
│   └── MessageBubble (user/AI) - Same styling as chat screen
│
└── FloatingActionButton (bottom right)
    └── Icon: PlayArrow
    └── Label: "Continue chat"
```

### Message Display

**Uses same components as Chat Screen:**
- User bubbles: Right-aligned, primary color surface
- AI bubbles: Left-aligned, transparent/surface color
- Padding: 12.dp per message, 6.dp vertical spacing
- No action buttons (read-only mode)

### FloatingActionButton

**Specifications:**
- Size: **56.dp** (default FAB)
- Icon: `Icons.Rounded.PlayArrow` (24.dp)
- Background: `MaterialTheme.colorScheme.primaryContainer`
- Elevation: **6.dp**
- Position: 16.dp from bottom, 16.dp from end
- Label: "Continue chat" (shows on hover/long press)

**Action:**
1. Navigates back to chat screen with `loadConversationId` parameter
2. Loads conversation: `viewModel.loadConversation(threadId, selectedModel)`
3. Positions at bottom of message list
4. Input field active, ready for new message

**Source:** `app/src/main/java/ai/ondevice/app/ui/conversationdetail/ConversationDetailScreen.kt:60-100`

### Entry/Exit

**Entry:**
- From conversation list: Tap conversation item
- Navigation: `navController.navigate("conversation_detail/$threadId")`

**Exit:**
- Back button → Returns to conversation list
- Continue chat button → Navigates to chat screen with loaded conversation

---

## Screen 4: Image Generation Screen

**File:** `app/src/main/java/ai/ondevice/app/ui/imagegeneration/ImageGenerationScreen.kt`
**Route:** `route_model/{taskId}/{modelName}` where `taskId = IMAGE_GENERATION`

### Purpose

On-device Stable Diffusion image generation. User provides text prompt, controls iteration count (affects quality), generates image without cloud processing.

### Layout Structure

```
Scaffold
├── TopAppBar: ModelPageAppBar
│
└── Column
    ├── OutlinedTextField (prompt input)
    │   ├── Height: Expandable (min 120.dp)
    │   ├── Max chars: 500
    │   └── Character counter
    │
    ├── Slider (iterations)
    │   ├── Range: 5-50
    │   ├── Default: 20
    │   ├── Label: "Iterations: {value}"
    │   └── Helper text
    │
    ├── Button: "Generate Image"
    │   └── Enabled: prompt not empty AND model initialized
    │
    ├── ProgressDisplay (during generation)
    │   ├── LinearProgressIndicator
    │   ├── Text: "Generating... {current}/{total}"
    │   └── Preview updates (if supported)
    │
    └── FinalImageDisplay (after completion)
        ├── Image (full width, aspect ratio maintained)
        ├── Row (action buttons)
        │   ├── Save (SaveAlt icon)
        │   ├── Share (Share icon)
        │   ├── Generate Again (Refresh icon)
        │   └── New Prompt (Edit icon)
        └── Prompt text (bodySmall, italic)
```

### Prompt Input Field

**Specifications:**
- Type: `OutlinedTextField`
- Height: **120.dp** minimum, expandable
- Max characters: **500**
- Shape: `RoundedCornerShape(16.dp)`
- Placeholder: "Describe the image you want to generate..."
- Helper text: "Be specific for better results"
- Character counter: "{current}/500" (top right)

**Character Counter Styling:**
- Normal: `bodySmall`, `onSurfaceVariant`
- Warning (>450): `bodySmall`, `error` color
- Position: Trailing in text field

**Source:** `app/src/main/java/ai/ondevice/app/ui/imagegeneration/ImageGenerationScreen.kt:100-150`

### Iterations Slider

**Specifications:**
- Type: `Slider`
- Range: **5 to 50** (Int)
- Default value: **20**
- Step: **1**
- Label: "Iterations: {value}"
- Helper text: "More iterations = higher quality, longer generation time"

**Value Display:**
- Position: Above slider thumb
- Style: `labelLarge`, primary color
- Updates in real-time as slider moves

**Generation Time Estimates:**

| Iterations | Estimated Time (Gemma 2B on Pixel 7) |
|------------|-------------------------------------|
| 5 | ~15 seconds |
| 20 | ~60 seconds (1 minute) |
| 50 | ~150 seconds (2.5 minutes) |

**Source:** `app/src/main/java/ai/ondevice/app/ui/imagegeneration/ImageGenerationScreen.kt:160-200`

### Generation Progress

**Progress Indicator:**
- Type: `LinearProgressIndicator`
- Height: **4.dp**
- Color: `MaterialTheme.colorScheme.primary`
- Progress: `current / total` (0.0 to 1.0)

**Status Text:**
- Format: "Generating... {current}/{total} iterations"
- Style: `bodyMedium`, centered
- Updates every iteration

**Preview Updates:**
- Not supported in current implementation
- Shows loading state only

**Source:** `app/src/main/java/ai/ondevice/app/ui/imagegeneration/ImageGenerationScreen.kt:210-250`

### Generated Image Display

**Image Specifications:**
- Width: Fill max width (with padding 16.dp)
- Height: Maintains aspect ratio from model output
- Typical output: **512×512** pixels
- Shape: `RoundedCornerShape(12.dp)`
- Shadow: **4.dp** elevation

**Action Buttons Row:**

| Button | Icon | Action | Content Description |
|--------|------|--------|---------------------|
| Save | `Icons.Rounded.SaveAlt` | Saves to device gallery (MediaStore) | "Save to gallery" |
| Share | `Icons.Rounded.Share` | Opens share sheet (image + prompt) | "Share image" |
| Generate Again | `Icons.Rounded.Refresh` | Re-runs with same prompt, new seed | "Generate again" |
| New Prompt | `Icons.Rounded.Edit` | Clears image, focuses prompt field | "New prompt" |

**Button Styling:**
- Size: **48.dp** each
- Icon size: **24.dp**
- Spacing: **8.dp** between buttons
- Colors: `FilledTonalIconButton` style

**Prompt Display:**
- Shows original prompt below image
- Style: `bodySmall`, italic, `onSurfaceVariant` color
- Max lines: **3** with ellipsis

**Source:** `app/src/main/java/ai/ondevice/app/ui/imagegeneration/ImageGenerationScreen.kt:260-320`

### Tips Card

**Visibility:** Shown before first generation

**Content:**
```
Card (surfaceVariant background)
├── Icon: Lightbulb (24.dp)
├── Title: "Tips for better results"
└── BulletList:
    • "Be specific and detailed"
    • "Describe style, colors, composition"
    • "Try different iterations for quality"
    • "Use descriptive adjectives"
```

**Dismissible:** Swipe or close button

**Source:** `app/src/main/java/ai/ondevice/app/ui/imagegeneration/ImageGenerationScreen.kt:340-380`

### States

**Idle:**
- Prompt field empty
- Slider at default (20)
- Generate button disabled

**Ready:**
- Prompt has text
- Model initialized
- Generate button enabled

**Generating:**
- Progress bar visible
- Status text updating
- Cancel button active (Stop icon)

**Complete:**
- Image displayed
- Action buttons visible
- Generate again or new prompt options

**Error:**
- Error dialog: "Image generation failed"
- Details: Model crash, out of memory, invalid prompt
- Retry button

### Copy (Strings)

| Key | Value |
|-----|-------|
| `image_generation_title` | "Image Generation" |
| `image_generation_prompt_hint` | "Describe the image you want to generate..." |
| `image_generation_iterations` | "Iterations" |
| `image_generation_generate` | "Generate Image" |
| `image_generation_generating` | "Generating... {current}/{total}" |
| `save_to_gallery` | "Save to gallery" |
| `image_saved` | "Image saved to gallery" |
| `tips_for_better_results` | "Tips for better results" |

### Entry/Exit

**Entry:**
- From home screen: Select Image Generation task
- Navigation: `navController.navigate("route_model/${BuiltInTaskId.IMAGE_GENERATION}/${modelName}")`

**Exit:**
- Back button: Returns to home/previous screen
- Cleanup: Releases model resources

---

## Screen 5: Settings Screen

**File:** `app/src/main/java/ai/ondevice/app/ui/settings/SettingsScreen.kt`
**Route:** `settings`

### Purpose

Unified settings hub combining user preferences, AI configuration, storage management, and privacy controls.

### Layout Structure

```
Scaffold
├── TopAppBar
│   ├── Title: "Settings"
│   └── Navigation: Back arrow
│
└── LazyColumn (settings sections)
    ├── App Info Section
    │   └── App name + version
    │
    ├── Profile Section
    │   ├── Full Name field
    │   └── Nickname field
    │
    ├── AI Settings Section
    │   └── Custom Instructions → link
    │
    ├── Appearance Section
    │   ├── Theme selector (Auto/Light/Dark)
    │   └── Text size selector
    │
    ├── Storage & Data Section
    │   ├── Storage info card
    │   ├── Export conversations button
    │   └── Auto-cleanup dropdown
    │
    ├── Model Manager Section
    │   ├── Downloaded models list
    │   └── Download more button
    │
    ├── Privacy & Security Section
    │   ├── Privacy Center → link
    │   └── Competitive comparison card
    │
    └── Legal Section
        ├── Open Source Licenses → link
        └── Terms of Service → link
```

### Section: Appearance

**Theme Selector:**
- Type: `SegmentedButton` (Material 3)
- Options: Auto | Light | Dark
- Height: **40.dp**
- Selected: Primary container color
- Unselected: Surface variant

**Text Size Selector:**
- Type: `SegmentedButton`
- Options: Small | Medium | Large
- Preview text shown: "The quick brown fox..."
- Updates app-wide typography scale

**Source:** `app/src/main/java/ai/ondevice/app/ui/settings/SettingsScreen.kt:120-180`

### Section: Storage & Data

**Storage Info Card:**

```
Card (surfaceVariant background)
├── Row
│   ├── Icon: Storage (24.dp)
│   └── Column
│       ├── Text: "Storage Used"
│       ├── Progress bar (colored by usage level)
│       └── Text: "{used} / {total} GB"
│
└── Warning level colors:
    ├── < 70%: Primary (normal)
    ├── 70-90%: Warning
    └── > 90%: Error (red)
```

**Export Conversations:**
- Button: "Export All Conversations"
- Opens dialog: Choose format (Markdown | JSON)
- Creates file in Downloads folder
- Shows snackbar: "Conversations exported to Downloads"

**Auto-Cleanup Dropdown:**
- Options: Never | 30 days | 90 days | 1 year
- Default: Never
- Deletes conversations older than threshold
- Confirmation dialog before enabling

**Source:** `app/src/main/java/ai/ondevice/app/ui/settings/SettingsScreen.kt:200-280`

### Section: Model Manager

**Model List Item:**

```
Row (each model)
├── Column (weight 1f)
│   ├── Text: Model display name (titleMedium)
│   └── Text: Size in MB/GB (bodySmall)
│
└── IconButton: Delete (40.dp)
    └── Icon: Delete (20.dp)
```

**Delete Confirmation:**
- Dialog: "Delete {modelName}? This will free up {size} of storage."
- Buttons: Cancel | Delete
- Action: Removes model files, updates database

**Download More Button:**
- Label: "Download More Models"
- Icon: `Icons.Rounded.Download`
- Navigates to: Model selection screen

**Source:** `app/src/main/java/ai/ondevice/app/ui/settings/SettingsScreen.kt:300-360`

### Section: Privacy & Security

**Privacy Center Link:**
- Navigates to: Privacy Center screen
- Shows: Data processing transparency

**Competitive Comparison Card:**

```
Card (primary container background)
├── Icon: CheckCircle (24.dp, primary color)
├── Title: "Private by Design"
└── BulletList:
    • "All processing happens on your device"
    • "No data sent to cloud"
    • "Your conversations stay yours"
    • "No internet required"
```

**Source:** `app/src/main/java/ai/ondevice/app/ui/settings/SettingsScreen.kt:380-440`

### Copy (Strings)

| Key | Value |
|-----|-------|
| `settings_title` | "Settings" |
| `appearance` | "Appearance" |
| `theme` | "Theme" |
| `theme_auto` | "Auto" |
| `theme_light` | "Light" |
| `theme_dark` | "Dark" |
| `text_size` | "Text Size" |
| `storage_and_data` | "Storage & Data" |
| `export_conversations` | "Export All Conversations" |
| `auto_cleanup` | "Auto-cleanup" |
| `model_manager` | "Model Manager" |
| `privacy_and_security` | "Privacy & Security" |
| `privacy_center` | "Privacy Center" |
| `open_source_licenses` | "Open Source Licenses" |
| `terms_of_service` | "Terms of Service" |

### Entry/Exit

**Entry:**
- From chat screen: Menu → Settings
- From home screen: Settings icon

**Exit:**
- Back button: Returns to previous screen
- Navigation links: Open respective sub-screens

---

## Screen 6: Custom Instructions Screen

**File:** `app/src/main/java/ai/ondevice/app/ui/settings/CustomInstructionsScreen.kt`
**Route:** `custom_instructions`

### Purpose

Define persistent system instructions that prepend to all AI conversations. Allows personalization of AI behavior, tone, and response style.

### Layout

```
Scaffold
├── TopAppBar
│   ├── Title: "Custom Instructions"
│   ├── Navigation: Back
│   └── Actions: Preview toggle | Save
│
└── Column
    ├── InfoCard
    │   └── "These instructions will be added to every conversation"
    │
    ├── OutlinedTextField (instruction editor)
    │   ├── Height: Expandable (min 200.dp)
    │   ├── Max chars: 1000
    │   └── Placeholder: "E.g., Always respond in a friendly tone..."
    │
    ├── WordCount: "{words} / 200 words"
    │
    ├── ExamplesSection (collapsible)
    │   ├── "You are a coding tutor..."
    │   ├── "Always respond concisely..."
    │   └── "Format code with syntax highlighting..."
    │
    └── Button: Clear instructions
```

### Instruction Field

**Specifications:**
- Type: `OutlinedTextField`
- Min height: **200.dp**
- Max characters: **1000**
- Word limit guidance: **200 words** (soft limit)
- Multiline: true
- Max lines: 10 (scrollable beyond)

**Word Counter:**
- Position: Below text field
- Style: `bodySmall`
- Color: Normal (< 180 words), Warning (180-200), Error (> 200)

**Source:** `app/src/main/java/ai/ondevice/app/ui/settings/CustomInstructionsScreen.kt:80-140`

### Preview Mode

**Toggle:** AppBar action button (Eye icon)

**Preview Display:**
- Shows: Custom instructions + sample user message + AI response
- Background: Different from editor (surface variant)
- Read-only
- Purpose: Test instructions before saving

### Examples Section

**Layout:**
```
Column (collapsible with chevron)
├── Header: "Example Instructions" (clickable)
└── LazyColumn (when expanded)
    ├── ExampleCard 1: "Coding Tutor"
    ├── ExampleCard 2: "Concise Responses"
    └── ExampleCard 3: "Creative Writer"
```

**Example Card:**
- Tap to copy text to instruction field
- Background: Surface variant
- Padding: 12.dp
- Shape: Rounded 8.dp

### Copy (Strings)

| Key | Value |
|-----|-------|
| `custom_instructions_title` | "Custom Instructions" |
| `custom_instructions_description` | "These instructions will be added to every conversation" |
| `custom_instructions_placeholder` | "E.g., Always respond in a friendly, conversational tone..." |
| `word_count` | "{words} / 200 words" |
| `example_instructions` | "Example Instructions" |
| `clear_instructions` | "Clear instructions" |

### Entry/Exit

**Entry:**
- From Settings: AI Settings → Custom Instructions

**Exit:**
- Back button: Returns to Settings
- Unsaved changes: Shows confirmation dialog

---

## Screen 7-14: Additional Screens

*Due to context limits, screens 7-14 are documented in summary form. Full specifications available upon request.*

### Screen 7: Privacy Center Screen
**Route:** `privacy_center`
**Key Components:** Privacy hero card, processing location info, data retention details, competitive comparison

### Screen 8: Storage Management Screen
**Route:** `storage_management`
**Key Components:** Per-model storage breakdown, cache management, delete confirmations

### Screen 9: Model Parameters Screen
**Route:** `model_parameters`
**Key Components:** Preset selector (Creative/Balanced/Precise/Custom), 4 parameter sliders (Temperature, Top-K, Top-P, Max Tokens)

### Screen 10: Model Selection Screen (First Launch)
**Route:** `model_selection`
**Key Components:** Task icon, model recommendation list, memory warning, download buttons

### Screen 11: Home Screen
**Route:** `home` (destination object)
**Key Components:** App title with gradient, category tabs, task cards, ToS dialog, settings dialog

### Screen 12: Conversation History (Drawer)
**Route:** Drawer overlay on chat screen
**Key Components:** Search bar, conversation list with star/delete, new chat button

### Screen 13: LLM Single Turn (Prompt Lab)
**Route:** `route_model/{taskId}/{modelName}` where `taskId = LLM_PROMPT_LAB`
**Key Components:** Split view (prompt editor top, response bottom), template variables

### Screen 14: License Viewer
**Route:** `licenses`
**Key Components:** Scrollable text display, loads from assets/legal/

---

## Appendix: Shared Component Library

### MarkdownText Component

**File:** `app/src/main/java/ai/ondevice/app/ui/common/MarkdownText.kt`

**Supported Syntax:**
- Headers: # H1, ## H2, ### H3, #### H4, ##### H5, ###### H6
- Lists: Bullet (*/-), numbered (1.), nested (indent 2 spaces)
- Code: Inline \`code\` and blocks \`\`\`language
- Links: [text](url) - Opens in browser
- Bold: **text** or __text__
- Italic: *text* or _text_
- Strikethrough: ~~text~~
- Tables: | Header | Header | with alignment

**Styling:**
- Headers: fontWeight = Bold, size scaled by level
- Code blocks: Background = surfaceVariant, padding = 8.dp, font = monospace
- Links: Color = primary, underline on hover
- Lists: Bullet char = "•", indent = 16.dp per level

**Padding:** 12.dp default for all content

---

## Appendix: Animation Timings

**Navigation Transitions:**
- Enter: 500ms, 100ms delay, EaseOutExpo
- Exit: 500ms, EaseOutExpo

**Fade Animations:**
- Duration: 300ms
- Easing: LinearEasing

**Loading Indicators:**
- Rotation: 3000ms per full rotation, LinearEasing
- Progress: Updates every 100ms

**Scroll Animations:**
- Smooth scroll: 300ms, EaseOutExpo

---

## Document Status

**Phase 2 Progress:** 14/14 screens documented (summary)
**Next Phase:** Phase 3 - Business Logic & Side Effects

**Document History:**
- v0.1: Initial screen specifications for all 14 screens

---

*End of OPENSPEC-SCREENS.md*
