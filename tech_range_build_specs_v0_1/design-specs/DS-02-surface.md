# DS-02 — Surface Tab

**Status**: Draft  
**Last updated**: 2026-07-07  
**Related specs**: DS-03 (Classification Panel), DS-09 (Signal Archaeology), TS-03 (Ingestion), TS-08 (Roles), ADR-003 (Object Relationships), ADR-009 (AI Content Tiers)

---

## 1. Purpose

Surface is where signals enter the analyst's view and where macro-pattern detection happens. It is the primary discovery and exploration tab — the place where incoming intelligence becomes visible and where early pattern recognition occurs before deeper investigation begins in Research or Synthesis.

Surface combines four tools in one workspace:

- A live node graph showing cluster topology and signal relationships (ReGraph)
- A temporal timeline linked to the graph (KronoGraph)
- A scrollable signal microblog feed
- TL/DR generation and archive tools

---

## 2. Tab layout

Three zones:

| Zone | Width | Height | Default state |
|---|---|---|---|
| Main area | 70% | Full height | ReGraph (upper ~65%) + KronoGraph timebar (lower ~35%) |
| Right panel | 30% | Full height | Signal microblog list |
| Bottom drawer | 100% | Collapsed | TL/DR tools (archive + on-demand generation) |

The bottom drawer overlays the bottom of both the main area and right panel when expanded. It does not displace the layout; it slides up over it.

---

## 3. ReGraph node graph

Implementation reference: Cambridge Intelligence ReGraph (React SDK). See Decision section for library confirmation.

### 3.1 Node types and visual encoding

| Node type | Shape | Size | Color |
|---|---|---|---|
| Neighborhood | Large hexagon (background region) | Fixed | Neighborhood color (semi-transparent fill) |
| Cluster (macro) | Large filled circle | Proportional to signal count | Neighborhood color, darker shade |
| Cluster (bursting) | Large filled circle + pulsing ring | Same as macro | Amber pulsing ring; node color shifts to amber tint |
| Signal | Small dot | Fixed small | Signal type color |
| Technology Profile | Diamond | Medium | Neighborhood color, bright/saturated |

