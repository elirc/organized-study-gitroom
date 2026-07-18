# 20 Junior Engineer User Stories With Senior Implementation Plans

This document is a product-owner style backlog for a junior engineer, with senior-engineer implementation guidance. Each story is scoped to teach one real pattern in this codebase while still producing a user-visible improvement.

Use the stories in order. Early tickets are UI-only; later tickets touch API, validation, persistence, and cache/state refresh. For every story, read the cited files before editing. The goal is not just to finish tickets; it is to learn how Postiz moves from UI state to API contracts to Prisma-backed persistence.

## Story 1: Improve The List View Empty State

**PM Story:** As a scheduler viewing posts in list mode, I want the empty state to mention filters so that I understand whether there are truly no posts or I am looking at a filtered result.

**Acceptance Criteria:**

- When list view has no posts, the message says there are no upcoming posts for the current filters.
- Loading behavior is unchanged.
- Day, week, and month views are unchanged.

**Primary Code Path:**

- `apps/frontend/src/components/launches/calendar.tsx:493-532` owns `ListView` loading and empty rendering.
- `apps/frontend/src/components/launches/calendar.context.tsx:285-291` derives `listPosts` and pagination totals.
- `apps/frontend/src/components/launches/filters.tsx:337-397` shows list pagination controls.

**Senior Implementation Plan:**

1. Open `Calendar.tsx` at the `ListView` component. Read from `ListView` declaration through the empty-state return (`apps/frontend/src/components/launches/calendar.tsx:493-532`).
2. Confirm that the `loading` branch returns before the empty-state branch. Do not move or refactor this logic.
3. Change only the fallback copy in the empty-state `t(...)` call. Keep the translation key stable if you want existing translations to continue working, or add a new key if the product wording is meaningfully different.
4. Keep the text compact. This is an operational tool; avoid explanatory paragraphs inside the app.
5. Manually verify by switching to list view with no data or a customer filter that returns no posts.

**Implementation Notes For Junior Engineer:**

- Do not touch backend code. The frontend already knows the empty state because `listPosts.length === 0` (`apps/frontend/src/components/launches/calendar.tsx:522-530`).
- Do not add another fetch. Empty state is purely UI.
- Keep Tailwind classes unchanged unless the copy overflows.

**Review Checklist:**

- Does `loading` still render first?
- Does the change only affect list view?
- Does the empty state still fit in the current centered layout?

## Story 2: Add A Manual Calendar Refresh Button

**PM Story:** As a scheduler, I want a refresh button near the calendar filters so that I can reload posts created by API, MCP, CLI, or other automations without refreshing the browser.

**Acceptance Criteria:**

- A refresh button appears near the existing calendar/list controls.
- Clicking it reloads both calendar and list data.
- The button does not add a new backend endpoint.
- The button uses existing calendar context behavior.

**Primary Code Path:**

- `apps/frontend/src/components/launches/calendar.context.tsx:316-320` defines `reloadCalendarView`.
- `apps/frontend/src/components/launches/calendar.context.tsx:325-345` exposes `reloadCalendarView` through context.
- `apps/frontend/src/components/launches/filters.tsx:43-487` owns the filter/header control UI.
- `apps/frontend/src/components/launches/launches.component.tsx:595-599` renders `Filters` above `Calendar`.

**Senior Implementation Plan:**

1. Open `filters.tsx` and find the top-level `Filters` component (`apps/frontend/src/components/launches/filters.tsx:43-487`).
2. Notice that it already calls `const calendar = useCalendar()` (`apps/frontend/src/components/launches/filters.tsx:43-45`). That means you already have access to `calendar.reloadCalendarView`.
3. Add a small button near the view toggle controls, not inside every view branch. A good location is after the calendar/list toggle around `filters.tsx:435-485`.
4. On click, call `calendar.reloadCalendarView()`.
5. If you add temporary loading state, keep it local to `Filters`; do not change `CalendarContext` for this simple interaction.
6. Use existing visual language: small height, border, `bg-newBgColorInner`, hover styles similar to neighboring controls.
7. Verify in both calendar view and list view. In list view, `reloadCalendarView` should call both `mutateCalendar` and `mutateList` (`apps/frontend/src/components/launches/calendar.context.tsx:316-320`), so you should not write separate logic for each view.

**Implementation Notes For Junior Engineer:**

- The button is a UI action over existing SWR mutate functions.
- Do not call `fetch('/posts')` manually from the button.
- Do not import SWR into `filters.tsx`; context already hides that detail.

**Review Checklist:**

- Does one click refresh both `/posts` and `/posts/list` paths?
- Does the button work when `display === 'list'`?
- Does the button avoid layout overflow on smaller widths?

## Story 3: Show Active Customer Filter Summary

**PM Story:** As a scheduler managing clients, I want to see when a customer filter is active so that I do not confuse a filtered calendar with an empty calendar.

**Acceptance Criteria:**

- When `calendar.customer` is set, the filters row shows a compact active-filter summary.
- The summary updates when customer changes.
- The summary does not trigger additional API calls.
- The summary disappears when no customer is selected.

**Primary Code Path:**

- `apps/frontend/src/components/launches/calendar.context.tsx:29-33` includes `customer` in context defaults.
- `apps/frontend/src/components/launches/calendar.context.tsx:258-281` updates filters and URL state.
- `apps/frontend/src/components/launches/filters.tsx:172-185` sets customer filters.
- `apps/frontend/src/components/launches/launches.component.tsx:405-422` groups integrations by customer.

