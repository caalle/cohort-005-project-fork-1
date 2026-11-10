# Is browser push worth including for Cadence?

Research for issue #4, part of map #1. Sources were read on 2026-11-10. Planning had no Knock account, so nothing here was tested against a live Knock or Firebase project.

This note gives facts and an assessment. The decision on whether to include push is made in a later ticket.

## Answer in brief

Browser push is technically viable on desktop and Android, and Knock can deliver it, but only through Firebase Cloud Messaging (FCM). It is weak at the job this work is for, reaching a Student who has already gone away:

- A Student can only be reached if they opted in earlier, in that browser, while they were on Cadence.
- On iPhone and iPad they must also have added Cadence to the Home Screen.
- Email has none of these preconditions.
- No independent evidence shows that push brings people back better than email. The favourable numbers come from companies that sell push.

Assessment: a poor candidate for a required Channel, and at most an optional extra alongside email. It also needs work Cadence has none of today: a service worker, a Firebase project, and handling for which browser belongs to which Student.

## Browser and phone support

| Platform                    | Support                                            | Limits                                                      |
| --------------------------- | -------------------------------------------------- | ----------------------------------------------------------- |
| Chrome, Edge, Firefox       | Chrome 42, Edge 17, Firefox 44 (desktop) [1]       | On desktop the browser must be running [2]                  |
| Android                     | Chrome and Firefox 48; Samsung Internet mirrors Chrome [1] | The OS wakes the browser, so it need not be open [2] |
| Safari on Mac               | Safari 16 on macOS 13 or later [1][3]              | Same permission rules as below                              |
| iPhone and iPad             | iOS and iPadOS 16.4 or later [1][3][4]             | **Only from a web app added to the Home Screen** [1][4]     |
| Android and iOS WebViews    | Not supported [1]                                  |                                                             |

MDN lists the Push API as widely available across browsers since March 2023 [5].

iOS details:

- A normal Safari tab cannot subscribe. The Student has to add Cadence to the Home Screen first, then grant permission from inside that installed web app [4].
- Before iOS 26 this needs a manifest with `display` set to `standalone` or `fullscreen`; otherwise the site is saved as a bookmark [4]. From iOS 26 every site added to the Home Screen opens as a web app by default, and a manifest is no longer required [6]. Cadence has no manifest today.
- Permission is per web app and managed in iOS Notification Settings [4].
- Other iOS browsers can only offer Add to Home Screen under specific conditions [4].
- EU: Apple planned to remove Home Screen web apps in iOS 17.4, then reversed that [7]. This comes from press coverage quoting Apple; Apple's current DMA page no longer mentions it [8]. WebKit's Declarative Web Push (iOS 18.4) removes the service worker requirement for displaying a notification, but the post does not say whether Home Screen install is still needed [9].

## The permission flow and what it demands of the app

**Platform requirements**

- HTTPS: Push is limited to secure contexts [10], and the FCM SDK works only on pages served over HTTPS [11].
- An active service worker [5]. Cadence has none: `app/` and `public/` contain no service worker or web manifest.
- VAPID keys: the app passes a public key to `subscribe()`, and the server signs with the private key [10][12]. Chrome and Edge require the key in practice [10]. Through FCM, the key pair lives in the Firebase console [11].
- Every push must show a visible notification. `userVisibleOnly: true` is required by Chrome and Edge [10]. Safari revokes permission if a push arrives and no notification is shown [3].

**The permission moment**

- Safari requires the request to come from a user gesture, called straight from the event handler [3]. MDN says the same direction applies to browsers generally [10].
- If the Student chooses Block, the app cannot ask again; they have to change the browser setting themselves [12].
- Prompts at page load are discouraged. Pre-prompts explaining the benefit are recommended [13].
- Chrome removes notification permission automatically for sites with very low engagement and high notification volume. Installed web apps are exempt [14]. Chrome also rate-limits sites it judges disruptive, and says nearly all sites are unaffected [15]. Neither post gives numeric thresholds.

**What Cadence would have to build, if Knock and FCM are used** [11][16]

