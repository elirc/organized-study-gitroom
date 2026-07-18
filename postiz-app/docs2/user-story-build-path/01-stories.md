# User Stories

## Story 1: Show Creation Method In The Composer Header
**Difficulty:** Easy  
**Estimated Time:** 1 hour  
**Skills You'll Practice:** reading props, conditional rendering, small UI changes  
**The Story:** As a super admin reviewing a post, I want to see how the post was created in the composer header so that I can tell whether it came from web, API, MCP, CLI, or autopost.  
**Acceptance Criteria:**
- [ ] Existing posts with a non-`UNKNOWN` creation method show a badge in the composer header.
- [ ] Draft/new posts without a creation method do not show a badge.
- [ ] The badge uses the existing `CreationMethodBadge` component.
**Files You'll Likely Touch:** `apps/frontend/src/components/new-launch/manage.modal.tsx` because the composer header already renders `CreationMethodBadge` around the create/edit title (`apps/frontend/src/components/new-launch/manage.modal.tsx:449-460`). `apps/frontend/src/components/launches/creation.method.badge.tsx` because it defines supported values and display behavior (`apps/frontend/src/components/launches/creation.method.badge.tsx:4-50`).  
**High-Level Implementation Plan:**
1. Inspect `ManageModal` around the header and existing `existingData?.posts?.[0]?.creationMethod` usage.
2. Confirm the badge receives the same creation method shape used by `CalendarItem`.
3. Adjust only the header display if needed; keep badge logic inside `CreationMethodBadge`.
**Tips:**
- `existingData` comes from `useExistingData` (`apps/frontend/src/components/launches/helpers/use.existing.data.tsx:1-19`).
- The schema enum lives in `schema.prisma:943-950`.
- Keep the badge hidden for `UNKNOWN`; the component already handles this.
**What Could Go Wrong:**
- Passing a missing `posts[0]` without optional chaining can crash new post creation.
- Duplicating badge styling instead of reusing the component creates drift.
**Stretch Goal:** Add a tooltip test plan for each creation method value.  
**Connects To:** Story 2 because both are small UI changes on existing post display.

## Story 2: Add A Clear Empty-State Line To Calendar List View
**Difficulty:** Easy  
**Estimated Time:** 1 hour  
**Skills You'll Practice:** reading conditional rendering, translation fallback, UI copy  
**The Story:** As a scheduler, I want the list view empty state to mention filters so that I know why no upcoming posts are visible.  
**Acceptance Criteria:**
- [ ] When list view has no posts, the empty state mentions there may be no upcoming posts for the current filters.
- [ ] Existing loading state remains unchanged.
- [ ] Calendar/week/month empty states are not affected.
**Files You'll Likely Touch:** `apps/frontend/src/components/launches/calendar.tsx` because `ListView` renders loading and empty states (`apps/frontend/src/components/launches/calendar.tsx:493-532`).  
**High-Level Implementation Plan:**
1. Locate `ListView` empty state.
2. Update the fallback string passed to `t(...)`.
3. Verify no other view shares that text block.
**Tips:**
- `ListView` reads `listPosts` from `useCalendar` (`apps/frontend/src/components/launches/calendar.tsx:493-500`).
- The existing empty state begins at `calendar.tsx:522`.
- Keep copy concise; this is a dense tool UI.
**What Could Go Wrong:**
- Editing a shared translation key may affect other languages unexpectedly.
- Changing loading logic can make empty state flash before SWR returns.
**Stretch Goal:** Include the selected customer name if available from calendar context.  
**Connects To:** Story 3 because it still modifies display without changing data.

