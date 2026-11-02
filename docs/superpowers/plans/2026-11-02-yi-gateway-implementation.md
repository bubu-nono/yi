# Yi Gateway Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a secure Go WebSocket gateway that accepts Yi iPhone audio, translates Chinese bidirectionally with English, Japanese, or Korean, and streams normalized subtitle and audio events back to the app.

**Architecture:** The gateway owns cloud credentials and exposes one stable client protocol. A conversation actor serializes state transitions, delegates English/Japanese speech-to-speech to the Volcengine AST adapter, and routes Korean through speech-to-text plus a TTS adapter. Provider-specific Protobuf and error codes never leak into the iOS domain model.

**Tech Stack:** Go 1.24, `net/http`, `github.com/coder/websocket`, generated Protobuf types, structured `log/slog`, table-driven Go tests.

## Global Constraints

- Never send a Volcengine API key to a client or include it in logs.
- Accept only PCM signed 16-bit little-endian, 16 kHz, mono audio frames.
- Treat one client binary frame as approximately 80 ms of audio (2,560 bytes).
- Do not persist raw audio.
- Support `zh` paired with exactly one of `en`, `ja`, or `ko`.
- English and Japanese use AST S2S; Korean uses AST S2T plus TTS.
- One conversation is half duplex: receiving and playback may not be active simultaneously.

---

## Planned File Structure

```text
gateway/
  cmd/server/main.go                 process setup and HTTP server lifecycle
  internal/config/config.go          validated environment configuration
  internal/protocol/messages.go      client JSON message types and validation
  internal/protocol/messages_test.go protocol contract tests
  internal/conversation/actor.go     half-duplex conversation state machine
  internal/conversation/actor_test.go deterministic state transition tests
  internal/provider/provider.go      provider-neutral translation interfaces
  internal/provider/volc/ast.go      Volcengine AST WebSocket adapter
  internal/provider/volc/ast_test.go adapter tests with fake WebSocket server
  internal/provider/volc/proto/      generated vendor Protobuf code
  internal/provider/tts/tts.go       Korean-capable TTS interface and adapter
  internal/server/websocket.go       iPhone WebSocket endpoint and frame pump
  internal/server/websocket_test.go  end-to-end protocol tests
  internal/security/limiter.go       install-token and concurrency limits
  .env.example                       safe configuration names only
  go.mod
```

### Task 1: Define and validate the Yi client protocol

**Files:**
- Create: `gateway/go.mod`
- Create: `gateway/internal/protocol/messages.go`
- Create: `gateway/internal/protocol/messages_test.go`

**Interfaces:**
- Produces: `protocol.Start`, `protocol.ClientControl`, `protocol.ServerEvent`, `protocol.Direction`, and `Start.Validate() error`.
- Consumes: none.

- [ ] **Step 1: Initialize the module and write failing validation tests**

```go
func TestStartValidate(t *testing.T) {
    valid := Start{
        Type: "start", ConversationID: "c-1",
        NativeLanguage: "zh", ForeignLanguage: "ja",
        Direction: ForeignToNative,
        Audio: AudioConfig{Encoding: "pcm_s16le", SampleRate: 16000, Channels: 1},
    }
    if err := valid.Validate(); err != nil { t.Fatalf("valid start: %v", err) }

    invalid := valid
    invalid.ForeignLanguage = "fr"
    if err := invalid.Validate(); err == nil { t.Fatal("expected unsupported language error") }
}
```

- [ ] **Step 2: Run the focused test and confirm the red state**

Run: `cd gateway && go test ./internal/protocol -run TestStartValidate -v`  
Expected: FAIL because `Start` and its validation do not exist.

- [ ] **Step 3: Implement the protocol types and strict validation**

