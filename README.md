# jama-velocity-reports

Reports for Jama Connect using Velocity/HTML. Each template exports data from specific Jama item types into either a Word (.doc) or Excel (.xls) file via MIME multipart HTML.

Each PROD template has a corresponding `DEV <name>.vm` counterpart configured for the DEV Jama environment, where item type IDs, field names, and pick list names differ. The mapping between environments is documented in `prod-to-dev-mapping.json`.

## Templates

### DIR Export

**Output format:** Word (.doc)

**Purpose:** Generates a Design Input Requirements report. Iterates through Folder items and renders their child requirement items in tables with ID, Requirement (name + description), and Rationale columns.

**Item types and fields:**
- **Folder** (type 32) — container; the template iterates each Folder's children
- **Text** (type 33) — rendered as headings with description
- **System Requirement** (type 149) — `rationale`
- **Subsystem Requirement** (type 150) — `rationale`
- **User Need** (type 148) — `rationale`
- **Component Requirement** (type 138) — `rationale`
- **Software Requirement Specification** (type 166) — uses `comments` field instead of `rationale`
- **Software Design Specification** (type 167) — `rationale`

Item types are filtered via `$validChildItemTypes = [149, 150, 148, 138, 166, 167]` (line ~3009). Only children matching these type IDs are gathered from each Folder.

**Parameters:** None. This template has no conditional parameters — it always renders the same columns.

**DEV template differences:**
- `$validChildItemTypes` changes to `[140, 138, 148, 150, 166, 167]` — System Requirement ID changes from 149 to 140, and the Subsystem/Component Requirement IDs (138/150) are swapped between environments.

**Adapting to your data dictionary:** To add or remove requirement types from the report, edit the `$validChildItemTypes` array. To change which field is used for the Rationale column, modify the `getValueForField` calls around line ~3040 (the `#if ($childItem.documentType.id == 166)` block controls which types use `comments` vs `rationale`).

---

### Risk Export

**Output format:** Excel (.xls)

**Purpose:** Generates a Fault/Failure Mode Effects Analysis (FMEA) risk report. For each Fault/Failure item, it traverses downstream relationships to Hazardous Situations and then to Harms, producing merged-row tables with risk scoring and color-coded risk evaluation cells.

**Item types and fields:**
- **Fault/Failure** (type 180) — `when_failure_occurs_phase_or_mode`, `occurrence_score_pre_risk_control`, `occurrence_score_post_risk_control`, `rationale` (Scoring Source), `risk_control_comments`, `hazards`, `partassembly_applicable`, `analysis_of_risk_controls__new_or_exacerbated_risk`, `legacy_id_if_applicable`
- **Hazardous Situation** (type 177) — `name`, `description` (rendered as Hazardous Situation text)
- **Harm** (type 158) — `severity_of_harm`
- **Risk Control** (type 182) — `description`, `analysis_of_risk_control`

**Parameters (Jama report parameters, set at export time):**
- `$postRisk` — when truthy, adds post-risk-control columns (Post Occurrence Score, Final Risk Evaluation). The template contains two full table sections: one for `$postRisk` mode and one without.
- `$riskCondition` — when truthy, adds columns for Risk Control Description, Risk Control(s), and Analysis of Risk Controls (new/exacerbated risk). Controls whether risk mitigation details appear.
- `$riskControlInterface` — when truthy, adds columns showing linked Risk Control items (ID + description) and their Analysis of Risk Control text. Traverses downstream relationships from Fault/Failure to find type-182 items.
- `$legacyId` — when truthy, adds a Legacy ID column.
- `$reportBaseline` / `$reportTestRuns` — standard baseline and test run mode toggles.

**Risk matrix:** A hardcoded `$riskMatrix` map (line ~938) maps Occurrence x Severity combinations to LOW/MOD/INT risk levels with color coding (green/yellow/red).

**DEV template differences:**
- `partassembly_applicable` field changes to `partcomponent_applicable`
- `risk_control_comments` field is removed (does not exist in DEV)
- `analysis_of_risk_controls__new_or_exacerbated_risk` field is removed
- `analysis_of_risk_control` (Risk Control type 182) changes to `riskbenefit_analysis`
- Pick list names gain `SPR Risk Management -` prefixes