## Story 3: Show Post Time In Month Calendar Cards
**Difficulty:** Easy  
**Estimated Time:** 2 hours  
**Skills You'll Practice:** component props, conditional UI, date formatting  
**The Story:** As a scheduler viewing month mode, I want each post card to show its time so that I can scan the month without opening the post.  
**Acceptance Criteria:**
- [ ] Month view cards show the local scheduled time.
- [ ] Week/day behavior is unchanged.
- [ ] Time formatting follows existing US/non-US helper behavior.
**Files You'll Likely Touch:** `apps/frontend/src/components/launches/calendar.tsx` because `CalendarItem` accepts `showTime` and formats time (`apps/frontend/src/components/launches/calendar.tsx:973-1178`), and `MonthView` renders `CalendarColumn` (`apps/frontend/src/components/launches/calendar.tsx:416-492`).  
**High-Level Implementation Plan:**
1. Trace how `display` reaches `CalendarItem` in `CalendarColumn`.
2. Set `showTime` for month display only.
3. Reuse existing `newDayjs(post.publishDate).local().format(...)` logic.
**Tips:**
- `CalendarItem` already has a `showTime` prop (`apps/frontend/src/components/launches/calendar.tsx:973-991`).
- Formatting uses `isUSCitizen()` at `calendar.tsx:1170-1173`.
- Avoid adding another date library call pattern.
**What Could Go Wrong:**
- Passing `showTime` for week/day could clutter dense cards.
- Formatting UTC directly instead of local time can show wrong times.
**Stretch Goal:** Add a tooltip with full localized date/time.  
**Connects To:** Story 4 because you will start reading existing calendar data more deeply.

## Story 4: Add A Channel Count Summary Above The Channel Menu
**Difficulty:** Medium  
**Estimated Time:** 3 hours  
**Skills You'll Practice:** derived state, existing data, sidebar UI  
**The Story:** As a scheduler, I want to see how many connected channels are active and disabled so that I can understand my account capacity at a glance.  
**Acceptance Criteria:**
- [ ] Sidebar shows total connected channels.
- [ ] Sidebar shows active vs disabled counts.
- [ ] The summary respects collapsed sidebar behavior.
- [ ] Counts update after integration mutations.
**Files You'll Likely Touch:** `apps/frontend/src/components/launches/launches.component.tsx` because it loads integrations, computes `totalNonDisabledChannels`, renders sidebar, and mutates after integration changes (`apps/frontend/src/components/launches/launches.component.tsx:352-431`, `apps/frontend/src/components/launches/launches.component.tsx:497-603`).  
**High-Level Implementation Plan:**
1. Add a derived disabled count near `totalNonDisabledChannels`.
2. Render a compact summary below the "Channels" header.
3. Hide or icon-only the summary when `collapseMenu === '1'`.
4. Confirm `mutate()` updates the counts after channel group/enable/disable actions.
**Tips:**
- Use the existing `integrations` value from `useIntegrationList` (`apps/frontend/src/components/launches/helpers/use.integration.list.tsx:7-23`).
- Follow Tailwind class style already used in the sidebar.
- Do not create another integration fetch.
**What Could Go Wrong:**
- Counting `undefined` while SWR is loading can render `NaN`.
- Showing long text in collapsed mode can overflow the 100px sidebar.
**Stretch Goal:** Add a tooltip in collapsed mode.  
**Connects To:** Story 5 because both derive useful UI from existing integration data.

## Story 5: Add Customer Filter Summary To Calendar Header
**Difficulty:** Medium  
**Estimated Time:** 3 hours  
**Skills You'll Practice:** context reading, filter state, non-mutating UI  
**The Story:** As a scheduler using customer groups, I want the calendar header to show when a customer filter is active so that I do not mistake filtered results for missing posts.  
**Acceptance Criteria:**
- [ ] When `customer` filter is set, a visible summary appears near filters.
- [ ] Clearing/changing filters updates the summary.
- [ ] The summary does not trigger extra API calls.
**Files You'll Likely Touch:** `apps/frontend/src/components/launches/filters.tsx` because it controls calendar filters, `apps/frontend/src/components/launches/calendar.context.tsx` because it stores `customer` and `setFilters` (`apps/frontend/src/components/launches/calendar.context.tsx:29-81`, `apps/frontend/src/components/launches/calendar.context.tsx:258-281`), and `apps/frontend/src/components/launches/launches.component.tsx` because integrations include customer grouping (`apps/frontend/src/components/launches/launches.component.tsx:405-422`).  
**High-Level Implementation Plan:**
1. Inspect `Filters` and `useCalendar`.
2. Read `customer` from context.
3. Derive customer name from integrations if needed.
4. Render the summary without changing fetch logic.
**Tips:**
- `CalendarContext` already exposes `customer` (`apps/frontend/src/components/launches/calendar.context.tsx:29-33`).
- URL state is updated in `setFiltersWrapper` (`apps/frontend/src/components/launches/calendar.context.tsx:258-281`).
- Customer grouping is already derived from integration customer ids.
**What Could Go Wrong:**
- Adding state in `Filters` that duplicates context can drift from URL state.
- Deriving customer name from a missing integration can render blank text.
**Stretch Goal:** Add a one-click clear filter control.  
**Connects To:** Story 6 because clearing filters may become a reusable state action.

