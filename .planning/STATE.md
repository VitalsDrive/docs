# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-05-11)

**Core value:** Prevent expensive vehicle failures — DTC codes, dead batteries, and overheating engines — before they cause breakdowns. Fleet Health, Simplified.
**Current focus:** Phase 1: Telemetry Pipeline

## Active Phase

**Phase:** 1 (Telemetry Pipeline)
**Status:** Not started
**Roadmap:** .planning/ROADMAP.md

## Sub-Repositories

This project uses 5 separate git repos:
- `docs/` — Documentation
- `packages/dashboard/` — Angular dashboard
- `packages/parser/` — TCP ingestion server
- `packages/simulator/` — Ghost fleet simulator
- `packages/supabase/` — Database schema & migrations
- `packages/auth-service/` — JWT exchange service

## Notes

- Brownfield project — core infrastructure already exists
- Config loaded from .planning/config.json
- Codebase analysis available in .planning/codebase/

---
*Initialized: 2026-05-11*
