# Lesson 35: The Dialog UI System

## Overview

Claude Code renders all interactive UI elements in the terminal using React and Ink.

## Four-Layer Architecture

1. **Launchers** (`dialogLaunchers.tsx`) - Async functions returning typed Promises
2. **Helpers** (`interactiveHelpers.tsx`) - `showDialog`/`showSetupDialog` wrappers
3. **Design System** - `Dialog`, `Pane`, `PermissionDialog` primitives
4. **Feature Components** - Tool-specific permission requests and wizard pages

## Design System Primitives

**Dialog** -- Standard chrome for confirm/cancel interactions.
**Pane** -- Borderless region for slash-command screens.
**PermissionDialog** -- Specialized frame for tool permission requests.

## Permission Requests

**BashPermissionRequest** is the most complex, featuring classifier integration, "allow prefix" rules, destructive command warnings, and sandbox fallback.

**ClassifierCheckingSubtitle** was extracted because a 20fps shimmer clock would re-render the entire 535-line dialog tree every 50ms.

## CustomSelect

Universal input widget with two variants:
- **TextOption** -- Standard labeled value option
- **InputOption** -- Embeds live text field inside selection list

## Wizard Pattern

- **WizardProvider** -- State container
- **useWizard** -- Hook for navigation
- **WizardDialogLayout** -- Per-step chrome with auto-computed title
- **WizardNavigationFooter** -- Contextual footer

## Key Takeaways

- Dialogs become async functions returning typed Promises
- `showSetupDialog` is sole location where `AppStateProvider` and `KeybindingSetup` are added
- `isCancelActive` resolves text input conflicts
- Permission requests use single `switch` routing
- Wizards manage their own exit lifecycle
