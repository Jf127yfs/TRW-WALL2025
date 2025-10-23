# App Code Description — Panopticon + Compatibility Stack

CheckInInterface.html and Checkincode should not be messed with as is working fine. 
Scope of analytics: Only run on guests with Checked-In = "Y" in Form Responses (Clean) column AB.
Function changes: add new behavior via new functions (e.g., foo_v2()), don’t silently refactor. Comment why.

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

🧠 Project README — The Panopticon / Compatibility Analytics System
Overview

This project is an AI-structured analytics environment built around a live guest dataset collected through Google Forms. It transforms raw event responses into interactive visualizations — including the Panopticon Wall and Compatibility Matrix — while keeping the original check-in workflow completely intact.

All analytics run on read-only copies of cleaned data and automatically document themselves for traceability, reversibility, and machine understanding.

Core Philosophy

This system is designed to:

Observe without interference — analytics never alter the check-in interface or raw data.

Track data evolution — every transformation from one sheet to another is logged in a Data Dictionary.

Allow modular expansion — new functions can bloom (expand) or collapse (contract) without breaking lineage.

Stay explainable to AI models — every function is documented, reversible, and discoverable.

Protected Sheets
Sheet	Description	Editable By
Form Responses 1	Raw Google Form feed.	System (check-in only)
Form Responses (Clean) (FRC)	Normalized guest data. Used for all analytics.	checkInGuest, updateGuestScreenName, uploadGuestPhoto only

⚠️ No analytics function may write to these sheets.
All reads must filter to guests where Checked-In = "Y" (Column AB).

Analytics Layers
1. Pan_Master

Encodes validated guest data numerically for statistical analysis.

Input: Form Responses (Clean)

Output: Pan_Master

Purpose: prepare data for association strength calculations (Cramér’s V).

2. Pan_Dict

Validation and translation reference for categorical mappings.

Defines all valid options and numeric codes.

Used by all encoding and similarity functions.

3. V_Cramer_Cat

Computes Cramér’s V matrix between categorical variables.

Purpose: measure association strength between demographics, traits, and interests.

4. Guest_Similarity

Generates pairwise guest similarity scores (for compatibility visuals).

Uses: Interests, Music Preference, “At Worst,” and Recent Purchase.

Excludes demographics for diversity.

Output: ranked similarity matrix for one-to-one match mapping.

5. Panopticon Wall Export

Transforms analytic sheets into real-time JSON or HTML assets for front-end visualization.

Displays dynamic guest clustering, crowd polarity, and collective metrics.

Documentation & Registry Sheets
Sheet	Purpose
Master_Desc	Auto-generated overview of all sheets, columns, data types, and samples.
Tool_Registry	Logs every function run (with version, inputs, and outputs).
Data_Dictionary	Documents from → to mappings at column-level granularity with notes on transformations.
Function Development Rules

Every new Apps Script or AI-generated function must:

Never modify Form Responses 1 or FRC.

Document itself in:

Tool_Registry: execution, input/output lineage.

Data_Dictionary: field-by-field mapping with notes.

Call Master_Desc after creation to refresh documentation.

Include inline descriptive comments explaining its intent and data flow.

Support parameters: {expand: true|false, contract: true|false} for adaptive detail.

Use archive/rollback utilities for safe reversion (archiveSheet_(), revertLast_()).

AI Integration Rules

For LLMs (e.g., Claude, GPT, Gemini) working on this repository:

Context Access

Models may read from all generated sheets except Form Responses 1 and Form Responses (Clean).

Must reference the latest Master_Desc or Data_Dictionary for structural awareness.

Code Modification Policy

AI assistants may only make function-by-function updates (no full rewrites).

Must preserve naming conventions, comments, and aesthetic layout.

Each new function must automatically append its metadata to Data_Dictionary.

Purpose of AI Collaboration

Automate analytics expansion (new similarity metrics, visual mappings).

Maintain human-readable documentation and system coherence.

Allow iterative “expansion/contraction” of analytics without losing traceability.

Example Data Dictionary Entry
Timestamp	Function	Source Sheet	Source Field	Target Sheet	Target Field	Transform	Notes
2025-10-22	buildPanMaster	Form Responses (Clean)	“Zodiac Sign”	Pan_Master	“code_zodiac”	code_map:zodiac	Converts to numeric code using Pan_Dict
