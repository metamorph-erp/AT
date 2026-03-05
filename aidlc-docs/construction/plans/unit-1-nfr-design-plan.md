# NFR Design Plan — Unit 1: Foundation & Data

## Plan Overview
Design patterns and logical components that implement the NFR requirements for ConfigManager, StateManager, DataPipeline, and Per-Module Logging.

## Execution Steps

- [x] **Step 1**: Design reliability patterns — WAL management, transaction boundaries, crash recovery, fail-fast startup
- [x] **Step 2**: Design resilience patterns — retry policies, circuit breaker for data sources, partial failure handling
- [x] **Step 3**: Design performance patterns — rate limiter, lazy loading, single-connection SQLite access, checkpoint budgeting
- [x] **Step 4**: Design security patterns — credential isolation, log scrubbing filter, input validation strategy
- [x] **Step 5**: Design observability patterns — structured logging, diagnostic framework, performance timing decorators
- [x] **Step 6**: Design testability patterns — fixture factory, mock adapters, test isolation, boundary test helpers

## Artifacts to Generate
1. `aidlc-docs/construction/unit-1-foundation/nfr-design/nfr-design-patterns.md`
2. `aidlc-docs/construction/unit-1-foundation/nfr-design/logical-components.md`

---

## Questions

**Assessment**: After reviewing NFR requirements and tech stack decisions for Unit 1, all pattern choices follow directly from prior decisions:

- **Reliability**: SQLite WAL + transactions + fail-fast — patterns are deterministic from R-001 through R-005
- **Resilience**: 3 distinct retry policies already specified (AV: 12/24/48s, Bhavcopy: 15min, Telegram: 5min) — implementation is a simple adapter-level concern
- **Performance**: Rate limiter is a timestamp tracker (tech stack decided custom), checkpoint budgeting already quantified (P-003)
- **Security**: Phase 1 plaintext secrets.yaml — pattern is trivial. Log scrubbing is a custom `logging.Filter` — no ambiguity
- **Observability**: 4 log files, stdlib logging, system `logrotate` as rotation owner — all decided
- **Testability**: pytest fixtures, in-memory SQLite, `responses` mock library — all decided

**No additional questions are needed** — designing directly from NFR requirements and tech stack decisions.
