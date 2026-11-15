# Acceptance run (Phase 8, 2026-11-08)

The final acceptance scenario from the build specification, run end to end in the browser.

**Environment:**
- a fresh MongoDB;
- **real ChromaDB 1.5.9**, with the **real `Xenova/multilingual-e5-small`** embedding model;
- Gnani in `AI_PROVIDER_MODE=mock`, because no credentials were available.

Speech recognition and extraction were therefore the clearly labelled mocks. Push-to-talk needs a real
microphone, so the voice step used the MP3 upload path. Push-to-talk was verified separately over its
WebSocket in Phase 4.

| # | Step | Result | Evidence |
|---|---|---|---|
| 1 | Admin logs in | ✅ | Seeded admin was forced to `/change-password`, then reached `/admin/dashboard` |
| 2 | Admin creates a field operator | ✅ | "Suresh Patil" (operator, Hindi) created in *Admin → Users*; the 14-character temporary password was shown once |
| 3 | Admin adds pump P-101 | ✅ | Added in *Admin → Equipment* (pump, high criticality) |
| 4 | Admin uploads an approved pump manual | ✅ | Uploaded in *Admin → Documents*, tagged P-101 / pump; status **Ready**, 7 pages / 7 passages in ChromaDB |
| 5 | Operator logs in | ✅ | Temporary password → forced change → `/operator/dashboard` |
| 6 | Reports a pump vibration in a supported language | ✅ | Dashboard defaulted to Hindi (the operator's preference) |
| 7 | Push-to-talk or MP3 | ✅ | MP3 upload from the dashboard's voice card |
| 8 | Prisma transcribes | ✅ (mock) | Labelled `[MOCK TRANSCRIPT] पंप P-101 में बहुत ज़्यादा वाइब्रेशन है…`, language Hindi |
| 9 | Evon extracts complaint details | ✅ (mock) | Draft: equipment **P-101 (registered)**, symptoms vibration and abnormal noise, category mechanical, severity "not assessed" (not guessed) |
| 10 | Follow-up questions if necessary | ✅ n/a | Nothing was missing, so no questions. The question path is covered by tests and the Phase 5 run |
| 11 | Operator reviews and edits | ✅ | Description clarified; severity set to High by the operator |
| 12 | Operator confirms | ✅ | Explicit Confirm, clicked twice |
| 13 | System creates **one** ticket | ✅ | `TKT-2026-00001`. Database: exactly 1 ticket, `source=voice`, `language=hi-IN`, transcript identical to the recording's; draft `CONFIRMED` and linked |
| 14 | Engineer sees the ticket | ✅ | Notification "1 unread"; ticket at the top of the work queue |
| 15 | Engineer assigns and updates | ✅ | Accepted → Under inspection (with comment) → In progress; history `created > assigned > UNDER_INSPECTION > IN_PROGRESS` |
| 16 | Engineer reads manual references | ✅ | "Documents tagged for this equipment": **P-101 Centrifugal Pump Manual, page 3 · "3 High vibration: common causes and checks"** |
| 17 | Operator sees the status | ✅ | 3 notifications (assigned, OPEN→UNDER INSPECTION, →IN PROGRESS); ticket on the dashboard |
| 18 | Audio files deleted | ✅ | 0 files in `AUDIO_UPLOAD_DIR`; 0 binary fields anywhere in MongoDB |
| 19 | Audit history records important actions | ✅ | `user.created`, `equipment.created`, `document.uploaded`, `complaint.draft_created/updated/confirmed`, `ticket.created/assigned`, `ticket.status_changed ×2`, `user.password_changed ×3`, logins and logouts |

## Not covered by this run

- **Live Gnani services.** Needs your `GNANI_API_KEY_ID` and `AI_PROVIDER_MODE=real`. The adapters
  are tested against stubs of the documented protocols only.
- **Evon in real mode.** Gnani publishes no Evon API, so extraction is the labelled mock.
- **A real microphone** in a browser for push-to-talk. Please try it once in Chrome on `localhost`.