```go
type Direction string
const (
    ForeignToNative Direction = "foreign_to_native"
    NativeToForeign Direction = "native_to_foreign"
)

type AudioConfig struct {
    Encoding  string `json:"encoding"`
    SampleRate int   `json:"sampleRate"`
    Channels   int   `json:"channels"`
}

type Start struct {
    Type            string      `json:"type"`
    ConversationID  string      `json:"conversationId"`
    NativeLanguage  string      `json:"nativeLanguage"`
    ForeignLanguage string      `json:"foreignLanguage"`
    Direction       Direction   `json:"direction"`
    Audio           AudioConfig `json:"audio"`
}

func (s Start) Validate() error {
    if s.Type != "start" || s.ConversationID == "" || s.NativeLanguage != "zh" {
        return errors.New("invalid start envelope")
    }
    if s.ForeignLanguage != "en" && s.ForeignLanguage != "ja" && s.ForeignLanguage != "ko" {
        return errors.New("unsupported foreign language")
    }
    if s.Direction != ForeignToNative && s.Direction != NativeToForeign {
        return errors.New("invalid direction")
    }
    if s.Audio != (AudioConfig{Encoding: "pcm_s16le", SampleRate: 16000, Channels: 1}) {
        return errors.New("unsupported audio format")
    }
    return nil
}
```

- [ ] **Step 4: Add JSON round-trip tests for `turn.end`, `direction.set`, `conversation.end`, transcript, playback, usage, and error events**

Use concrete event names: `session.ready`, `transcript.partial`, `translation.final`, `playback.started`, `playback.finished`, `direction.changed`, `usage`, and `error`.

- [ ] **Step 5: Run and commit**

Run: `cd gateway && go test ./internal/protocol -v`  
Expected: PASS.  
Commit: `git commit -am "feat(gateway): define client protocol"`

### Task 2: Implement the deterministic conversation actor

**Files:**
- Create: `gateway/internal/conversation/actor.go`
- Create: `gateway/internal/conversation/actor_test.go`
- Create: `gateway/internal/provider/provider.go`

**Interfaces:**
- Consumes: `protocol.Direction` and normalized client controls.
- Produces: `Actor.Handle(Event) ([]Command, error)` and provider-neutral `Translator`/`Synthesizer` interfaces.

- [ ] **Step 1: Write failing table-driven state tests**

```go
func TestActorCompletesOneTurn(t *testing.T) {
    a := NewActor(protocol.ForeignToNative)
    mustState(t, a, WaitingForSpeech)
    mustHandle(t, a, SpeechStarted{})
    mustState(t, a, Receiving)
    mustHandle(t, a, SpeechEnded{})
    mustState(t, a, Translating)
    mustHandle(t, a, TranslationReady{Text: "你好", HasAudio: true})
    mustState(t, a, Playing)
    cmds := mustHandle(t, a, PlaybackFinished{})
    mustState(t, a, WaitingForSpeech)
    if cmds[0] != (SetDirection{Direction: protocol.NativeToForeign}) {
        t.Fatalf("unexpected command: %#v", cmds[0])
    }
}
```

- [ ] **Step 2: Confirm the test fails**

Run: `cd gateway && go test ./internal/conversation -run TestActorCompletesOneTurn -v`  
Expected: FAIL because the actor is undefined.

- [ ] **Step 3: Implement explicit states and events**

```go
type State uint8
const (
    WaitingForSpeech State = iota
    Receiving
    Translating
    Playing
    Recovering
    Closed
)

type Actor struct { state State; direction protocol.Direction }

func (a *Actor) Handle(event Event) ([]Command, error) {
    switch e := event.(type) {
    case SpeechStarted:
        if a.state != WaitingForSpeech { return nil, ErrInvalidTransition }
        a.state = Receiving
    case SpeechEnded:
        if a.state != Receiving { return nil, ErrInvalidTransition }
        a.state = Translating
    case TranslationReady:
        if a.state != Translating { return nil, ErrInvalidTransition }
        if e.HasAudio { a.state = Playing } else { return nil, errors.New("audio unavailable") }
    case PlaybackFinished:
        if a.state != Playing { return nil, ErrInvalidTransition }
        a.direction = a.direction.Opposite()
        a.state = WaitingForSpeech
        return []Command{SetDirection{Direction: a.direction}}, nil
    }
    return nil, nil
}
```

