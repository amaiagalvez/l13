## Why
Changing a project's dates does not revalidate its epics, so an epic can end up outside its
project's dates. The owner decided not to block the project change (code review BUG-012,
2026-10-21) but to make those epics visible in the epic list.

## What Changes
- Saving an epic still requires its dates to fall within its project's dates.
- Editing a project's dates is allowed even if it leaves epics outside them.
- The epic list shows the dates of such an epic in red, with a text alternative.