**Senior Implementation Plan:**

1. In `filters.tsx`, inspect `setCustomer` (`apps/frontend/src/components/launches/filters.tsx:172-185`) and `SelectCustomer` rendering (`apps/frontend/src/components/launches/filters.tsx:399-403`).
2. Derive `selectedCustomerName` from `calendar.integrations`. Find the first integration whose `customer?.id` equals `calendar.customer`, then use `customer.name`.
3. Render a small summary next to `SelectCustomer`, such as `Filtered: Acme`, only when `calendar.customer` is truthy.
4. Keep the summary derived from existing context. Do not add new state.
5. Make sure this works in list view and calendar view, because the filter row is shared.
6. If no matching name is found, render a generic `Filtered` label rather than crashing.

**Implementation Notes For Junior Engineer:**

- `calendar.integrations` already exists in context (`apps/frontend/src/components/launches/calendar.context.tsx:40-42`, `apps/frontend/src/components/launches/calendar.context.tsx:333-334`).
- The URL updates are already handled by `setFiltersWrapper`; do not edit history logic unless your UI needs a clear action.
- Start with display only. A clear-filter button can be a follow-up.

**Review Checklist:**

- Does it render only when `calendar.customer` exists?
- Does it handle unknown/missing customer names safely?
- Does it avoid a second integration fetch?

## Story 4: Add Active/Disabled Channel Counts

**PM Story:** As a scheduler, I want to see active and disabled channel counts in the sidebar so that I understand how many connected channels are currently usable.

**Acceptance Criteria:**

- Sidebar shows active channel count.
- Sidebar shows disabled channel count.
- Counts update after integration enable/disable/group changes.
- Collapsed sidebar remains visually clean.

**Primary Code Path:**

- `apps/frontend/src/components/launches/launches.component.tsx:352-365` loads integrations.
- `apps/frontend/src/components/launches/launches.component.tsx:366-371` derives `totalNonDisabledChannels`.
- `apps/frontend/src/components/launches/launches.component.tsx:511-546` renders sidebar header and create buttons.
- `apps/frontend/src/components/launches/launches.component.tsx:423-431` refreshes integration data after updates.

**Senior Implementation Plan:**

1. Open `LaunchesComponent` and read the existing `totalNonDisabledChannels` derived value (`apps/frontend/src/components/launches/launches.component.tsx:366-371`).
2. Add `totalDisabledChannels` with `useMemo`, using the same `integrations` dependency.
3. Render the counts below or near the `Channels` heading (`apps/frontend/src/components/launches/launches.component.tsx:511-537`).
4. When `collapseMenu === '1'`, hide text labels or use a tooltip so the sidebar does not overflow. The collapsed width is set at `launches.component.tsx:501-504`.
5. Do not add a new API endpoint; integration state already comes from `useIntegrationList`.
6. Verify that `mutate()` after `changeItemGroup` and other menu actions updates counts (`apps/frontend/src/components/launches/launches.component.tsx:372-397`, `apps/frontend/src/components/launches/launches.component.tsx:423-431`).

**Implementation Notes For Junior Engineer:**

- Be defensive when `integrations` is fallback data. `useIntegrationList` provides `fallbackData: []` (`apps/frontend/src/components/launches/helpers/use.integration.list.tsx:14-22`).
- Use short labels: `Active 4`, `Disabled 1`.
- Do not count deleted integrations; the backend list endpoint should already filter what it returns, so count only what the UI receives.

**Review Checklist:**

- No `NaN` while loading.
- No text overflow in collapsed sidebar.
- Counts update after `mutate`.

## Story 5: Show Scheduled Time On Month Cards

**PM Story:** As a scheduler using month view, I want each post card to show its scheduled time so that I can scan upcoming posts without opening each one.

**Acceptance Criteria:**

- Month view post cards show local scheduled time.
- Day and week view behavior does not regress.
- Time format follows existing US/non-US logic.

**Primary Code Path:**

- `apps/frontend/src/components/launches/calendar.tsx:416-492` defines `MonthView`.
- `apps/frontend/src/components/launches/calendar.tsx:583-972` defines `CalendarColumn`.
- `apps/frontend/src/components/launches/calendar.tsx:973-1178` defines `CalendarItem`, including `showTime`.
- `apps/frontend/src/components/launches/calendar.tsx:1170-1173` renders time when `showTime` is true.

**Senior Implementation Plan:**

1. Start at `CalendarItem` and confirm `showTime` already exists (`apps/frontend/src/components/launches/calendar.tsx:973-991`).
2. Find where `CalendarColumn` renders `CalendarItem` (`apps/frontend/src/components/launches/calendar.tsx:863-876` from prior inspection).
3. Identify the `display` value passed to `CalendarItem`. If it already knows `display === 'month'`, pass `showTime={display === 'month'}`.
4. Do not change time formatting logic. It already uses `newDayjs(post.publishDate).local().format(isUSCitizen() ? 'hh:mm A' : 'HH:mm')` (`apps/frontend/src/components/launches/calendar.tsx:1170-1173`).
5. Verify month view with multiple posts on one date; the time should not crush the text.

**Implementation Notes For Junior Engineer:**

- This is likely a one-line prop change once you find the render site.
- Do not duplicate date formatting.
- Do not show seconds; the existing format is already right for this app.

**Review Checklist:**

- Month card content still clamps cleanly.
- Week/day do not unexpectedly show extra time unless already intended.
- Time is local, not UTC.

