# Yi iOS App Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an iOS 17 SwiftUI app for automatic, half-duplex face-to-face translation between Chinese and English, Japanese, or Korean through Yi Gateway.

**Architecture:** A pure Swift client separates microphone capture, gateway transport, audio playback, conversation state, and SwiftUI rendering behind protocols. A deterministic `ConversationStore` owns all state changes on the main actor. Development starts against a mock transport, so UI and audio behavior can be tested before live cloud credentials exist.

**Tech Stack:** Swift 6, SwiftUI, AVFoundation, URLSessionWebSocketTask, XCTest, XcodeGen, iOS 17+.

## Global Constraints

- Support iOS 17 and newer.
- Never embed a Volcengine credential in source, plist files, build settings, or network logs.
- Capture PCM signed 16-bit little-endian, 16 kHz, mono audio in approximately 80 ms frames.
- Do not upload microphone audio while translated audio is playing.
- Support Chinese paired with English, Japanese, or Korean.
- Keep the main conversation usable without normal manual direction switching.
- Store no transcript after conversation end unless the user explicitly saves it.
- Use Chinese user-facing copy in the first release.

---

## Planned File Structure

```text
ios/
  project.yml
  Yi/App/YiApp.swift
  Yi/App/AppEnvironment.swift
  Yi/Conversation/ConversationModels.swift
  Yi/Conversation/ConversationStore.swift
  Yi/Conversation/ConversationView.swift
  Yi/Conversation/PartnerPanel.swift
  Yi/Conversation/LocalPanel.swift
  Yi/Conversation/ConversationStatusView.swift
  Yi/Audio/MicrophoneCapture.swift
  Yi/Audio/AudioPlaybackQueue.swift
  Yi/Audio/AudioSessionController.swift
  Yi/Gateway/GatewayClient.swift
  Yi/Gateway/GatewayMessages.swift
  Yi/History/TranscriptExport.swift
  Yi/Resources/Assets.xcassets/
  Yi/Resources/Info.plist
  YiTests/ConversationStoreTests.swift
  YiTests/GatewayMessagesTests.swift
  YiTests/AudioFrameTests.swift
  YiUITests/ConversationFlowUITests.swift
```

### Task 1: Create the project and domain model

**Files:**
- Create: `ios/project.yml`
- Create: `ios/Yi/App/YiApp.swift`
- Create: `ios/Yi/App/AppEnvironment.swift`
- Create: `ios/Yi/Conversation/ConversationModels.swift`
- Create: `ios/Yi/Resources/Info.plist`
- Test: `ios/YiTests/GatewayMessagesTests.swift`

**Interfaces:**
- Produces: `LanguagePair`, `Direction`, `ConversationPhase`, `Turn`, and dependency protocols.
- Consumes: none.

- [ ] **Step 1: Define XcodeGen targets and write failing model tests**

```swift
func testDirectionOpposite() {
    XCTAssertEqual(Direction.foreignToNative.opposite, .nativeToForeign)
    XCTAssertEqual(Direction.nativeToForeign.opposite, .foreignToNative)
}

func testSupportedLanguagePairs() {
    XCTAssertEqual(LanguagePair.supported.map(\.foreignCode), ["en", "ja", "ko"])
}
```

- [ ] **Step 2: Generate the project and confirm the test target fails to compile**

Run on macOS: `cd ios && xcodegen generate && xcodebuild test -scheme Yi -destination 'platform=iOS Simulator,name=iPhone 16'`  
Expected: FAIL because domain types do not exist.

- [ ] **Step 3: Implement the domain model**

```swift
enum Direction: String, Codable, Sendable {
    case foreignToNative = "foreign_to_native"
    case nativeToForeign = "native_to_foreign"
    var opposite: Self { self == .foreignToNative ? .nativeToForeign : .foreignToNative }
}

struct LanguagePair: Equatable, Sendable {
    let foreignCode: String
    let displayName: String
    static let supported = [
        Self(foreignCode: "en", displayName: "英语"),
        Self(foreignCode: "ja", displayName: "日语"),
        Self(foreignCode: "ko", displayName: "韩语")
    ]
}

enum ConversationPhase: Equatable, Sendable {
    case idle, connecting, listening(Direction), translating(Direction)
    case playing(Direction), reconnecting(attempt: Int), failed(message: String)
}
```