1. A Firebase project, a service account for Knock, and a Knock FCM channel. This is a second third-party service alongside Knock.
2. `firebase-messaging-sw.js` at the site root (for Cadence, in `public/`). Knock's tutorial reads `process.env` in that file, and a static file in `public/` may not have access to it [16].
3. Client code to register the worker, ask permission from a button, get a token and save it to Knock as channel data.
4. A way to unsubscribe. A script cannot revoke permission; the app clears the token instead [16].
5. Handling for a push arriving while the Student is on the page (`onMessage`) as well as in the background.
6. iOS-specific guidance to install to the Home Screen, plus a manifest for iOS versions before 26.
7. Token removal on sign-out and handling for stale tokens. Knock's tutorial does not cover sign-out or expiry [16]. Knock can optionally remove invalid tokens when a message bounces [17].

**A Cadence-specific risk.** A push subscription belongs to a browser, not a Student. Cadence's identity is a cookie session that can be switched (`app/routes/api.switch-user.ts`, `app/lib/session.ts`) and has no real login. Unless a token is removed on switch or sign-out, one Student's Notifications will appear on a browser now used by another Student, including on the lock screen or desktop while the app is closed. This is my inference from the code and the sources above, and it adds to the privacy limit the map already requires the spec to state.

## Knock support

- Knock states: "Knock does not have a first-party web push channel." Browser delivery goes through FCM [16].
- Knock's FCM integration page says FCM can "send mobile or web push notifications" and offers a `webpush` override [17]. It needs the Firebase project ID and service account JSON [17].
- Knock's push overview still labels the integration "Firebase Cloud Messaging (Android)" [18]. The dedicated FCM page and the tutorial are the more specific sources.
- Knock's OneSignal integration page describes mobile push only and does not address web [19]. Amazon SNS and APNs are listed for native push [18].
- Knock's own tutorial handles iPhone by requiring the Home Screen install before subscribing [16].
- Knock's Node.js article sends browser push with a separate library, not through Knock's platform [20], so it is not evidence of Knock support.

## Effectiveness compared with email

**No controlled comparison was found.** Two web searches turned up no peer-reviewed or randomised comparison of browser push and email for bringing people back to a site or a course. The searches were not exhaustive: academic databases were not searched directly.

| Evidence                                                                                                      | Source and interest                        | What it is worth                                                                    |
| ------------------------------------------------------------------------------------------------------------- | ------------------------------------------ | ----------------------------------------------------------------------------------- |
| "Less than 1% of all notifications receive any interaction" [14]                                              | Google/Chrome, browser vendor              | All web notifications, including low-quality senders. Not a ceiling for Cadence      |
| Notification prompts granted 10% on desktop, 21% on Android, from Chrome telemetry [21]                       | Google researchers, peer-reviewed (USENIX Security 2021) | Strongest number found. Every site, older data, before the quieter prompt |
| Desktop permission prompts allowed 12% without a signal of intent, 30% after interaction [13]                 | Chrome telemetry via web.dev               | Covers all permission types, not only notifications                                  |
| Education email: 3.02% click, 0.18% unsubscribe, December 2023 [22]                                           | Mailchimp, **email vendor**                | Click rate per delivered email. Mailchimp warns open rates are unreliable             |
| Web push vs email on one blog: 0.64% vs 0.90% clicked, 44.97% "read" vs 5.73% opened, 9.76% vs 0.17% lost [23] | Prompty, **push vendor**                   | One site, no control. "Read" means shown on push and opened on email, so not comparable. Lists were small and collected differently |
| Push "3 to 10x higher returns" than email [24]                                                                | PushEngage, **push vendor**                | Own platform data, no method given. Disregard                                        |

Search excerpts (not fully read, all from push or marketing vendors) put web push click rates at 2.25% to 6% or higher, with definitions that differ between sources [25]. These are not comparable with email figures and should not be used to rank the Channels.

**What can be said.** Rates per message sent cannot answer this ticket on their own, because the two Channels have different audiences. Email reaches every Student who has an address on file. Push reaches only those who granted permission, in a browser that is still running or installed. Even at the 30% allow rate above, most Students asked would stay unreachable by push, and a Student who has already left cannot be asked at all. That reach argument is my reasoning from the sources, and no source measures it for a product like Cadence.

## What the sources do not settle

