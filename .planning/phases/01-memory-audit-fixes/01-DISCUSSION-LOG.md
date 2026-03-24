# Phase 1: Memory Audit & Fixes - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions captured in CONTEXT.md — this log preserves the discussion.

**Date:** 2026-03-24
**Phase:** 01-memory-audit-fixes
**Mode:** discuss

## Gray Areas Discussed

All four areas selected by user.

## Decisions

### Audit scope
| Question | Answer |
|----------|--------|
| What should the audit flag? | Owning raw pointers only — allocations where no Qt parent AND no smart pointer. Skip correctly-parented Qt objects. |

### Smart pointer migration
| Question | Answer |
|----------|--------|
| Smart pointers for what? | Non-Qt types only. Qt-derived classes stay on Qt parent-child. |

### Fix approach
| Question | Answer |
|----------|--------|
| Structure of work | Audit first, fix in one pass. Full catalogue of file:line issues first, then apply all fixes. |

### Risk tolerance
| Question | Answer |
|----------|--------|
| High-traffic classes (Host, TMainConsole, TLuaInterpreter) | Fix all, rely on build + tests. Fixes are mechanical — no behavioral changes. |

## No Corrections

All recommended options were selected — no deviations from defaults.