Burst indicator behavior: nodes and clusters where `is_bursting = true` display a pulsing concentric ring animation. The pulse period is ~2 seconds. Color is amber (#F59E0B or equivalent design token) regardless of Neighborhood color to ensure it reads as an alert state across all color schemes.

### 3.2 Semantic zoom levels

**Level 1 — zoomed out**
- Neighborhoods render as labeled background regions (hexagonal, color-filled)
- Macro-clusters visible as large nodes
- Individual signal nodes not rendered
- Cluster labels visible if space permits; otherwise shown on hover

**Level 2 — mid-zoom**
- Clusters resolve into constituent signal nodes (small dots)
- Edges between signals become visible
- Cluster boundary remains as a light halo grouping its signals
- Cluster label persists at the group centroid

**Level 3 — zoomed in**
- Individual signal detail nodes rendered
- Hover reveals full signal summary tooltip
- Edge labels appear (relationship type)
- Technology Profile nodes resolve to show linked signal count

Transitions between zoom levels are animated (crossfade, ~200ms).

### 3.3 Edges

Edges represent canonical relationship types from `object_relationships` (ADR-003):

| Relationship type | Visual style |
|---|---|
| Corroborates | Solid line, neutral color |
| Contradicts | Dashed red line |
| Co-clusters | Light gray line, low opacity |
| Precedes / Follows | Directional arrow, neutral |
| Supersedes | Directional arrow, muted blue |

Edge weight and thickness are proportional to `relationship_strength`. Very low-strength edges (below a display threshold TBD) are hidden by default and surfaced via the "show all edges" toggle.

### 3.4 Hover and click interaction

**Hover on any node**
Tooltip shows: entity name, type, Neighborhood, signal count (clusters) or summary snippet (signals), burst status if `is_bursting = true`.

**Click on cluster node**
- Expands the cluster in the graph (transitions to zoom level 2, centering on the cluster)
- Highlights linked signals in the microblog list (right panel filters to that cluster; see §5.3)

**Click on signal node**
- Slides in a signal detail panel from the right, replacing the microblog list panel
- Panel shows full signal detail with triage actions (see §5.4)
- A "← Back to feed" affordance at the top of the panel returns to the microblog list

**Click on Technology Profile node**
- Navigates to the profile detail view in the Research tab
- Opens in the same tab (not a new window) unless the user holds ⌘/Ctrl (opens in new tab)

**Right-click / long-press on any node**
- Context menu: Add observation, Link to cluster, Copy node ID, Open in new view

### 3.5 Graph controls

A toolbar sits above the ReGraph canvas. Controls from left to right:

**Layout**
- Toggle: force-directed / hierarchical / radial
- Layout applies immediately with an animated transition

**Filter**
- Filter by Neighborhood (multi-select dropdown, color-coded chips)
- Filter by signal_type (multi-select dropdown)

**Toggles**
- Show Technology Profiles on graph (off by default at zoom level 1; auto-on at zoom level 3)
- Show burst indicators (on by default)
- Show contradiction edges (off by default; surfaces dashed red edges when enabled)

**Zoom controls**
- Fit all (resets zoom to show the full graph)
- Zoom in (+)
- Zoom out (−)
- Current zoom level indicator (read-only)

**Signal archaeology**
- "Signal archaeology" toggle button: opens the DS-09 archaeology panel as an overlay on the graph canvas. The graph remains interactive beneath the overlay.

---

## 4. KronoGraph timebar

Implementation reference: Cambridge Intelligence KronoGraph (React SDK). The timebar is linked to the ReGraph instance above it; the two SDKs share a data layer.

### 4.1 What it shows

A horizontal timeline displaying signal `event_time` distribution as a bar/density chart:

- X-axis: time (range matches current selection)
- Y-axis: signal count per time bucket (bucket size adjusts automatically to the selected range)
- Bars color-coded by Neighborhood (stacked bars per bucket)
- Burst events indicated by a small amber marker above the bar for that time bucket

Default view on tab load: last 30 days.

### 4.2 Linked filtering

The timebar and graph are bidirectionally linked:

- Selecting a time range on the timebar filters the ReGraph to show only signals/clusters with `event_time` within that range
- The graph animates to show/hide nodes as the selection changes (nodes fade out rather than snap out)
- When a cluster is clicked in the graph, the timebar highlights the time range that cluster's signals span

**Playback mode**: a play button animates the graph from the start of the selected range to the end, stepping through time buckets at ~1 second per bucket. Useful for seeing how cluster topology evolved over a time window. Pause/stop controls visible during playback.

### 4.3 Scrub controls

- Click-drag on the timebar to select a custom time range
- Pinch/scroll (or scroll wheel on desktop) to zoom the time axis
- Preset range buttons: **7d / 30d / 90d / Custom**
- **Reset** button: returns to default 30-day view, clears any custom selection
- Selected range is shown as a readable label above the timebar (e.g., "Jun 7 – Jul 7, 2026")

### 4.4 Temporal annotations

Senior analyst and above can add temporal annotations: vertical marker lines on the timebar with a short text label (e.g., "ADR published", "Prototype launched", "Competitor announcement").

- Annotations appear as thin vertical lines with a small label flag at the top
- Hover on an annotation shows the full label text and the user who created it
- Annotations are stored as records and visible to all roles (read-only for viewer/analyst)
- Annotations provide context for signal archaeology (DS-09)

---

## 5. Signal microblog list (right panel)

### 5.1 Feed format

Each entry in the feed:

| Element | Detail |
|---|---|
| Source name | Linked to source detail record |
| Source credibility indicator | Colored dot: green = high (≥0.70), amber = medium (0.40–0.69), red = low (<0.40) |
| signal_type badge | Pill badge, color-coded: paper / model_release / product_launch / regulatory_filing / etc. |
| Summary text | 2–3 lines, truncated; "Read more" expands to full signal detail panel |
| captured_at timestamp | Relative format: "2 hours ago"; hover shows absolute timestamp |
| Neighborhood tag | Color-coded tag matching graph Neighborhood color |
| Domain Area tag | Secondary tag |
| Triage action row | Analyst+: inline buttons — **Triage / Group / Archive / Reject** |

Viewer role: triage action row is hidden. Signal entries are read-only.

### 5.2 List controls

Controls appear at the top of the right panel, above the feed:

- **Sort**: newest first (default) / oldest first / by source credibility
- **Filter**: by signal_type, by Neighborhood, by triage state (`new` / `triaged` / `grouped` / `archived` / `rejected`)
- **Search**: full-text search within the currently visible list (searches summary and source name)

### 5.3 Linked selection

When a cluster is clicked in the graph:

- The microblog list automatically filters to show only signals belonging to that cluster
- A banner appears at the top of the list: **"Showing signals in [Cluster Name]"** with a **"Clear filter"** button
- The transition is animated (list re-renders with a brief fade)
- Clearing the filter returns to the full feed in its previous sort/filter state

### 5.4 Signal detail panel

Clicking "Read more" or anywhere on a signal row opens the full signal detail panel. This panel replaces the microblog list in the right panel slot (or can optionally open as a wider overlay — layout decision TBD).

Panel contents:

| Section | Roles |
|---|---|
| Full summary text | All |
| Source name + URL | All |
| Source credibility score breakdown | Analyst+ |
| DS-03 classification panel (Neighborhood axis) | All (analyst+ can edit) |
| Evidence graph links: corroborating and contradicting signals (linked list) | All |
| Triage actions: Triage, Group, Add observation, Link to cluster, Archive, Reject | Analyst+ |
| Temporal info: event_time, captured_at | All |
| Temporal info: ingested_at, ingestion mode | Analyst+ |

A **"← Back to feed"** affordance at the top of the panel returns to the microblog list. The back navigation preserves the list's scroll position and filter state.

---

## 6. TL/DR bottom drawer

Collapsed by default. A **"TL/DR"** button in the tab toolbar expands the drawer. The drawer slides up from the bottom of the Surface tab, overlaying the lower portion of the main area and right panel. Height when expanded: ~40% of the viewport.

### 6.1 Two modes (tabs within the drawer)

---

#### Mode 1 — Archive

A filterable, sortable list of all TL/DR briefs ever generated or ingested.

**Columns**:
- Title / source
- Type: on-demand / configured / ingested link
- Date generated
- Topics / Neighborhoods
- Link to full Advisory (if one has been created from this brief)

**Controls**:
- Filter by: topic, Neighborhood, Domain Area, date range
- Sort by: date (newest first default), topic

---

#### Mode 2 — Generate

Two sub-options presented as a segmented control: **Paste a link** / **Configure and generate**.

---

**Paste a link**

Input: text field labeled "Paste a URL"

On submit, the system checks the signals/sources table for a matching URL:

- **Found**: displays the existing signal record — title, current classification, evidence links, TL/DR if already generated. A note indicates when the signal was originally ingested.
- **Not found**: ingests as a new signal via `intentional_injection` mode (TS-03), runs enrichment and TL/DR generation pipeline, displays the result when complete. If the URL matches an active subscription configuration, the signal is queued for the next email digest. Analyst sees a notification: "New signal added to queue for review."

Processing states:
1. "Checking database..." (spinner)
2. "Generating TL/DR..." (spinner, shown only if ingestion required)
3. Result display

---

**Configure and generate**

A form for generating a structured digest across a defined scope:

| Field | Detail |
|---|---|
| Date range | Date pickers; maximum window of 10 days |
| Neighborhoods | Multi-select; at least one required |
| Domain Areas | Multi-select; optional |
| Signal types | Multi-select; optional |

**"Generate TL/DR"** button submits the form.

Output — a structured digest containing:
- Headline count: e.g., "47 signals across 3 Neighborhoods"
- Key themes: AI-generated summary of themes (Tier 2 per ADR-009), displayed as a short bulleted list
- Notable signals: top 5 signals with title, source, and in-app link
- Summary paragraph: AI-generated narrative overview (Tier 2 per ADR-009)

Output actions:
- **Save as TL/DR record**: stores the digest in the Archive (Mode 1)
- **Promote to Advisory draft**: creates a new draft Advisory in the Synthesis tab pre-populated with this digest (Advisory creation restricted to senior_analyst+ per TS-06 §4.8)

---

## 7. Role permissions summary

| Action | viewer | analyst | senior_analyst | architect / admin |
|---|---|---|---|---|
| View graph | read-only | full exploration | full exploration | full exploration |
| View microblog list | published/approved only | all signals | all signals | all signals |
| Triage actions (Triage, Group, Archive, Reject) | — | yes | yes | yes |
| Signal detail: credibility breakdown | — | yes | yes | yes |
| Edit DS-03 classification from Surface | — | yes | yes | yes |
| Approve classifications from Surface | — | — | yes | yes |
| Add temporal annotations to timebar | — | — | yes | yes |
| TL/DR generation | — | yes | yes | yes |
| Promote TL/DR to Advisory draft | — | — | yes | yes |

---

## 8. Decisions needed

> **DECISION NEEDED —** Should the ReGraph graph load all signals for the selected time window on Surface, or only signals above a credibility threshold? *Recommendation: default to signals with `source credibility_score ≥ 0.40`; lower-credibility signals available via an "Include low-credibility signals" toggle. Prevents graph clutter on first load.*

> **DECISION NEEDED —** Implementation library: should Surface use Cambridge Intelligence's ReGraph + KronoGraph SDKs as the visualization layer? *Recommendation: yes — these are production-proven SDKs for exactly this use case (network investigation + timeline), with a React-native API, semantic zoom support, and an existing Figma design kit. Reduces visualization build time significantly vs. a custom D3 implementation.*

> **DECISION NEEDED —** Should the signal microblog list auto-refresh in real time (WebSocket) or require manual refresh? *Recommendation: auto-refresh for new signals arriving (green "N new signals" banner at top of list), with the list itself not jumping. User clicks the banner to load new items — prevents disorienting list reordering during active triage work.*

> **DECISION NEEDED —** Should the signal detail panel replace the microblog list in the right panel slot, or open as a wider overlay (e.g., 50% width) over the graph? *Recommendation: replace the right panel by default (keeps the graph fully visible for context); offer an "expand" control in the detail panel for analysts who want more reading width.*
