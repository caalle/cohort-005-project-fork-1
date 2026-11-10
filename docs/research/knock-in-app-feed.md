# How Knock delivers in-app Notifications

Research for [#2](https://github.com/caalle/cohort-005-project-fork-1/issues/2), under the map [#1](https://github.com/caalle/cohort-005-project-fork-1/issues/1). Read on 2026-11-10.

Sources are docs.knock.app, knock.app/pricing, and Knock's official repos. Source links are pinned to [`knocklabs/javascript@7bc6b68`][js] and [`knocklabs/knock-node@06707db`][node] (`@knocklabs/node` 1.41.0). `@knocklabs/react` is 0.15.1. No Knock account was used.

Each claim is marked as one of:

- no mark: stated by the cited source
- **(source)**: read from Knock's code, not stated in the docs
- **(probe)**: observed by running the published package locally
- **(inference)**: my conclusion from the cited facts, not stated by Knock

## Answer in brief

Knock stores the feed and pushes updates. The Cadence server does two things: it signs a per-Student JWT in a loader and calls `workflows.trigger` from route actions. In the browser, `@knocklabs/react` provides either styled components or headless hooks over the same feed client. Headless hooks suit shadcn/Tailwind best. Feed content is client-only, so server rendering gives an empty shell that fills in after hydration.

The trigger call is a short HTTP request and the workflow runs asynchronously. A Knock outage therefore reaches the caller only as a thrown error or a slow request, and Cadence has to contain both.

## The feed and its React components

**Packages.** `@knocklabs/react` provides the pre-built components, and "a complete set of React hooks" for building your own UI ([React overview](https://docs.knock.app/in-app-ui/react/overview)). It accepts React 17 to 19, and `next` and `@tanstack/react-router` are optional peers, so React Router needs neither ([package.json][react-pkg] **(source)**).

**Two levels of UI**

| | Pre-built components | Headless hooks |
| --- | --- | --- |
| API | `KnockProvider`, `KnockFeedProvider`, `NotificationIconButton`, `NotificationFeedPopover`, `NotificationFeed`, `NotificationCell` ([feed docs](https://docs.knock.app/in-app-ui/react/feed)) | `useKnockClient`, `useNotifications`, `useNotificationStore` ([custom UI docs](https://docs.knock.app/in-app-ui/react/custom-notifications-ui)) |
| Knock fixes | Markup, `rnf-*` classes, popover, header and filter, default cell | Only the data shape (`FeedItem`, `metadata`) |
| Left to us | Theme variables, `renderItem`, `renderHeader`, click handlers | All markup and behaviour |
| Marking seen | Opening the popover marks all items as seen by default | "you're responsible for marking items as seen" |
| Free-plan branding | "Powered by Knock" footer when `branding_required` is true ([source][feed-tsx] **(source)**) | Our UI, so the component does not render it **(inference)**; the docs do not say whether the plan terms still require it |

**Fit with shadcn/Tailwind 4**

- The pre-built styling is `@knocklabs/react/dist/index.css`, themed with `--rnf-` variables. It can also be replaced entirely, because every class starts with `rnf-` ([feed docs](https://docs.knock.app/in-app-ui/react/feed)).
- The stylesheet has no `@layer` rules **(probe)**. Tailwind 4 utilities live in a cascade layer, and unlayered CSS beats layered CSS, so a utility class will not override an `rnf-*` rule **(inference)**.
- Dark mode is a `colorMode` prop that defaults to `light`, and it is separate from Cadence's theme. The default font is Inter ([theme.css][theme] **(source)**).
- A headless bell and list built from shadcn `Popover`, `Button` and `Badge` avoid Knock's CSS. The docs' headless examples import no stylesheet, but they do not state that it's optional.
- Notification body HTML comes from `item.blocks[0].rendered`, which the docs render with `dangerouslySetInnerHTML`. Headless UI can use that, or render from the workflow's `data` instead.

**Server rendering.** The docs do not mention React Router or plain SSR. I rendered the components with `renderToString` on `@knocklabs/react` 0.15.1 with `fetch` stubbed **(probe)**:

- `KnockProvider`, `KnockFeedProvider`, `NotificationIconButton`, a closed `NotificationFeedPopover` and `NotificationFeed` all rendered without throwing.
- **With default props, rendering sent `PUT /v1/users/student-1` from the server.** The provider authenticates during render, and the default `inline` identification calls `identify` when `user` is an object ([useAuthenticatedKnockClient][auth-hook], [knock.ts][knock-ts] **(source)**).
- With `identificationStrategy="skip"` or `enabled={false}`, the server made no requests.
- Feed items load in a `useEffect` ([NotificationFeed.tsx][feed-tsx] **(source)**), so server HTML never contains them. The socket is only created when `window` exists ([api.ts][api-ts] **(source)**).

Implication **(inference)**: pass `identificationStrategy="skip"`, identify Students from the server, and treat the badge and list as content that appears after hydration. Hydrating the client was not probed.

## Real-time updates and read/unread state

- **Transport.** Clients receive "notifications over a websocket when new messages are sent to an in-app feed channel" ([feeds overview](https://docs.knock.app/in-app-ui/feeds/overview)). It's a Phoenix socket, hosted by Knock ([api.ts][api-ts] **(source)**), so Cadence's single server process holds no connections.
- **Flow.** The `new-message` event carries badge counts only, not content, and it "Might be throttled" ([API overview](https://docs.knock.app/in-app-ui/api-overview)). The client then fetches items newer than its head ([feed.ts][feed-ts] **(source)**).
- **Hooking in sonner.** `feedClient.on("items.received.realtime", handler)` is the documented hook. The docs' example uses `react-hot-toast`, and Knock has no toast component ([toasts docs](https://docs.knock.app/in-app-ui/react/toasts)). Using sonner instead is not documented but only replaces the display call **(inference)**.
- **Engagement states.** Seen, read, interacted and archived are separate, and Knock persists them ([feeds overview](https://docs.knock.app/in-app-ui/feeds/overview)). The feed client has `markAsSeen`, `markAsRead`, `markAsInteracted`, `markAsArchived`, unset versions of most of these, and bulk versions such as `markAllAsRead` ([feed.ts][feed-ts] **(source)**).
- **Counts.** `metadata` has `total_count`, `unread_count` and `unseen_count`. The icon button counts unseen by default, and `badgeCountType="unread"` changes that. Archived items are excluded by default ([API overview](https://docs.knock.app/in-app-ui/api-overview)).
- **Messages can't be edited.** Content "can't be changed once Knock generates it". To change one, archive it and trigger the workflow again ([feeds overview](https://docs.knock.app/in-app-ui/feeds/overview)).
- **Tabs.** Hidden tabs disconnect their socket by default and reconnect when visible ([api.ts][api-ts] **(source)**). Syncing state across tabs uses `__experimentalCrossBrowserUpdates`, marked experimental in the docs.

## Identifying a Student from a session-based login

Cadence's `userId` is a number in a signed cookie (`app/lib/session.ts`). Knock takes it as a string ID, and recommends "your internal user identifier" ([identifying recipients](https://docs.knock.app/managing-recipients/identifying-recipients)).

- **Default mode is open.** "Without enhanced security mode, a request with your public key can act as any user." The docs recommend turning it on for production ([security docs](https://docs.knock.app/in-app-ui/security-and-authentication)).
- **Enhanced mode** requires a JWT signed RS256 by our backend. `sub` is required, and `iat` and `exp` are recommended. The key is shown once in the dashboard and conventionally stored as `KNOCK_SIGNING_KEY`.
- **Signing.** `signUserToken(String(userId), { expiresInSeconds })` from `@knocklabs/node` is local and makes no network call, so it can run in a loader. The default expiry is one hour ([tokenSigner.ts][signer] **(source)**). The docs import it from `@knocklabs/node/lib/tokenSigner`, while the current README imports it from `@knocklabs/node`.
- **Passing it to the client.** A loader returns the token, and it goes to `KnockProvider` as `userToken` next to `apiKey` and `user`. Docs suggest a serialized user property or a cookie.
- **Expiry.** Match it to the session if possible, otherwise use `onUserTokenExpiring` to fetch a new one. Cadence's cookie has no `maxAge`, so a refresh route is the fit **(inference)**.
- **Revalidation trap.** Any change to `userToken` tears down and rebuilds the socket and feeds ([knock.ts][knock-ts] **(source)**). `iat` changes every second ([tokenSigner.ts][signer] **(source)**), and React Router re-runs loaders after actions. Signing a fresh token in a loader would therefore reconnect after every action **(inference)**. Keep the token stable until near expiry, such as by caching it in the session, or stop that loader revalidating.
- **DevUI user switching** changes the user and token, which the provider handles by reinitialising and remounting the feed subtree ([useAuthenticatedKnockClient][auth-hook] **(source)**).
- **Limit.** Knock trusts whatever `sub` the server signs. Enhanced mode stops browsers from impersonating Students through the public key, but a Student's Notifications stay only as private as the Cadence session, as the map says. That session's cookie secret is currently the hardcoded `cadence-dev-secret`.

## Triggering a workflow and identifying recipients

- `POST /v1/workflows/{key}/trigger`, or `client.workflows.trigger(key, {...})` in the Node SDK, with a secret key on the server. "All workflow runs are executed asynchronously" ([API reference](https://docs.knock.app/api-reference/workflows/trigger)). The response is a `workflow_run_id`, not a delivery result ([trigger docs](https://docs.knock.app/send-notifications/triggering-workflows/api)).
- `recipients` takes string IDs, object references, or inline objects, up to 1000 per call. There is no bulk trigger.
- Inline recipients such as `{ id, name, email }` are "guaranteed to be identified **before** the workflow is triggered". This keeps Knock's profile current without a separate identify step, and email steps need the `email` property ([identifying recipients](https://docs.knock.app/managing-recipients/identifying-recipients)).
- Other fields: `data` (JSON for templates, up to 10MB in total), `actor`, `tenant`, `cancellation_key`, and `settings` with `sandbox_mode`, which generates messages without delivering them to providers.
- Workflow step conditions and Student preferences are evaluated inside Knock, so a successful trigger does not mean a Notification was sent ([overview](https://docs.knock.app/send-notifications/triggering-workflows/overview)).

## When Knock is unavailable

**At the trigger (Cadence's route actions)**

- The Node SDK retries connection errors, 408, 409, 429 and 5xx twice by default, with short backoff. Requests time out after 1 minute, and timeouts are retried too ([README][node-readme]).
- With defaults, an awaited trigger can block an action for about three minutes before throwing **(inference)**. A short `timeout` and low `maxRetries` are needed.
- On failure the SDK throws, for example `APIConnectionError` or `RateLimitError`. The docs describe no fallback or client-side queue, so the Notification is lost unless Cadence stores it **(inference)**.
- Rate limiting returns `429 Too Many Requests` ([rate limits](https://docs.knock.app/api-reference/overview/rate-limits)). Trigger is "Rate limit tier: 5" ([API reference](https://docs.knock.app/api-reference/workflows/trigger)).
- `Idempotency-Key` is honoured only on trigger, up to 255 characters, for 24 hours. Only 2xx responses are cached, and reusing a key with different parameters is an error ([idempotent requests](https://docs.knock.app/api-reference/overview/idempotent-requests)).
- The SDK generates a key per call and reuses it across its own retries ([client.ts][node-client] **(source)**). Any retry by Cadence needs a key it derives itself, such as from event and Student IDs **(inference)**.
- Non-outage errors also throw. For example, an inactive workflow returns `workflow_inactive` ([overview](https://docs.knock.app/send-notifications/triggering-workflows/overview)).

**In the browser**

- `fetch` sets the store's `networkStatus` to `error` and resolves rather than throwing ([feed.ts][feed-ts] **(source)**). `NotificationFeed` then shows its empty state because it has no error state of its own ([NotificationFeed.tsx][feed-tsx] **(source)**).
- The API client retries up to 3 times. The socket reconnects with backoff up to 10 minutes, and promptly when the browser comes back online ([api.ts][api-ts] **(source)**).
- Cadence pages keep working because the feed is a client-side enhancement **(inference)**.

The pricing page lists a 99.9% uptime SLA ([pricing](https://knock.app/pricing)).

## Free plan limits

Knock's pricing page labels the $0 tier "Free". "Developer" appears only in a signup link.

| Limit | Free plan |
| --- | --- |
| Messages | 10,000 per month |
| Counting | One user on one Channel is one message. "an in-app message and an email ... counts as two messages sent". Bounced and failed sends are excluded. |
| Over the limit | "We don't hard cap your messages." Knock contacts you about billing for the next month. |
| Users, seats, Channels | Unlimited |
| Feed retention | Unlimited: "In-app feed notifications are available indefinitely" ([data retention](https://docs.knock.app/manage-your-account/data-retention)) |
| Log retention | 30 days |
| Branding | Knock branding stays; removal needs Starter at $250/month |

A Notification sent to a Student on both email and in-app costs two messages, so the free plan covers about 5,000 per month (**inference**). Whether that is enough depends on Cadence's volume, which the fan-out decisions will set.

## What the docs do not settle

1. **React Router and hydration.** No documentation covers it. Server rendering was probed, but client hydration and Vite's handling of the CSS import weren't.
2. **Rate limit for trigger.** The page lists 1, 5, 60, 200 and 1,000 requests per second, but the page text I could read doesn't connect tier numbers to those values. Tier 5 being 1,000 per second is a guess. No `Retry-After` header is documented.
3. **Triggering for a string ID Knock has never seen.** No page says what happens. Inline recipient objects avoid the question.
4. **Exact error codes for an expired or invalid user token.** The list on the [error codes page](https://docs.knock.app/api-reference/overview/error-codes) could not be read with the tools used here.
5. **Whether headless UI must keep Knock branding** on the free plan. Only the pre-built footer is visible in code.
6. **Whether a free-plan workspace is ever throttled** before billing is arranged. The pricing page says only that there is no hard cap.
7. **Real-time delivery latency and socket connection limits.** I found neither documented.
8. **Status on batched messages.** How engagement status applies across a batched message isn't described.

## Glossary note

The task and map refer to `GLOSSARY.md`, but no such file exists on `main` or this branch. Student, Notification and Channel are used as the map uses them.

[js]: https://github.com/knocklabs/javascript/tree/7bc6b685628e7156d60e66e5f038d34820dad578
[node]: https://github.com/knocklabs/knock-node/tree/06707db328d5b7bdabbcec51357ad5898073a8c2
[react-pkg]: https://github.com/knocklabs/javascript/blob/7bc6b685628e7156d60e66e5f038d34820dad578/packages/react/package.json
[theme]: https://github.com/knocklabs/javascript/blob/7bc6b685628e7156d60e66e5f038d34820dad578/packages/react/src/theme.css
[feed-tsx]: https://github.com/knocklabs/javascript/blob/7bc6b685628e7156d60e66e5f038d34820dad578/packages/react/src/modules/feed/components/NotificationFeed/NotificationFeed.tsx
[auth-hook]: https://github.com/knocklabs/javascript/blob/7bc6b685628e7156d60e66e5f038d34820dad578/packages/react-core/src/modules/core/hooks/useAuthenticatedKnockClient.ts
[knock-ts]: https://github.com/knocklabs/javascript/blob/7bc6b685628e7156d60e66e5f038d34820dad578/packages/client/src/knock.ts
[api-ts]: https://github.com/knocklabs/javascript/blob/7bc6b685628e7156d60e66e5f038d34820dad578/packages/client/src/api.ts
[feed-ts]: https://github.com/knocklabs/javascript/blob/7bc6b685628e7156d60e66e5f038d34820dad578/packages/client/src/clients/feed/feed.ts
[signer]: https://github.com/knocklabs/knock-node/blob/06707db328d5b7bdabbcec51357ad5898073a8c2/src/lib/tokenSigner.ts
[node-readme]: https://github.com/knocklabs/knock-node/blob/06707db328d5b7bdabbcec51357ad5898073a8c2/README.md
[node-client]: https://github.com/knocklabs/knock-node/blob/06707db328d5b7bdabbcec51357ad5898073a8c2/src/client.ts