## Story 6: Add A "Current Page Has X Posts" List Summary

**PM Story:** As a scheduler in list view, I want to know how many posts are on the current page so that I can understand whether pagination is working.

**Acceptance Criteria:**

- List view shows the number of posts on the current page.
- It appears only in list view.
- It updates when moving between pages.
- It does not require backend changes.

**Primary Code Path:**

- `apps/frontend/src/components/launches/calendar.context.tsx:187-231` fetches list data.
- `apps/frontend/src/components/launches/calendar.context.tsx:288-291` derives `listPosts` and `listTotalPages`.
- `apps/frontend/src/components/launches/filters.tsx:337-397` renders list pagination.
- `apps/frontend/src/components/launches/calendar.tsx:493-532` renders list view.

**Senior Implementation Plan:**

1. Decide where the count belongs. I recommend `filters.tsx` near the list pagination text, because it is a control/status row.
2. Extend the pagination display around `filters.tsx:365-368` to include `calendar.listPosts.length` if `listPosts` is exposed in context. It is exposed by `CalendarContext.Provider` (`apps/frontend/src/components/launches/calendar.context.tsx:339-343`).
3. If TypeScript complains about context default typing, update the default context shape in `calendar.context.tsx:29-81` rather than using `any`.
4. Show a compact label such as `25 posts`.
5. Verify with list pages: first page, middle page, empty page.

**Implementation Notes For Junior Engineer:**

- This is derived UI; no endpoint changes.
- Keep `Math.max(1, calendar.listTotalPages)` behavior intact (`apps/frontend/src/components/launches/filters.tsx:365-368`).
- Avoid counting `calendar.posts`, because that belongs to calendar view, not list view.

**Review Checklist:**

- Count changes when `listPage` changes.
- Count is hidden outside list view.
- No stale count while loading is acceptable only if the loading state hides the list.

## Story 7: Add A Clear Customer Filter Action

**PM Story:** As a scheduler filtering by customer, I want one click to clear the filter so that I can return to all channels quickly.

**Acceptance Criteria:**

- A clear action appears only when a customer filter is active.
- Clicking it sets `customer` to `null`.
- Current date range and display mode are preserved.
- URL query string is updated by existing filter logic.

**Primary Code Path:**

- `apps/frontend/src/components/launches/filters.tsx:172-185` sets customer filters.
- `apps/frontend/src/components/launches/calendar.context.tsx:258-281` updates filters and URL.
- `apps/frontend/src/components/launches/calendar.context.tsx:165-185` uses customer in `/posts` params.
- `apps/frontend/src/components/launches/calendar.context.tsx:187-199` uses customer in `/posts/list` params.

**Senior Implementation Plan:**

1. In `Filters`, add a `clearCustomer` callback next to `setCustomer`.
2. The callback should call `calendar.setFilters` with the current `startDate`, `endDate`, `display`, and `customer: null`.
3. Render the clear action near `SelectCustomer` (`apps/frontend/src/components/launches/filters.tsx:399-403`) only when `calendar.customer` exists.
4. Use a small icon/text button. Do not add a modal.
5. Verify the URL loses the customer parameter. Existing `setFiltersWrapper` builds the URL and filters out empty values (`apps/frontend/src/components/launches/calendar.context.tsx:274-280`).

**Implementation Notes For Junior Engineer:**

- Do not manually call `window.history.replaceState`; context already does this.
- Do not reset date range or display mode.
- Use `null`, not empty string, to match context default (`apps/frontend/src/components/launches/calendar.context.tsx:32`).

**Review Checklist:**

- Clear action appears only while filtered.
- Calendar data refetches with all customers.
- List view also clears correctly.

## Story 8: Add A Compact "Created Via" Indicator To List View Rows

**PM Story:** As a product owner reviewing automation usage, I want list view rows to show how each post was created so that I can distinguish web-created posts from API/MCP/CLI/autopost posts.

**Acceptance Criteria:**

- List view rows show existing `CreationMethodBadge` when `creationMethod` is not `UNKNOWN`.
- Calendar day/week/month behavior is unchanged unless already using the badge.
- Badge does not show for normal unknown values.

**Primary Code Path:**

- `apps/frontend/src/components/launches/calendar.tsx:493-567` defines list view grouping/rendering.
- `apps/frontend/src/components/launches/calendar.tsx:1008-1060` shows existing badge behavior in `CalendarItem`.
- `apps/frontend/src/components/launches/creation.method.badge.tsx:4-50` defines the badge.
- `libraries/nestjs-libraries/src/database/prisma/schema.prisma:943-950` defines `CreationMethod`.

**Senior Implementation Plan:**

1. Inspect list view rendering in `calendar.tsx` beginning at `ListView` (`apps/frontend/src/components/launches/calendar.tsx:493-567`). Find the row/card component used for each list post.
2. Import or reuse `CreationMethodBadge`; it is already imported near the top for calendar items (`apps/frontend/src/components/launches/calendar.tsx:53-55` from inspection).
3. Render the badge near title/content/time, using `post.creationMethod`.
4. Keep the same visibility rules as `CalendarItem`: do not show unknown values. You can rely on `CreationMethodBadge` returning `null` (`apps/frontend/src/components/launches/creation.method.badge.tsx:21-23`).
5. Verify list view with posts created from different methods if data exists; otherwise inspect rendered DOM with mocked/dev data.

**Implementation Notes For Junior Engineer:**

