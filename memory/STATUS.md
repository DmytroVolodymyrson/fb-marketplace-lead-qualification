# STATUS.md — Project State

## Current State
- **Phase:** Phase 2 planning / project source-of-truth setup
- **Last updated:** 2026-06-25 by Cedric auto git sweep
- **Blocked by:** Nothing

## Done
- [x] Captured active n8n AI lead qualification workflow notes
- [x] Added Phase 2 seller profile scraping plan
- [x] Added repo sync scaffolding and private artifact ignore rules

## In Progress
- [ ] Verify Phase 1 workflow in production before enabling seller profile scraping

## Next Up
- [ ] Install/test n8n Puppeteer community node if Phase 2 is approved
- [ ] Export fresh Facebook cookies into private n8n credential storage
- [ ] Add seller-profile scraping toggle to the active workflow

## Key Context
- Facebook anonymous scraping does not expose reliable seller identity data; dealer/private classification currently relies on text and image signals.
- Cookie files and workflow backups can contain live secrets and must stay ignored/local.