## Story 6: Add A Save-As-Draft Confirmation Preference
**Difficulty:** Medium  
**Estimated Time:** 5 hours  
**Skills You'll Practice:** new user setting, SWR hook, frontend/backend endpoint  
**The Story:** As a frequent drafter, I want to control whether saving a draft asks for confirmation so that I can speed up my workflow.  
**Acceptance Criteria:**
- [ ] A setting exists in the settings UI with values `ASK` and `NO`.
- [ ] Composer checks the preference before saving draft.
- [ ] Preference persists per organization.
- [ ] Existing schedule/post-now behavior is unchanged.
**Files You'll Likely Touch:** `apps/frontend/src/components/settings/shortlink-preference.component.tsx` as the pattern for settings hook/UI (`apps/frontend/src/components/settings/shortlink-preference.component.tsx:16-116`), `apps/backend/src/api/routes/settings.controller.ts` because settings endpoints live there (`apps/backend/src/api/routes/settings.controller.ts:12-56`), `libraries/nestjs-libraries/src/database/prisma/schema.prisma` because org settings include `shortlink` (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:11-50`, `libraries/nestjs-libraries/src/database/prisma/schema.prisma:952-956`), and `apps/frontend/src/components/new-launch/manage.modal.tsx` because draft action uses `schedule('draft')` (`apps/frontend/src/components/new-launch/manage.modal.tsx:591-607`).  
**High-Level Implementation Plan:**
1. Add schema enum/field or reuse a simple string field after deciding migration shape.
2. Add service/repository methods near organization settings.
3. Add `GET`/`POST` routes in `SettingsController`.
4. Add a hook/component mirroring `useShortlinkPreference`.
5. In `ManageModal`, intercept `type === 'draft'` and preference `ASK` before submit.
**Tips:**
- Match `ShortLinkPreference` design if adding enum (`schema.prisma:952-956`).
- Keep the hook's SWR options consistent with `useShortlinkPreference`.
- Do not change `CreatePostDto.type`; draft already exists (`libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:93-96`).
**What Could Go Wrong:**
- Persisting per user instead of per org would not match existing org setting pattern.
- Forgetting to mutate the setting cache after save can show stale UI.
**Stretch Goal:** Add a small "do not ask again" option in the confirmation dialog.  
**Connects To:** Story 7 because you will add another stateful API-backed behavior.

## Story 7: Add A Revalidate Calendar Button
**Difficulty:** Medium  
**Estimated Time:** 4 hours  
**Skills You'll Practice:** context actions, UI controls, SWR mutation  
**The Story:** As a scheduler, I want a manual refresh button in the calendar header so that I can reload posts after external automation creates them.  
**Acceptance Criteria:**
- [ ] Button appears near calendar filters.
- [ ] Button calls existing calendar reload behavior.
- [ ] Button shows a loading/disabled state during reload if practical.
- [ ] No new backend endpoint is added.
**Files You'll Likely Touch:** `apps/frontend/src/components/launches/calendar.context.tsx` because it exposes `reloadCalendarView` (`apps/frontend/src/components/launches/calendar.context.tsx:316-345`), `apps/frontend/src/components/launches/filters.tsx` because filters/header controls live there, and `apps/frontend/src/components/launches/launches.component.tsx` because filters are rendered above `Calendar` (`apps/frontend/src/components/launches/launches.component.tsx:595-599`).  
**High-Level Implementation Plan:**
1. Inspect `Filters` and confirm it can call `useCalendar`.
2. Add a refresh button using existing UI/button conventions.
3. Call `reloadCalendarView`.
4. Optionally add transient local loading state around the click.
**Tips:**
- `reloadCalendarView` already mutates both calendar and list SWR data (`apps/frontend/src/components/launches/calendar.context.tsx:316-320`).
- Do not duplicate `mutateCalendar` and `mutateList` outside context.
- Keep button text/icon compact.
**What Could Go Wrong:**
- Calling only calendar mutate would not refresh list view.
- Adding state to context for a local spinner may be overkill.
**Stretch Goal:** Show last refreshed time.  
**Connects To:** Story 8 because external automation becomes more visible when refresh is reliable.