- Avoid creating a second badge component.
- Keep the badge small; list rows are dense.
- Make sure list query selects `creationMethod`; calendar query already selects it (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:173-183`). If list query does not, update backend select as part of this story.

**Review Checklist:**

- Badge appears only when data exists.
- No runtime error for old posts.
- Backend read model includes `creationMethod` for list rows if needed.

## Story 9: Add Shortlink Preference Save Feedback

**PM Story:** As an admin changing shortlink settings, I want clear feedback while the setting is saving so that I do not click repeatedly or wonder whether the change worked.

**Acceptance Criteria:**

- Shortlink preference control disables or shows saving state during save.
- Success toast still appears after save.
- Local state and SWR cache stay in sync.
- Failed save restores prior value or reports failure.

**Primary Code Path:**

- `apps/frontend/src/components/settings/shortlink-preference.component.tsx:16-31` defines the SWR hook.
- `apps/frontend/src/components/settings/shortlink-preference.component.tsx:33-116` renders and saves the setting.
- `apps/backend/src/api/routes/settings.controller.ts:51-66` exposes backend shortlink endpoints.
- `libraries/nestjs-libraries/src/database/prisma/organizations/organization.repository.ts:401-420` persists the preference.

**Senior Implementation Plan:**

1. Open `ShortlinkPreferenceComponent`. Read `localValue`, sync effect, and `handleChange` (`apps/frontend/src/components/settings/shortlink-preference.component.tsx:33-64`).
2. Add a local `saving` boolean.
3. In `handleChange`, capture the previous value before setting the new one.
4. Set `saving` true, optimistically set local state, send `POST /settings/shortlink`, call `mutate({ shortlink: newValue })`, then set `saving` false.
5. Wrap the fetch in `try/catch`. On catch, restore previous local value, optionally `mutate({ shortlink: previousValue })`, and show a warning toast.
6. Pass disabled state to `Select` if the shared component supports it; if not, wrap interaction with a guard.
7. Verify that backend route is admin-gated (`apps/backend/src/api/routes/settings.controller.ts:56-66`), so a non-admin failure should not leave optimistic UI stuck.

**Implementation Notes For Junior Engineer:**

- Do not change the backend unless you discover response shape problems.
- Keep SWR key `shortlink-preference` unchanged (`apps/frontend/src/components/settings/shortlink-preference.component.tsx:23-30`).
- Use the existing toaster pattern from the same file (`apps/frontend/src/components/settings/shortlink-preference.component.tsx:60-63`).

**Review Checklist:**

- Saving flag clears in success and error.
- UI is not left with a value the server rejected.
- No duplicate toasts on one change.

## Story 10: Add Email Notification Save Error Handling

**PM Story:** As a user changing email notification settings, I want the UI to recover if saving fails so that the toggles do not lie about my preferences.

**Acceptance Criteria:**

- Toggle still updates optimistically.
- If the save fails, UI returns to previous settings.
- Failure toast appears.
- Existing successful-save behavior stays the same.

**Primary Code Path:**

- `apps/frontend/src/components/settings/email-notifications.component.tsx:16-31` loads settings.
- `apps/frontend/src/components/settings/email-notifications.component.tsx:33-76` handles local state and saving.
- `apps/frontend/src/components/settings/global.settings.tsx:16-24` renders this component inside global settings.
- `apps/backend/src/api/routes/users.controller.ts:148-153` exposes email notification endpoints from prior route inspection.

**Senior Implementation Plan:**

1. Read `updateSetting` carefully (`apps/frontend/src/components/settings/email-notifications.component.tsx:56-76`).
2. It currently uses `settingsRef.current` to avoid stale closure problems (`apps/frontend/src/components/settings/email-notifications.component.tsx:45-47`). Keep that pattern.
3. Store `previousSettings` before calling `setLocalSettings(newData)`.
4. Wrap the `fetch('/user/email-notifications', ...)` call in `try/catch`.
5. On success, keep the existing success toast.
6. On failure, call `setLocalSettings(previousSettings)`, update `settingsRef.current`, and show a warning toast.
7. Consider adding a local `savingKey` if you want to disable only the active slider; keep it simple if the shared `Slider` does not expose disabled state.

**Implementation Notes For Junior Engineer:**

- Do not remove the ref; it is there to keep latest state available inside callbacks.
- Do not refetch settings just to recover; you already have `previousSettings`.
- Keep copy short.

**Review Checklist:**

- A failed request does not leave local state wrong.
- Fast toggles do not use stale data.
- Success path still shows `settings_updated`.

## Story 11: Add A "No Channels Yet" CTA To Connect A Channel

**PM Story:** As a new user with no channels connected, I want the empty state to directly offer connecting a channel so that I know the next action.

**Acceptance Criteria:**

- Empty sidebar state includes a clear connect action.
- Action reuses existing add-provider flow.
- Empty state appears only when there are zero integrations and sidebar is expanded.
- Collapsed sidebar remains unchanged.

**Primary Code Path:**

- `apps/frontend/src/components/launches/launches.component.tsx:538-546` shows `AddProviderButton` and `NewPost`.
- `apps/frontend/src/components/launches/launches.component.tsx:547-568` renders no-channel empty state.
- `apps/frontend/src/components/launches/add.provider.component.tsx` owns add-provider behavior.

**Senior Implementation Plan:**

1. Inspect `LaunchesComponent` empty state around `sortedIntegrations.length === 0 && collapseMenu === '0'` (`apps/frontend/src/components/launches/launches.component.tsx:547-568`).
2. Confirm `AddProviderButton` is already imported and rendered above the empty state (`apps/frontend/src/components/launches/launches.component.tsx:3`, `apps/frontend/src/components/launches/launches.component.tsx:538-540`).
3. Add a small CTA inside the empty state. Prefer reusing `AddProviderButton update={() => update(true)}` rather than inventing a new route.
4. Make sure there are not two confusing add buttons too close together. If the existing top add button is already visible, consider making the empty state copy point toward it instead of duplicating it.
5. Verify expanded empty sidebar and collapsed sidebar.

**Implementation Notes For Junior Engineer:**

- The product goal is clarity, not another button at all costs.
- Reuse existing add-provider wiring so integration refresh behavior remains consistent.
- Do not change `NewPost`; it intentionally only appears when integrations exist (`apps/frontend/src/components/launches/launches.component.tsx:541-544`).

**Review Checklist:**

- No duplicate confusing CTAs.
- CTA uses existing provider modal/flow.
- Empty state remains centered and readable.

## Story 12: Add A Visible Draft Label In List View

**PM Story:** As a scheduler reviewing upcoming content, I want draft posts to be clearly labeled in list view so that I do not mistake them for queued posts.

**Acceptance Criteria:**

- Draft posts show a visible `Draft` label in list view.
- Non-draft posts are unchanged.
- Existing calendar card draft behavior remains unchanged.

**Primary Code Path:**

- `apps/frontend/src/components/launches/calendar.tsx:493-567` renders list view.
- `apps/frontend/src/components/launches/calendar.tsx:1160-1167` already prefixes draft cards in `CalendarItem`.
- `libraries/nestjs-libraries/src/database/prisma/schema.prisma:903-908` defines `State`.
- `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:173-196` selects `state` for calendar reads.

**Senior Implementation Plan:**

1. Find the list row/card body inside `ListView`.
2. Add a conditional `post.state === 'DRAFT'` label near the content summary.
3. Match existing translation style using `t('draft', 'Draft')`, because `CalendarItem` already uses that fallback (`apps/frontend/src/components/launches/calendar.tsx:1160-1162`).
4. Keep the label visually compact. A small pill or prefix is enough.
5. Verify list rows for `QUEUE`, `DRAFT`, `ERROR`, and `PUBLISHED` if sample data allows.

**Implementation Notes For Junior Engineer:**

- Do not alter backend state names.
- Do not infer draft from date or missing release id; use `state`.
- If list endpoint does not include `state`, add it to the select in repository.

**Review Checklist:**

- Draft label is visible.
- No labels on queued posts.
- No layout shift for rows without labels.

## Story 13: Add Delete Confirmation Details For Post Groups

**PM Story:** As a scheduler deleting a post, I want the confirmation message to mention that the whole post group/thread may be deleted so that I understand the impact.

**Acceptance Criteria:**

- Delete confirmation text is clearer for grouped posts.
- Delete still calls the existing endpoint.
- Calendar refresh behavior is unchanged.
- Composer delete and calendar delete both get reviewed for copy.

**Primary Code Path:**

- `apps/frontend/src/components/launches/calendar.tsx:197-220` deletes posts from calendar actions.
- `apps/frontend/src/components/new-launch/manage.modal.tsx:180-200` deletes an existing post from the composer.
- `apps/backend/src/api/routes/posts.controller.ts:205-211` maps `DELETE /posts/:group`.
- `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:602-630` shows group/update behavior; delete service path should also be inspected.

**Senior Implementation Plan:**

1. Inspect both frontend delete flows. Calendar delete uses `deleteDialog` and then `fetch(`/posts/${post.group}`, { method: 'DELETE' })` (`apps/frontend/src/components/launches/calendar.tsx:197-220`). Composer delete does the same with `existingData.group` (`apps/frontend/src/components/new-launch/manage.modal.tsx:180-200`).
2. Update copy to mention "post or thread" or "this scheduled group" depending on product language.
3. Keep endpoint unchanged. The backend expects a group value (`apps/backend/src/api/routes/posts.controller.ts:205-211`).
4. Verify both paths still call `mutate()` or `reloadCalendarView()` after deletion.
5. Do not change deletion semantics in this story.

**Implementation Notes For Junior Engineer:**

- This is copy plus careful trace, not a backend behavior change.
- The key learning is that `group` is the deletion identifier, not necessarily a single row id.
- Use existing `deleteDialog` helper.

**Review Checklist:**

- Both delete surfaces use improved wording.
- Delete still works.
- Calendar reload still happens.

## Story 14: Add Search Clear Button In Media Library

**PM Story:** As a user searching media, I want to clear the search quickly so that I can return to browsing all media.

**Acceptance Criteria:**

- Clear button appears when search text is non-empty.
- Clicking clear resets search and reloads all media.
- Existing pagination/search behavior remains intact.

**Primary Code Path:**

- `apps/frontend/src/components/media/media.component.tsx:1-51` imports media dependencies.
- `apps/frontend/src/components/media/media.component.tsx` owns media list/search UI and uses SWR/fetch.
- `apps/backend/src/api/routes/media.controller.ts:181-188` exposes `GET /media?page&search`.

**Senior Implementation Plan:**

1. Search within `media.component.tsx` for `search`, `setSearch`, and `params`. The backend route expects `page` and optional `search` (`apps/backend/src/api/routes/media.controller.ts:181-188`).
2. Find the search input UI.
3. Add a clear button or icon inside/next to the search control when search value is non-empty.
4. On click, set search to empty string and reset page to zero if page state exists.
5. Make sure SWR key or fetch params update naturally from state. Do not manually call backend unless existing pattern requires it.
6. Verify with a search that returns results and a search that returns no results.

**Implementation Notes For Junior Engineer:**

- Keep backend unchanged.
- Use existing icons from `apps/frontend/src/components/ui/icons` if an appropriate close icon exists; `media.component.tsx` already imports `CloseCircleIcon` (`apps/frontend/src/components/media/media.component.tsx:39-50`).
- Watch for debounce behavior if present.

**Review Checklist:**

- Clear button only appears with search text.
- Clear resets page/search consistently.
- No extra duplicate requests beyond normal SWR state update.

## Story 15: Add Media Alt Text Missing Indicator

**PM Story:** As a scheduler attaching images, I want to see when media is missing alt text so that I can improve accessibility before posting.

**Acceptance Criteria:**

- Media items with no `alt` show a subtle warning indicator.
- Media items with `alt` do not show the warning.
- No post submission is blocked in this story.
- Existing media selection behavior is unchanged.

**Primary Code Path:**

- `libraries/nestjs-libraries/src/database/prisma/schema.prisma:209-231` defines `Media.alt`.
- `apps/backend/src/api/routes/media.controller.ts:122-127` saves media information.
- `apps/frontend/src/components/media/media.component.tsx` renders media items.
- `apps/frontend/src/components/new-launch/manage.modal.tsx:385-398` serializes media alt into post payload.

**Senior Implementation Plan:**

1. Find where individual media tiles/cards are rendered in `media.component.tsx`.
2. Confirm the media object type includes `alt`; it imports `Media` from Prisma (`apps/frontend/src/components/media/media.component.tsx:18`).
3. Add a small warning mark when `!media.alt`.
4. Use tooltip text if the app's tooltip system is available in that component tree.
5. Do not block uploads or post submission. This ticket only creates visibility.
6. Verify that post payload already carries `alt` when present (`apps/frontend/src/components/new-launch/manage.modal.tsx:390-397`).

**Implementation Notes For Junior Engineer:**

- This is an accessibility nudge, not validation.
- Avoid loud red error styling; missing alt can be improved later.
- Do not change backend schema. `alt` already exists.

**Review Checklist:**

- Indicator appears only when `alt` is missing.
- Media selection still works.
- No new backend call is added.

## Story 16: Add A Composer Warning For Empty Alt Text

**PM Story:** As a scheduler composing posts with images, I want a warning when attached images have no alt text so that I can decide whether to fix them before publishing.

**Acceptance Criteria:**

- Composer detects attached media without alt text.
- Warning appears before schedule/post-now, but does not block draft saving.
- Warning does not block scheduling in this first version unless user explicitly cancels.
- Payload shape remains unchanged.

**Primary Code Path:**

- `apps/frontend/src/components/new-launch/manage.modal.tsx:248-334` performs validation before scheduling.
- `apps/frontend/src/components/new-launch/manage.modal.tsx:372-401` builds payload.
- `apps/frontend/src/components/new-launch/store.ts:10-48` defines media shape currently used in store.
- `apps/frontend/src/components/new-launch/manage.modal.tsx:390-397` serializes `alt`.

**Senior Implementation Plan:**

1. Read the validation section in `ManageModal` from `checkAllValid` through max length checks (`apps/frontend/src/components/new-launch/manage.modal.tsx:248-334`).
2. After existing blocking validation passes, derive whether any `checkAllValid` values contain media with missing `alt`.
3. Only run this warning for `type === 'schedule'` or `type === 'now'`; drafts should stay fast.
4. Use existing `deleteDialog` pattern to ask whether to continue without alt text.
5. If user cancels, set loading false and return.
6. Do not change the payload builder. It already includes `alt` when present (`apps/frontend/src/components/new-launch/manage.modal.tsx:390-397`).

**Implementation Notes For Junior Engineer:**

- Be careful where you place this: after content/settings checks, before shortlink prompt is reasonable.
- Avoid double prompts if no media is attached.
- You may need to widen local media types in `store.ts` to include `alt`, because current `Values.media` type only lists `id`, `path`, and optional `thumbnail` (`apps/frontend/src/components/new-launch/store.ts:10-15`).

**Review Checklist:**

- Draft save does not warn.
- Schedule/post-now warns only when needed.
- Loading state resets when user cancels.

## Story 17: Add A Default Calendar View Setting

**PM Story:** As a scheduler, I want to choose my default calendar view so that the calendar opens in my preferred mode.

**Acceptance Criteria:**

- User can choose default view: day, week, month, or list.
- Calendar uses that default when there is no URL display parameter.
- Current URL parameter still overrides default.
- Existing `calendar-display` cookie behavior is respected or intentionally replaced.

**Primary Code Path:**

- `apps/frontend/src/components/launches/calendar.context.tsx:141-163` reads URL/cookie display and initializes filters.
- `apps/frontend/src/components/launches/filters.tsx:90-170` switches between day/week/month/list.
- `apps/frontend/src/components/settings/global.settings.tsx:16-24` renders global settings components.
- `apps/frontend/src/components/settings/shortlink-preference.component.tsx:33-116` provides a settings UI pattern.

**Senior Implementation Plan:**

1. First decide storage. For a junior-friendly first version, use the existing `calendar-display` cookie rather than adding backend persistence. It is already read with `useCookie('calendar-display', 'week')` (`apps/frontend/src/components/launches/calendar.context.tsx:141-143`).
2. Add a `DefaultCalendarViewComponent` under settings, modeled after `ShortlinkPreferenceComponent` but writing to the same cookie.
3. Render it in `GlobalSettings` after or before shortlink preference (`apps/frontend/src/components/settings/global.settings.tsx:16-24`).
4. Confirm `CalendarWeekProvider` already prioritizes URL param over saved cookie (`apps/frontend/src/components/launches/calendar.context.tsx:141-143`).
5. Verify changing setting affects a fresh visit to `/launches` without `display` query.
6. Do not add backend code for this version.

**Implementation Notes For Junior Engineer:**

- This is a frontend preference; product can later decide whether it must sync across devices.
- Keep allowed values aligned with `display: 'week' | 'month' | 'day' | 'list'` (`apps/frontend/src/components/launches/calendar.context.tsx:56-60`).
- Make sure list view initializes a sensible date range (`apps/frontend/src/components/launches/filters.tsx:35-40`).

**Review Checklist:**

- URL query still wins over cookie.
- Setting persists after reload.
- TypeScript union includes all four values.

## Story 18: Add Draft Save Confirmation Preference

**PM Story:** As a frequent content drafter, I want to turn off draft-save confirmation so that I can save drafts quickly without extra clicks.

**Acceptance Criteria:**

- New organization setting stores draft confirmation preference.
- Settings UI lets admins change it.
- Composer reads the preference before saving drafts.
- Schedule and post-now flows are unchanged.

**Primary Code Path:**

- `libraries/nestjs-libraries/src/database/prisma/schema.prisma:11-50` defines organization settings fields.
- `apps/backend/src/api/routes/settings.controller.ts:51-66` shows the shortlink settings endpoint pattern.
- `libraries/nestjs-libraries/src/database/prisma/organizations/organization.service.ts:123-132` delegates shortlink setting updates.
- `libraries/nestjs-libraries/src/database/prisma/organizations/organization.repository.ts:401-420` reads/writes shortlink setting.
- `apps/frontend/src/components/settings/shortlink-preference.component.tsx:16-116` is the frontend settings pattern.
- `apps/frontend/src/components/new-launch/manage.modal.tsx:591-607` triggers draft save.

**Senior Implementation Plan:**

1. Add a Prisma enum or Boolean field to `Organization`. A junior-friendly option is an enum like `DraftSavePreference { ASK NO }`, matching `ShortLinkPreference` style (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:952-956`).
2. Add DTO under settings, following `ShortlinkPreferenceDto` usage in `SettingsController` (`apps/backend/src/api/routes/settings.controller.ts:7-8`, `apps/backend/src/api/routes/settings.controller.ts:56-66`).
3. Add repository methods `getDraftSavePreference` and `updateDraftSavePreference`, mirroring `getShortlinkPreference` and `updateShortlinkPreference` (`libraries/nestjs-libraries/src/database/prisma/organizations/organization.repository.ts:401-420`).
4. Add service pass-through methods, mirroring `OrganizationService` shortlink methods (`libraries/nestjs-libraries/src/database/prisma/organizations/organization.service.ts:123-132`).
5. Add `GET /settings/draft-save` and `POST /settings/draft-save` in `SettingsController`; admin-gate the POST like shortlink (`apps/backend/src/api/routes/settings.controller.ts:56-66`).
6. Add a frontend hook/component modeled after `ShortlinkPreferenceComponent`.
7. Render the setting in `GlobalSettings` (`apps/frontend/src/components/settings/global.settings.tsx:16-24`).
8. In `ManageModal`, call the hook and, inside `schedule('draft')`, ask confirmation only when preference is `ASK`.
9. Keep draft payload unchanged; only the prompt behavior changes.