- [ ] **Step 4: Add `NSMicrophoneUsageDescription` and foreground-only audio capabilities**

Use copy: `Yi 需要使用麦克风，把面对面对话实时翻译成双方能听懂的语言。`

- [ ] **Step 5: Run and commit**

Run: the Task 1 `xcodebuild test` command.  
Expected: model tests PASS.  
Commit: `git commit -am "feat(ios): create Yi app domain model"`

### Task 2: Implement microphone capture and playback isolation

**Files:**
- Create: `ios/Yi/Audio/MicrophoneCapture.swift`
- Create: `ios/Yi/Audio/AudioPlaybackQueue.swift`
- Create: `ios/Yi/Audio/AudioSessionController.swift`
- Test: `ios/YiTests/AudioFrameTests.swift`

**Interfaces:**
- Produces: `AsyncThrowingStream<Data, Error>` of 2,560-byte PCM frames and `AudioPlaying` callbacks.
- Consumes: start/stop commands from `ConversationStore`.

- [ ] **Step 1: Write failing PCM framing tests**

```swift
func testFrameSizeIsEightyMilliseconds() {
    XCTAssertEqual(PCMFormat.bytesPerFrame, 2_560)
}

func testFramerKeepsRemainder() {
    var framer = PCMFramer(frameSize: 4)
    XCTAssertEqual(framer.append(Data([1,2,3])).count, 0)
    XCTAssertEqual(framer.append(Data([4,5,6])).first, Data([1,2,3,4]))
    XCTAssertEqual(framer.remainder, Data([5,6]))
}
```

- [ ] **Step 2: Confirm red state**

Run: `xcodebuild test -scheme Yi -only-testing:YiTests/AudioFrameTests -destination 'platform=iOS Simulator,name=iPhone 16'`  
Expected: FAIL because `PCMFormat` and `PCMFramer` do not exist.

- [ ] **Step 3: Implement framing and capture**

```swift
enum PCMFormat {
    static let sampleRate: Double = 16_000
    static let channels: AVAudioChannelCount = 1
    static let bytesPerFrame = 16_000 * 2 * 80 / 1_000
}

struct PCMFramer {
    let frameSize: Int
    private(set) var remainder = Data()
    mutating func append(_ data: Data) -> [Data] {
        remainder.append(data)
        var frames: [Data] = []
        while remainder.count >= frameSize {
            frames.append(remainder.prefix(frameSize))
            remainder.removeFirst(frameSize)
        }
        return frames
    }
}
```

Configure `AVAudioSession` as `.playAndRecord`, mode `.voiceChat`, options `[.defaultToSpeaker, .allowBluetoothHFP]`. Convert the input tap to 16 kHz mono PCM using `AVAudioConverter`.

- [ ] **Step 4: Implement serialized playback**

`AudioPlaybackQueue` must call `onPlaybackStarted` before scheduling the first buffer and `onPlaybackFinished` only after the last buffer drains. `ConversationStore` stops capture before enqueueing audio and restarts only after the callback.

- [ ] **Step 5: Run and commit**

Run: `xcodebuild test -scheme Yi -only-testing:YiTests/AudioFrameTests -destination 'platform=iOS Simulator,name=iPhone 16'`  
Expected: PASS.  
Commit: `git commit -am "feat(ios): add isolated capture and playback"`

### Task 3: Implement the Yi Gateway transport

**Files:**
- Create: `ios/Yi/Gateway/GatewayMessages.swift`
- Create: `ios/Yi/Gateway/GatewayClient.swift`
- Test: `ios/YiTests/GatewayMessagesTests.swift`

**Interfaces:**
- Consumes: `StartMessage`, PCM `Data`, `turn.end`, `direction.set`, and `conversation.end`.
- Produces: `AsyncThrowingStream<GatewayEvent, Error>`.

- [ ] **Step 1: Write failing JSON contract tests**

