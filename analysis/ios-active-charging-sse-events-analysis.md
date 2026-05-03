# iOS Active Charging SSE Events Analysis

Date: 2026-05-03

## Problem

The iOS app active charging screen is not receiving charging events from the backend.

The backend was recently changed so active charging should be driven by Server-Sent Events from:

```text
GET https://dev.electrahub.com:8443/session/api/v1/sessions/active/stream
Accept: text/event-stream
X-Account-Id: <driver account id>
```

The screen can still pull current active charging state from:

```text
GET https://dev.electrahub.com:8443/session/api/v1/sessions/active
GET https://dev.electrahub.com:8443/session/api/v1/sessions/{sessionId}/current
```

## Current Backend State

### Deployed Services

- `session-service:21`
- `api-gateway:15`
- `web-socket-connector:7`

ArgoCD status observed for `session-service`: `Synced Healthy`.

### Relevant Backend Commits

`session-service`:

- `a9dfe6f feat: stream active charging sessions`
- `9208450 fix: calculate delivered energy from meter delta`
- `22e583c feat: index realtime charging sessions`
- `9177e91 fix: make session indexing non-blocking`

`api-gateway`:

- `2703b49 fix: stream server-sent events through gateway`

`k8s-platform`:

- `3a1f7bc chore: deploy non-blocking session indexing`

## Backend Contract

### SSE Endpoint

```http
GET /session/api/v1/sessions/active/stream
Accept: text/event-stream
X-Account-Id: 0322b0cc-5419-41b1-bd99-c1be65ab8004
```

Important: gateway only switches to streaming proxy mode when the request `Accept` header contains:

```text
text/event-stream
```

If iOS calls the stream endpoint with `Accept: application/json`, the gateway may treat it like a regular REST request instead of an SSE stream.

### Event Types

The backend emits named SSE events:

```text
event: snapshot
event: session
event: terminal
```

The JSON payload also has an internal `type` field:

```json
{
  "type": "SESSION_UPDATED",
  "session": {
    "id": "0896750b-d7fa-48bd-b144-6e0413fa8567",
    "stationId": "8ea3966b-8279-315c-a5b2-001ab12b75a7",
    "stationName": "US*EHB*LOC*SFO001",
    "connectorId": "CON-SFO-001",
    "connectorType": "CON-SFO-001",
    "startedAt": "2026-05-03T19:04:22Z",
    "energyDeliveredKwh": 10.275,
    "currentPowerKw": 7.2,
    "estimatedCost": 3.5963,
    "batteryPercent": null,
    "estimatedTimeRemainingMin": null,
    "status": "CHARGING"
  },
  "sessions": null,
  "emittedAt": "2026-05-03T19:27:13.136392047Z"
}
```

For `snapshot`, `session` is `null` and `sessions` contains the current active sessions:

```json
{
  "type": "SNAPSHOT",
  "session": null,
  "sessions": [
    {
      "id": "0896750b-d7fa-48bd-b144-6e0413fa8567",
      "energyDeliveredKwh": 10.2,
      "currentPowerKw": 7.2,
      "estimatedCost": 3.57,
      "status": "CHARGING"
    }
  ],
  "emittedAt": "2026-05-03T19:27:04.299817959Z"
}
```

## Live Validation Already Performed

### Public SSE Validation

Command:

```bash
curl -k -sS --max-time 12 -N \
  -H 'Accept: text/event-stream' \
  -H 'X-Account-Id: 0322b0cc-5419-41b1-bd99-c1be65ab8004' \
  https://dev.electrahub.com:8443/session/api/v1/sessions/active/stream
```

Observed output:

```text
event:snapshot
data:{"type":"SNAPSHOT","session":null,"sessions":[{"id":"0896750b-d7fa-48bd-b144-6e0413fa8567","stationId":"8ea3966b-8279-315c-a5b2-001ab12b75a7","stationName":"US*EHB*LOC*SFO001","connectorId":"CON-SFO-001","connectorType":"CON-SFO-001","startedAt":"2026-05-03T19:04:22Z","energyDeliveredKwh":10.2,"currentPowerKw":7.2,"estimatedCost":3.57,"batteryPercent":null,"estimatedTimeRemainingMin":null,"status":"CHARGING"}],"emittedAt":"2026-05-03T19:27:04.299817959Z"}

event:session
data:{"type":"SESSION_UPDATED","session":{"id":"0896750b-d7fa-48bd-b144-6e0413fa8567","stationId":"8ea3966b-8279-315c-a5b2-001ab12b75a7","stationName":"US*EHB*LOC*SFO001","connectorId":"CON-SFO-001","connectorType":"CON-SFO-001","startedAt":"2026-05-03T19:04:22Z","energyDeliveredKwh":10.275,"currentPowerKw":7.2,"estimatedCost":3.5963,"batteryPercent":null,"estimatedTimeRemainingMin":null,"status":"CHARGING"},"sessions":null,"emittedAt":"2026-05-03T19:27:13.136392047Z"}
```

