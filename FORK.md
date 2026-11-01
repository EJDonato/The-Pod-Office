# Fork notes

Soft fork of [stablyai/orca](https://github.com/stablyai/orca). Plan: [the-pod-office-docs/PRD.md](the-pod-office-docs/PRD.md).

## Seams (edits to Orca's own code)

| Seam | Change                                                                                                 | Why                                                                                                    |
| ---- | ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| S0   | Add `ORCA_USER_DATA_PATH` to `WORKER_ENV_ALLOWLIST` in `src/main/plugins/workers/plugin-worker-env.ts` | Without it, a plugin worker's `orca` CLI call reaches the default profile, not the app that spawned it |

Planned: S1, see PRD §6.

## After merging upstream

Re-run the PRD Phase 0 smoke test: load a plugin, launch an agent, confirm status events and the worker's CLI call.