```swift
func testStartEncodingMatchesGatewayContract() throws {
    let value = StartMessage(conversationId: "c1", foreignLanguage: "ja", direction: .foreignToNative)
    let json = try JSONSerialization.jsonObject(with: JSONEncoder().encode(value)) as! [String: Any]
    XCTAssertEqual(json["type"] as? String, "start")
    XCTAssertEqual(json["nativeLanguage"] as? String, "zh")
    XCTAssertEqual((json["audio"] as? [String: Any])?["sampleRate"] as? Int, 16_000)
}
```

- [ ] **Step 2: Confirm red state, then implement exact `Codable` messages**

Model every event defined by the gateway plan and decode unknown future events as `.unsupported(type:)` rather than crashing.

- [ ] **Step 3: Implement `URLSessionWebSocketTask` transport**

```swift
protocol GatewayConnecting: Sendable {
    func connect(start: StartMessage) async throws -> AsyncThrowingStream<GatewayEvent, Error>
    func sendAudio(_ data: Data) async throws
    func send(_ control: ClientControl) async throws
    func disconnect() async
}
```

Use bearer install token authorization, enforce `wss` outside Debug builds, and never log frame bodies or authorization headers.

- [ ] **Step 4: Add cancellation and reconnect tests with `URLProtocol`/mock transport**

Verify one receive loop, one close, and no frame send after cancellation.

- [ ] **Step 5: Run and commit**

Run: `xcodebuild test -scheme Yi -only-testing:YiTests/GatewayMessagesTests -destination 'platform=iOS Simulator,name=iPhone 16'`  
Expected: PASS.  
Commit: `git commit -am "feat(ios): add Yi Gateway client"`

### Task 4: Build the conversation store with TDD

**Files:**
- Create: `ios/Yi/Conversation/ConversationStore.swift`
- Test: `ios/YiTests/ConversationStoreTests.swift`

**Interfaces:**
- Consumes: `GatewayConnecting`, `MicrophoneCapturing`, `AudioPlaying`, and gateway events.
- Produces: observable `phase`, `direction`, `sourceText`, `translatedText`, `turns`, and user intents.

- [ ] **Step 1: Write a failing successful-turn test**

```swift
@MainActor
func testPlaybackCompletionAutomaticallyFlipsDirection() async throws {
    let gateway = GatewayMock()
    let audio = AudioPlayerMock()
    let store = ConversationStore(gateway: gateway, microphone: MicrophoneMock(), player: audio)
    await store.start(language: .supported[0])
    gateway.emit(.sessionReady)
    gateway.emit(.translationFinal(source: "Hello", translated: "你好", direction: .foreignToNative))
    gateway.emit(.playbackStarted)
    XCTAssertEqual(store.phase, .playing(.foreignToNative))
    audio.finish()
    XCTAssertEqual(store.direction, .nativeToForeign)
    XCTAssertEqual(store.phase, .listening(.nativeToForeign))
}
```

- [ ] **Step 2: Confirm the test fails, then implement the minimum reducer**

All public intents run on `@MainActor`; gateway and audio callbacks are converted to explicit `Action` values before mutation.

- [ ] **Step 3: Add tests for manual correction, reconnect attempts, TTS text-only fallback, audio interruption, empty speech, and end conversation**

Assert that the microphone mock is never active while `phase` is `.playing`.

- [ ] **Step 4: Implement capped reconnect behavior**

Use delays of 0.5, 1, and 2 seconds through an injected clock. After the third failure, set `.failed(message: "网络连接不稳定，请稍后重试。")`.

- [ ] **Step 5: Run and commit**

Run: `xcodebuild test -scheme Yi -only-testing:YiTests/ConversationStoreTests -destination 'platform=iOS Simulator,name=iPhone 16'`  
Expected: PASS.  
Commit: `git commit -am "feat(ios): add conversation state store"`

### Task 5: Implement the face-to-face SwiftUI interface

**Files:**
- Create: `ios/Yi/Conversation/ConversationView.swift`
- Create: `ios/Yi/Conversation/PartnerPanel.swift`
- Create: `ios/Yi/Conversation/LocalPanel.swift`
- Create: `ios/Yi/Conversation/ConversationStatusView.swift`
- Modify: `ios/Yi/App/YiApp.swift`
- Test: `ios/YiUITests/ConversationFlowUITests.swift`