**Implementation Notes For Junior Engineer:**

- Make one vertical slice first: backend GET returns default, frontend displays it.
- Then add POST save.
- Then integrate composer behavior.
- Run Prisma generate after schema changes.

**Review Checklist:**

- Setting is org-scoped, not global.
- Non-admin cannot update it.
- Draft save works even if preference fetch is still loading; default to safe `ASK`.

## Story 19: Add Retry Failed Post Action

**PM Story:** As a scheduler, I want to retry a failed post from the calendar so that I can recover publishing errors without recreating the content.

**Acceptance Criteria:**

- Retry action appears only on posts with `ERROR` state.
- Backend verifies the post belongs to the current org.
- Retry moves the post back to queue and restarts publishing workflow.
- Calendar refreshes after retry.
- Existing error tooltip remains useful before retry.

**Primary Code Path:**

- `apps/frontend/src/components/launches/calendar.tsx:1031-1052` renders error state marker.
- `apps/frontend/src/components/launches/calendar.tsx:1077-1139` renders hover actions.
- `apps/backend/src/api/routes/posts.controller.ts:213-221` shows a state-changing post endpoint pattern.
- `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:814-841` changes state and restarts workflow for status changes.
- `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:843-875` restarts workflow after date changes.
- `libraries/nestjs-libraries/src/database/prisma/schema.prisma:903-908` defines `ERROR` and `QUEUE`.