## Story 8: Add Notes To Scheduled Post Groups
**Difficulty:** Hard  
**Estimated Time:** 10 hours  
**Skills You'll Practice:** database migration, DTO validation, composer UI, calendar display, repository read/write  
**The Story:** As a scheduler, I want to add a private note to a scheduled post group so that I remember campaign context without putting it in the public post.  
**Acceptance Criteria:**
- [ ] Composer can add/edit a private note for a post group.
- [ ] Note is stored in the database and scoped to the organization.
- [ ] Calendar card can show a private-note indicator.
- [ ] Editing a post preserves the existing note.
- [ ] Note is never included in public preview content.
**Files You'll Likely Touch:** `schema.prisma` because `Post` currently has title/description but no explicit private note (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:393-440`), `create.post.dto.ts` because create payload validation lives there (`libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:93-125`), `manage.modal.tsx` because payload is built there (`apps/frontend/src/components/new-launch/manage.modal.tsx:372-401`), `posts.repository.ts` because create/read selects live there (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:173-196`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`), and `calendar.tsx` because cards render post content and actions (`apps/frontend/src/components/launches/calendar.tsx:973-1178`).  
**High-Level Implementation Plan:**
1. Decide whether note belongs on first post row in a group or every row; prefer first row to match tag behavior.
2. Add a nullable `privateNote` field to `Post`.
3. Add optional DTO field and preserve it through `mapTypeToPost`.
4. Add UI input in composer settings or footer.
5. Include field in `data` payload.
6. Write field in repository upsert for first post.
7. Select/read field for calendar and group edit.
8. Render an indicator without exposing note publicly.
**Tips:**
- Tags are only applied to first post in group (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:563-599`).
- Public preview uses separate public controller paths, so verify you do not add note there (`apps/backend/src/api/routes/public.controller.ts:56-81`).
- Existing edit data loads group posts through `/posts/group/:group` (`apps/frontend/src/components/launches/calendar.tsx:109-173`).
**What Could Go Wrong:**
- Saving note on child rows can duplicate or desync it.
- Selecting note in public endpoints can leak private context.
- Updating old groups may soft-delete the row carrying the note if not copied forward.
**Stretch Goal:** Add note search/filter in list view.  
**Connects To:** Story 9 because both require full-stack persistence and read-model changes.

## Story 9: Add A Failed Post Retry Action
**Difficulty:** Hard  
**Estimated Time:** 12 hours  
**Skills You'll Practice:** backend state transition, calendar action, workflow restart, permission review  
**The Story:** As a scheduler, I want to retry a failed post from the calendar so that I can recover publishing errors without recreating the post.  
**Acceptance Criteria:**
- [ ] Failed post cards show a retry action.
- [ ] Retry action is only shown for `ERROR` state.
- [ ] Backend endpoint changes state back to queue and restarts workflow.
- [ ] UI refreshes after retry.
- [ ] Error message is preserved or cleared intentionally.
**Files You'll Likely Touch:** `calendar.tsx` because `CalendarItem` marks `ERROR` posts and renders actions (`apps/frontend/src/components/launches/calendar.tsx:1031-1140`), `posts.controller.ts` because post state endpoints live there (`apps/backend/src/api/routes/posts.controller.ts:213-221`), `posts.service.ts` because `changeState`, `changePostStatus`, and workflow restart patterns exist (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:814-841`), `posts.repository.ts` because state changes persist there (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:393-439`), and `schema.prisma` because `State` defines `ERROR` and `QUEUE` (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:903-908`).  
**High-Level Implementation Plan:**
1. Add a backend endpoint such as `POST /posts/:id/retry`.
2. In service, load post by id/org and verify current state is `ERROR`.
3. Change state to `QUEUE`, decide whether to clear `error`, and restart workflow.
4. Add frontend action in `CalendarItem` only when `state === 'ERROR'`.
5. Call the endpoint with `useFetch`, show toaster result, and call mutate/reload.
6. Verify old draft/update semantics are not affected.
**Tips:**
- Existing error marker is rendered at `calendar.tsx:1044-1052`.
- Workflow restart code exists in `changePostStatus` and `changeDate` (`posts.service.ts:831-838`, `posts.service.ts:861-871`).
- Keep org scoping through `GetOrgFromRequest`.
**What Could Go Wrong:**
- Retrying a post from another org if repository lookup is not org-scoped.
- Starting workflow without changing state can leave UI in error.
- Clearing error too early can erase useful debugging context.
**Stretch Goal:** Add retry count tracking.  
**Connects To:** Story 10 because retry/caching/queue behavior forces architectural thinking.

