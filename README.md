# new-fixed-issues-showcase

Reproductions for how Codacy attributes **New** and **Fixed** issues to a diff,
and how the same issue can be counted twice.

## Background

An issue's identity is `md5(filename + patternInternalId + lineText + sourceId)`,
with all whitespace stripped from the line text. **Line number and message are
deliberately excluded**, so moving code does not re-identify its issues.

New and Fixed are then computed per commit against the first parent:

1. **Scope to changed files** — issues in untouched files can never be reported.
2. **Raw set difference** by identity.
3. **Diff-line confirmation** — a raw new issue is New only if its start line is a
   line the diff *added*; a raw fixed issue is Fixed only if its start line is a
   line the diff *removed*.
4. **Cancellation** — if `startLine` **and** `patternId` **and** `fileDataId` all
   match across the two raw sets, both are dropped from the confirmed buckets and
   fall through to Possible.

Step 4 is the one that decides whether churn inflates the counters.

## The commits

| Commit | Change | What it shows |
|---|---|---|
| `5c939ca` | Create `test1.js` | Everything is genuinely New. |
| `8e5e1cb` | `'Hey'` → `'Heey'` | In-place edit, line 1 stays line 1. **Cancellation fires**, so nothing is confirmed and it all lands in Possible. |
| `0dd9813` | `'Heeeeey'` → `'Heeeeeey'` | Same again. Symmetric `+34 / -34` Possible with **zero** confirmed — the signature of total cancellation. |
| `5d3075e` | Edit line 1, drop a semicolon, add a blank line | Mixed: line 1 still cancels, the `"Bye"` line changes *and* shifts so it escapes — but a real issue is also introduced, so the demo is ambiguous. |
| `31e3880` | Add `inflation.js` | Baseline: five distinct one-line statements. |
| `65e9ef0` | Rename labels, prepend a comment | **The inflation case.** See below. |

## The inflation case (`65e9ef0`)

```diff
+// Renamed the log labels. Nothing was fixed, nothing was introduced.
-console.log('alpha')
+console.log('alpha1')
 ... and the same for bravo, charlie, delta, echo
```

Every line does two things at once:

- **its text changes**, so the issue gets a new identity, and
- **its number shifts by one**, so `startLine` no longer matches across the raw sets.

Cancellation therefore cannot pair them up, and both halves are confirmed:

| | Before | After |
|---|---|---|
| Rules firing per line | `no-console`, `quotes`, `semi` | identical |
| Issues in the file | N | N |
| Genuinely fixed | — | **0** |
| Genuinely introduced | — | **0** |
| Codacy reports | — | **~N New and ~N Fixed** |

Nothing was repaired and nothing was added, but the commit contributes ~N to
*both* counters. Summed over a month of commits, this is what makes a repository
holding 40 issues report thousands "fixed".

### Why the earlier commits do *not* inflate

They are in-place edits that keep the line number, so all three cancellation
keys match and the pair is suppressed. Cancellation only catches that narrow
case — churn escapes it whenever **any** of the three keys moves:

| Situation | Cancels? | Result |
|---|---|---|
| Edit a line in place, no shift | yes | nothing counted |
| Edit a line **and** shift it | no — `startLine` moved | **1 New + 1 Fixed** |
| Tool upgrade renames a rule | no — `patternId` differs | **1 New + 1 Fixed** |
| Rename or move the file | no — `fileDataId` differs | New only; Fixed stays silent |

In real repositories the second row is close to universal: almost any commit
shifts line numbers somewhere in the file.

## Checking the results

`onlyPotential` is a switch, not an "include", so confirmed and possible need
two separate calls. The public `deltaType` is only `Added`/`Fixed` and does not
say which bucket a result came from — the request does.

```bash
export CODACY_API_TOKEN=...
BASE=https://app.codacy.com/api/v3/analysis/organizations/gh/troubleshoot-codacy/repositories/new-fixed-issues-showcase

# confirmed New / Fixed
curl -s -H "api-token: $CODACY_API_TOKEN" "$BASE/commits/<sha>/deltaIssues?status=all" \
  | jq -r '.data[] | "\(.deltaType)\t\(.commitIssue.patternInfo.id)\t\(.commitIssue.lineNumber)"' \
  | sort | uniq -c

# the possible buckets
curl -s -H "api-token: $CODACY_API_TOKEN" "$BASE/commits/<sha>/deltaIssues?status=all&onlyPotential=true" \
  | jq -r '.data[] | "\(.deltaType)\t\(.commitIssue.patternInfo.id)\t\(.commitIssue.lineNumber)"' \
  | sort | uniq -c
```

## Caveats

- The comment line added in `65e9ef0` exists only to shift the line numbers. If
  it trips a rule of its own it adds a small constant to New; it does not affect
  the paired New/Fixed inflation being demonstrated.
- Identical lines collapse to one identity (same filename, pattern and line
  text), which is why `inflation.js` uses five *distinct* labels. A file of
  repeated lines would dedup and show far fewer issues than expected.
