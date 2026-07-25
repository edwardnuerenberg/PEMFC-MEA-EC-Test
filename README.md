# PEMFC-MEA-EC-Test-Sequence-Builer

A single self-contained HTML dasboard for planning PEMFC (Proton Exchange Membrane Fuel Cell) MEA (Membrane Electrode Assembly) test campaigns — break-in, electrochemical characterization, and accelerated stress testing — and exporting the finished procedure as an eLabFTW-compatible metadata JSON.

No build step, no server, no dependencies to install. Open [`PEMFC MEA TestingV2.html`](PEMFC%20MEA%20TestingV2.html) directly in a browser.

## What it's for

You use it to:
1. **Compose a test procedure** in the Procedure Builder — pick break-in protocols, add POL+EC (polarization + electrochemical) blocks before and after a stress test, pick which stress-test protocols to run.
2. **Fine-tune the exact parameters** of each measurement (temperatures, scan rates, voltage windows, frequency ranges, ...) in the dedicated section for that measurement type.
3. **Review the assembled campaign** in the Overview page, either as a static example template or as the actual procedure you've built.
4. **Export** the finished procedure as a machine-readable JSON ready to paste into eLabFTW.

## Sections

- **Overview / Pipeline** — visual campaign timeline. Toggle between a static **Template** (illustrative example, always the same) and **Current Procedure** (built live from whatever you've actually configured). Click any block to jump to its section.
- **Break-in / BOL** — the 5 available break-in protocols (Galvanostatic Ramp, Alternating Voltage Cycling, OCV Sweep, Stoichiometric Sweep, Convergence Loop) with their reference parameters and a schematic timeline.
- **Electrochemical Tests** — one tab per POL+EC block. Each tab holds:
  - the block's nominal gas/RH/T/P **Conditions**
  - **Polarization Curve** parameters (own λ/RH/T/P overrides, OCV pre-hold, hold time, averaging, current steps)
  - **Cyclic Voltammetry** — its own independent temperature (separate from Pol Curve/EIS), and any number of CV measurements (add/remove down to zero to skip the test entirely)
  - **EIS** — any number of measurements, each with DC current, frequency max/min (Hz), AC amplitude, points/decade, integration cycles
  - **H₂ Crossover** — enable/disable toggle plus its own parameters
- **Stress Tests (AST)** — Catalyst (Pt dissolution), Support (carbon corrosion), and Membrane (OCV hold) degradation protocols with reference parameters.
- **Procedure Builder** — the structural composer:
  - select which BOL protocols run, and which AST protocols run (or none — AST can be skipped entirely)
  - add/remove/reorder POL+EC blocks, split into **Pre-AST** and **Post-AST** groups
  - **Clone to Post-AST**: copy a pre-AST block's exact conditions into a new post-AST block, for stability/aging comparisons (e.g. re-running POL1's conditions as POL4 after the stress test)
  - the **eLabFTW Export Metadata** fields (Project, Owner, Experimenter, Sample Name, Teststand Nr., Results storage path, Procedure shortname)
  - a live preview of the assembled campaign
  - the **Export to eLabFTW** button

## Architecture note

The Procedure Builder and the Electrochemical Tests section read and write the *same* underlying data (`builderBlocks` in the script) — there's no separate copy to keep in sync. The Builder only controls structure (how many blocks, how many CV/EIS measurements, which protocols are selected); the EC section is where you edit the actual values. The Break-in/BOL page (`bolProtocols[i].params`) and the Stress Tests/AST page (`astProtocolDefaults[i]`) work the same way — editing their parameter tables persists directly into those arrays, which the eLabFTW export reads from. This is deliberate throughout: each parameter lives in exactly one place, so no view can silently drift out of sync with what gets exported.

## eLabFTW export

"Export to eLabFTW" downloads a JSON matching eLabFTW's `extra_fields` metadata format (paste it into an experiment's Metadata JSON editor in eLabFTW):

```json
{
  "extra_fields": {
    "Field Name": { "type": "number", "value": "80", "unit": "°C", "units": ["°C"], "group_id": 3, "position": 5 }
  },
  "elabftw": {
    "display_main_text": false,
    "extra_fields_groups": [ { "id": 1, "name": "Metadata" }, ... ]
  }
}
```

Key points:
- **Every parameter gets its own field** (not squashed into a descriptive string) — numeric parameters are `"type": "number"` with a `unit`/`units` pair; everything else is `"type": "text"`. This makes the export queryable/filterable per field in eLabFTW, not just human-readable.
- **One group per logical section**: Metadata, one group per selected BOL protocol, one group per POL+EC block (pre-AST, then AST, then post-AST), one group per selected AST protocol.
- **Field names are globally unique** — since `extra_fields` is one flat object shared across every group, every field is prefixed with its block/protocol identifier (e.g. `POL1 Conditions: λ`, `BOL: Galvanostatic Ramp (GSTEP): Temperature`). Skipped tests (0 CVs, 0 EIS points, H₂X disabled, no AST selected) still get one field explicitly saying so, rather than disappearing silently.
- The 7 **Metadata** fields are always present and unaffected by the procedure; everything else is generated live from whatever you've built.

## Known limitations

- The AST/BOL/EC schematic plots are illustrative, not computed from your actual entered values.

## Browser requirements

Any modern browser (tested against headless Chromium/Edge during development). Uses Google Fonts and Font Awesome via CDN for styling/icons — an internet connection is needed for those to load, though the app itself works fine without it (icons/fonts just won't render).


## Proposed future work

- Allow export, safe and import procedures to and from local database, or directly next point
- Incorporate API to eLabFTW to oversee existing procedures, import and export directly from and to eLabFTW instead of local database. Adjust naming and count to ZBT system: 41234_ENG_ECM_123_456(_nickname). Allow connection to MEA-sample and the teststation.
- Simplify eLabFTW entry by separating main descriptor parameters from technical descriptors
- Adjust parameter naming along official ZBT consensus
- Allow higher flexibility in the EC procedure, like e.g. temperatures variation between CVs in one measurement, changing of order of the hitherto fixed POL1, CV1,2,x, EIS1,2,y, H2P. To e.g. POL1, CV1, EIS1, CV2, EIS2, ...
- Expand the script to electrolyzer testing 
- Add another sublayer hidden from the user, which is covering the technical parameters of the Lifestation script, like e.g. heating, cooling, breakin before EIS, etc. by parametrization of Lifestation scripts. Allow generation and export of those scripts.
- Assign duration to the respective procedures or blocks to approximate measurement duration
- Design optimization: couple hitherto static graphical representation to real values (not that easy)