Conclusion: backend and gateway can stream events publicly when the client sends `Accept: text/event-stream`.

### Current Session Pull Validation

Command:

```bash
curl -k -sS \
  -H 'Accept: application/json' \
  -H 'X-Account-Id: 0322b0cc-5419-41b1-bd99-c1be65ab8004' \
  https://dev.electrahub.com:8443/session/api/v1/sessions/0896750b-d7fa-48bd-b144-6e0413fa8567/current
```

Observed response:

```json
{
  "id": "0896750b-d7fa-48bd-b144-6e0413fa8567",
  "stationId": "8ea3966b-8279-315c-a5b2-001ab12b75a7",
  "stationName": "US*EHB*LOC*SFO001",
  "connectorId": "CON-SFO-001",
  "connectorType": "CON-SFO-001",
  "startedAt": "2026-05-03T19:04:22Z",
  "energyDeliveredKwh": 10.125,
  "currentPowerKw": 7.2,
  "estimatedCost": 3.5438,
  "batteryPercent": null,
  "estimatedTimeRemainingMin": null,
  "status": "CHARGING"
}
```

### Elasticsearch Validation

Index:

```text
session-current-sessions
```

Document id:

```text
0896750b-d7fa-48bd-b144-6e0413fa8567
```

Observed `_source` fields:

```json
{
  "accountId": "0322b0cc-5419-41b1-bd99-c1be65ab8004",
  "chargerId": "EH-SFO-CHG-001",
  "connectorId": "CON-SFO-001",
  "connectorType": "CON-SFO-001",
  "currency": "USD",
  "currentPowerKw": 7.2,
  "energyDeliveredKwh": 10.05,
  "estimatedCost": 3.5175,
  "latestMeterRegisterWh": 1210050,
  "meterStartWh": 1200000,
  "sessionId": "0896750b-d7fa-48bd-b144-6e0413fa8567",
  "status": "CHARGING"
}
```

Conclusion: meter values are flowing, realtime cost is calculated, and the current-session index is being updated.

## Most Likely Causes On iOS

### 1. Missing `Accept: text/event-stream`

The gateway checks the request `Accept` header to detect SSE:

```java
private boolean isEventStreamRequest(HttpServletRequest request) {
    String accept = request.getHeader(HttpHeaders.ACCEPT);
    return accept != null && accept.toLowerCase(Locale.ROOT).contains(MediaType.TEXT_EVENT_STREAM_VALUE);
}
```

If iOS uses the existing JSON client defaults:

```text
Accept: application/json
```

then it may not receive streaming behavior from the gateway.

### 2. iOS Client May Be Waiting For JSON Response Completion

SSE is an infinite/long-lived response. If the active charging screen uses the normal API client path that waits for a full JSON response, it will not process events incrementally.

The iOS implementation should use a streaming-capable API such as `URLSessionDataDelegate` and parse chunks as they arrive.

### 3. iOS Parser May Ignore Named Events

Backend emits:

```text
event:snapshot
data:{...}

event:session
data:{...}
```

The iOS parser must handle named events and not assume every event is unnamed `message`.

### 4. Snapshot Shape Differs From Session Update Shape

Initial snapshot:

```json
{
  "type": "SNAPSHOT",
  "session": null,
  "sessions": []
}
```

Live update:

```json
{
  "type": "SESSION_UPDATED",
  "session": {},
  "sessions": null
}
```

If iOS only reads `session`, it may ignore the first snapshot and show empty UI until the next meter event. If it only reads `sessions`, it may ignore live updates.

### 5. Account Id Mismatch

Backend groups SSE emitters by account id. The account id comes from `AccountContextResolver`, usually via:

```text
X-Account-Id
```

The active session was validated with:

```text
X-Account-Id: 0322b0cc-5419-41b1-bd99-c1be65ab8004
```

If iOS uses a different account id, no matching session events will be published to that stream.

### 6. Connection Lifecycle May Be Tied To View Refresh

If SwiftUI recreates the view model or the active charging screen reconnects repeatedly, the SSE connection may close before the next meter tick. Meter events arrive about every 10 seconds in the simulator.

The stream should be owned by a stable session/charging state object, not by a transient view render.

### 7. Timeout/Buffering In iOS Networking Layer

The backend stream timeout is 30 minutes:

```java
private static final long STREAM_TIMEOUT_MS = 30 * 60 * 1000L;
```

If iOS has a shorter request timeout or default timeout on the stream request, it may disconnect.

## Backend Files To Inspect

Session SSE controller:

```text
/Users/amolsurjuse/development/projects/session-service/src/main/java/com/electrahub/session/web/ChargingSessionController.java
```

Session stream service:

```text
/Users/amolsurjuse/development/projects/session-service/src/main/java/com/electrahub/session/service/DriverSessionStreamService.java
```

Realtime session index:

