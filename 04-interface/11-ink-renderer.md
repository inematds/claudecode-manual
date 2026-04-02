# Lesson 11: The Ink Renderer

## Overview

Claude Code's Ink rendering engine converts React component trees into terminal output through seven sequential stages, achieving microsecond performance with zero garbage collection pressure.

## The Seven-Stage Rendering Pipeline

### Stage 1 -- React Reconciler (reconciler.ts)
A custom `react-reconciler` host connects to Ink's own DOM instead of the browser. When `<Text>` nests inside another `<Text>`, the inner node becomes `ink-virtual-text` -- a ghost node without Yoga backing.

Event handlers stored separately in `node._eventHandlers`, preventing unnecessary repaints when parent components re-render with new callback references.

### Stage 2 -- Virtual DOM (dom.ts)
Plain TypeScript object tree with `DOMElement` nodes. The dirty flag signals whether re-rendering is needed -- clean unchanged nodes are blitted from the previous frame, delivering O(changed cells) performance.

**Node Types:** ink-root, ink-box, ink-text, ink-virtual-text, ink-raw-ansi, ink-link, ink-progress

### Stage 3 -- Yoga Layout (layout/)
Yoga is a C++ Flexbox engine compiled to WebAssembly. The `LayoutNode` interface decouples the codebase from Yoga's WebAssembly bindings.

**Critical Detail:** Since `ink-virtual-text`, `ink-link`, and `ink-progress` lack Yoga nodes, DOM index != yoga index. Manual counting of preceding siblings with Yoga nodes is necessary.

### Stage 4 -- renderNodeToOutput (render-node-to-output.ts)
Recursive tree walk converting laid-out nodes into Output operations. **Blit Fast Path:** if node is clean, position unchanged, and prevScreen available, emit a single `output.blit()` call.

**ScrollBox Hardware Scroll Optimization:** Emits DECSTBM + SU/SD sequence for O(1) scroll frames.

### Stage 5 -- Output Operations Queue (output.ts)
Command-buffer abstraction collecting operations: write, blit, clear, clip/unclip, shift, noSelect. Two-pass replay: Pass 1 expands damage bounding box, Pass 2 executes operations.

### Stage 6 -- Screen Buffer (screen.ts)
**Packed Int32Array**: 2 words per cell (8 bytes total).
- word0: charId (32 bits)
- word1: styleId [31:17] + hyperlinkId [16:2] + width [1:0]

Eliminates per-cell object allocation. A 200x120 terminal would otherwise allocate 24,000 objects per frame.

**StylePool** intern, transition, and cache system. `transition(fromId, toId)` pre-computes ANSI escape strings.

**Wide Character Handling:** CellWidth enum: Narrow, Wide, SpacerTail, SpacerHead.

### Stage 7 -- Cell Diff -> Terminal
Diff walks the **damage bounding box** only. For each changed cell, emits minimal ANSI sequence.

## Key Takeaways

1. **The dirty flag is the core optimization.** Steady-state frames touch O(changed cells).
2. **Yoga is a WebAssembly Flexbox engine.** Same layout algorithm as React Native.
3. **The screen buffer is a typed array, not objects.** 2 Int32s per cell eliminates GC pressure.
4. **The Output class is a command buffer.** Two-pass design for correct clip/damage interaction.
5. **Alt-screen mode has three interlocking invariants.** Height clamped, viewport faked, cursor clamped.
6. **The reconciler avoids marking dirty for event handlers.**
7. **The charCache in Output is the tokenizer hot path accelerator.** Most lines don't change between frames.
