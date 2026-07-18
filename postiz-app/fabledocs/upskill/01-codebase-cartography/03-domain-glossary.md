# Domain Glossary

Product nouns, their code homes, and the confusions between them.

| Term | Meaning here | Code home | Watch out |
| --- | --- | --- | --- |
| **Organization** | The tenant. Everything hangs off it | [schema L11-L50](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L11-L50) | Users belong to N orgs via `UserOrganization`; the active org is a *request-time* choice (`showorg`) |
| **Integration** | A connected social **channel** (one X account, one LinkedIn page) | [schema L314-L356](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L314-L356) | UI says "channel", code says "integration", and *provider* means the platform class — three words, three concepts |
| **Provider** | The platform implementation (XProvider, TelegramProvider) | [integrations/social/](../../../libraries/nestjs-libraries/src/integrations/social) | `providerIdentifier` (string like `'x'`) links Integration row → provider class |
| **internalId** | The platform's own account id | Integration column | Unique with orgId ([L345](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L345)) — reconnecting the same account updates instead of duplicating |
| **rootInternalId** | Parent account for page-type channels (e.g. FB page under a user) | Integration column L335 | |
| **Post** | One platform-post-sized unit. A thread = chain of Posts | [schema L393-L446](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L393-L446) | What the UI calls "a post" is a **group** of Post rows |
| **group** | UUID string tying a thread + its edits together | Post.group | Delete/read operate on group, not id ([DELETE /posts/:group](../../../apps/backend/src/api/routes/posts.controller.ts#L205-L211)) |
| **state** | QUEUE → PUBLISHED / ERROR, or DRAFT | [enum L903-L908](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L903-L908) | No enforced transition rules |
| **releaseId / releaseURL** | The platform's id/url for the published post | Post columns | `releaseId === 'missing'` is a sentinel meaning "published but id unknown" ([getMissingContent](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L80-L138)) |
| **Launches** | The calendar UI for scheduled posts | [components/launches/](../../../apps/frontend/src/components/launches) | Legacy name — API says posts |
| **Plug** | Post-publish automation (auto-repost at N likes, repost by teammates) | `@Plug` decorator metadata ([x.provider L110](../../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts#L110)); rows in `Plugs` ([schema L549-L561](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L549-L561)); executed in [workflow L318-L437](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L318-L437) | "internal plug" = per-post settings-driven; "global plug" = channel-level recurring |
| **Sets** | Saved channel+content template combos | [schema L631-L641](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L631-L641), sets.controller | |
| **Autopost** | RSS/URL watcher that auto-creates posts | [schema L610-L629](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L610-L629), [autopost.workflow.ts](../../../apps/orchestrator/src/workflows/autopost.workflow.ts) | Different from "scheduled post" |
| **Customer** | An agency's client grouping channels | [schema L301-L312](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L301-L312) | Not a Stripe customer! Stripe linkage is `Organization.paymentId` |
| **Marketplace (Orders/Messages/MessagesGroup)** | Buyer hires seller-influencer; posts submitted for approval | [schema L463-L547](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L463-L547) | Large dormant-looking subsystem; `submittedForOrder*` fields on Post belong to it |
| **Signature** | Reusable content footer | [schema L358-L371](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L358-L371) | |
| **Webhooks (model)** | *Outgoing* URLs fired on publish | [schema L595-L608](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L595-L608) | Incoming Stripe webhook is a different thing entirely |
| **Third-party** | External generation services (video etc.) keyed per org | [schema L643-L658](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L643-L658) | Encrypted apiKey via `fixedEncryption` |
| **isBetweenSteps** | Two-phase connect: OAuth done, page/board selection pending | Integration column L330 | Posts fail while true |
| **refreshNeeded** | Token refresh failed; user must reconnect | Integration column L331 | Sweep excludes these ([repo L36-L64](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L36-L64)) |
| **Streak** | Gamified consecutive-posting counter | `Organization.streakSince`, [streak.workflow.ts](../../../apps/orchestrator/src/workflows/streak.workflow.ts) | |
| **gitroom** | The project's former name — the npm package name ([package.json L2](../../../package.json#L2)), every `@gitroom/*` import alias, Stripe metadata `service: 'gitroom'` ([stripe.controller L31](../../../apps/backend/src/api/routes/stripe.controller.ts#L27-L35)) | everywhere | Postiz = product name, gitroom = code name; don't let it confuse greps |

Near-synonym traps: *channel/integration/provider* (above); *comment* meaning thread-item-posted-as-reply ([postComment activity](../../../apps/orchestrator/src/activities/post.activity.ts#L158-L200)) vs the `Comments` table (team feedback on drafts, [schema L373-L391](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L373-L391)); *customer* (agency client) vs Stripe customer; *order* (marketplace) vs sort order.