- [ ] **Step 4: Cover manual direction correction, empty speech, recoverable provider errors, terminal errors, and close**

Expected invariants: audio frames are accepted only in `Receiving`; provider playback is accepted only in `Playing`; a manual direction change is accepted only while waiting.

- [ ] **Step 5: Run and commit**

Run: `cd gateway && go test ./internal/conversation -v`  
Expected: PASS.  
Commit: `git commit -am "feat(gateway): add conversation state machine"`

### Task 3: Add the Volcengine AST adapter

**Files:**
- Create: `gateway/internal/provider/volc/ast.go`
- Create: `gateway/internal/provider/volc/ast_test.go`
- Generate: `gateway/internal/provider/volc/proto/*.pb.go`

**Interfaces:**
- Consumes: `provider.Request{Direction, ForeignLanguage, Audio}`.
- Produces: ordered `provider.Event` values: ready, source text, translation text, audio, usage, finished, or failure.

- [ ] **Step 1: Add a fake provider WebSocket and failing handshake test**

```go
func TestASTSendsRequiredHeaders(t *testing.T) {
    got := make(chan http.Header, 1)
    srv := newFakeASTServer(t, func(h http.Header) { got <- h })
    client := NewAST(ASTConfig{URL: srv.URL, APIKey: "secret", ResourceID: "volc.service_type.10053"})
    session, err := client.Open(context.Background(), provider.Request{Source: "ja", Target: "zh", Mode: provider.SpeechToSpeech})
    if err != nil { t.Fatal(err) }
    defer session.Close()
    headers := <-got
    if headers.Get("X-Api-Key") != "secret" { t.Fatal("missing API key") }
    if headers.Get("X-Api-Resource-Id") != "volc.service_type.10053" { t.Fatal("missing resource id") }
}
```

- [ ] **Step 2: Confirm the adapter test fails**

Run: `cd gateway && go test ./internal/provider/volc -run TestASTSendsRequiredHeaders -v`  
Expected: FAIL because `AST` is undefined.

- [ ] **Step 3: Implement connection, StartSession, and event mapping**

Use `X-Api-Key` and `X-Api-Resource-Id`; capture `X-Tt-Logid`; wait for event 150 (`SessionStarted`) before allowing audio event 200 (`TaskRequest`). Map source 651/652, translation 654/655, TTS 352, usage 154, finish 152, and failure 153 into provider-neutral events.

- [ ] **Step 4: Add framing tests**

Verify that every audio frame is 16 kHz PCM and that `FinishSession` event 102 is sent once after the client ends a turn. Verify provider errors `45000001`, `45000002`, `45000081`, `45000151`, `55000031`, and `550xxxxx` map to stable Yi error categories.

- [ ] **Step 5: Run and commit**

Run: `cd gateway && go test ./internal/provider/volc -v`  
Expected: PASS with no real cloud call.  
Commit: `git commit -am "feat(gateway): add Volcengine AST adapter"`

### Task 4: Route Korean through S2T plus TTS

**Files:**
- Create: `gateway/internal/provider/tts/tts.go`
- Create: `gateway/internal/provider/tts/tts_test.go`
- Create: `gateway/internal/provider/router.go`
- Create: `gateway/internal/provider/router_test.go`

**Interfaces:**
- Consumes: `provider.Request` and finalized AST translated text.
- Produces: the same normalized audio events as S2S.

- [ ] **Step 1: Write failing routing tests**

```go
func TestRouterModes(t *testing.T) {
    cases := []struct{ foreign string; want Mode }{
        {"en", SpeechToSpeech}, {"ja", SpeechToSpeech}, {"ko", SpeechToTextThenTTS},
    }
    for _, tc := range cases {
        if got := SelectMode(tc.foreign); got != tc.want {
            t.Errorf("%s: got %v want %v", tc.foreign, got, tc.want)
        }
    }
}
```