## Story 10: Add Calendar Response Caching With Safe Invalidation
**Difficulty:** Expert  
**Estimated Time:** 20 hours  
**Skills You'll Practice:** caching design, invalidation, API contracts, performance, correctness  
**The Story:** As a high-volume scheduler, I want the calendar to load faster for stable date ranges so that large teams can navigate without waiting on repeated database reads.  
**Acceptance Criteria:**
- [ ] `GET /posts` responses can be cached by org, date range, display, and customer.
- [ ] Cache invalidates after creating, updating, deleting, retrying, or rescheduling posts.
- [ ] Recurring posts remain correct for the requested range.
- [ ] Auth/org isolation is preserved.
- [ ] There is a documented fallback when Redis is unavailable.
**Files You'll Likely Touch:** `calendar.context.tsx` because it constructs query params and SWR keys (`apps/frontend/src/components/launches/calendar.context.tsx:165-231`), `posts.controller.ts` because cacheable endpoints and invalidating mutations live there (`apps/backend/src/api/routes/posts.controller.ts:111-229`), `posts.service.ts` because reads/writes and workflow starts are centralized (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:324-337`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-875`), `posts.repository.ts` because expensive reads and recurrence expansion live there (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:129-220`), `redis.service.ts` because Redis client/service exists (`libraries/nestjs-libraries/src/redis/redis.service.ts`), and `schema.prisma` for indexes supporting cache misses (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:431-440`).  
**High-Level Implementation Plan:**
1. Define cache key shape: org id, start date, end date, display, customer.
2. Add service-level read-through cache around `getPostsMinified`.
3. Keep repository unchanged initially; cache should wrap, not replace, correctness.
4. Add invalidation helper for org/date/customer ranges or broad org calendar cache.
5. Call invalidation after create, delete, date change, retry, and update flows.
6. Add logging/metrics around hit/miss if local pattern exists.
7. Document fallback: if Redis errors, query repository directly and do not fail user request.
8. Manually verify list view is not accidentally served calendar cache unless explicitly designed.
**Tips:**
- `AppModule` already uses Redis-backed throttler storage (`apps/backend/src/app.module.ts:35-43`).
- Calendar SWR already reduces frontend revalidation churn (`apps/frontend/src/components/launches/calendar.context.tsx:201-231`).
- Cache below auth middleware, never above org resolution.
- Recurrence expansion depends on requested end date (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:199-219`).
**What Could Go Wrong:**
- Cache key missing `customer` can leak or mix filtered results.
- Invalidation after write can miss update/delete paths and show stale posts.
- Caching expanded recurring posts for one range and reusing for another range can be incorrect.
**Stretch Goal:** Add ETag-style response headers for calendar reads.  
**Connects To:** Story 9 because retry is another mutation that must invalidate cached calendar reads.