**Senior Implementation Plan:**

1. Backend first. Add a controller method such as `@Post('/:id/retry')` to `PostsController`.
2. It should use `@GetOrgFromRequest() org` and `@Param('id') id`.
3. Add `PostsService.retryPost(orgId, id)`.
4. In service, load the post using an org-scoped repository method. If current helper does not return exactly what you need, add one; do not trust client data.
5. If the post is not found or is not `ERROR`, throw a clear `BadRequestException`.
6. Change state to `QUEUE`, decide whether to clear the error field. For first version, keep error until successful publish unless product asks to clear it.
7. Restart workflow using the same provider identifier pattern in `changePostStatus`/`changeDate` (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:831-838`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:861-871`).
8. Frontend: in `CalendarItem`, add a retry hover action only when `state === 'ERROR'`.
9. The action calls `fetch(`/posts/${post.id}/retry`, { method: 'POST' })`, shows toast, and calls existing mutate/reload path through `usePostActions`.
10. Verify the action does not show for `DRAFT`, `QUEUE`, or `PUBLISHED`.

**Implementation Notes For Junior Engineer:**

- This is a full-stack story; implement backend and manually test with a forced/error data row if needed.
- Do not use `group` for retry unless product wants to retry every post in a thread.
- Keep org scoping strict.