- [ ] **Step 2: Confirm red state, then implement `SelectMode` and `Synthesizer`**

```go
type Synthesizer interface {
    Synthesize(ctx context.Context, language, text string) (<-chan []byte, error)
}

func SelectMode(language string) Mode {
    if language == "ko" { return SpeechToTextThenTTS }
    return SpeechToSpeech
}
```

- [ ] **Step 3: Implement text-to-audio streaming and fallback**

When Korean synthesis fails, emit `translation.final` followed by a recoverable `error` with code `tts_unavailable`; do not discard text or close the conversation.

- [ ] **Step 4: Run and commit**

Run: `cd gateway && go test ./internal/provider/... -v`  
Expected: PASS.  
Commit: `git commit -am "feat(gateway): route Korean through TTS"`

### Task 5: Expose the authenticated WebSocket endpoint

**Files:**
- Create: `gateway/internal/security/limiter.go`
- Create: `gateway/internal/security/limiter_test.go`
- Create: `gateway/internal/server/websocket.go`
- Create: `gateway/internal/server/websocket_test.go`
- Create: `gateway/internal/config/config.go`
- Create: `gateway/cmd/server/main.go`
- Create: `gateway/.env.example`

**Interfaces:**
- Consumes: client JSON controls and binary PCM frames.
- Produces: `/healthz`, `/readyz`, and `/v1/conversations`.

- [ ] **Step 1: Write failing endpoint tests**

Test that missing/invalid bearer tokens return HTTP 401, a third concurrent conversation for one install token returns 429, invalid start JSON closes with policy violation, 2,560-byte binary frames reach the provider, and server events return in order.

- [ ] **Step 2: Implement safe configuration loading**

Required variables: `YI_LISTEN_ADDR`, `YI_INSTALL_TOKEN_SECRET`, `VOLC_AST_API_KEY`, `VOLC_AST_RESOURCE_ID`, `YI_MAX_CONVERSATIONS_PER_INSTALL`, and the selected Korean TTS credentials. Validation must fail at process start without printing secret values.

- [ ] **Step 3: Implement the WebSocket frame pump**

Use separate read and write goroutines connected to a single conversation actor. Limit control frames to 16 KiB, binary frames to 4 KiB, idle listening to 60 seconds, and total conversation duration to 30 minutes. Cancel provider sessions immediately when the client disconnects.

- [ ] **Step 4: Add structured operational signals**

Log only conversation ID hash, language pair, direction, durations, stable Yi error code, and provider Log ID. Emit metrics for active sessions, turn latency, provider failures, input audio milliseconds, and output audio milliseconds.

- [ ] **Step 5: Run all gateway checks and commit**

Run: `cd gateway && gofmt -w . && go vet ./... && go test -race ./...`  
Expected: all commands exit 0.  
Commit: `git commit -am "feat(gateway): expose secure conversation endpoint"`

### Task 6: Add local integration fixtures and operator documentation

**Files:**
- Create: `gateway/internal/server/testdata/conversation-en.jsonl`
- Create: `gateway/internal/server/testdata/conversation-ja.jsonl`
- Create: `gateway/internal/server/testdata/conversation-ko.jsonl`
- Create: `gateway/README.md`
- Create: `gateway/Makefile`

**Interfaces:**
- Consumes: all gateway packages.
- Produces: reproducible local validation without cloud credentials.

- [ ] **Step 1: Add three deterministic recorded-provider fixtures**

Each fixture must include session ready, partial source, final source, final translation, audio or Korean TTS audio, usage, and session finish. Use generated synthetic PCM, not recorded human speech.

- [ ] **Step 2: Add `make test`, `make test-race`, and `make run` targets**

- [ ] **Step 3: Document local setup, environment names, health checks, protocol examples, privacy behavior, and production deployment requirements**

- [ ] **Step 4: Run final verification and commit**

Run: `cd gateway && make test-race`  
Expected: PASS with zero external requests.  
Commit: `git commit -am "docs(gateway): add integration fixtures and runbook"`