**Adapting to your data dictionary:** To change which fields appear in the report, modify the `getValueForField` calls around lines ~1129-1138. To adjust the risk matrix scoring, edit the `$riskMatrix` hash map. To change the color thresholds, modify the `$riskEvalStyle` conditionals that check for `LOW`, `MOD`, and `INT`.

---

### STRIDE Export

**Output format:** Excel (.xls)

**Purpose:** Generates a STRIDE (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege) cybersecurity threat/vulnerability analysis report. Each STRIDE item is rendered as a single flat row — no merged-row logic.

**Item types and fields:**
- **STRIDE Threat/Vulnerability** (type 183) — `stride_category`, `data_flow`, `potential_safety_impact`, `access_vector`, `access_complexity`, `authentication`, `confidentiality_impact`, `integrity_impact`, `availability_impact`, `collateral_damage_potential`, `premitigated_cvss_score`, `notescomments`
- **Risk Control** (type 182) — `description`, `analysis_of_risk_control` (accessed via downstream relationships)
- **Fault/Failure** (type 180) — document key only (listed in the Safety Related column)

**Parameters (Jama report parameters, set at export time):**
- `$postRisk` — when truthy, adds 8 post-mitigation columns: `post_access_vector`, `post_access_complexity`, `post_authentication`, `post_confidentiality_impact`, `post_integrity_impact`, `post_availability_impact`, `post_collateral_damage_potential`, `postmitigated_cvss_score`.
- `$riskControlInterface` — when truthy, adds two columns showing linked Risk Control items and their analysis text.
- `$legacyId` — when truthy, adds a Legacy ID column (`legacy_id`).
- `$safetyRelated` — when truthy, adds a Fault/Failure ID(s) column showing linked type-180 items.

**DEV template differences:**
- `potential_safety_impact` field changes to `potential_safety__efficacy_impact`
- `analysis_of_risk_control` (Risk Control type 182) changes to `riskbenefit_analysis`
- `notescomments` and `legacy_id` fields do not exist in DEV — the `$legacyId` parameter should not be used

**Adapting to your data dictionary:** To change which CVSS-related fields appear, edit the `getValueForField` calls around lines ~1077-1088. To add or remove conditional column groups, modify the `#if ($postRisk)`, `#if ($riskControlInterface)`, `#if ($legacyId)`, and `#if ($safetyRelated)` blocks.

---

### URRA Export

**Output format:** Excel (.xls)

**Purpose:** Generates a Use-Related Risk Analysis report. The hierarchy is Use Task > Use Error > Hazardous Situation > Harm, with merged rows spanning the full chain. Includes risk scoring with color-coded evaluation.

**Item types and fields:**
- **Use Task** (type 176) — `description`, `critical_taskQ`
- **Use Error** (type 154) — `occurrence_score_pre_risk_control`, `occurrence_score_post_risk_control`, `scoring_sourcecomments`, `comments`, `risk_control_description`, `analysis_of_risk_controls__new_or_exacerbated_risk`, `legacy_id`
- **Hazardous Situation** (type 177) — `name`
- **Harm** (type 158) — `severity_of_harm`
- **Risk Control** (type 182) — `description`, `analysis_of_risk_control`

The template traverses: Use Task (176) → Use Error (154) → Hazardous Situation (177) → Harm (158), all via downstream relationships.

**Parameters (Jama report parameters, set at export time):**
- `$postRisk` — when truthy, adds Post Occurrence Score and Final Risk Evaluation columns.
- `$riskCondition` — when truthy, adds Risk Control Description, Risk Control(s), and Analysis of Risk Controls columns.
- `$riskControlInterface` — when truthy, adds Risk Control interface columns showing linked type-182 items and their analysis.
- `$legacyId` — when truthy, adds a Legacy ID column.

**Risk matrix:** Same hardcoded `$riskMatrix` as Risk Export, mapping Occurrence x Severity to LOW/MOD/INT with color coding.

**DEV template differences:**
- `critical_taskQ` (Use Task) changes to `critical_task`
- `comments`, `risk_control_description`, and `analysis_of_risk_controls__new_or_exacerbated_risk` fields are removed from Use Error (do not exist in DEV)
- `analysis_of_risk_control` (Risk Control type 182) changes to `riskbenefit_analysis`
- Pick list names gain `SPR Risk Management -` prefixes