- **Whether push beats email at bringing Students back.** No controlled evidence either way, and none for course platforms.
- **Cadence's device mix.** I found no device or user-agent data in the app, so the share of Students on iPhone is unknown. This has the largest effect on how much iOS limits matter.
- **How many iOS Students would install to the Home Screen.** No public figure found. In 2024 Apple was reported to cite "very low user adoption" of Home Screen web apps; that was seen only in press excerpts, and it came with Apple defending a decision.
- **Current grant rates.** The peer-reviewed figures predate later Chrome prompt changes, and none are specific to a site where people are already paying customers.
- **FCM token type.** Firebase now marks registration-token APIs as deprecated in favour of Firebase Installation IDs [11]. Knock's tutorial uses the older `getToken` [16]. Whether Knock's `tokens` channel data accepts the newer IDs is not documented. A Knock account or a question to Knock support would settle it.
- **Safari through FCM, in practice.** Knock's FAQ says "all major modern browsers" without versions [16]. The iPhone branch in its tutorial implies support, but this has not been tested.
- **What Knock's delivery status means for browser push.** The docs do not define whether it means shown on the device [18].
- **Cost.** The FCM page read does not state pricing [26], and Knock's pricing for push steps was not checked.
- **Chrome's revocation thresholds.** No numbers are published, so how a low-volume site with absent Students would be treated is unknown. The stated condition of high notification volume suggests it would not apply.
- **Declarative Web Push.** Whether it drops the Home Screen requirement on iOS, and whether it could be used through FCM.

## Sources

Primary and first-party unless labelled.

1. MDN browser-compat-data, `api/PushManager`: https://github.com/mdn/browser-compat-data/blob/main/api/PushManager.json
2. web.dev, Push notifications FAQ: https://web.dev/articles/push-notifications-faq
3. Apple, Sending web push notifications in web apps and browsers: https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers
4. WebKit, Web Push for Web Apps on iOS and iPadOS: https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
5. MDN, Push API: https://developer.mozilla.org/en-US/docs/Web/API/Push_API
6. WebKit, WebKit features in Safari 26.0: https://webkit.org/blog/17333/webkit-features-in-safari-26-0/
7. MacRumors (secondary, quoting Apple), 2024-03-01: https://www.macrumors.com/2024/03/01/apple-walks-back-decision-to-disable-eu-web-apps/
8. Apple, DMA and apps in the EU: https://developer.apple.com/support/dma-and-apps-in-the-eu/
9. WebKit, Meet Declarative Web Push: https://webkit.org/blog/16535/meet-declarative-web-push/
10. MDN, PushManager.subscribe(): https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe
11. Firebase, Set up a JavaScript FCM client: https://firebase.google.com/docs/cloud-messaging/js/client
12. web.dev, Subscribing a user: https://web.dev/articles/push-notifications-subscribing-a-user
13. web.dev, Permissions best practices: https://web.dev/articles/permissions-best-practices
14. Google/Chromium, automatic notification permission revocation, 2025-10-10: https://blog.google/chromium/automatic-notification-permission/
15. Chrome for Developers, web push rate limits, 2026-01-06: https://developer.chrome.com/blog/web-push-rate-limits
16. Knock, Sending web push notifications with FCM: https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm
17. Knock, Firebase Cloud Messaging integration: https://docs.knock.app/integrations/push/firebase
18. Knock, Push overview: https://docs.knock.app/integrations/push/overview
19. Knock, OneSignal integration: https://docs.knock.app/integrations/push/one-signal
20. Knock blog, browser push from Node.js, 2023-11-20: https://knock.app/blog/how-to-send-browser-push-notifications-from-nodejs
21. Bilogrevic et al., "Shhh...be quiet!", USENIX Security 2021 (abstract read, via Google Research): https://research.google/pubs/shhhbe-quiet-reducing-the-unwanted-interruptions-of-notification-permission-prompts-on-chrome/
22. Mailchimp benchmarks (email vendor): https://mailchimp.com/resources/email-marketing-benchmarks/
23. Prompty (push vendor): https://www.prompty.io/web-push-notifications-vs-email-which-is-better-for-getting-return-visitors-to-your-blog/
24. PushEngage (push vendor): https://www.pushengage.com/why-push-notifications-for-browsers-beats-email-hands-down-in-roi/
25. Vendor benchmarks, search excerpts only: https://docs.pushowl.com/en/articles/6361107-web-push-notification-benchmarks and https://clevertap.com/?p=113014 and https://www.digitalmarketer.com/?p=120411
26. Firebase Cloud Messaging overview: https://firebase.google.com/docs/cloud-messaging
