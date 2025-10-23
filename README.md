# App Code Description — Panopticon + Compatibility Stack
0) Golden Rules (apply everywhere)

Scope of analytics: Only run on guests with Checked-In = "Y" in Form Responses (Clean) column AB.

Function changes: add new behavior via new functions (e.g., foo_v2()), don’t silently refactor. Comment why.

Aesthetic: minimal UI, clear toasts, 2-space indent, <100-char lines, consistent constants.

Safety: no external calls; batch read/write; < 6-min runtime; catch & log warnings, proceed best-effort.

1) Sheets & Constants (single source of truth)
// Sheet ids
const SRC_SHEET = 'Form Responses (Clean)'; // AB = "Y" is the gate
const MASTER_DESC_SHEET = 'Master_Desc';
const PAN_LOG_SHEET = 'Pan_Log';

const OUT = {
  MASTER: 'Pan_Master',           // numeric encodings for Cramér’s V
  DICT: 'Pan_Dict',               // validation dictionary (codes → labels)
  VCAT: 'V_Cramer_Cat',           // Cramér’s V matrix (categoricals)
  SIM: 'Guest_Similarity',        // pairwise similarity matrix or top-N
  EDGES: 'Edges_Top_Sim'          // sparse edges (top matches for front-end)
};

// Required headers in SRC_SHEET (simplified; keep in sync with Master_Desc)
const H = {
  TS: 'Timestamp',
  BDAY: 'Birthday (MM/DD)',
  ZODIAC: 'Zodiac Sign',
  AGE: 'Age Range',
  EDU: 'Education Level',
  ZIP: 'Current 5 Digit Zip Code',
  ETH: 'Self Identified Ethnicity',
  GENDER: 'Self-Identified Gender',
  ORIENT: 'Self-Identified Sexual Orientation',
  IND: 'Employment Information (Industry)',
  ROLE: 'Employment Information (Role)',
  KNOW_HOSTS: 'Do you know the Host(s)?',
  KNOWN_LONGEST: 'Which host have you known the longest?',
  KNOW_SCORE: 'If yes, how well do you know them?',
  INT1: 'Interest_1', INT2: 'Interest_2', INT3: 'Interest_3',
  MUSIC: 'Music Preference',
  RECENT: "Recent purchase you're most happy about",
  WORST: 'At your worst you are...',
  STANCE: 'Which best describes your general social stance?',
  SCREEN: 'Screen Name',
  UID: 'UID',
  CHECKED: 'Checked-In',
  CHECKED_TIME: 'Check-in Time'
  Photo_URL: 'PHOTO_URL_COL'
};

2) Entry Points (UI + Orchestration)
onOpen()

Purpose: add a menu for ops.

Output: UI menu: 📋 Documentation → Generate Master_Desc, Refresh All Analytics.


3) Logging & Utilities
logToPan(message, ctx = {})

Purpose: append a timestamped row to Pan_Log.

Inputs: message string; optional context object (serialized).

Output: row: [ISO time, message, JSON(ctx)].

Why: provenance; every analytic step is traceable.

getHeaderMap_(sheet)

Purpose: map header text → column index (1-based).

Output: { header: colIndex }.

readFilteredRows_()

Purpose: batch-read only checked-in rows from SRC_SHEET.

Filter: H.CHECKED === "Y".

Output: { rows, headerMap }.

ensureSheet_(name, rows, cols)

Purpose: idempotent output creation; clears & sizes, but doesn’t delete.

4) Validation Dictionary (Pan_Dict)
buildPanDict()

Purpose: produce clean, ordered codebooks (labels → numeric codes) for all categorical fields used by analytics.

Sources: distinct values from checked-in rows.

Normalize: trim, case-safe, collapse variants like (N/A) into explicit "N/A".

Outputs: OUT.DICT with columns:

key (e.g., zodiac, age_range, education, ethnicity, gender, orientation, industry, role, know_hosts, known_longest, recent_purchase, at_worst, social_stance, music_pref, interest)

label

code (1..N per key)

valid (TRUE/FALSE) — only labels that pass formatting rules

Bloom: codes are first born here (the seed for Pan_Master encodings).

Tracked: Pan_Log entry with counts per key.

Tip: keep interests unified under a single interest key; you’ll encode Int1/Int2/Int3 using the same dictionary.

5) Numeric Encodings (Pan_Master)
buildPanMaster()

Purpose: convert all categorical responses to numeric codes using Pan_Dict and pass through numeric/date fields. Invalid/missing → N/A.

Inputs: checked-in rows, Pan_Dict maps.

Columns (example):

UID | Screen Name | timestamp | zodiac | age_range | education | zip | ethnicity |
gender | orientation | industry | role | know_hosts | known_longest | know_score |
interest_1 | interest_2 | interest_3 | music_pref | recent_purchase | at_worst |
social_stance | checked_in_time


Encoding:

categorical → dictionary codes

numeric pass-through: zip, know_score, social_stance

dates standardized as ISO strings or serials

Bloom: analysis-ready table (every var is numeric or normalized).

Stacked: sheet Pan_Master, versioned by rebuild; note counts of valid vs N/A.