```text
/Users/amolsurjuse/development/projects/session-service/src/main/java/com/electrahub/session/service/CurrentSessionIndexService.java
/Users/amolsurjuse/development/projects/session-service/src/main/java/com/electrahub/session/elasticsearch/document/CurrentSessionDocument.java
```

Gateway SSE proxy:

```text
/Users/amolsurjuse/development/projects/api-gateway/src/main/java/com/electrahub/gateway/route/GatewayProxyController.java
```

## iOS Files To Inspect Next

Start in:

```text
/Users/amolsurjuse/development/projects/driver-portal-ios
```

Search terms:

```bash
rg -n "active/stream|sessions/active|EventSource|URLSessionDataDelegate|text/event-stream|Accept|Charging|ActiveCharging" /Users/amolsurjuse/development/projects/driver-portal-ios
```

Specific checks:

- Confirm the active charging screen calls `/session/api/v1/sessions/active/stream`, not only `/sessions/active`.
- Confirm request header `Accept: text/event-stream`.
- Confirm request includes the same auth/account context as normal REST calls.
- Confirm parser handles `event:` and `data:` lines.
- Confirm parser handles both `SNAPSHOT.sessions[]` and `SESSION_UPDATED.session`.
- Confirm the stream is retained while the charging screen is visible and is not deallocated by SwiftUI view refresh.
- Confirm updates are dispatched onto the main actor before changing published UI state.

## Recommended iOS Implementation Shape

Use a dedicated stream client instead of the normal JSON request method:

```swift
final class ActiveChargingStreamClient: NSObject, URLSessionDataDelegate {
    private var task: URLSessionDataTask?
    private var buffer = ""

    func connect(accountId: String) {
        var request = URLRequest(url: URL(string: "https://dev.electrahub.com:8443/session/api/v1/sessions/active/stream")!)
        request.setValue("text/event-stream", forHTTPHeaderField: "Accept")
        request.setValue(accountId, forHTTPHeaderField: "X-Account-Id")
        request.timeoutInterval = 0

        let session = URLSession(configuration: .default, delegate: self, delegateQueue: nil)
        task = session.dataTask(with: request)
        task?.resume()
    }

    func urlSession(_ session: URLSession, dataTask: URLSessionDataTask, didReceive data: Data) {
        // Append chunk, split on blank line, parse event/data fields.
    }
}
```

Parsing rules:

- SSE event boundary is a blank line.
- `event:` gives the event name.
- `data:` contains JSON payload.
- Multiple `data:` lines should be joined with newline if present.
- Decode payload into:

```swift
struct SessionStreamEvent: Decodable {
    let type: String
    let session: ActiveChargingSession?
    let sessions: [ActiveChargingSession]?
    let emittedAt: String
}
```

UI update rules:

- On `SNAPSHOT`, replace local active sessions from `sessions`.
- On `SESSION_UPDATED`, upsert `session`.
- On `SESSION_TERMINAL`, mark session complete or remove from active view after final state is shown.

## Backend Follow-Ups If iOS Is Correct

If iOS already sends the correct header and parses SSE correctly, check these backend/gateway items next:

1. Add backend logs for stream subscribe and publish account id:
   - `subscribe(accountId)`
   - `publish(accountId, eventName)`
   - number of emitters for account id
2. Add a heartbeat/comment event every 15 seconds so clients and proxies know the stream is alive:

```text
: heartbeat

```

3. Make gateway route detect SSE by path as well as `Accept` header:

```text
/session/api/v1/sessions/active/stream
```

This would make the gateway more forgiving if clients omit `Accept: text/event-stream`.

4. Consider changing backend event names to canonical names if iOS expects `message`:
   - either emit unnamed events
   - or ensure iOS listens to `snapshot`, `session`, and `terminal`

## Fast Reproduction Checklist For Next Thread

1. Verify backend stream still works:

```bash
curl -k -sS --max-time 12 -N \
  -H 'Accept: text/event-stream' \
  -H 'X-Account-Id: 0322b0cc-5419-41b1-bd99-c1be65ab8004' \
  https://dev.electrahub.com:8443/session/api/v1/sessions/active/stream
```

2. Verify wrong header behavior:

```bash
curl -k -i --max-time 12 -N \
  -H 'Accept: application/json' \
  -H 'X-Account-Id: 0322b0cc-5419-41b1-bd99-c1be65ab8004' \
  https://dev.electrahub.com:8443/session/api/v1/sessions/active/stream
```

3. Verify active pull fallback:

```bash
curl -k -sS \
  -H 'Accept: application/json' \
  -H 'X-Account-Id: 0322b0cc-5419-41b1-bd99-c1be65ab8004' \
  https://dev.electrahub.com:8443/session/api/v1/sessions/active
```

4. Inspect iOS request headers and stream parser logs.

## Working Hypothesis

Backend SSE is functional. The most likely issue is the iOS active charging screen is using the normal JSON networking path or missing `Accept: text/event-stream`, so it never enters a streaming parse flow. The second most likely issue is the iOS parser only expects a plain JSON response or unnamed `message` events and ignores `event:snapshot` / `event:session`.

