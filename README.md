# Meridian Voice Receptionist

A real-time conversational voice receptionist built in a single TypeScript file (`receptionist.ts`). The system answers incoming calls, responds to common business questions, maintains multi-turn conversational context, checks and schedules clinic appointments, and transfers complex requests to human departments.

## Capabilities

- **Call Answering**: Greets callers automatically upon connection over a full-duplex WebSocket audio stream.
- **Business Knowledge Base**: Answers questions about clinic hours, location, parking, service pricing, accepted insurance plans, and cancellation policy via structured tool lookups.
- **Conversational Context & History**: Tracks speaker turns, timestamps, interruption counts, and executed actions across the entire call session.
- **Appointment Scheduling**: Verifies slot availability, prevents double-booking, books new appointments with confirmation IDs, and supports cancellations.
- **Department Call Transfers**: Escalates billing disputes, clinical triage questions, urgent matters, and manager requests to the right extension with a structured handoff summary.
- **Speech Input, Speech Output & Interruption Handling**: Streams 16 kHz PCM microphone audio upstream, schedules gapless 24 kHz PCM audio playback downstream, and immediately halts active audio playback when the caller speaks over the receptionist.

## Requirements

- Node.js 20 or newer
- `GEMINI_API_KEY` environment variable
- Browser with Web Audio API and microphone permission (text input fallback is also included)

## Setup & Execution

Install dependencies:

```bash
npm install express ws dotenv @google/genai
npm install -D tsx typescript @types/express @types/ws @types/node
```

Set your API key and launch the single-file server:

```bash
export GEMINI_API_KEY="your_api_key_here"
npx tsx receptionist.ts
```

Open `http://localhost:3000` in a browser and click **Answer Call** to start a live voice session.

## Architecture

### Audio Pipeline

1. **Upstream Capture**: The browser captures microphone audio at `16000 Hz`, converts Float32 frames into 16-bit little-endian PCM, base64-encodes each buffer, and sends it over `/ws/call`.
2. **Downstream Playback**: Incoming `24000 Hz` PCM audio frames are decoded into Web Audio `AudioBufferSourceNode` instances and scheduled sequentially using a running `nextStartTime` clock to eliminate gaps and overlaps.
3. **Barge-In Interruption**: When an interruption event arrives from the server or the caller triggers a manual interrupt, all active `AudioBufferSourceNode` instances are stopped immediately and `nextStartTime` resets to `outputCtx.currentTime`.

### Tool Functions

| Function | Purpose |
| :--- | :--- |
| `get_business_info` | Returns hours, address, parking, services, pricing, insurance, or policy details |
| `check_availability` | Lists unbooked time slots for a requested `YYYY-MM-DD` date |
| `schedule_appointment` | Books a slot, checks for conflicts, and returns a confirmation ID |
| `cancel_appointment` | Cancels an existing confirmed booking by ID or caller name |
| `transfer_call` | Logs an escalation to Billing, Clinical Triage, Manager, or Urgent Care |

### HTTP & WebSocket Endpoints

- `GET /` — Embedded voice console UI.
- `WS /ws/call` — Full-duplex real-time audio and transcript stream.
- `POST /api/turn` — Turn-based text and audio endpoint for scripted testing or environments without microphone access.
- `GET /api/state` — Returns current business profile, appointments, transfer log, and call history.

## Verification Scenarios

1. **General Inquiry**: Ask for clinic hours, parking directions, or whether Delta Dental is accepted.
2. **Context Retention**: Ask about open times tomorrow, then say "Book the 2 PM slot under Jordan Lee for a dental cleaning" without repeating the date.
3. **Interruption Handling**: While the receptionist is reading back clinic services or hours, speak mid-sentence to change the topic; audio playback should stop immediately and address the new question.
4. **Escalation & Transfer**: State that you were double-billed on a recent statement or need clinical advice for severe pain; verify that a transfer record is created with a concise summary.
