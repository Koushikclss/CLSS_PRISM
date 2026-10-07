CLSS Analytics Suite
A metadata-driven analytics platform for SAP. Administrators design dashboards in a drag-and-drop Designer; end
users explore them with filters, drill-down, cross-filtering and export. Data comes from SAP BW (BEx queries) through
a Node.js gateway and a Python MCP bridge — the browser never talks to SAP directly.
Status: working prototype / demo-grade. See Known limitations before using it with real users or production
SAP systems.
Contents
• Features
• Architecture
• Backend at a glance
• Frontend at a glance
• Repository layout
• Prerequisites
• Getting started
• Using the platform
• API overview
• Configuration
• Troubleshooting
• Known limitations
• Roadmap
Features
For administrators (Designer)
• Modules → side-menu groups → pages, all created and reordered from the UI (nothing is hard-coded)
• Drag-and-drop canvas: add, move and resize widgets; page size and page-level settings
• 27 widget types: KPI card, gauge, stat card, metric strip, table, pivot table, bar / line / area / donut / radar / scatter
/ stacked / combo / waterfall / funnel / treemap / histogram / box plot / Pareto / map / heatmap, plus a business
health strip, a "needs attention" list, and filter widgets
• Properties panel with live preview; Top N, thresholds / conditional formatting, drill-through, per-widget
cross-filter and visualization-switching controls
• Dataset Explorer: browse registered sources, search fields, preview data, create calculated measures, override
metadata, define field- and row-level security, define cross-source relationships
• KPI Designer, draft → published lifecycle, version history with restore, refresh settings
• Every characteristic of a BEx query (Rows, Columns and Free) is available as a dimension
For users (Dashboard)
• Filter bar with one dropdown per characteristic: multi- or single-select, sort order, show ID / description / both,
and a per-filter choice between all pages and this page only
• Cross-filter highlighting between widgets, drill-down tables with sub/grand totals, drill-through
• Switch a visualization at runtime (where the admin allows it), focus mode, view underlying data
• Compact number formatting (e.g. 36.4Cr), per-widget insight sentences
• Export Preview page, then download as Excel / CSV; the whole dashboard as a one-page A4 PDF or PowerPoint
with the active filters printed in the header
Platform
• Two-stage request model: metadata first, data second
• Normalised metadata and query contract regardless of the underlying SAP source
• Server-side cache (20 min), startup pre-warm and background refresh of BW row sets
• Structured errors with correlation IDs, audit log, retry / timeout policy, versioned API (/api/v1)
Architecture
┌──────────────┐ HTTP/JSON ┌───────────────────┐ MCP (HTTP) ┌──────────────┐ RFC / BICS ┌─────────┐
│ Frontend │ ───────────► │ SAP.js gateway │ ───────────► │ mcp-bw │ ───────────► │ SAP BW │
│ SAPUI5 │ │ Node + Express │ │ Python │ │ (BEx) │
│ :8080 │ ◄─────────── │ :4000 /api/v1 │ ◄─────────── │ :8831 /mcp │ ◄─────────── │ │
└──────────────┘ └───────────────────┘ └──────────────┘ └─────────┘
 │
 ├── ABAP ADT adapter → SAP tables / CDS views (optional)
 └── Mock adapter → built-in demo sources