6) Association Strength (V_Cramer_Cat)
buildVCramers()

Purpose: compute Cramér’s V between all selected categorical columns from Pan_Master.

Scope:
Demographics: zodiac, age_range, education, ethnicity, gender, orientation
Employment: industry, role
Social: know_hosts, known_longest, recent_purchase, at_worst
Interests: music_pref, interest_1, interest_2, interest_3

Computation (per pair X,Y):

Build contingency table on non-N/A.

χ² test statistic.

phi2 = chi2 / n.

V = sqrt( phi2 / min(k-1, r-1) ), clip [0,1].

Output: OUT.VCAT matrix; row/col headers are keys; cells numeric (0..1).

Bloom: association map (the field’s magnetic field).

Tracked: write matrix + diagonals = 1; log skipped pairs with low sample size.

7) Pairwise Compatibility (Guest_Similarity + Edges)
buildGuestSimilarity()

Purpose: compute pairwise similarity among checked-in guests, excluding demographics to promote diversity.

Included factors (default):

Interests: interest_1..3 (same dictionary)

music_pref, recent_purchase, at_worst

Metric (default hybrid):

Jaccard on {interest_1, interest_2, interest_3} sets

Exact-match bonus: +0.2 for music_pref

Soft bonus: +0.1 if recent_purchase matches

Soft bonus: +0.1 if at_worst matches

Cap at 1.0

Outputs:

OUT.SIM (optional full matrix) or

OUT.EDGES sparse table: uid_a | uid_b | sim | reasons

reasons: compact explainer (e.g., int:2,music,aworst)

Bloom: edges for front-end “compatibility verse.”

Stacked: rank top-N per UID (e.g., N=5) for fast UI.

Swap metric easily by adding scoreGuestPair_v2_(a, b, weights).

8) Documentation (Master_Desc)
generateMasterDesc()

Purpose: self-document every sheet: name, columns, inferred types, samples, row counts, and lineage notes.

Inferences: date vs number vs text vs code; formula presence.

Special tags by sheet name:

Pan_Master → “Numeric encodings for Cramér’s V”

Pan_Dict → “Validation dictionary”

V_Cramer_Cat → “Association matrix”

Guest_Similarity / Edges_Top_Sim → “Pairwise compatibility outputs”

Extras:

If source has Checked-In, compute coverage summary (count + %).

Bloom: meta-knowledge for devs & Claude.

Stacked: re-run after every build; frozen header, alternating stripes.

9) Build Orchestration (Pan Sheets)
buildPanSheets()

Purpose: deterministic pipeline: Dict → Master → V → Sim → Docs.

Order matters (stack):

buildPanDict()

buildPanMaster()

buildVCramers()

buildGuestSimilarity()

generateMasterDesc()

Tracked: single log line with durations per phase.

10) Error Handling & Data Hygiene
guardHeaders_(headerMap, requiredList)

Warn on missing; continue with available subset.

normalizeLabel_(s)

Trim, unify whitespace, map common (N/A) variants, lowercase for dictionary keys while preserving display case.

safeCramers_(x, y)

Returns {v, n, k, r, warn}; if n<minN set v = N/A, log.

11) Performance Tactics

One getDataRange().getValues() per sheet; no per-row I/O.

Build maps in memory; write in one setValues() per output sheet.

Use sparse edges instead of full NxN when N grows.

12) Front-End Read Model (for visuals)

Panopticon (system layer):

From V_Cramer_Cat: heatmap cells as brightness.

From Pan_Master: density of codes per category for “old-school terminal” bars.

From Pan_Log: recent events ticker (“Photo uploaded… Similarity spikes…”).

Compatibility (human layer):

From Edges_Top_Sim: draw 1-to-1 lines; pulse weight by sim.

No individual PII beyond Screen Name + UID you already collect; avoid demographics.

13) Testing Hooks

test_readFilteredRows_() – asserts only AB="Y" rows returned.

test_dictRoundTrip_() – encode→decode label equality.

test_cramersSmoke_() – small synthetic contingency with known V.

test_similarityEdgeCases_() – duplicates, missing interests, ties.

14) Deployment & Ops

Keep all builders in Pan_Builder.gs.

Keep docs in Master_Desc.gs.

Keep UI in Menu_UI.gs.

Add a time-driven trigger (optional) to run refreshAllAnalytics() hourly during the party window.

15) Comment Style (example)

/**

PURPOSE: Build numeric feature table for association & similarity.

INPUT: Form Responses (Clean) — checked-in only (AB="Y"), Pan_Dict.

OUTPUT: Pan_Master sheet with codes + cleaned numerics/dates.

DEPENDS ON: buildPanDict, readFilteredRows_, getHeaderMap_.

NOTES: Invalid labels → "N/A" code; dates normalized to ISO.
*/

TL;DR Data “blooms, tracks, stacks”

Blooms in this order: Pan_Dict (codes) → Pan_Master (encodings) → V_Cramer_Cat (associations) → Edges_Top_Sim (matches).

Tracked in Pan_Log for every orchestration step & warning.

Stacked as layered sheets, rebuilt deterministically by buildPanSheets() and documented by generateMasterDesc().
