# Yi Conversation Screen Override

This page overrides the light marketing defaults in `../MASTER.md` for the in-call, face-to-face conversation experience.

## Rationale

The conversation screen is used in stations, restaurants, vehicles, and evening travel. A dark, low-glare surface keeps attention on live language status and makes a phone placed between two people less visually intrusive. The master blue/teal translator palette remains the semantic foundation.

## Semantic Colors

| Role | Value | Usage |
|---|---:|---|
| Background | `#061016` | Full conversation screen |
| Surface | `#102832` | Partner and local panels |
| Primary | `#2563EB` | Start and primary actions |
| Live | `#2DD4BF` | Listening, connection, waveform |
| Translating | `#22D3EE` | Translation in progress |
| Accent | `#F97316` | Attention and recoverable warning only |
| Text | `#F8FAFC` | Primary transcript and translation |
| Secondary text | `#A8BBC5` | Source text and labels; contrast >= 3:1 |
| Border | `#29434E` | Dividers and control outlines |
| Destructive | `#FF6B5F` | End/error actions |

## Typography

- Native app: San Francisco system type with Dynamic Type text styles.
- HTML prototype: `-apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif`.
- Translation: bold title hierarchy; source text: body hierarchy.
- Never render essential status text below 13 px / `.caption`.

## Interaction

- Minimum target: 44 x 44 pt; minimum gap: 8 pt.
- Use outline SF Symbols in the native app and matching outline SVG in the prototype.
- Animate only the listening waveform and status transition, 150-300 ms.
- With Reduce Motion enabled, replace the waveform animation with a static level mark.
- The top partner panel is visually rotated 180 degrees. VoiceOver uses an alternate non-rotated reading order.

