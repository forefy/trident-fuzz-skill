---
name: trident-fuzz
description: Set up and write Trident fuzz tests for Solana/Anchor programs in 5 phases - setup, invariant mapping, test construction, validation, and result analysis. Use when the user asks to fuzz, set up fuzz tests, write property-based tests for Solana programs, or references Trident/trident. Also triggered by /trident-fuzz or /fuzz.
---

# Trident Fuzz Testing Skill

**Current API version: Trident v0.12.0** - see [references/trident-api-v0.12.md](references/trident-api-v0.12.md) for method signatures. If Trident updates and APIs change, only that file needs replacing.

## What Trident Does

Trident is a **stateful fuzzer** for Solana programs. It runs programs inside a local BanksClient (no validator needed), randomly picks transaction "flows", and checks invariants after each iteration:

```text
iteration = init() -> [random flow1, flow2, ..., flowN] -> end()
                        ^ fuzzer picks randomly each time
```

## When Fuzzing Is Worth It (and When It's Not)

**High ROI:**
- Instructions where inputs are numeric ranges (amounts, percentages)
- Cross-program shared state (two protocols touching the same account)
- Operations that should be commutative/associative (order shouldn't matter)
- Post-event operations (after liquidation, after migration)

**Low ROI / architectural blockers:**
- Instructions where each input requires a unique pre-initialized PDA (e.g., borrow amount determines tick, tick is a PDA that must exist). Random inputs need random PDAs - impractical.
- Pure admin/config instructions with no user inputs
- Programs with no state persistence between transactions

**When you hit a low-ROI path:** Don't force it. Scope the campaign to what CAN be fuzzed effectively (e.g., deposit/withdraw only), and note the gap for manual audit.

## Guidance for First-Time Users

Fuzzing is NOT "throw random inputs and see what breaks." For Solana programs, fuzzing is:

1. **Build a valid protocol state** (the hard part - Phase 1 is 60% of the work)
2. **Define what "correct" means** (invariants - Phase 2)
3. **Let the fuzzer find paths to "incorrect"** (flows - Phase 3)

**What to expect:**
- Phase 1 will take the longest - hidden prerequisites (version gates, supply checks, duration bounds) require iterative fixes.
- First campaigns should be narrow. Start with deposit/withdraw on one instruction.
- Most flows will revert (40-60% success rate is ideal).
- "No violations found" is the likely outcome for well-audited code. Value is confidence + regression safety.
- Some paths are architecturally unfuzzable (dynamic-PDA problem) - acknowledge and cover manually.

**Most common mistakes:**
1. Putting everything in `test_fuzz.rs` - use `common/` modules from the start.
2. Not asserting setup success - silent failures make all invariants trivially pass.
3. Only checking one invariant level - need both cross-protocol AND within-protocol.
4. Skipping the sensitivity test - verify invariants CAN fail.
5. Ignoring revert logs - unexpected reverts mean broken setup.

## Phases

Execute in **5 sequential phases**. Complete each phase, present output to user, **wait for approval** before the next.

| Phase | Name | Goal | Reference |
| ----- | ---- | ---- | --------- |
| 1 | Setup & Account Graph | Map dependencies, generate scaffolding, write setup modules | [references/trident-phase-1-setup.md](references/trident-phase-1-setup.md) |
| 2 | Invariant Mapping | Derive testable properties at multiple detection scopes | [references/trident-phase-2-invariants.md](references/trident-phase-2-invariants.md) |
| 3 | Test Construction | Write modular flows + invariant assertions | [references/trident-phase-3-construction.md](references/trident-phase-3-construction.md) |
| 4 | Dry Run & Validation | Compile, run short campaign, verify flows execute | [references/trident-phase-4-validation.md](references/trident-phase-4-validation.md) |
| 5 | Result Analysis & Iteration | Interpret output, assess coverage, decide next steps | [references/trident-phase-5-analysis.md](references/trident-phase-5-analysis.md) |

**When to stop or loop back:**
- Phase 1 fails to compile → Fix setup. Do not move to Phase 2.
- Phase 4 shows 0% flow success → Setup broken. Return to Phase 1.
- Phase 4 shows 100% revert on specific flows → PDA seed mismatch or missing prerequisite. Debug in Phase 4, fix in Phase 1.
- Phase 5 coverage gaps are significant → Design a new campaign (`fuzz_N/`) targeting the gap. Restart from Phase 1, reuse setup modules.

## Modular File Structure

**Do not put everything in `test_fuzz.rs`.** Split into reusable modules:

```text
trident-tests/
  Cargo.toml
  Trident.toml
  common/              <-- shared across campaigns
    mod.rs
    setup.rs           <-- reusable init functions (create_mints, init_oracle, etc.)
    instructions.rs    <-- instruction builders (do_operate, do_deposit, etc.)
    invariants.rs      <-- assertion functions (check_accounting, check_bitmap, etc.)
    constants.rs       <-- program IDs, VAULT_ID, MIN_TICK, etc.
  fuzz_0/
    test_fuzz.rs       <-- FuzzTest struct + #[flow] + #[end] only
    fuzz_accounts.rs
    types.rs           <-- auto-generated
  fuzz_1/
    test_fuzz.rs       <-- different flows, different invariants, reuses common/
    fuzz_accounts.rs
    types.rs           <-- auto-generated
```

## Progressive Invariant Fuzzing

This methodology proves that defined properties hold across all reachable states - not traditional crash fuzzing. Each campaign's blind spots inform the next.

**Refinement loop:** Campaign N passes → ask "what can this campaign NOT catch?" → identify blind spot → design Campaign N+1 → reuse `common/` modules.

### Four stages

| Stage | Name | Invariants | Catches | Misses |
| ----- | ---- | ---------- | ------- | ------ |
| 1 | **Foundation** | Within-protocol accounting | Basic state inconsistencies | Cross-program corruption, adversarial transitions |
| 2 | **Integration** | Cross-protocol conservation + within-protocol | Shared state corruption between programs | CPI-consistent bugs (both sides agree on wrong number) |
| 3 | **Scenario** | Bug-specific + general invariants at all scopes | Edge cases, regression testing | Time-dependent bugs |
| 4 | **Temporal** | Exchange price drift, interest accrual | Time-dependent accounting bugs | Dynamic-PDA paths |

**Key question after each stage:** "If there were a bug in [accounting / cross-protocol / CPI-consistent / time-dependent], would this campaign have caught it?"

## Sufficiency Checklist

Run through this after Phase 5.

**Coverage:** Every in-scope instruction has a flow · Both success and revert paths exercised (20-80% success) · Cross-protocol flows interleave · Adversarial transitions exercised

**Invariant quality:** Multiple detection scopes · Each invariant sensitivity-tested · Non-triviality guards present · No trivially-true invariants

**Iteration depth:** 1000+ iterations for Foundation, 200+ for Scenario · Multiple flows succeed per iteration

**Gap documentation:** Coverage matrix lists what was NOT tested · Each gap has risk assessment · High-risk gaps have documented alternatives

**Reproducibility:** Seed and commands documented · Setup is deterministic

**Sufficient** means you can write: *"We executed N iterations with M flows across K campaigns. Our invariants verified [properties] at [scopes]. Flow success rates: X-Y%. Not covered: [list]. Gaps addressed by: [manual audit / accepted risk]."*
