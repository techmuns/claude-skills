## Auth Standards

### Munshot Dashboard SDK (Required)

All dashboards generated using this skill are embedded inside the Munshot platform as iframes.

Dashboard-to-host communication must use the Munshot Dashboard SDK.

SDK Script:

```html
<script src="https://munshot.s3.ap-south-1.amazonaws.com/SDK+script/munshot-dashboard-sdk.v1.0.0.min.js"></script>
```

Claude must integrate the SDK into every generated dashboard.

Do not implement custom iframe messaging when the SDK can be used.

---

### Authentication Model

Authentication is owned by the Munshot host application.

Generated dashboards must:

- Consume authentication and user context from the SDK.
- Assume the host application manages login, sessions, and token lifecycle.
- Use SDK-provided session information when available.

Generated dashboards must not:

- Create standalone login pages.
- Implement username/password authentication.
- Store credentials in localStorage.
- Embed API keys, secrets, or hardcoded tokens.
- Require users to authenticate separately from Munshot.

---

### Required SDK Lifecycle

Every generated dashboard must:

1. Load the Munshot Dashboard SDK.
2. Initialize the dashboard SDK client during application startup.
3. Register dashboard metadata.
4. Signal dashboard readiness.
5. Request initial host context.
6. Subscribe to host context updates.
7. Handle SDK disconnect and error scenarios.

---

### Context Consumption

Dashboards should expect context from the host application, including:

- User information
- Organization information
- Session information
- Selected ticker or symbol
- Active filters
- Application navigation state

Dashboards must react to context updates without requiring a page refresh.

---

### Host Context Contract

The Munshot host automatically passes session and market context to embedded dashboards. Dashboards must treat this as the authoritative source for authentication, user identity, organization, and selected ticker.

#### Contract 1: JWT Session

On every context sync, the host sends:

```typescript
context.session = {
  token: string | null;    // JWT bearer token for Munshot APIs
  userName: string | null; // e.g. "Rahul Sharma"
  email: string | null;    // e.g. "rahul@acme.com"
  orgId: string | null;    // organization identifier
  orgName: string | null;  // e.g. "Acme Capital"
};
```

Token delivery rules:

- On iframe load, the host sends `host:init` with full context, including `session.token`.
- On login/session refresh, the host sends `host:context:update` with the new token.
- On logout, `session.token` becomes `null` and a context update is pushed immediately.
- The SDK queues messages until the dashboard sends `dashboard:ready`, so context should be available even if the dashboard takes time to boot.
- The host forwards the user's JWT token regardless of dashboard domain origin. The dashboard domain must still be allowlisted in Munshot server-side CORS for API requests.

Dashboards must use the token as a bearer header:

```typescript
fetch(apiUrl, {
  headers: {
    Authorization: `Bearer ${session.token}`,
    "Content-Type": "application/json",
  },
});
```

If `session.token` is `null`, show a small non-blocking waiting state inside the relevant widget. Do not show a full-page error because token null can be transient on load.

```tsx
if (!session.token) {
  return (
    <div style={{ padding: 16, textAlign: "center", color: "#9ca3af", fontSize: 13 }}>
      Waiting for session...
    </div>
  );
}
```

#### Contract 2: Selected Ticker

Whenever the user selects, changes, or clears a stock in the Munshot host UI, the host pushes:

```typescript
context.market = {
  selectedTicker: string | null;        // e.g. "AAPL", "RELIANCE"
  selectedTickerCompany: string | null; // e.g. "Apple Inc."
  selectedTickerCountry: string | null; // e.g. "US", "IN"
  selectedSymbol: string | null;        // TradingView format, e.g. "NASDAQ:AAPL"
};
```

Ticker rules:

- If no stock is selected, all market fields may be `null`.
- Show an empty state when `selectedTicker === null`.
- Do not attempt ticker-dependent API calls until both `selectedTicker` and `session.token` exist.
- Re-fetch all ticker-dependent data when either `selectedTicker` or `session.token` changes.
- Never hardcode a ticker in production dashboards.
- Show the selected ticker prominently in the header using the indigo ticker pill from UI Standards.
- If embedding TradingView, use `context.market.selectedSymbol` instead of `selectedTicker`.

#### Authoritative React Hook

Dashboards should centralize host context access in one hook:

```tsx
import { useEffect, useState } from "react";
import { sdk } from "../lib/sdk";

interface SessionContext {
  token: string | null;
  userName: string | null;
  email: string | null;
  orgId: string | null;
  orgName: string | null;
}

export function useHostContext() {
  const [session, setSession] = useState<SessionContext>({
    token: null,
    userName: null,
    email: null,
    orgId: null,
    orgName: null,
  });
  const [ticker, setTicker] = useState<string | null>(null);
  const [tickerCompany, setTickerCompany] = useState<string | null>(null);
  const [tickerCountry, setTickerCountry] = useState<string | null>(null);
  const [selectedSymbol, setSelectedSymbol] = useState<string | null>(null);

  const sync = () => {
    const ctx = sdk.getContext() as any;
    if (!ctx) return;

    if (ctx.session) setSession({ ...ctx.session });

    if (ctx.market) {
      setTicker(ctx.market.selectedTicker ?? null);
      setTickerCompany(ctx.market.selectedTickerCompany ?? null);
      setTickerCountry(ctx.market.selectedTickerCountry ?? null);
      setSelectedSymbol(ctx.market.selectedSymbol ?? null);
    }
  };

  useEffect(() => {
    sync();
    return sdk.onMessage(sync);
  }, []);

  return { session, ticker, tickerCompany, tickerCountry, selectedSymbol };
}
```

Use it in widgets like this:

```tsx
function MyWidget() {
  const { session, ticker } = useHostContext();

  useEffect(() => {
    if (!ticker || !session.token) return;
    fetchMyData(ticker, session.token);
  }, [ticker, session.token]);

  if (!ticker) {
    return <EmptyState message="No stock selected" hint="Pick a stock from the portfolio panel." />;
  }

  if (!session.token) {
    return <div style={{ padding: 16, textAlign: "center", color: "#9ca3af", fontSize: 13 }}>Waiting for session...</div>;
  }

  return <div>Chart for {ticker}</div>;
}
```

Important: always include `session.token` in `useEffect` dependency arrays alongside ticker or other context values. If the session refreshes, widgets must re-fetch automatically.

---

### Communication Standards

Use SDK request/response patterns for:

- Context retrieval
- User selections
- Host-controlled actions
- Operations requiring acknowledgment

Use SDK publish/subscribe patterns for:

- Filter changes
- Dashboard interactions
- Widget events
- Analytics events
- Dashboard telemetry

Topic names must be namespaced.

Examples:

```text
portfolio.ticker.select
analytics.filter.change
dashboard.metric
dashboard.error
```

---

### Visual UI Snapshots For Export

In addition to data snapshots, the Munshot host may request a visual image snapshot of the dashboard's current UI on the dedicated SDK request channel:

```text
dashboard.capture.visual
```

When this request is received, the dashboard must capture Zone 2, the scrollable main content area, and return the image as a native `Blob`.

Implementation rules:

- Use the `html-to-image` library.
- Capture the stable main content target: `#dashboard-main`, `[data-dashboard-capture-root="true"]`, or the main scrollable container.
- Handle capture asynchronously.
- Return a native `Blob`, not a Base64 string.
- Use `pixelRatio: 2` for a sharper export.
- Throw a clear error if the main content container cannot be found or capture fails.

Required dependency:

```bash
npm install html-to-image
```

Example pattern:

```typescript
import { toBlob } from "html-to-image";

client.onRequest("dashboard.capture.visual", async () => {
  const mainElement =
    document.querySelector("#dashboard-main") ||
    document.querySelector("[data-dashboard-capture-root='true']") ||
    document.querySelector("main");

  if (!mainElement) {
    throw new Error("Main content container not found for visual snapshot");
  }

  try {
    const imageBlob = await toBlob(mainElement as HTMLElement, {
      pixelRatio: 2,
    });

    if (!imageBlob) {
      throw new Error("Visual snapshot capture returned an empty Blob");
    }

    return {
      visualSnapshot: imageBlob,
      capturedAt: new Date().toISOString(),
    };
  } catch (err) {
    console.error("Failed to capture visual snapshot:", err);
    throw new Error("Failed to capture visual snapshot");
  }
});
```

Do not convert the visual snapshot to Base64 unless the host explicitly requires it. `Blob` transfer is faster and avoids SDK payload size limits because it can move through the browser structured clone algorithm.

---

### Security Requirements

Generated dashboards must:

- Trust only SDK-authorized host communication.
- Validate all incoming payloads.
- Handle missing or malformed context safely.
- Respect SDK origin validation.
- Use HTTPS for all external communication.
- Avoid exposing sensitive session data.
- Avoid transmitting secrets through dashboard state.

---

### Error Handling

Generated dashboards must:

- Handle SDK initialization failures.
- Handle missing host context.
- Handle request timeouts.
- Display user-friendly error states.
- Publish dashboard errors through SDK mechanisms when appropriate.

---

### Dashboard Generation Rules

When generating dashboards, Claude must:

- Include SDK integration by default.
- Use SDK communication instead of custom postMessage implementations.
- Use host-provided context whenever possible.
- Implement `dashboard.capture.visual` for export workflows.
- Keep dashboard logic independent of authentication implementation details.
- Assume dashboards run inside a Munshot iframe environment.