**Review Checklist:**

- Retry endpoint is authenticated via `ApiModule` because `PostsController` is in authenticated controllers (`apps/backend/src/api/api.module.ts:48-68`).
- No retry button on non-error states.
- Workflow restart uses provider identifier correctly.

## Story 20: Add Private Notes To Scheduled Post Groups

**PM Story:** As a scheduler, I want to add a private note to a scheduled post group so that I can remember campaign context without exposing it in the public post.

**Acceptance Criteria:**

- Composer supports adding/editing a private note.
- Note persists with the scheduled post group.
- Calendar shows a small note indicator when a note exists.
- Public preview does not expose the note.
- Updating a post preserves the note.

**Primary Code Path:**

- `libraries/nestjs-libraries/src/database/prisma/schema.prisma:393-440` defines `Post`.
- `libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:93-125` validates create/update payloads.
- `apps/frontend/src/components/new-launch/manage.modal.tsx:372-401` builds the post payload.
- `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630` creates/updates post rows.
- `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:173-196` selects fields for calendar reads.
- `apps/frontend/src/components/launches/calendar.tsx:973-1178` renders post cards.
- `apps/backend/src/api/routes/public.controller.ts:56-81` exposes public preview/comment paths.

**Senior Implementation Plan:**

1. Decide data model. Because a group can contain multiple rows, store the private note on the first/root post row, similar to how tags are applied only to the first post (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:563-599`).
2. Add `privateNote String?` to the `Post` model (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:393-440`).
3. Run Prisma generate/db push during implementation, not in this documentation task.
4. Add optional `privateNote` to `CreatePostDto` (`libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:93-125`).
5. Add local UI state in `ManageModal` or in `useLaunchStore`. For a junior, start with local state in `ManageModal` unless note must be edited across child components.
6. When editing an existing post, initialize the note from `existingData.posts[0].privateNote` after making sure group fetch selects it.
7. Add a compact textarea/input in the composer footer or settings area. Keep it private-facing; label it "Private note".
8. Include `privateNote` in the top-level payload built at `manage.modal.tsx:372-401`.
9. In `PostsService.mapTypeToPost`, ensure the field survives validation (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:243-292`).
10. In `PostsRepository.createOrUpdatePost`, write `privateNote` only for the first/root post. Be careful with updates: previous group rows are soft-deleted after new rows are created (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:617-628`), so the new root row must receive the note.
11. Add `privateNote` to calendar read select (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:173-196`) and group edit read if separate.
12. In `CalendarItem`, show a small note icon/indicator when `post.privateNote` exists. Do not render full note text on crowded cards.
13. Review public controller response paths and do not include `privateNote` in public preview (`apps/backend/src/api/routes/public.controller.ts:56-81`).

**Implementation Notes For Junior Engineer:**

- This is the hardest story in this list. Work in slices: schema/backend write, backend read, composer UI, calendar indicator, public leak check.
- Do not store notes in `content`; public post content and private campaign context must stay separate.
- Add a simple manual test plan before coding.

**Review Checklist:**

- Note is org-scoped through post ownership.
- Note survives edit/update.
- Note indicator appears in authenticated calendar only.
- Public preview does not expose private note.

## General Senior Guidance For All 20 Stories

Before coding, answer these questions in your branch notes:

1. What is the user-visible behavior?
2. Is this UI-only, API-backed, or persisted?
3. What existing component already owns the nearest behavior?
4. What endpoint, DTO, service, repository, and Prisma model are involved, if any?
5. What state must refresh after the change?
6. What can fail, and how does the UI recover?

Preferred trace order for this repo:

1. Route/component owner: `apps/frontend/src/app` or `apps/frontend/src/components`.
2. State owner: `CalendarWeekProvider`, `useLaunchStore`, SWR hook, or local component state.
3. Fetch owner: `useFetch` and `customFetch` (`libraries/helpers/src/utils/custom.fetch.func.ts:46-84`).
4. Backend route: `apps/backend/src/api/routes`.
5. Auth/org context: `AuthMiddleware` and request decorators (`apps/backend/src/services/auth/auth.middleware.ts:33-104`, `libraries/nestjs-libraries/src/user/org.from.request.ts:1-8`).
6. Service/business logic: usually under `libraries/nestjs-libraries/src/database/prisma`.
7. Repository/Prisma write: repository class and `schema.prisma`.
8. UI refresh: SWR `mutate`, context reload, or local optimistic state.
