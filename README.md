# Mabinso CSV → OpenStreetMap Interactive App

Single-file (or small multi-file) JavaScript + HTML application that imports a semicolon-delimited CSV of vehicle events and visualizes them on OpenStreetMap (Leaflet). The implementation must follow SOLID principles applied to frontend JavaScript design.

## 1. Project overview (short)

Web browser-based app that:
- Imports semicolon (;) delimited CSV files (with quoted values and # comment lines).
- Allows defining vehicle capacity that is used for anomaly detection.  Default value when not defined is 50.
- Computes POINT_INDEX, stop LFD_NR, SALDO and ANOMALY per business rules.
- Groups rows to map points and draws journey polylines.
- Shows an interactive table and a resizable map/table layout with filtering, selection and CSV export (CSV-only).
- Is modular and designed according to SOLID principles.

## 2. Key functional requirements
### 2.1 CSV parsing

Accept only CSV files (UTF-8).
- Delimiter: ; (explicitly forced).
- Header row is present and preserved exactly for display (original header names must be used in table column titles).
- Quoted fields supported (").
- Lines starting with # are comments and must be ignored (even if the first line begins with #).
- Empty lines skipped.
- On parse error: show clear UI error message (modal or visible banner) with details.

### 2.2 Required columns (expected in header)

The CSV header will include the following columns (case and spacing preserved in header string, but processing should normalize keys internally):
FAHRZEUG_NR, DATUM, BETRIEBSTAG, UHRZEIT, GPS_LON, GPS_LAT, Vehicle type, Linie, Fahrt ID, Halt NR, Number of Doors, TUER_ID, SENSOR_STATUS, EINSTEIGER, AUSSTEIGER.

 If required columns are missing: show a validation error listing missing headers.

### 2.3 Data types / formats

- DATUM, BETRIEBSTAG: strings in YYYYMMDD (display as received, optionally format to YYYY-MM-DD for human readability).
- UHRZEIT: integer seconds since midnight. Display as HH:MM:SS.
- GPS_LON, GPS_LAT: floats (WGS84B). Rows with missing/invalid coordinates must be skipped for mapping but remain in CSV table (with indicator).
- EINSTEIGER, AUSSTEIGER: integers (treat missing as 0).

### 2.4 Business rules — LFD_NR, SALDO, POINT_INDEX

- Journey key: Bus ID + Linie + Fahrt ID. When either changes, a new journey begins.
- POINT_INDEX: calculated as ordered position of records grouped by geographical location within a journey.
   - For each journey, initialize POINT_INDEX = 1 for the event/record.  
   - If geographical position (GPS_LON + GPS_LAT) changes compared to previous row in the same journey, increment POINT_INDEX.
   - If Linie or Fahrt ID changes, reset POINT_INDEX to 1.
- LFD_NR (stop ordinal):
   - For each journey, initialize LFD_NR = 1 for the first Halt NR.
   - If Halt NR changes compared to previous row in the same journey, increment LFD_NR.
   - If Linie or Fahrt ID changes, reset LFD_NR to 1.
- SALDO (passenger balance):
  - When journey starts, if SALDO of previous stops is negative consider the saldo as 0 (before applying events).
  - For each row within the journey: SALDO_current = SALDO_previous + EINSTEIGER - AUSSTEIGER.
  - SALDO computed per row and stored for table and tooltip.
- DATA_ANOMALY (Anomaly Type)
  -   **GPS location unknown**: GPS_LON or GPS_LAT either empty or contains only `0` digit (or character `.`).
  -   **Negative Values**: `Einsteiger < 0` or `Aussteiger < 0`.
  -   **Impossible Saldo**: cumulative saldo \< 0.
  -   **Exceeding Capacity**: `Einsteiger` or `Aussteiger` \>
	vehicle capacity.
  -   **Saldo Drift**: end saldo ≠ 0 (expected empty vehicle).\


### Grouping for map

Map grouping key per point: Linie, Fahrt ID, POINT_INDEX.
All CSV rows that share that grouping represent a single map point.
Each grouped point should show the aggregated tooltip (see UI requirements) and be represented by a single marker and label (index).

### Map behavior & markers

Map provider: OpenStreetMap tiles via Leaflet.
Journey polylines: events belonging to same Linie|Fahrt ID are ordered by timestamp and connected with a polyline in the journey color.
Each journey uses a different color (consistent across markers and polyline).

Point markers must:
- Display the point index (order within journey) inside the marker.
- Marker background color is the journey color.
- Marker index is always visible (not only on hover).
- Tooltip on hover must show a compact table with: UHRZEIT, LFD_NR, Halt NR, TUER_ID, EINSTEIGER, AUSSTEIGER, SALDO.
  Tooltip can contain more than one record in a table.
- Click on a marker: select that row in the CSV table (scroll to and highlight corresponding table row), and open the tooltip/popup.
- Clicking a table row must focus + select the corresponding map marker and open its tooltip.

Journey polylines must: 
- Tooltip on hover must show a following information: LINIE, FAHRT_ID, UHRZEIT of the first and last record within a journey.

### Table UI (CSV Data)

Table visible by default, showing original header names as column titles.
Column headers are following: "FAHRZEUG_NR","DATUM","BETRIEBSTAG","UHRZEIT","GPS_LON","GPS_LAT","VEHICLE_TYPE","LINIE","FAHRT_ID","LFD_NR","HALT_NR","NUMBER_OF_DOORS","TUER_ID","SENSOR_STATUS","EINSTEIGER","AUSSTEIGER",'SALDO', "DATA_ANOMALY"
All imported data rows are shown in the table (raw CSV rows).

Special formatters:
- DATUM, BETRIEBSTAG: display (optionally human-readable).
- UHRZEIT: display as HH:MM:SS.
- SALDO: 
  - Show text in red if negative.
  - If SALDO on the last record of a journey ≠ 0, the SALDO cell background is light orange.

Row background alternation:
- Alternate background colors between journeys, not rows: e.g., first journey rows white, next journey rows #f0f0f0, and so on.
- Header color must be preserved.
- SALDO-warning (light orange) has priority override over row background for that cell.
- When table row is selected then background of table row is changed (standard table row highlight background).

Clicking a table row must focus + select the corresponding map marker and open its tooltip.
Hovering a map point shows tooltip with aggregated data; hovering a marker can also highlight row in the table (UX choice).
Table is paginated for large datasets (client-side) with configurable page size (e.g., 100 rows per page) to avoid UI freezes.

### Filters & controls

Journey filter area:
- Checkboxes for each journey (Linie|Fahrt ID) horizontally laid out and wrapping to next line when space is limited.
- Buttons: Select All and Deselect All.
- Changing a checkbox: show/hide markers and polylines for that journey.
- Selecting a journey checkbox should also focus the map on that journey’s first point and open its tooltip.

Map/Table splitter:
- Resizable horizontal splitter between map and table; supports drag to change heights.
- Button/option to hide/show the table entirely.

Export:
- Export CSV-only (CSV of current table including computed columns).
- Keep original header names in exported CSV header.
- Include LFD_NR and SALDO as last two columns.

### Error handling and user notifications

Clear validation messages for:
- Missing required headers.
- Parse errors (show row and error message).
- Non-blocking notifications (toast/banner) preferred.

UI must not crash on malformed files — handle gracefully and present meaningful errors.

# Non-functional requirements
## SOLID-oriented code architecture

- Single Responsibility (SRP): each module/class should have one reason to change (e.g., CSV Parser, Data Processor, Map Renderer, Table Renderer, UI Controller, Exporter).
- Open/Closed (OCP): modules should be open for extension and closed for modification (use strategy objects, or small adapters for alternative parsers or visualizations).
- Liskov Substitution (LSP): abstractions should allow interchangeable implementations (e.g., a MarkerRenderer interface with Leaflet implementation).
- Interface Segregation (ISP): prefer small interfaces — consumers should only depend on needed methods.
- Dependency Inversion (DIP): high-level modules should not depend on low-level modules directly — use inversion via dependency injection or module parameters.

Note: In browser JS, SOLID maps to small ES modules/classes, clear interfaces (objects with methods), and dependency injection via constructor params or factory functions.

## Performance & scalability

- Must handle moderately large CSVs (10k — 100k rows) gracefully:
- Use grouping and incremental DOM updates.
- Use client-side pagination on table.
- Consider Web Workers for heavy parsing/processing if performance issues arise (design to allow swap-in).
- Minimize reflows: only update DOM elements that changed; reuse marker layers when possible.

## Usability & Accessibility

- Responsive UI for desktop and tablet (mobile optional but preferable).
- Keyboard/navigable table rows and checkboxes (a11y).
- Tooltips accessible (on focus and hover).
- Colors should have sufficient contrast; color-blind friendly palette suggested (or allow user to choose palette).

## Security

- Only local file uploads (no network upload of user CSV). No external execution of CSV content.
- Sanitize any string inserted into the DOM to avoid XSS if CSV contains HTML (escape text in table and tooltips).

## Libraries / Tools (allowed)

- Leaflet (for map) — required.
- PapaParse (recommended) — for robust CSV parsing with comments and delimiter options.
- No heavy UI frameworks required; plain ES6 + small utility libraries allowed.
- Styling: CSS (optionally Tailwind if requested), but deliver a self-contained HTML/CSS/JS bundle as default.

## Architecture / Module design (high-level)

Top-level modules (each a separate JS class/module)

- AppController (high-level): orchestrates UI events, wiring between modules. Depends on abstractions, not concrete classes.
- CSVParser:
  - Responsibility: parse CSV text/file, validate headers, skip comments.
  - Interface: parseFile(file) -> Promise<ParseResult>, parseText(text) -> Promise<ParseResult>.
- DataProcessor:
  - Responsibility: normalize headers to internal keys, order rows by timestamp per journey, compute LFD_NR, compute SALDO, compute POINT_INDEX, build grouping keys, call AnomalyDetector to annotate rows with anomaly flags.
  - Interface: process(rows, headers) -> ProcessedData (includes allData, journeyKeys, groupedPoints, visibleJourneys).
- AnomalyDetector:
  - Responsibility: core logic (pure, testable), supports pluggable detection strategies, responsible for detecting anomalies.
  - Interface: detect(processedData) -> { processedDataWithAnomalies, report }
- MapRenderer:
  - Responsibility: render markers, labels, polylines, handle selection, coloring per journey.
  - Interface: render(processedData), focusOnPoint(pointKey), setJourneyVisibility(journeyKey, visible).
- TableRenderer:
  - Responsibility: render table header and paginated body, only render rows where Linie+FahrtID is in visibleJourneys, handle row select events, SALDO formatting, alternate backgrounds per journey.
  - Interface: render(allData, headers), selectRow(rowIndex) etc.
- FilterPanel:
  - Responsibility: render journey checkboxes, select/deselect all, emit events when changed.
- Exporter:
  - Responsibility: create CSV export from current data and trigger download.
- NotificationService:
  - Responsibility: show validation/errors to user.
  - Dependency wiring: AppController receives instances of implementations (constructor injection). This satisfies DIP.
- Data contracts
  - RowRaw: object with keys from original header (strings).
  - RowNormalized: object with normalized keys (e.g., LINIE, FAHRT_ID, GPS_LAT, GPS_LON, etc.) plus computed LFD_NR, SALDO, POINT_INDEX, DATA_ANOMALY and supporting fields
  - ProcessedData: { allData: RowNormalized[], journeyKeys: string[], groupedPoints: { [pointKey]: RowNormalized[] } }

## UI/UX design (detailed) 
- Top bar: Vehicle capacity text input, File input, Load sample, Export CSV.
- Journey filter: below controls, horizontal wrapping checkboxes, Select All / Deselect All buttons.
- Content: vertical stack — Map (top), splitter, Table (bottom).
  - Map: Leaflet with markers and polylines; clickable markers; cluster not necessary (grouping handles duplicates).
  - Table: fixed header, scrollable body, pagination controls at bottom.

Interactions:
- Enter vehicle capacity for anomaly detection. Text input if filled with default value.
- Upload CSV → parse → process → render (map + table + filters).
- Hover marker → tooltip shows aggregated rows for that point (table within tooltip).
- Click marker → select and scroll to the first corresponding table row.
- Click table row → highlight and focus map center on corresponding marker, open popup.
- Change journey checkbox → show/hide corresponding polyline, markers and apply filtering in table.
- Click "Select all" button -> mark all checkbox's as checked and show all polyline, markers and data in table 
- Click "Deselect all" button -> mark all checkbox's as unchecked and hide all polyline, markers and data in table

## Acceptance criteria / test cases
### Parsing + validation

- Upload a CSV with semicolons, quoted values, and comment lines starting # — parser ignores comment lines and reads the rest.
- Missing required header produces a clear error listing missing fields.
- Malformed rows cause parse error(s) surfaced to user.

### LFD_NR and SALDO correctness

- Given sample data per journey (ordered by UHRZEIT), LFD_NR increments only when Halt NR value changes; reset on new journey.
- SALDO computed cumulatively and reset at each journey start when previous saldo is negative. Negative SALDO values appear in red. Last SALDO non-zero shows light orange background.

### Map & markers
- Markers show index numbers in journey order and use the same color as their journey.
- Polylines connect points in chronological order.
- Clicking markers selects table row and opens tooltip.
- Selecting journey checkboxes hides/shows markers, polylines and table rows.

### Table behaviors
- Table shows original headers and computed columns (LFD_NR, SALDO).
- Alternate background per journey implemented.
- Clicking table row centers map and opens marker popup.
- Table is paginated and responsive for large datasets.
- Table shows rows where Linie+FahrtID is checked in journey filter.

### Export

Exported CSV contains original headers in the same order, with LFD_NR and SALDO appended, and values matching what's shown in the table.

## Coding standards & deliverable format

Provide a self-contained HTML file (or modular set of JS files) with clear module boundaries mapped to the design above.
Use ES6 modules/classes where possible.
Add inline documentation (JSDoc) for public methods.
Avoid global mutable state; prefer AppController and dependency injection.
CSS should be modular and avoid inline styles where possible, but small inline allowed for icon HTML.
Tests: include a minimal set of unit-testable logic (data processor) as separate JS functions with examples in comments.

### Example CSV (for dev/testing)

```
# this is a commented line and must be ignored
FAHRZEUG_NR;DATUM;BETRIEBSTAG;UHRZEIT;GPS_LON;GPS_LAT;Vehicle type;Linie;Fahrt ID;Halt NR;Number of Doors;TUER_ID;SENSOR_STATUS;EINSTEIGER;AUSSTEIGER
1001;20250911;1;0;13.405;52.52;Bus;10;100;1;2;1;OK;5;0
1001;20250911;1;300;13.406;52.521;Bus;10;100;2;2;1;OK;3;1
1002;20250911;1;0;13.407;52.522;Bus;20;200;1;2;1;OK;2;0
```

Expected: two journeys (10|100, 20|200), computed LFD_NR / SALDO values, correct grouping and markers.

# Implementation notes for SOLID in JS

- SRP: Make CSVParser only do parsing/validation; DataProcessor only compute business logic; MapRenderer only do map tasks; TableRenderer only DOM table tasks.
- OCP: Keep parser/config options injectable (delimiter, commentChar). If new export formats are needed later, implement new Exporter classes rather than editing the existing one.
- LSP: If using an abstract Renderer interface, a LeafletRenderer must adhere to that interface; a future MapboxRenderer can be substituted.
- ISP: Don’t force TableRenderer to implement map functions; provide only table-specific methods.
- DIP: AppController should receive instances like new AppController({ parser, processor, mapRenderer, tableRenderer, exporter }) rather than constructing them itself.

## Deliverables

Fully working HTML/JS/CSS application meeting the requirements above (single-file or small module set).
README with: How to run/open the file.
CSV format constraints and sample CSV.
Notes on SOLID design choices and module map.
Known limitations and further extension suggestions (e.g., WebWorkers for >100k rows).
Optional: small set of unit-test snippets for DataProcessor logic (in comments or separate file).
