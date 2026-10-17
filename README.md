# pcmedicalist-resilience-engineering

Circuit breakers, bulkheads, backpressure, timeouts, retries, chaos drills and error-budget discipline for failure-tolerant services.

## Standalone usage

This skill is a [Hermes](https://github.com/NousResearch/hermes-agent) `SKILL.md`.
Drop it into your agent's skill library:

```bash
# Hermes
mkdir -p ~/.hermes/skills/<category>/pcmedicalist-resilience-engineering
cp SKILL.md ~/.hermes/skills/<category>/pcmedicalist-resilience-engineering/SKILL.md
```

Or port the markdown into any other agent framework that ingests
instruction files (Claude Code `CLAUDE.md`, Cursor `.cursorrules`,
CrewAI/YAML skill packs, etc.) — the content is framework-agnostic.

## Topics covered

See `SKILL.md` for the full topic list and remediation patterns.

## Monetized access (optional)

A paid tier of this skill is available via x402 (USDC, Base mainnet):

- **STANDARD** — individual `SKILL.md` fetch billed per retrieval.
- **PRO** — all PCMedicalist platform-engineering skills as a bundle.

Discovery (free): `GET https://x402.bankr.bot/<PAYEE_WALLET>/pcmedicalist-skills-discovery`
Per-skill: `GET https://x402.bankr.bot/<PAYEE_WALLET>/pcmedicalist-resilience-engineering`

## License

CC-BY-4.0 — attribution to **PCMedicalist** required. See `LICENSE`.