Component Responsibility
Frontend
(sap-fiori-frontend)
SAPUI5 app: Dashboard (user view), Designer (admin), Dataset Explorer, KPI Designer, Export
Preview. Knows only the gateway API.
Gateway (sap-js-gateway) Request validation, role checks, metadata normalisation, query engine (filter / group /
aggregate / Top N / calculated measures), KPI and export services, caching, audit,
configuration persistence.
mcp-bw
(sap-js-gateway/mcp-bw)
Python bridge to SAP BW. Opens BEx queries with BICS_PROV_* function modules over RFC,
can re-drill a query onto any characteristic, and returns labelled rows. Exposes MCP tools
(bw_execute, bw_get_characteristics, …).
Request flow
1 Metadata: the frontend asks the gateway for modules and data sources → gateway → mcp-bw → SAP returns the
query's characteristics and key figures → normalised and returned.
2 Data: a widget sends query intent (dimensions, measures, filters, sort, Top N) → the gateway fetches the needed
rows (drilled onto the characteristics the query uses) and applies filters, grouping and calculated measures →
normalised result.
Backend at a glance
The backend is two programs: the gateway (Node.js / TypeScript / Express) and mcp-bw (Python).
Gateway (sap-js-gateway/src)
Layer What it does
routes/ Versioned REST API (/api/v1); thin handlers that validate and delegate
middleware/ Role checks (requireRole), correlation IDs, structured error handler
services/queryService Query engine: validates dimensions, measures and filters against metadata, then filters,
groups, aggregates, sorts, applies Top N and pages. Calculated measures (ratio, percent,
difference, sum, variance %) are computed on the aggregated rows.
services/metadataService Normalised metadata per source, metadata overrides, field-level security, calculated measures
merged in
services/kpiService KPI value, target, variance, status and thresholds, using the same query engine
services/exportService Export preview and server-side Excel / CSV generation (50,000-row safety limit)
services/cacheService 20-minute TTL cache; query keys include the role
Layer What it does
services/configService Modules, pages, widgets, KPIs, security rules, version history — persisted to
data/config.json
services/monitoringServic
e, util/auditLog
Request timings, cache hit rate, concurrency, audit trail
adapters/bw Real BW access through mcp-bw: per-source row cache, startup pre-warm, 12-minute
background refresh, and drilling a query onto any of its characteristics (one or several at a
time)
adapters/abap Optional ADT adapter for SAP tables and CDS views
adapters/mock Built-in demo sources that follow the same contract as the real ones
mcp-bw (sap-js-gateway/mcp-bw/mcp_bw) — opens BEx queries through the BICS_PROV_* RFC function modules and
returns labelled rows. Tools include bw_list_systems, bw_list_queries, bw_get_metadata, bw_get_characteristics,
bw_execute (with row_characteristics to re-drill a query) and bw_get_variables. Several SAP systems can be
configured side by side.
Design principles: no dataset is hard-coded; the frontend sends query intent and receives one normalised response
shape; the gateway is the only entry point; every mutating route is role-checked and audited.
Frontend at a glance
The frontend is a SAPUI5 (1.120) single-page app in plain JavaScript and XML views, served by a small Express server.
Charts use Chart.js; PDF / PowerPoint / Excel generation is done in the browser.
Area What it contains
Views Login, Dashboard (user view), Designer (admin), Dataset Explorer, KPI Designer, Export Preview
Widget framework
(widgets/)
One file per widget type, created through WidgetFactory; every widget is driven by a
configuration object (source, dimensions, measures, filters, style)
Services (service/) GatewayService (all API calls, client cache, retry-without-filters), PageConfigService (pages /
widgets / publish), AuthService (session and roles)
Dashboard Loads the published page, builds the filter bar, renders widgets on a fixed-size canvas scaled to
the screen, handles cross-filter, drill-through, auto-refresh and the A4 PDF / PowerPoint export
Designer Palette, canvas with drag and resize, properties panel, filter-bar editor, versions, page and
module management, refresh and slicer settings
Filter system Filter bar and Filter widgets share one state: selection kept across pages, per-filter scope /
selection mode / sort / ID-or-description display
Utilities (util/) Compact number formatting, chart mounting and theming, canvas layout, colour palette,
threshold logic, insight sentences
Design principles: pages, menus and widgets come from configuration, not code; admins can build a new analytical
page without changing source files; designer-only controls are hidden from normal users.
Repository layout
.
├── sap-fiori-frontend/ # SAPUI5 application (static files + tiny Express server)
│ ├── server.js
│ └── webapp/
│ ├── controller/ view/ # Dashboard, Designer, Dataset Explorer, KPI Designer, Export Preview
│ ├── widgets/ # one file per widget type + WidgetFactory
│ ├── service/ # GatewayService, PageConfigService, AuthService
│ ├── util/ # formatting, chart mounting, layout, palette
│ └── css/
└── sap-js-gateway/ # backend
 ├── src/
 │ ├── routes/ # REST API (/api/v1)
 │ ├── services/ # query, metadata, KPI, export, cache, config, monitoring
 │ ├── adapters/ # bw (via mcp-bw), abap (ADT), mock
 │ └── middleware/ # role checks, error handling
 ├── data/ # persisted configuration + audit log (runtime, git-ignored)
 └── mcp-bw/ # Python SAP BW bridge (MCP server)
