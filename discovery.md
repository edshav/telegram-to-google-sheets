# Telegram-to-GoogleSheets Archiver Specification

## Executive Summary

A serverless integration using Google Apps Script that automatically archives a Telegram group chat (between a Product Manager and Developers) into Google Sheets and Google Drive. The primary goal is to maintain an accurate, threaded history of product discussions and media (screenshots/PDFs), transforming raw chat data into a decoupled, 'Standardized Project Knowledge Base'. While NotebookLM is the primary intended consumer layer for semantic analysis, the data schema remains strictly consumer-agnostic.

## Problem Statement

Important product discussions, feature implementation details, and shared mockups occur in a Telegram group. Extracting this context later for documentation or AI analysis is tedious and error-prone due to the unordered nature of chat, edited messages, and detached media files. This system provides a fully automated, structured, and continuously updated repository of these conversations.

## Success Criteria

- 100% capture rate of all supported messages (text, screenshots, PDFs) up to ~100 messages/day.
- Existing messages are accurately updated when an `edited_message` webhook is received.
- Contextual threading (replies) is explicitly mapped to ensure downstream consumers (like NotebookLM) can reconstruct the conversation flow.
- Zero data corruption or lost messages during simultaneous webhook events.

## User Personas

- **Primary Users:** Product Manager and 2 Developers. They interact naturally in Telegram without changing their workflow.
- **Admin/Consumer:** The person feeding the resulting Standardized Project Knowledge Base into downstream tools (e.g., NotebookLM) for context synthesis.

## User Journey

1. **Setup (Admin):** The Admin creates a Google Sheet and a Google Drive folder manually to ensure strict permission control. Admin registers the Telegram Bot, disables Privacy Mode via BotFather, and configures the Google Apps Script environment properties.
2. **Operation (Users):** Users chat normally in the designated Telegram group.
3. **Capture (System):** The Bot immediately receives webhooks. It saves text to the Sheet and uploads media to Drive, linking everything together.
4. **Correction (System):** If a user edits a message to fix a typo or clarify a feature, the system instantly locates the original row in the Sheet and updates the content.

## Functional Requirements

### Must Have (P0)

- **Webhook Listener:** Securely receive and validate incoming updates from Telegram using `X-Telegram-Bot-Api-Secret-Token`.
- **Concurrency Control:** Utilize `LockService` to prevent race conditions and skipped rows when multiple messages arrive simultaneously.
- **Message Insertion:** Append new rows to Google Sheets containing message text and metadata.
- **Message Editing:** Process `edited_message` webhooks, use `TextFinder` on the ID column to rapidly locate the row, and update the text/status.
- **Thread Tracking:** Capture and store the `reply_to_id` for every message that is a reply.
- **Media Handling:** Download supported media (Screenshots/Photos, PDFs), save them to the configured Google Drive folder, and store the Drive URL in the spreadsheet. _Note: Telegram sends multiple photo sizes; the implementation must select the last element in the `photo` array for the highest resolution to ensure optimal analysis by downstream AI consumers._

### Should Have (P1)

- **BotFather Setup Guide:** Clear documentation on disabling Privacy Mode (`Bot Settings` -> `Group Privacy` -> `Turn Off`) so the bot can read all group messages. _Important: If the bot was already added to the group before disabling privacy mode, you must kick it out and re-add it for the change to take effect._
- **Error Logging:** Basic error handling within GAS to capture failed API calls or Drive upload failures without crashing the main thread.

### Nice to Have (P2)

- Video support (Out of scope for v1, potentially considered for v2).

## Technical Architecture

### Data Model

**Google Sheet Schema:**

- `id` (String/Integer) - The unique message ID from Telegram.
- `created_at` (Timestamp) - The original timestamp of the message.
- `updated_at` (Timestamp) - The timestamp of the last edit. _(Crucial for downstream consumers like NotebookLM: allows sorting the Sheet by "Last Modified" to quickly surface recently clarified/updated project decisions)._
- `author` (String) - Username or First/Last name of the sender.
- `reply_to_id` (String/Integer) - The ID of the message this message replies to (if applicable).
- `message_text` (String) - The body of the message.
- `media_link` (String) - URL to the file in Google Drive.
- `status` (String) - Current state (e.g., "received", "edited").

### System Components

1. **Telegram Bot API:** Emits webhooks on new and edited messages.
2. **Google Apps Script (Web App):** The serverless backend receiving `POST` requests.
3. **Google Sheets API:** The database layer. Uses `SpreadsheetApp.getActiveSheet().appendRow()` for ingestion and `TextFinder` for performant lookups.
4. **Google Drive API:** Media storage layer. Uses `UrlFetchApp` to download from Telegram, and `DriveApp` to save files.

### Integrations

- **Telegram -> GAS:** Authenticated via standard Webhook mechanism + Custom Header validation (`X-Telegram-Bot-Api-Secret-Token`).
- **GAS -> Telegram API:** Needed to fetch file paths for media downloads via `getFile` endpoint.

### Security Model

- **Webhook Validation:** Rejects any incoming POST requests lacking the correct `X-Telegram-Bot-Api-Secret-Token`.
- **Infrastructure:** Employs a 'Configuration-First' approach. Credentials, Folder IDs, and Sheet IDs are injected via ScriptProperties.
- **Permissions:** Because the Drive Folder and Sheet are created manually, access control is tightly locked to the designated Google accounts directly in Google Workspace, securely limiting ingestion to authorized downstream consumers (e.g., NotebookLM).

## Non-Functional Requirements