**Adapting to your data dictionary:** To change which Use Error fields appear, modify the `getValueForField` calls around lines ~1107-1115. The relationship traversal logic (Use Task → Use Error → Hazardous Situation → Harm) is built around the `documentType.id` checks — if your item type IDs differ, update the `== 176`, `== 154`, `== 177`, and `== 158` comparisons.

---

### Use Task Export

**Output format:** Excel (.xls)

**Purpose:** Generates a use task/scenario report. Renders Use Task items in a structured table showing step number, subtask ID, subtask description, user group, use environment, and parts/assemblies. Supports two hierarchy modes.

**Item types and fields:**
- **Folder** (type 32) — container; used as Use Scenario grouping
- **Text** (type 33) — rendered as heading rows above the table
- **Use Task** (type 176) — `user_groups`, `use_environments`, `partsassemblies`; also uses `name` (step name), `documentKey` (subtask ID), and `description` (subtask text)

**Parameters (Jama report parameters, set at export time):**
- `$report2Hierarchy` — when truthy, uses a 3-level hierarchy: Folder → Folder → Use Task (the outer Folder is the "Use Scenario", inner Folders are "Tasks", and Use Tasks are "Subtasks/Steps"). When falsy, uses a 2-level hierarchy: Folder → Use Task (Folder is "Use Scenario", Use Tasks are direct children).

**DEV template differences:**
- `user_groups` changes to `user_group` (Multi Select → Text Box, pick list removed)
- `use_environments` changes to `use_environment` (Multi Select → Text Box, pick list removed)
- `partsassemblies` changes to `associated_component` (Multi Select → Text Box, pick list removed)

**Adapting to your data dictionary:** The field names for user, environment, and component data are in the `getValueForField` calls around lines ~5393-5395 (3-level mode) and ~5480-5482 (2-level mode). If your Use Task item type uses different field names, update these strings. The `$validChildItemTypesL1` and `$validChildItemTypesL2` arrays control which item types are gathered at each level — adjust if your hierarchy uses different types.

---

### User Needs Export

**Output format:** Word (.doc)

**Purpose:** Generates a User Needs traceability report. Similar structure to DIR Export — iterates Folders and renders child items in tables — but with different columns: ID, Name, User Need (description), Rationale, and User.

**Item types and fields:**
- **Folder** (type 32) — container; the template iterates each Folder's children
- **Text** (type 33) — rendered as headings with description
- **System Requirement** (type 149/140) — `rationale`, `user`
- **Subsystem Requirement** (type 150) — `rationale`, `user`
- **User Need** (type 148) — `rationale`, `user`
- **Component Requirement** (type 138) — `rationale`, `user`

Item types are filtered via `$validChildItemTypes = [140, 149, 150, 148, 138]` (line ~3009).

**Parameters:** None. This template has no conditional parameters.

**DEV template differences:**
- `$validChildItemTypes` changes to `[140, 150, 148, 138]` — System Requirement ID 149 is removed (DEV uses 140), and the array is adjusted for the 138/150 ID swap.

**Adapting to your data dictionary:** To add or remove requirement types, edit the `$validChildItemTypes` array. To change which fields appear in the table columns, modify the `getValueForField` calls around lines ~3041-3042 (for `rationale` and `user`). The table header (lines ~3031-3037) should be updated to match any column changes.

## PROD vs DEV Environment

Each template has a `DEV` version (e.g., `DEV Risk Export.vm`). The differences are primarily:

1. **Item type ID swaps** — System Requirement (PROD 149 → DEV 140), Component Requirement and Subsystem Requirement (IDs 138/150 are swapped)
2. **Field name changes** — e.g., `partassembly_applicable` → `partcomponent_applicable`, `critical_taskQ` → `critical_task`, `analysis_of_risk_control` → `riskbenefit_analysis`
3. **Removed fields** — some PROD fields (e.g., `risk_control_comments`, `comments` on Use Error, `benefitrisk_analysis`) do not exist in DEV
4. **Pick list name changes** — many pick lists gain `SPR Risk Management -` or `Medical Device Framework -` prefixes in DEV

See `prod-to-dev-mapping.json` for the complete field-by-field mapping.

## Reference

- [Jama Velocity Documentation](https://velocity.jamasoftware.com/latest/ProxiedVelocity-9-5/)
- `Data Dictionary.xlsx` — field and item type reference for the PROD Jama environment
