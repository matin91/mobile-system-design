# Identity Verification Flow — Scan ID + Selfie & Liveness Check (react-query, redux, coordinator, Pusher)

Short visual description:
- Two mobile screens side-by-side:
    - Left: "Scan your ID" — camera-based frame for scanning passport/driver's license, guides to align document, detects MRZ (Machine Readable Zone) or PDF417 barcode, shows OCR results (name/DOB/expiry), flash/help/next controls.
    - Right: "Selfie & Liveness Check" — guided live selfie (blink/smile commands), selfie preview, compare captured ID photo vs. selfie, result "Match" badge, Accept & Continue or Retry, with face mismatch help.

---

## 1) Requirements

- Functional
    - Step 1: Scan ID (passport/driver license) — camera view, guides to position document, detect and extract MRZ or PDF417 barcode, OCR fields (name, DOB, expiry).
    - Step 2: Selfie & Liveness — prompt user to blink, move, or smile; capture live selfie; compare with ID photo for face match; show match result.
    - Show extracted OCR data for confirmation; allow retry/help/flash; show security statement on privacy/data handling.
    - Liveness: block static image spoof; follow movement instructions for anti-fraud.
    - Persist results securely and flag failed matches for manual review (support retry).
    - Sync data across device/platform if multi-step (mobile ↔ web).
- Non-functional
    - Secure image processing (encrypt at rest/in transit).
    - Fast, reliable OCR & face matching, tuned for mobile network conditions.
    - UX tuned for accessibility (camera/face instructions, retry flow).
    - High-throughput for KYC flows, support concurrency with API/ML model scaling.

Pros / Cons — On-device vs server-side OCR/face match
- On-device:
    - Pros: fast, instant feedback, privacy (user-side).
    - Cons: device compatibility, model size limits, inconsistent lighting.
- Server-side:
    - Pros: consistent accuracy, easier ML iteration, scalable compute.
    - Cons: latency, network required, privacy (requires upload).

---

## 2) Caching, offline & sync strategy

- Camera & OCR: UI must handle slow network (retry, local cache temp images if interrupted).
- OCR/scan results: store locally in Redux, resend/upload when device regains connectivity (offlineQueue).
- Liveness/face images: short-term local persistence (MMKV/AsyncStorage), erase after upload/processing.
- react-query:
    - useMutation for scan/upload, track API progress, optimistically show results when available.
    - useQuery for scan result status (e.g., ongoing verification), polling or subscribe for result.
- Pusher/WebSocket:
    - Subscribe to verification status events (verification.{sessionId}) for “match”, “mismatch”, manual review needed.
- Coordinator pattern: orchestrates scan step navigation, file upload, subscriptions — see example below.

Pros / Cons — Offline queue + retry
- Pros: supports network interruptions, avoids data loss mid-flow.
- Cons: privacy risk (must erase when done), complexity in failing gracefully if persistently offline.

---

## 3) Data models (shared types)

```ts
// ID scan result
interface IDScanResult {
  sessionId: string;
  userId?: string;
  idType: 'passport'|'license';
  mrzData?: string;
  pdf417Data?: string;
  ocr: { name: string; dob: string; expiry: string; };
  rawImageUrl?: string;
  status: 'pending'|'success'|'failed';
  error?: string;
}

// Face & liveness check
interface LivenessCheckResult {
  sessionId: string;
  selfieUrl: string;
  livenessPassed: boolean;
  faceMatchScore: number; // e.g., between 0 and 1
  match: boolean; // recommended threshold
  instructions: string[]; // blink, smile, etc.
  status: 'pending'|'success'|'failed';
  error?: string;
}

// Offline queue for uploads/scan results
interface OfflineAction {
  id: string;
  type: 'uploadID'|'uploadSelfie'|'submitScanResult'|'submitLiveness';
  payload: any;
  timestamp: string;
  retries?: number;
}
```

---

## 4) REST endpoints (mapping from the UI)

- POST /kyc/id-scan
    - body: { sessionId, userId?, rawImage, cameraParams }
    - returns: { scanId, status, ocr data, MRZ/PDF417 (if found) }
- GET /kyc/id-scan/{scanId}
    - scan result/status; error if failed, ocr data
- POST /kyc/selfie
    - body: { sessionId, selfieImage, instructionsCompleted }
    - returns: { liveness status, match result, faceMatchScore }
- GET /kyc/liveness/{sessionId}
    - current match/liveness status
- POST /kyc/retry
    - for the user to retry either step
- POST /kyc/manual-review
    - trigger manual review on persistent failure
- POST /kyc/help-request
    - log help/support request for funnel analysis
- Realtime (Pusher)
    - channel: verification.{sessionId} -> events: scan.status.updated, liveness.status.updated, manual_review.requested

Pros / Cons — Push vs polling for result status
- Pusher/WebSocket:
    - Pros: instant feedback, saves battery, smooth flow.
    - Cons: requires server push infra & session/channel management.
- Polling:
    - Pros: simpler, universally available.
    - Cons: wasteful, introduces latency; may miss rapid events.

---

## 5) High‑level architecture (narrative & mermaid ordering)

- UI Layer (first)
    - ScanIDScreen: camera frame, guidance overlays, flash/help/next controls, OCR result display.
    - LivenessCheckScreen: facebox + preview, liveness instructions (“blink twice”, etc.), ID photo vs. live selfie, Accept/Retry, face match status/help.
    - Small components: FramingGuide, FlashButton, InstructionPrompt, MatchResultBadge.