Prerequisites
Tool Version Used for
Node.js 18 or newer gateway and frontend server
Python 3.10 or newer mcp-bw
SAP NW RFC SDK matching your OS required by pyrfc (not on PyPI)
pyrfc 3.x RFC calls to SAP BW
Network access to SAP
BW
— RFC gateway port, e.g. 33<nn>
Internet access (for now) — the frontend loads SAPUI5, Chart.js and the PDF
/ Excel libraries from public CDNs
Getting started
Start the three components in this order: mcp-bw → gateway → frontend.
1. mcp-bw (SAP BW bridge)
cd sap-js-gateway\mcp-bw
python -m venv venv
venv\Scripts\python.exe -m pip install -r requirements.txt
:: install pyrfc against your SAP NW RFC SDK (see https://github.com/SAP/PyRFC)
venv\Scripts\python.exe -m pip install pyrfc
copy .env.example .env :: then fill in your SAP systems (see Configuration)
set SAPNWRFC_HOME=C:\nwrfcsdk
set PATH=%SAPNWRFC_HOME%\lib;%PATH%
set SERVER_MODE=mcp-http
set MCP_HTTP_HOST=0.0.0.0
set MCP_HTTP_PORT=8831
venv\Scripts\python.exe server.py
The server is ready when it listens on http://localhost:8831/mcp.
2. Gateway
cd sap-js-gateway
npm install
copy .env.example .env :: set MCP_BW_URL=http://localhost:8831/mcp
npm run dev :: development (auto-reload)
:: or: npm start
The gateway listens on http://localhost:4000 (API under /api/v1). On start it re-registers the BW sources saved in
data/config.json, retrying every 30 s until mcp-bw answers.
3. Frontend
cd sap-fiori-frontend
npm install
npm start
Open http://localhost:8080.
First run
1 Sign in (the login is a demo login — see Known limitations). Choose the role Admin to get the Designer.
2 Open Datasets → Add Data Source, choose BEx Query, enter SYSTEM:QUERY_NAME (for example
SIMQA:ZBEX_PMD_QUERY) and register it.
3 Open Admin Designer, create a module and a page, add widgets, pick the source, dimensions and measures,
then Publish.
Using the platform
Roles
Role Can
Admin everything: design, publish, security rules, audit / monitoring
Designer design and save drafts; cannot publish
Analyst / User view published pages, filter, drill, export
Filters
• Pages show a filter bar. By default each page gets up to four filters picked from its data (year, plant, region, …).
An admin can instead pin specific filters by adding Filter (Slicer) widgets in the Designer — they appear in the
bar, not on the canvas.
• Each filter has a small menu: Filter type (all pages / this page only), Display (description / ID / both), Selection
(multiple / single), Sort by, Clear selection.
• A filter set to all pages keeps its selection as you move between pages and modules.
• Only roles that can open the Designer see Add filter and Remove filter.
Export
• Export → Export Preview shows exactly what will be exported; download as Excel or CSV from there.
• Export Dashboard → PDF / PowerPoint produces a single A4 page with the page title, the filters currently
applied, and the whole dashboard.
API overview
All endpoints are under /api/v1. Mutating routes require a role (see Known limitations).
Area Endpoints
Data sources / metadata GET /datasources, GET /datasources/:id/metadata, POST /datasources, DELETE
/datasources/:id
Query POST /query — { sourceId, dimensions[], measures[], filters[], sort[], topN, page
}
Area Endpoints
Modules, pages, widgets /modules, /pages, /pages/:id/widgets/:widgetId, POST /pages/:id/publish,
/pages/:id/versions
KPIs, calculated measures /kpis, /kpi/:kpiId/compute, /calculated-measures
Governance /metadata-overrides, /field-security, /row-security, /relationships
Export POST /export/preview, POST /export/download
Admin /admin/audit-log, /admin/monitoring
BW GET /bw/systems, GET /bw/healthcheck/:system
Filter operators: EQ NE GT LT GE LE BETWEEN IN NOT_IN CONTAINS STARTS_WITH ENDS_WITH REL_MTD REL_FTM
Errors use one structured shape:
{ "error": { "errorCode": "ERR_UNKNOWN_DIMENSION", "message": "...", "category": "VALIDATION",
 "requestId": "...", "correlationId": "...", "timestamp": "...", "retryable": false } }
Configuration
Gateway (sap-js-gateway/.env)
Variable Purpose
PORT gateway port (default 4000)
MCP_BW_URL URL of mcp-bw, e.g. http://localhost:8831/mcp
SAP_URL, SAP_USER,
SAP_PASSWORD, SAP_CLIENT,
SAP_LANGUAGE
ADT connection for the optional table / CDS adapter
BW_SYSTEMS optional fallback list of BW systems
mcp-bw (sap-js-gateway/mcp-bw/.env)
Variable Purpose
SERVER_MODE mcp-http for the gateway (also mcp = stdio, http = REST)
MCP_HTTP_HOST,
MCP_HTTP_PORT
bind address and port (use 8831 for the gateway)
SAPNWRFC_HOME path to the SAP NW RFC SDK
SAP_DEFAULT_SYSTEM system used when none is given
SAP_<SYS>_HOST, _SYSNR,
_CLIENT, _USER, _PASS
one block per SAP system (IDS, DEV, CLSS, SIMQA, HANA, …)
Never commit .env files. Use the provided .env.example files as templates.
Persistence
Dashboard configuration (modules, pages, widgets, KPIs, calculated measures, security rules, registered sources,
version history) is stored in sap-js-gateway/data/config.json; the audit trail in data/audit.log. Both are runtime
files. Back up config.json — it is your dashboards.
Troubleshooting
Symptom Cause / fix
Gateway log: mcp-bw
unreachable at
http://localhost:8831/mcp
mcp-bw is not running. Start it (step 1); the gateway retries every 30 s on its own.
A data source shows only one
dimension
It registered while mcp-bw was down or busy. Restart the gateway after mcp-bw is up.
Filter dropdown stuck on
"Loading…"
The first load of a characteristic runs an SAP query (can take 10–30 s); later loads are cached.
pyrfc import / DLL error SAPNWRFC_HOME not set, or the SDK's lib folder is not on PATH.
Sign-in page appears after a
restart
The demo session lives in browser session storage; sign in again.
Known limitations
This is a prototype. Before real-world use, be aware of:
• Authentication is a demo. The login page accepts any username and password; the role is sent by the browser
as an x-user-role header and trusted by the gateway. Anyone who can reach the API can claim any role. CORS is
open.
• Row-level security fails open when a request carries no role. (Rules are supported but none are configured by
default.)
• SAP access uses one technical user per system, so all dashboard users see the same BW data; per-user SAP
authorization is not propagated.
• KPI trend / comparison is synthetic (a fixed offset), not computed from a real prior period.
• Filtering and aggregation run in the gateway, not inside SAP; the default fetch of a query's own view is capped at
200 rows.
• No OData adapter, no custom-query support, SAP hierarchies are not exposed for BW sources, and the
calculation set is limited (ratio, percent, difference, sum, variance %).
• Configuration is a single JSON file; the audit log is not rotated.
• No automated tests; the frontend loads libraries from public CDNs.
Roadmap
• Real authentication (SSO / SAP principal propagation) and server-side role resolution
• Push filters and aggregation into SAP; connection pooling in mcp-bw
• Time intelligence (YTD / QTD / MTD, prior period, year-over-year) and a formula editor
• BW hierarchies, OData V2/V4 adapter, custom queries
• More widgets (bullet, scorecard, progress, matrix, tree table) and richer property panels
• Designer tooling: copy / paste, align, group, undo; page sections; a testing stage
• Automated tests, self-hosted front-end libraries, translations (i18n)
License
Add a license of your choice (for example MIT) as LICENSE in the repository root.
