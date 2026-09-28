# Aligned (teamaligned.com) — context notes

Captured 2026-09-28 from https://metrofloors.teamaligned.com/ in a read-only, logged-in session (user: Arron, User role, limited plan).
Companion to the HTML series in this folder (start at `index.html`). Nothing was saved, submitted, invited or created.
One app-initiated write was observed: `PUT /api/v1/itemMetadata/:id` fires when a task side panel is opened (read tracking).

Product in one line: a B2B digital sales room — per-deal, tabbed, branded micro-sites shared with buyers, with typed content sections,
a mutual action plan, comments, buyer AI assistant, seller-only internal tabs, and engagement analytics.

Object model: Account → Users → Room (dealmap/project) → Tabs (GENERAL | MAP | FILES) → Sections (typed) → Content items → Resources (Library).
Cross-cutting: Comments (public / internal note), Analytics events, Templates, Permissions & plan limits, AI (Agent, Client Assist, Deal Builder, Deal Insights).

Legend: "gated" = behind an upgrade prompt on this account; "inferred" = not confirmed by a tooltip or payload.

## Tech
- Next.js pages router (served under /__njs/_next/), pages: [userType]/dealmaps, [userType]/analytics/[dealMapId]; root #__next
- Styling: CSS Modules (Button_root__hash), Tailwind with `tw-` prefix and `!` important, Bootstrap dropdowns (.dropdown-menu.show), emotion (css), font Nunito Sans
- 3rd party: Segment, Intercom, Clarity, HubSpot, GTM, LinkedIn, FB, Bing, Cello (referrals, "Earn $5000")
- REST API /api/v1/... (XHR). Mongo ObjectId ids (24 hex). ?avk= query param appended.
- Rooms page API: accounts/:id, featuretoggles/:id/list, projects/room/my/all, projects/room/latestActivity/all, projects/room/team/all, integration/getIntegration/default/ACCOUNT/:id, permissions/:id/dealmap/:id, integration/groupsIntegrations, users/:id/account/:id, referral/tracking, cello/*, projects, projects/:id/alignedTemplates, projects/userRecentTemplates, groups/:id, billing/:id/info, limits/:id/getCurrentUsageByLimitType/CRM_ROOM_PER_USER_HUBSPOT/resource/:id, library, library/v2, upload, ipstack/check, accounts/:id/features, users/tutorialRoom, userChoices/list, aiDealBuilder/isEnabledForUser
## Rooms (/seller/dealmaps?filterByUser=latestActivity&group=all&roomStatus=all)
- Top: CRM connect banner (dismiss), onboarding progress stepper (Create a room > Customize > Share > Analytics; hover expands to cards with links), Upgrade, HubSpot/Salesforce icons, Invite Team
- Create a room (collapsible): AI Deal Builder (Beta), Duplicate room, template cards (Customer Onboarding, Digital Sales Room, Customer Onboarding), All templates ->
- Table toolbar: user filter dropdown (search user; Recent activity, User groups >, Archived rooms, Starred, My Rooms, [team members]); Sort dropdown (Order: Unread comments first / Follow sort order; Sort by: Starred, Name, CRM, Last Viewed, Total Time, Trend, Stage, Close Date, Team, Amount, Visitors, Progress, Plan Stage, Created Date); status (All Rooms, Active (CRM), Closed Won, Closed Lost - CRM gated); Column picker (Add team column; checkboxes CRM, Last Viewed, Total Time, Trend, Stage, Close Date, Team, Amount, Visitors, Progress, Plan Stage, Created Date; drag handles; CRM-gated ones have "Connect CRM" badge); Search; New Room
- Columns: Name (logo, hover: checkbox+star, comment count bubble, AI sparkle, kebab), Last viewed, Total time, Trend (Heating pill), Team (avatars + add), Visitors, Progress (% bar), Plan Stage, Created Date
- Row kebab: Duplicate, Share, Save as template, Analytics, | Settings, Archive, Delete
- Share = right drawer, tabs Share externally / Share with my team (URL ?tab=My+Team). External: Share with link + Copy Link, include thumbnail checkbox, Share via email (input + Preview & Send), Notifications accordion (Enable room notifications toggle, Send 'Room update' notification, Advanced notification settings link), Room access accordion radio: Anyone with link / ask for name / require email / only permitted users & companies / password protected. Team: invite by emails + Add to team, member list with role (Owner) / Add, Send a copy of this room to teammates.
- Room settings page /room/:id/settings?view=edit&tab=notifications|permissions|integrations. Notifications matrix: events x channel (Email/Slack/MS Teams): Enable room notifications, Room visited, Room is cold, Room not visited, Room is hot, Comment made, Allow Aligned to send client reminders, Prospect downloaded file, Prospect uploaded file/video, Visitor engaged HubSpot quote. Permissions: Enable comments (Upgrade badge), Enable AI Client Assist. Integrations: Connect a CRM first.
## Room editor (/room/:id/:tabUri?view=edit|preview)
- Top bar: logo, "< Menu" (back), segmented Edit/Preview/Analytics toggle (view= query), Agent button (AI), HubSpot/Salesforce icons, Comments counter (Alt+T notifications), presence avatar (S), Share (purple), user avatar
- Banner image (Edit banner), dual logos (seller + buyer, "+"), title "Seller - Buyer" with dropdown chevron, settings gear, star
- Tab strip: Overview, Mutual Action Plan, Quick-start guide, Internal (seller-only), File Sharing (hidden/greyed), + add tab, search icon
- Left sticky section TOC (scroll-spy), sections with headline + edit pencil; section toolbar: lock (restrict, inferred), ➤ Move to tab (confirmed via popover), duplicate, save-to-library, visibility eye, delete, drag handle; "Edit this section with AI" tooltip
- Per-section footer: Comment, Share (deep-link to section). "+" divider between sections to insert.
- Floating AI sparkle (AI Client Assist, Beta) bottom right; "Comments? Questions? click here to join the conversation"
- Room API on load: featuretoggles, comments/:roomId, projects/:id/project/:id(/settings), settings/notification/default, permissions, limits USER_PER_ACCOUNT, projects/project/:id/token, userContacts/list, analytics/dealmap/:id, projects/template/my|team, tabs/:id/overview/:tabUuid, tabs/all/:id, featuretoggles/room/:id/ROOM_USE_AI_CHAT_V2, projects/project/:id/users, resources/:id, integration ROOM, sellerAssist/dealEditor/debugEnabled, ai/hasChatV2/:id/:uuid
### Data model (from GET /api/v1/tabs/...)
- Tab: {_id, roomId, uri, type: GENERAL|MAP|FILES, title, isHidden, isInternal, allowedList[], tabContent}
- Room tabs here: Internal(GENERAL, internal, isInternal), Overview(GENERAL), Quick-start guide(GENERAL, overview-2), Mutual Action Plan(MAP, plan), File Sharing(FILES, resources, hidden)
- tabContent: {_id, roomId, tabId, sections[]}
- Section: {_id, type, headline, content[], comments, commentsCount, commentsNotesCount, enableHeadlineEditing, enableIsHidden, isHidden, isPreviewEditable, metadataId}
- Section types seen: SELLER (welcome card: description + ownerUserId), TEAM (stakeholders: sellerUser[], buyerUser[{index,user}]), SINGLE_EMBEDDED (resource), TIMELINE (items {title, description, isDone}), MULTIPLE_EMBEDDED (items {resource, headline}), LIST (items {title, description, resource})
- Content item generic: {_id, title, description, resource, resources[], subtasks[], sectionUser[], buyerUser[], sellerUser[], comments[], isDone, headline, ownerUserId}
- Resource: {_id, name, description, url, embeddedCodeSnippet, type: EMBEDDED_LINK|PDF|..., metaType, status, access, folderId, isArchived, isInternal, owner, parentResourceId}
- User: {_id, name, email, userType, role{key,title,department,hierarchyLevel,seniority}, accountId{_id,accountType}, picture, avatarColor, status, defaultView, anonymousEmail}
- Add Section drawer (react-bootstrap Offcanvas "NewSectionDrawer"): tabs Sections | AI sections | Integrations | Saved sections (scroll-anchored), search box.
  - Sections: Large Section, Carousel Section, Links & Examples, Call To Action, Next Steps, Text, Our Team, Process Overview, Files Sharing, Mobile View, Welcome
  - AI sections (Try Free): Business Case, Meeting Recap, Executive Summary, Onboarding Plan, POC Plan, Project Plan
  - Integrations: HubSpot Quote, PandaDoc, Calendly, Chili Piper, Gong (upgrade), Gong Embed (upgrade), G.Workspace, Hubspot Calendar, Sendspark, Loom, Miro, Highspot, Seismic, Outreach, Tolstoy, Vidyard, Vimeo, YouTube
  - Saved sections: team's saved sections (library)
- Empty media section shows paged grid picker: From Library, Loom, Embedded Link, YouTube, File Upload, Gong(upgrade), Gong Embed(upgrade), G.Workspace, Google Drive (+ more pages)
- PDF viewer: page counter 1/7, Page Width, View PDF; placeholder dashed "Replace me with your company deck"
- MULTIPLE_EMBEDDED: left viewer + right list of items (Calix/Apex/Driftline Case Study) + "Add Feature or Case Study"; extra toolbar icon (swap/layout)
- TIMELINE: horizontal stepper w/ done checks, current pin marker, dashed future, alternating above/below labels, horizontal scroll
- LIST: rows with logo + title + description, "Add element" placeholder row
## Mutual Action Plan tab (/room/:id/plan)
- Header: "Plan" + edit pencil, % progress bar, buttons "Reschedule due dates" (calendar) & "Notify assignees by email" (bell)
- Stage card (collapsible, "Steps", count done/total of *client-visible* steps 0/5 while 7 rows incl. 2 Internal)
- Step row: drag handle, circular checkbox, title (inline edit), Internal chip (toggle on hover), Expand, Date link (date picker), assignee avatar picker, kebab; "+ Add Step"; "+ Add Stage" (dashed)
- Expand => right side panel (URL ?actionItemId=): status dropdown (Not Started / On Track / At Risk / Delayed, colored dots), Set Milestone flag, Owner avatar, title, Start/End date, Files (Add Files & Links, empty state), Description (textarea), Subtasks (+ Add Subtask), footer tabs Comments | Activity; composer with visibility dropdown (Public comment / Internal note), Add file, @Mention, Emoji, send; hint "Tag @username to notify specific person only..."
- Opening task panel fires GET /activities/:actionItemId and PUT /itemMetadata/:id (read tracking)
- Data: tab type MAP -> section type ACTION_ITEMS with contentRef {stages[], sectionId, dealmapId, tabId}; stage {headline, isHidden, actionItems[]}; actionItem {headline, isDone, assignee, assignee2, date, startDate, status, metadataId, commentsCount, commentsNotesCount}
- GET /api/v1/tabs/:roomId/:tabUri/:tabId ; GET /api/v1/tabs/all/:roomId
## Other tabs
- Quick-start guide (uri overview-2, GENERAL): TEXT section with TinyMCE 6.3.2 (menubar File/Edit/Insert/Format/Table; toolbar undo/redo, B/I/U, block format, font family Nunito Sans, size 12pt, lists, text color, highlight, more); MULTIPLE_EMBEDDED "Getting Started" with YouTube + side list of items (▶ Step 1..); section toolbar lock icon appears on some
- Internal tab (isInternal): purple dismissible banner "This section is for the internal team only and can't be shared with clients"; CRM Management section (Opportunity Name, Stage select, Close Date datepicker, Amount, Lead Source, Next Step - disabled until Connect CRM; delete icon); Internal Tasks (ACTION_ITEMS) w/ stage status "Not Started"; Handoff Notes (TinyMCE)
- Tab strip: drag handle on hover (reorder), tab kebab: Tab icons >, Hide tab, Save tab (premium), Copy shareable link, Edit tab permissions (premium), Delete. "+" => New tab / From a template (premium)
- Room title chevron => room switcher popover: Search Room..., Duplicate this Room, Create New Room, Save Room as Template, MY ROOMS list
- Agent (AI) docked right panel, pushes content: header "Agent Beta [room chip]", Clear chat, help, close; empty state "Arron, how can I help?" w/ suggestion chips (Update next steps, Add case study from library, Restructure the room); composer with "Internal" chip, "Ask our AI...", + attach, @ mention, send; disclaimer "AI can make mistakes"
- Comments button => "Room comments" right panel listing threads per section (General comments, Stakeholders w/ last message preview, count, See comments ->); thread view (URL ?sectionId=) with chat bubbles, timestamp, composer (Public comment/Internal note, Add file, Mention, Emoji)
- Vendors: Chargebee (billing iframe), Intercom, YouTube embeds
## Preview (buyer view) (/room/:id/:tab  no view param / view=preview)
- Document title changes to "▶ Seller + Buyer". Internal tab + hidden File Sharing tab disappear; empty sections (Demo) hidden; section toolbars, edit pencils, "+" inserters, tab kebabs removed; Internal MAP steps hidden (5 of 7 shown). Comment/Share per section remain; Reply button on welcome; stakeholders email icon.
- AI Client Assist floating button: suggestion bubbles pop above ("Is there an NDA file in this room?", "Who's the room owner?"); opens chat card: header "AI Client Assist Beta", settings gear, close; banner "AI Client Assist is now visible to Clients"; date/last updated; welcome message; suggestion chips (+ "What are the next steps"); "Ask a question..." input
## Room Analytics (/seller/analytics/:roomId?analyticsTab=overview|timeline|visits|orgchart)
- Header: Analytics, room selector dropdown (right), mini stats (Room visits, Interactions, Total time); tabs Overview | Timeline | Visits | Stakeholders map
- Overview: room card + Room trend gauge (Heating); KPI strip (Time viewed, Total visits, Visitors, Interactions, Comments, AI Chats) with ? tooltips; AI Deal Insights (Beta) with 3 colored tab cards Opportunities/Flags/Risks + blurred sample insight card behind "Connect CRM to unlock deal insights" gate; Recent engagement line chart (High/Low, dates); Most visited tabs (bar list %); CRM Insights (Deal name/Stage/Close date/Amount); AI Highlights (Coming Soon, blurred sample summary); Most active visitors (See more); Latest actions table (Name, Action, Time spent); Most engaged content table (File name, total views, last viewed, downloads, engaged users) w/ empty state "No content visited yet, invite your clients in"
- Timeline: range toggles 1m/3m/9m + Today; legend filter; Filter by person; "Room Engagement" swimlane with dot markers per day; Activity Log: day group (SEP 22, 2026) list of sessions (Someone entered, location Canada, time, duration) + detail pane with events (Viewed "Overview" tab w/ thumbnail)
- Visits: companies panel (empty), engagement concentric rings (High engagement / Medium / Active) with avatars; Visitors (1\2) table: Name (+LinkedIn), Job title, Location, First entry, Last activity (sortable), Time spent, Actions, CRM enrichment; "Invited" pill; Connect CRM
- Stakeholders map: React Flow (xyflow) org chart canvas: search/add stakeholders, Filter; node cards with Select role dropdown, name + LinkedIn, title, kebab, connection handles top/bottom; bottom toolbar (fit, auto-layout, zoom in/out, fullscreen, undo/redo)
- API: /analytics/overview/:roomId -> {topVisitors[], companies[], roomAnalytics{visitors,lastViewed,firstViewed,trendDateList{date:..},totalInteractions,linksOpened,totalVisits,totalTime,totalAiAssisted,trend}, uniqueVisits, totalComments, visitedTabs[], crmInsights{isCrmAvailable}, latestActions[], mostEngagedContent[], engagementTrend{scope,dataPoints,isTruncated}, engagedVisitors[], aiDealInsights{aiDealInsights,aiDealInsightsInfo}, dealMap{_id,owner,clientCompanyName,clientLogo,analytics,tabs,closedDealRef,didCelebrateWon,closedStatus}, aiHighlightsSummary}; /stakeholderRoles/:id, /stakeholders/:id, /projects/compact, /aiDealInsights/:id/mark-viewed
- Visitor user object includes anonymousTokenId, publicId, visitedDates, sessionFirstActivity/LastActivity (anonymous visitor tracking via token)
## Templates (/seller/templates?filterByUser=aligned)
- Tabs Rooms | Tabs (premium -> "Upgrade to unlock tab templates" modal w/ illustration, 3 check bullets, Contact Us / Upgrade Now)
- Owner filter dropdown: search user, My Templates, All Templates, Aligned Templates, [team members]; Search; "Create Room Template"
- Expert Gallery hero banner (purple gradient, circular avatars w/ name + company)
- Tables: Name (kebab: Use template, Duplicate), Owner avatar, Last used (sortable), Date created, Rooms created (info tip); sections "Aligned" and "Expert Gallery"
- Clicking template => Templates modal (URL ?galleryModal=templates&modalTab=aligned&dealMapId=): left list with segmented Team templates / Aligned templates + EXPERT GALLERY group; right live preview of template room (scaled), "Skip template", "+ Use template"
## Task Manager (/seller/myTasks) — premium gated
- Filters: room dropdown (Latest Rooms / list), All users, Internal|External segmented, Search; KPI filter cards Open 7 / Overdue 0 / Completed 0 / All 7 (selected outline)
- Table: Task (checkbox circle), Room (logo+name), Section, Due date (sortable, Date link), assignee; attachment & comment icons; skeleton rows + "Upgrade plan to manage all your next steps..." gate
## Analytics (sidebar) => jumps straight to latest room's analytics (/seller/analytics/:roomId)
## Library (/seller/library?folderId=)
- Left rail: Company Library / My Library; Team Folders tree; Labels > Content Area (General), AI Labels (Business Case, Case Study, Competitive Comparison, Contract/Legal...) checkbox filters; "Manage labels" footer
- Top: global search w/ Advanced search modal (search + filters Content labels, Content type, Owner, User groups; Reset/Search)
- Main: breadcrumb "Company Library / Case Studies ▾", archive toggle icon, list/grid toggle, New (Add file or link / Create folder); in-folder filter chips (Content labels, Content type, Owner, User groups); grid view adds sort select (LAST MODIFIED)
- Table: checkbox, Name (icon/thumbnail), Owner, Modified (sort), Type (Folder/PDF/PPTX); row kebab Open, Rename, Download, Move to, Archive, Delete (permission-disabled states)
- File viewer: fullscreen overlay using @react-pdf-viewer (rpv): zoom out/in with % dropdown, Actual size, Page fit, Page width, close
## Settings (/seller/settings/members|profile|notifications|general|integrations|mcp)
- Sidebar "Integrations" and "MCP" both land in Settings tabs. Tabs: Members, My info, My notifications, My settings, My integrations, MCP
- Members: KPI filter pills All 2 / Active 2 / Pending 0 / Invites Sent 0, Invite button; table #, Name+email, CRM, Department (dropdown), Role (Admin/User), Status (Active green, sortable), Group (Choose Group dropdown)
- My info: profile image + Change image, Full name, Title (select), Email, Phone, LinkedIn, Personal email, Room action button label + link (e.g. Book a meeting -> Calendly); Save
- My notifications: account-default matrix (same events as room) "Changes to these settings will not effect existing rooms"
- My settings: AI Client assist in rooms toggle; link card "Manage in AI Center"
- My integrations: Account integrations banner (HubSpot, Salesforce, Zapier, Gong, Dynamics, etc. admin-managed); Personal: Calendly, Slack, Microsoft Teams, Google Drive (upgrade), Chili Piper, Outreach Calendar, HubSpot Calendar — each card w/ logo, description, Connect
- MCP: "MCP Server — Use your Aligned tools inside Claude, ChatGPT, and other assistants" cards Claude / ChatGPT / Glean (Beta) with Enable; upsell card
## Overview (/seller/odashboard) — Manager's Dashboard, gated: blurred sample (date range pills Last 30/60/90 days/12 months/All time, All groups, Manager's Dashboard / Content Dashboard, KPI tiles, Top 5 rooms, bar chart "time spent by top buyers", most engaged visitors table) under "Upgrade to unlock the Manager's View" modal
## Buying Hub (/seller/buying-hub) — New: buyer-side workspace landing: headline, 3 check bullets, "Invite vendor to Aligned" CTA, product illustration (vendor accordions w/ AI Vendor Brief / Notes / Private tasks, Share / Go to room)
## Global chrome
- Left sidebar 187px: logo, nav w/ icons, divider, Buying Hub w/ "New" badge; bottom: Help (popover: Help Center, Give Feedback, Support), Earn $5000 (Cello referral, red dot), user (popover: Invite team (primary), Settings, Admin Center, Upgrade, Log out)
- Top-right utility: Upgrade (orange pill), HubSpot/Salesforce connect icons, Invite Team (modal: invite link w/ Copy, invite by emails + role select User, Invite team)
- Top: dismissible CRM banner; onboarding checklist bar with strike-through completed steps, hover expands to 4-card stepper
- Toasts (react-toastify) w/ progress bar, e.g. "Only admins can access this page"
- Upsell pattern: "Upgrade plan" orange badges inline; blurred sample content + modal "Upgrade to unlock X" (illustration, 3 benefits, Contact Us / Upgrade Now)
- New Room menu: AI Deal Builder (Beta, gradient outline), Create from scratch, Create from template, Duplicate from existing room
- AI Deal Builder modal: mad-libs progressive sentence "I'm building a room for [an Enterprise deal|a Transactional deal|a QBR|Client Onboarding|Other]. The deal is in a [Discovery|Demo|POC/Pilot|Security Review|Legal/Procurement|Waiting for signature] stage" -> content dropzone (Upload Files / Add from library / Skip content) -> prompt textarea (prefilled "I'd like to map stakeholders, and set an evaluation timeline.") + Add content + Generate ->
- Section toolbar icons (inferred): lock (restrict), arrow "Move to tab" (popover listing tabs), duplicate, save to library (data-testid save-section), eye (hide from client), trash, grip "Drag to reorder"
## Design tokens
- font Nunito Sans; body bg #eff3fa; text #404965 (gray-500); primary #7f4ee7 (hover #ba99ff, disabled #d7c5ff); button radius 6px, h 40px, 16px/700
- gray: 100 #dee5fa 200 #a9b5db 300 #7c8dc1 400 #576694 500 #404965 600 #272f4a 700 #1f242f
- purple 50 #f2edfd ... 500 #7f4ee7 600 #663eb9 700 #4c2f8b; positive-green 600 #3bcab0 700 #15c691; teal-500 #06e6cc; red-500 #fe3711; orange ramp; blue-500 #3485ff; stage-1 dark #cb6506 light #ffe5cc
- Libraries: Next.js (pages), React, react-bootstrap (dropdown, offcanvas, modal), Tailwind (tw- prefix), CSS Modules, emotion, TinyMCE 6.3.2, @react-pdf-viewer, React Flow (xyflow), react-toastify, Intercom, Segment, Chargebee, Cello, Clarity, HubSpot tracking

## Screenshot index (screenshots/)
- 01-rooms-dashboard-progress-overlay.jpg
- 02-rooms-dashboard.jpg
- 03-rooms-filter-user-dropdown.jpg
- 04-rooms-sort-dropdown.jpg
- 05-rooms-status-dropdown.jpg
- 06-rooms-column-picker.jpg
- 07-rooms-row-kebab-menu.jpg
- 08-share-drawer-external.jpg
- 09-share-drawer-notifications.jpg
- 10-share-drawer-team.jpg
- 11-room-settings-notifications.jpg
- 12-room-settings-permissions.jpg
- 13-room-settings-integrations.jpg
- 14-room-editor-overview-top.jpg
- 15-room-editor-video-and-deck.jpg
- 16-room-editor-empty-media-picker.jpg
- 17-room-editor-timeline-section.jpg
- 18-room-editor-links-list-section.jpg
- 19-add-section-drawer-sections.jpg
- 20-add-section-drawer-ai.jpg
- 21-add-section-drawer-integrations.jpg
- 22-mutual-action-plan.jpg
- 23-map-step-row-hover.png
- 24-map-task-detail-panel.jpg
- 25-map-task-status-menu.png
- 26-map-comment-visibility-menu.png
- 27-quickstart-tab-rich-text.jpg
- 28-internal-tab-crm-and-tasks.jpg
- 29-add-tab-menu.jpg
- 30-tab-options-menu.jpg
- 31-room-switcher-dropdown.jpg
- 32-agent-ai-panel.jpg
- 33-room-comments-panel.jpg
- 34-comment-thread.jpg
- 35-preview-buyer-view-overview.jpg
- 36-preview-buyer-view-plan.jpg
- 37-ai-client-assist-chat.jpg
- 38-room-analytics-overview-top.jpg
- 39-room-analytics-overview-mid.jpg
- 40-room-analytics-overview-bottom.jpg
- 41-room-analytics-timeline.jpg
- 42-room-analytics-visits.jpg
- 43-room-analytics-stakeholders-map.jpg
- 44-templates-list.jpg
- 45-templates-owner-filter.jpg
- 46-templates-row-menu.jpg
- 47-template-gallery-modal.jpg
- 48-upgrade-gate-modal.jpg
- 49-task-manager.jpg
- 50-library-company.jpg
- 51-library-new-menu.jpg
- 52-library-row-menu.jpg
- 53-library-advanced-search.jpg
- 54-library-folder-list.jpg
- 55-library-folder-grid.jpg
- 56-library-pdf-viewer.jpg
- 57-settings-my-integrations.jpg
- 58-settings-members.jpg
- 59-settings-my-info.jpg
- 60-settings-my-notifications.jpg
- 61-settings-my-settings-ai.jpg
- 62-settings-mcp.jpg
- 63-overview-managers-dashboard-gate.jpg
- 64-buying-hub.jpg
- 65-help-menu.jpg
- 66-user-menu.jpg
- 67-toast-admin-only.jpg
- 68-new-room-menu.jpg
- 69-ai-deal-builder-step1.jpg
- 70-ai-deal-builder-step2.jpg
- 71-ai-deal-builder-step3-content.jpg
- 72-ai-deal-builder-step4-prompt.jpg
- 73-invite-teammates-modal.jpg
- 74-section-move-to-tab-menu.jpg