- Presentation & Hooks
    - useIDScan (useMutation) — upload image, poll/subscribe for OCR/MRZ result.
    - useLivenessCheck (useMutation) — upload selfie, poll/subscribe for liveness/match result.
    - VerificationCoordinator (code example below): manages step navigation, upload, Pusher subscription, offlineQueue integration, error handling, retry logic.

- Network & Realtime
    - ApiClient (axios/fetch) — REST endpoints for scan/selfie/results/retry/manual review.
    - Pusher/WebSocket: subscribe to verification.{sessionId} channel for result updates.
    - Background worker for offlineQueue replay.

- State Layer
    - react-query: mutation/responses cache for pending/in-progress results.
    - redux:
        - offlineSlice: queued uploads and results for network failures.
        - uiSlice: navigation/progress/step info, error state.
        - securitySlice: erase temp images after process complete.
    - persistQueryClient + redux-persist for resilience.

- Local Persistence
    - MMKV/AsyncStorage: temporarily store images and actions for offlineSync/network issues; wipe securely when done.
    - LocalDB (Realm/SQLite, optional): for more advanced analytics/funnel tracking or large enterprise KYC.

- Backend Services
    - KYC Verification Service: handles scan/upload, OCR/MRZ extraction, liveness check, face match ML.
    - Manual Review Service: triggers for failed/uncertain cases.
    - Event Bus & Realtime Worker: pushes scan/liveness/manual review status to Pusher.
    - Analytics Worker: logs funnel metrics, errors, help requests.
    - File Storage (S3/Blob): for raw images (ephemeral).

---

## 6) Coordinator Example (VerificationCoordinator.ts)

```ts
import { QueryClient } from '@tanstack/react-query';
import Pusher from 'pusher-js/react-native';

export class VerificationCoordinator {
  private qc: QueryClient;
  private pusher?: Pusher;
  private subs: string[] = [];
  private sessionId: string;

  constructor(queryClient: QueryClient, sessionId: string, pusher?: Pusher) {
    this.qc = queryClient;
    this.sessionId = sessionId;
    this.pusher = pusher;
  }

  // Manage step navigation (scan → liveness → manual review)
  nextStep(step: 'scan'|'liveness') {
    // UI: navigate to correct screen
    // e.g., router.navigate(step)
  }

  // Upload image/make mutation; optimistically transition if local result available
  async submitIDScan(imageData: Blob) {
    // fire useMutation for /kyc/id-scan
    await this.qc.fetchMutation({
      mutationKey: ['idScan', this.sessionId],
      mutationFn: async () =>
        fetch('/api/kyc/id-scan', { method: 'POST', body: imageData }),
    });
    // UI: show "Processing" spinner and display OCR preview if available
  }

  // Subscribe to scan/liveness realtime events and update caches
  subscribeToVerification() {
    if (!this.pusher) return;
    const chName = `verification.${this.sessionId}`;
    const ch = this.pusher.subscribe(chName);

    ch.bind('scan.status.updated', (payload: any) => {
      // Find react-query cache for scan and update status
      this.qc.setQueryData(['idScan', this.sessionId], (old: any) => ({ ...(old || {}), ...payload }));
    });

    ch.bind('liveness.status.updated', (payload: any) => {
      this.qc.setQueryData(['liveness', this.sessionId], (old: any) => ({ ...(old || {}), ...payload }));
    });

    ch.bind('manual_review.requested', (payload: any) => {
      // UI: surface manual review dialog and notify user
    });

    this.subs.push(chName);
  }

  teardown() {
    if (!this.pusher) return;
    for (const ch of this.subs) this.pusher.unsubscribe(ch);
    this.subs = [];
  }
}
```

---

## 7) Nuanced Concepts — Short Pros & Cons

- On-device OCR vs server OCR
    - Pros (on-device): instant feedback, privacy, no upload if no network.
    - Cons: device compatibility, lower accuracy, no ongoing ML/AI updates.
    - Use: fallback for connectivity loss, or in privacy-sensitive apps.

- Face liveness detection (anti-spoof)
    - Pros: high security against attacks, blocks still image replays; reduces false positives.
    - Cons: can cause UX friction, failure for users with disabilities, increased latency if server-side.
    - Mitigation: clear instructions, retry flow, fallback to manual review.

- Real-time result sync (Pusher) vs polling
    - Pros (realtime): instant status, better UX, reduces battery/network for polling.
    - Cons: costs/infra, requires reliable server/event delivery.
    - Use: for synchronous KYC flows where immediate feedback is critical.

- Offline queue for uploads/scan results
    - Pros: prevents data loss mid-verification, supports flaky networks.
    - Cons: must clean up temp data securely, risk if queue isn’t cleared.

---

## 8) Sequence flows (brief)

- Scan ID flow:
    - User aligns ID in camera frame; MRZ/PDF417 detected; image uploaded via useMutation; OCR fields display as soon as available; user verifies; upon success, coordinator.nextStep('liveness').
- Liveness check:
    - User faces camera; performs movements per instructions; live selfie captured and uploaded; server performs liveness/face match; UI shows match result.
- Real-time:
    - Server emits scan.status.updated and liveness.status.updated events to verification.{sessionId}; UI patches cache and navigates accordingly.
- Error/retry:
    - If match fails, UI offers Retry; persistent failure triggers manual review; offline actions are queued and replayed on reconnect.

