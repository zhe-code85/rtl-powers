# Implementation Trace Template

Use this template for the lightweight code-paired implementation record. Keep it short and implementation-focused. Do not restate the full `rtl-design` content.

## 0. Metadata

### 0.1 Date
<!-- YYYY-MM-DD -->

### 0.2 Revision
<!-- v1, revA, or project-specific change identifier -->

### 0.3 Related Change
<!-- Optional commit, review, ticket, or branch reference -->

## 1. Implementation Target

### 1.1 Module Or Block
<!-- Name of the implemented object. -->

### 1.2 RTL Output Files
<!-- Table: File | Type (top/wrapper/helper) | Notes -->

## 2. Input Sources

### 2.1 Design Inputs Used
<!-- Table: Source | Type (rtl-design/user/project-rule/catalog) | Path or Reference | Notes -->

### 2.2 Confirmed Assumptions
<!-- Table: Topic | Confirmed Value | Source | Impact On RTL -->

### 2.3 Comment Language
<!-- Record the user-confirmed language for RTL comments. -->

## 3. Reuse Decisions

### 3.1 CBB/IP Preference
<!-- Required / preferred / not required -->

### 3.2 Reuse Summary
<!-- Table: Candidate | Decision (direct reuse / wrapper / custom) | Why | Key Parameters Or Wrapper Scope -->

## 4. Key Implementation Choices

### 4.1 Structural Choices
<!-- FSM split, datapath/storage ownership, wrapper boundaries, pipeline cuts, arithmetic strategy, reset scope, CDC structure. -->

### 4.2 Design-Intent Refinements
<!-- Only record implementation-level refinements or clarifications. Do not duplicate rtl-design sections. -->

## 5. Verification And Integration Follow-Up

### 5.1 Verification Focus
<!-- Table: Area | Why It Matters | Suggested Check -->

### 5.2 Integration Or Synthesis Notes
<!-- Critical paths, inference-sensitive logic, CDC/reset assumptions, constraint-sensitive boundaries. -->

## 6. Open Items

<!-- Unresolved risks, waivers, deferred cleanup, or assumptions that still need confirmation. -->
