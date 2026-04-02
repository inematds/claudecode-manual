# mdENG — Lesson 01 — Claude Code Boot Sequence

## Overview

The lesson explores the multi-phase startup pipeline that executes when a user types `claude` in their terminal, covering everything from initial keystroke to the first interactive REPL prompt.

## Three-Layer Boot Architecture

**Layer 1: CLI Entrypoint** (`cli.tsx`)
Zero-cost fast paths, environment preparation, and argument dispatch

**Layer 2: Main Function** (`main.tsx`)
Commander parsing, initialization, migrations, and permission validation

**Layer 3: Setup + REPL** (`setup.ts` + `replLauncher.tsx`)
Session wiring and React-based terminal UI rendering

## Key Startup Optimizations

The instruction notes that "MDM policy reads (via `plutil`) and keychain reads take 20–65ms," and these launch during module evaluation to run concurrently with the import chain. This parallelization prevents sequential reads from adding latency to the critical startup path.

## Setup Phase Sequence

Critical ordering enforced throughout includes `setCwd()` executing before `captureHooksConfigSnapshot()` to ensure correct project hooks are loaded. The `tengu_started` analytics beacon fires immediately after the analytics sink attaches, creating an early "process started" signal for health monitoring.

## Bare Mode Optimization

The `--bare` flag strips UDS messaging, teammate snapshots, session memory initialization, plugin hooks, and all deferred prefetches for scripted/SDK scenarios where latency sensitivity is highest.

## Global State Management

`bootstrap/state.ts` maintains session-scoped state including identity, usage tracking, mode flags, and telemetry. "Latch fields" like `afkModeHeaderLatched` keep API headers stable to prevent prompt cache busts during mid-session setting changes.