- **Performance:** Near-instant processing. `TextFinder` ensures O(1)-like finding speeds regardless of Sheet row length. Media processing is optimized by downloading only the highest-resolution version of a file, minimizing UrlFetchApp execution time.
- **Scalability:** Built for ~100 messages/day. The architecture (LockService + TextFinder) inherently supports much higher limits (thousands per day) within GAS quota boundaries.
- **Reliability:** The system handles simultaneous webhooks via LockService. It specifically addresses the 'Edit Gap' by providing graceful degradation when an edited message refers to a non-existent row.
- **Security:** Hardened webhook endpoint to prevent spoofed data injections.

## Out of Scope (Known Limitations)

- **Deleted Messages:** The Telegram Bot API does not send webhooks for standard deleted messages in regular groups to standard bots. This is a known limitation. Deleted messages will remain in the Sheet unless manually purged.
- **Video Files:** Large video files are excluded from v1 to avoid Apps Script execution timeout limits and complex chunking logic.

## Open Questions for Implementation

- _None at this time. All requirements and constraints are clearly defined._

## Appendix: Implementation Notes

- **LockService Implementation:** Ensure the `try/catch/finally` block releases the lock unconditionally to prevent deadlocks in the container.
- **Historical Data Ingestion:** To populate the archive with discussions that occurred before the bot was added, the Admin will perform a one-time Telegram JSON Export. This data will be mapped to the Google Sheet schema manually or via a helper utility to ensure downstream consumers have a complete project timeline from day one.

## Reliability Reference

### Pseudocode for Core Webhook Logic

#### 1. Global Webhook Listener (The Gatekeeper)

**INPUT:** Incoming POST request from Telegram.
**EXTERNAL DEPENDENCIES:** Google Apps Script `LockService`.
**SIDE EFFECTS:** None (Routing only).

- **VALIDATE** request by checking the `X-Telegram-Bot-Api-Secret-Token` header.
- **IF** token is missing or invalid, **TERMINATE** with 401 Unauthorized.
- **ACQUIRE** a Global Script Lock (wait up to 30 seconds).
- **PARSE** the incoming JSON payload.
- **IF** payload contains a new message: **EXECUTE** `Process New Message`.
- **ELSE IF** payload contains an edited message: **EXECUTE** `Process Message Edit`.
- **FINALLY:** **RELEASE** the lock and return a 200 OK to Telegram to prevent retries.

---

#### 2. Process New Message

**INPUT:** Telegram Message Object.
**EXTERNAL DEPENDENCIES:** Google Drive API, Telegram `getFile` API, Google Sheets API.
**SIDE EFFECTS:** Appends a new row to the Spreadsheet; Creates a new file in Google Drive.

- **CHECK** for media attachments (Photos or Documents).
- **IF** media exists: **EXECUTE** `Media Storage Handler` and retrieve the Drive URL.
- **IDENTIFY** the message author and timestamp.
- **EXTRACT** the reply-to ID (if the message is part of a thread).
- **APPEND** a new record to the Knowledge Base with the following state:
  - _Timestamps:_ `created_at` and `updated_at` set to current.
  - _Content:_ Plain text or media caption.
  - _Status:_ Mark as "received".
  - _Reference:_ Link to the Drive URL (if any).

---

#### 3. Process Message Edit

**INPUT:** Telegram Edited Message Object.
**EXTERNAL DEPENDENCIES:** Google Sheets `TextFinder`.
**SIDE EFFECTS:** Modifies an existing row in the Spreadsheet.

- **SEARCH** the unique `id` column in the Spreadsheet for the Telegram Message ID.
- **IF** ID is found:
  - **UPDATE** the `message_text` or `caption` column with the new content.
  - **SET** the `updated_at` timestamp to the edit date provided by Telegram.
  - **CHANGE** the `status` to "edited".
- **ELSE** (ID not found):
  - **LOG** a "History Gap" warning (occurs if the original message predates the bot).

---

#### 4. Media Storage Handler (Black Box)

**INPUT:** Telegram `file_id`.
**EXTERNAL DEPENDENCIES:** Telegram API (`getFile` and `file_download`), Google Drive API.
**SIDE EFFECTS:** Consumes Google Drive storage; Uses `UrlFetchApp` quota.

- **STEP 1:** Request the direct file path from Telegram using the `file_id`.
- **STEP 2:** **IF** multiple versions exist (e.g., Photos), **SELECT** the highest resolution version only.
- **STEP 3:** Fetch the file binary from Telegram's servers.
- **STEP 4:** Stream the file into the designated Google Drive Folder.
- **STEP 5:** **RETURN** the shareable View-URL of the new file for the Spreadsheet.

---

### Implementation Guardrails

- **Media Resolution Logic:** To optimize for downstream AI analysis (e.g., NotebookLM), the system **MUST** iterate through the Telegram `photo` array and select only the last (largest) `file_id`.
- **Lock Service Timeout:** The `waitLock(30000)` is a hard limit of Google Apps Script. If high-concurrency bursts occur, the script should return a 500 error to Telegram to trigger a temporary retry, rather than failing silently.
- **The "Historical Edit" Gap:** If `Process Message Edit` fails to find a Message ID (common for messages sent before bot installation), the system must log the event to `ScriptProperties` or a dedicated "Logs" sheet instead of throwing a terminating error.

---

## QA Checklist

| Scenario | Expected Logic |

| **User sends a 4MB Photo** | Logic identifies the largest `file_id`, downloads once, and saves to Drive. |
| **Two users message at 0.01s apart** | `LockService` queues the second request; both rows are appended correctly. |
| **User edits a message from 1 month ago** | `TextFinder` scans Column A; if not found, it logs a "History Gap" and exits safely. |

---