**Interfaces:**
- Consumes: `ConversationStore` observable state and intents.
- Produces: launch, language selection, conversation, error, and transcript interfaces.

- [ ] **Step 1: Write failing accessibility UI tests**

```swift
func testConversationScreenExposesCurrentStateAndCorrection() {
    let app = XCUIApplication()
    app.launchArguments = ["-ui-test-state", "listening-english"]
    app.launch()
    XCTAssertTrue(app.staticTexts["正在听对方说话"].exists)
    XCTAssertTrue(app.buttons["纠正翻译方向"].exists)
    XCTAssertTrue(app.buttons["结束对话"].exists)
}
```

- [ ] **Step 2: Confirm the UI test fails**

Run: `xcodebuild test -scheme Yi -only-testing:YiUITests/ConversationFlowUITests -destination 'platform=iOS Simulator,name=iPhone 16'`  
Expected: FAIL because the screen is absent.

- [ ] **Step 3: Implement the split orientation layout**

```swift
struct ConversationView: View {
    @Bindable var store: ConversationStore
    var body: some View {
        ZStack {
            Color.yiBackground.ignoresSafeArea()
            VStack(spacing: 0) {
                PartnerPanel(store: store).rotationEffect(.degrees(180))
                ConversationStatusView(phase: store.phase, direction: store.direction)
                LocalPanel(store: store)
            }
        }
    }
}
```

Use the permanent prototype as the visual reference. Add explicit accessibility labels and do not rotate VoiceOver reading order; when VoiceOver is active, offer a non-rotated accessible presentation.

- [ ] **Step 4: Implement visual tokens and state-specific motion**

Background `#061016`, surface `#102832`, active mint `#49E2C6`, primary text `#F5FBFF`, secondary text `#8FA7B5`, recoverable error `#FF806E`. Respect Reduce Motion by replacing waveform animation with a static level indicator.

- [ ] **Step 5: Run UI tests and commit**

Run: `xcodebuild test -scheme Yi -only-testing:YiUITests -destination 'platform=iOS Simulator,name=iPhone 16'`  
Expected: PASS.  
Commit: `git commit -am "feat(ios): add face-to-face conversation UI"`

### Task 6: Add transcript handling, privacy, and release verification

**Files:**
- Create: `ios/Yi/History/TranscriptExport.swift`
- Create: `ios/PRIVACY.md`
- Create: `ios/README.md`
- Modify: `ios/Yi/Conversation/ConversationView.swift`
- Test: `ios/YiTests/TranscriptExportTests.swift`

**Interfaces:**
- Consumes: completed in-memory `Turn` values.
- Produces: explicit user-triggered UTF-8 text export; otherwise no persistence.

- [ ] **Step 1: Write failing transcript formatting and deletion tests**

Verify source and translation text are formatted with language and timestamp, raw audio is never accepted by the exporter, and `ConversationStore.end()` clears in-memory turns after an optional export completes.

- [ ] **Step 2: Implement explicit export only**

Present `ShareLink` only after the user chooses “保存本次文字记录”. Never write a transcript file during a live conversation.

- [ ] **Step 3: Document privacy and setup**

Explain cloud audio processing, no default persistence, required gateway URL/token setup, simulator limitations, and real-device audio test steps.

- [ ] **Step 4: Run the full automated suite**

Run: `cd ios && xcodegen generate && xcodebuild test -scheme Yi -destination 'platform=iOS Simulator,name=iPhone 16'`  
Expected: all unit and UI tests PASS.

- [ ] **Step 5: Run the required real-device matrix**

Test iPhone speaker, wired/USB-C audio, AirPods Bluetooth HFP, audio interruption, screen rotation lock, English/Japanese/Korean 20-turn conversations, weak network, and gateway restart. Record median and p95 sentence-end-to-audio latency; median must be at most 2.5 seconds.

- [ ] **Step 6: Commit**

Commit: `git commit -am "docs(ios): add privacy and release verification"`
