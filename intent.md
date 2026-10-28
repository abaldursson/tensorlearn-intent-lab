# Intent

## Goal
A support coordinator needs to preview tickets from a CSV file before a later import is
approved. Add `previewTicketCsv(csvText)`, a pure function that returns either all tickets
or a single error code. It is all-or-nothing and changes no data.

## Scope
Create only `src/import-tickets.mjs` and `tests/import-tickets.test.mjs`. Do not edit
`src/tickets.mjs`, `.github/`, `CLAUDE.md`, `package.json` or this document. No UI,
database, network, file I/O, dependencies, or the import step itself. If anything seems to
need another file or an extra feature, stop and ask.

## Constraints
Return a plain object, never throw, and have no side effects. Use Node built-ins only and
parse CSV by hand, including quoted commas and doubled quotes. `ticket()` from
`src/tickets.mjs` may be reused, but it throws and does not default a blank priority.
Rules, in order:
1. Header must be `title` or `title,priority` (either order, trimmed, case-sensitive).
   Other, missing or duplicate names, empty input, a wrong field count or an unterminated
   quote give `unknown-column`.
2. More than 100 data rows gives `row-limit`, checked before any value is validated.
3. Trim values. An empty title gives `empty-title`; blank or missing priority becomes
   `normal`; anything but `low`, `normal`, `urgent` gives `invalid-priority`. If a row has
   both errors, report `empty-title`.
4. Accept CRLF and one trailing newline. A header-only CSV gives `{"tickets":[]}`. Ask the
   user about any other edge case.

## Plan
1. Restate the scope and rules and ask the user to approve before writing any code.
2. Read `src/tickets.mjs` and `tests/tickets.test.mjs` to match their style.
3. Write `tests/import-tickets.test.mjs` first, one test per acceptance case plus CRLF,
   header-only and error-precedence cases.
4. Implement `previewTicketCsv`: parse, check header, check row limit, then build tickets
   in input order.
5. Run the verification commands and report what changed.

## Acceptance
Success is `{"tickets":[{"title":"...","priority":"normal"}]}` in input order. Failure is
`{"error":"error-code"}` with no partial tickets. The contract below covers explicit and
defaulted priority, a quoted comma, empty title, unknown column, invalid priority and 101
rows (`repeat` means 101 copies of the one data row).

```json
{
  "schema": 1,
  "change": "ticket-csv-preview",
  "allowed_changes": ["src/import-tickets.mjs", "tests/import-tickets.test.mjs"],
  "protected_paths": ["src/tickets.mjs", ".github/", "CLAUDE.md"],
  "non_goals": ["ui", "database", "network"],
  "requirements": {
    "columns": ["title", "priority"],
    "default_priority": "normal",
    "valid_priorities": ["low", "normal", "urgent"],
    "reject_unknown_columns": true,
    "reject_empty_title": true,
    "max_rows": 100,
    "side_effects": "preview-only"
  },
  "verification": ["npm test", "git diff --check"],
  "approval": "ask-before-implementation",
  "acceptance_cases": [
    {
      "csv": "title,priority\nPrinter issue,urgent\n",
      "expected": {"tickets":[{"title":"Printer issue","priority":"urgent"}]}
    },
    {
      "csv": "title\nPassword reset\n",
      "expected": {"tickets":[{"title":"Password reset","priority":"normal"}]}
    },
    {
      "csv": "title,priority\nVPN down,\n",
      "expected": {"tickets":[{"title":"VPN down","priority":"normal"}]}
    },
    {
      "csv": "title,priority\n\"Printer, floor 2\",low\n",
      "expected": {"tickets":[{"title":"Printer, floor 2","priority":"low"}]}
    },
    {
      "csv": "title,priority\n   ,low\n",
      "expected": {"error":"empty-title"}
    },
    {
      "csv": "title,owner\nPrinter issue,sam\n",
      "expected": {"error":"unknown-column"}
    },
    {
      "csv": "title,priority\nPrinter issue,high\n",
      "expected": {"error":"invalid-priority"}
    },
    {
      "csv": "title\nPrinter issue\n",
      "repeat": 101,
      "expected": {"error":"row-limit"}
    }
  ]
}
```

## Verification
Run `npm test` (all tests must pass) and `git diff --check` (no output). Confirm with
`git status` that only the two allowed files changed. Ask the user before implementing,
before expanding scope, and before any commit or push. Report failing or skipped checks
honestly.
