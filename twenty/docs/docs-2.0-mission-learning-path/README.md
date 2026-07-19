# Docs 2.0 - Mission Learning Path

This folder is a fresh mission-style learning path for the Twenty codebase. It is intentionally separate from any earlier documentation so it can be compared as a different prompt style.

## How To Use This Path

Read missions in order. Each mission is meant to force an active engineering behavior: trace, annotate, question, explain, debug, or review. The Socratic checkpoints are written as static pause points because this is a document set: answer the questions before reading the self-grade rubric and strong-answer notes.

## Files

- `01-tier-1-junior-missions.md` - Missions 1-8
- `02-tier-2-mid-level-missions.md` - Missions 9-17
- `03-tier-3-senior-missions.md` - Missions 18-24
- `04-mission-25-reference-artifacts.md` - Mission 25 permanent reference docs
- `JOURNAL.md` - reasoning notes from this fresh documentation pass

## Core Fresh-Look Mental Model

Twenty is a metadata-driven CRM workspace. The browser boots into a provider-heavy React shell, loads workspace/user/config/metadata state, and then renders CRM routes. The server boots into a NestJS engine that attaches auth/workspace context, exposes GraphQL and REST APIs, and routes many user actions through metadata-aware services rather than simple one-table CRUD.

Start with these anchors:

- Frontend entry: `packages/twenty-front/src/index.tsx:1-14`
- Frontend app shell: `packages/twenty-front/src/modules/app/components/App.tsx:1-46`
- Frontend router: `packages/twenty-front/src/modules/app/hooks/useCreateAppRouter.tsx:109-260`
- Frontend API transport: `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:98-365`
- Backend entry: `packages/twenty-server/src/main.ts:1-96`
- Backend app module: `packages/twenty-server/src/app.module.ts:49-146`
- Simple backend route: `packages/twenty-server/src/modules/dashboard/controllers/dashboard.controller.ts:1-25`
- Dynamic REST controller: `packages/twenty-server/src/engine/api/rest/core/controllers/rest-api-core.controller.ts:24-163`

