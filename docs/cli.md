# snp CLI reference

## snp version

```text
Usage:
  snp version

Description:
  Print the version of this snp on one line. It never calls snpd, and snp
  --version prints the same.
  snp sends its version with every request, and snpd refuses one older than its
  floor with CLIENT_TOO_OLD (exit 12). snp root shows that floor as
  min_client_version.

Output:
  The version on stdout: the tag, such as v0.1.0, on a release build;
  <contract>+dev.<short commit>, such as v0.1.0+dev.7781cfe, on any other build
  made with scripts/build-snp.sh; and dev on a plain go build, which snpd
  refuses.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value, the wrong number of arguments. snp sent
      nothing; stderr says which, so fix the command and run it again.

Examples:
  snp version
  snp --version
```

## snp mem get

```text
Usage:
  snp mem get <id> [--rev-only] [--json]

Description:
  Read one document as it is at snpd's latest commit, with its rev. The rev is
  the document's git blob id, 40 hex characters. Pass it to --expect-rev on snp
  mem put or snp mem set, so that the write fails with CONFLICT instead of
  overwriting a change you have not read.
  To find ids, use snp mem list or snp mem search.

Arguments:
  <id>: the document's path, such as projects/demo/notes.md.
  snp checks every id before it sends anything, and exits 5 (INVALID) unless:
  - it is not empty
  - it is at most 512 bytes
  - it is UTF-8
  - it does not start with /
  - it does not contain ..
  - it has no control character
  - it has no empty or . segment (no // and no /./)
  - it has no .git segment, in any case (.GIT too)
  - it is not _search or _commit (a nested x/_search is fine)
  Those are the only rules a read applies. The rule for writes is stricter, and
  mem get does not apply it: agents write only projects/<slug>/…/<name>.md or
  .yaml, with a lowercase slug and letters, digits, spaces, dots, dashes or
  underscores in each name.

Flags:
  --json
      print {content, rev}
  --rev-only
      print only the rev

Output:
  - Default: the content, byte for byte, on stdout, and the rev on stderr as one
    line.
  - --json: {"content":"…","rev":"…"} on one line on stdout.
  - --rev-only: only the rev on stdout. It wins over --json.
  On an error, stderr gets one line, snp: <CODE>: <message>, and stdout gets
  snpd's error JSON when snpd sent one.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value, the wrong number of arguments. snp sent
      nothing; stderr says which, so fix the command and run it again.
  NOT_FOUND (exit 4)
      There is no document at that id. Check the id with snp mem list, or create
      the document with snp mem put and --expect-rev <id>=new.
  INVALID (exit 5)
      The id breaks a rule above, or SNP_ORG is missing, or SNP_URL is not an
      http or https URL with a host. stderr says which; fix it and run it again.
  UNAUTHENTICATED (exit 6)
      SNP_TOKEN is malformed, unknown, revoked or expired, so snpd refused it.
      Ask your admin for a new token. When SNP_TOKEN is not set, snp sent none:
      set it, or add the snpd network secret to the cloud environment.
  FORBIDDEN (exit 7)
      SNP_TOKEN was issued for another org than SNP_ORG. Set SNP_ORG to the
      token's org, or use a token issued for this one.
  INSUFFICIENT_SCOPE (exit 7)
      The token lacks the mem scope. Ask your admin for a token that has it.
  TRANSPORT (exit 10)
      No answer from snpd, or an answer from something else, such as a proxy,
      after 2 retries. Check the network and that SNP_URL points at snpd, then
      run it again.
  CLIENT_TOO_OLD (exit 12)
      This snp is too old: snpd accepts only a newer one. Install the version
      the message names, then run it again.
  LIMIT (exit 14)
      The token went over its request rate. Wait the seconds the message names,
      then run it again.
  INTERNAL (exit 1)
      The request failed inside snpd in a way it does not expect, after 2
      retries. Run it again; if it fails again, send the message to your admin.

Environment:
  SNP_URL: snpd's base URL, such as https://host:8484. A url= line in
  ~/.snp/config works too, and SNP_URL wins over it. With neither, snp uses
  https://snp.tesselabs.com.
  SNP_ORG: your org. Required; an org= line in ~/.snp/config works too, and
  SNP_ORG wins over it.
  SNP_TOKEN: your token, read only from the environment, never from argv or a
  file. Leave it unset in a cloud session whose network secret adds it; snp then
  sends no token of its own.
  SNP_SESSION, or else CLAUDE_CODE_REMOTE_SESSION_ID, or else
  CLAUDE_CODE_SESSION_ID: optional. A claude.ai cloud session sets the second,
  which stays the same across /clear and worker restarts. snp sends the first
  one set with every request, and a write records it on the commit as the
  Snp-Session trailer; a write without a valid one is recorded as undeclared.
  When SNP_ORG is missing, or SNP_URL is not an http or https URL with a host,
  snp exits 5 (INVALID) and sends nothing.

Examples:
  # Print the content, with the rev on stderr:
  snp mem get projects/demo/notes.md
  # Keep only the rev, for a later --expect-rev:
  rev=$(snp mem get projects/demo/notes.md --rev-only)
  # No document at this id: exit 4, and stderr says
  #   snp: NOT_FOUND: no document at projects/demo/missing.md; check the id, …
  snp mem get projects/demo/missing.md
```

## snp mem put

```text
Usage:
  snp mem put --file <id>=<path> --expect-rev <id>=<rev|new> [--file …
      --expect-rev …] [--reason …] [--md]

Description:
  Write one or more whole documents as one commit: they all land, or none does.
  Give each document a --file <id>=<path> and an --expect-rev <id>=<rev|new>.
  The rev is the one snp mem get printed; new means the document must not exist
  yet. If any document's rev is not current, snpd writes nothing and answers
  CONFLICT, naming each stale id with the rev it has now.
  One commit holds up to 50 documents, and the whole request is at most 1 MiB.
  --reason is recorded on the commit as the Snp-Reason trailer when it is 1 to
  200 characters of letters, digits, spaces and . _ : @ / -. Any other reason is
  left out, and the write still lands.
  To change one frontmatter key, use snp mem set. To add text to the end of a
  document, use snp mem append.

Arguments:
  None besides the flags. --file and --expect-rev each take <id>=<value>: the id
  is everything before the first =, the value everything after it. Every --file
  needs an --expect-rev for the same id, and each id appears once.
  <path> is a local file of UTF-8 text.
  snp checks every id before it sends anything, and exits 5 (INVALID) unless:
  - it is not empty
  - it is at most 512 bytes
  - it is UTF-8
  - it does not start with /
  - it does not contain ..
  - it has no control character
  - it has no empty or . segment (no // and no /./)
  - it has no .git segment, in any case (.GIT too)
  - it is not _search or _commit (a nested x/_search is fine)
  snpd then applies the rule for writes: agents write only
  projects/<slug>/…/<name>.md or .yaml, with a lowercase slug and letters,
  digits, spaces, dots, dashes or underscores in each name. In any letter case,
  MEMORY.md, a memory/ folder and a top .snp/ folder are refused with
  INSUFFICIENT_SCOPE, and the names CLAUDE.md, AGENTS.md and SKILL.md with
  INVALID. The issue and milestone folders, projects/<slug>/issues/ and
  projects/<slug>/<iniciativa>/milestones/, are refused with INSUFFICIENT_SCOPE
  too: write them through snp work / snp state. An id under a path that is
  already a document is INVALID.

Flags:
  --expect-rev <id>=<rev|new> (repeatable)
      <id>=<rev|new>: the rev <id> has now, or new when it must not exist
  --file <id>=<path> (repeatable)
      <id>=<path>: write the local file <path> as document <id>
  --md
      print markdown
  --reason string
      why, recorded as Snp-Reason

Output:
  - Default: snpd's JSON on one line,
    {"commit","parent","revs","replayed","overwrote"}. commit is the new commit
    and parent the one before it. revs maps each id to its new rev. overwrote
    maps each id to the rev it replaced, or null when the document is new.
    replayed is true when this request had already landed, so nothing new was
    written.
  - --md: "Wrote commit <commit> on top of <parent>." (or, when replayed, "This
    write had already landed as commit <commit>; nothing new was written."),
    then one "- <id>: rev <rev> (was <old rev>)" line per id, with "(new
    document)" in place of "(was …)" for a new one.
  After the write, snp moves your projection, if you have one, to the new
  commit. If that fails, stderr gets "snp: PROJECTION_STALE: …" and snp still
  exits 0, because the write landed; run snp pull.
  On an error, stderr gets one line, snp: <CODE>: <message>, and stdout gets
  snpd's error JSON when snpd sent one.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value, an argument besides the flags, no
      --file, a --file or --expect-rev without =, an id given twice, or a --file
      without its --expect-rev, or an --expect-rev without its --file. snp sent
      nothing; stderr says which, so fix the command and run it again.
  CONFLICT (exit 3)
      A document's rev is not the one you passed: it changed since you read it,
      it exists and you passed new, or it does not exist and you passed a rev.
      Nothing was written. stderr names each id with its expected and current
      rev; read them again with snp mem get, merge, and write with the current
      revs.
  INVALID (exit 5)
      An id breaks a rule above, a --file is not readable UTF-8, a rev is
      neither 40 hex characters nor new, the commit is over 50 documents, the
      whole request is over 1 MiB, or SNP_ORG is missing, or SNP_URL is not an
      http or https URL with a host. Nothing was written; stderr says which, so
      fix it and run it again.
  UNAUTHENTICATED (exit 6)
      SNP_TOKEN is malformed, unknown, revoked or expired, so snpd refused it.
      Ask your admin for a new token. When SNP_TOKEN is not set, snp sent none:
      set it, or add the snpd network secret to the cloud environment.
  FORBIDDEN (exit 7)
      SNP_TOKEN was issued for another org than SNP_ORG. Set SNP_ORG to the
      token's org, or use a token issued for this one.
  INSUFFICIENT_SCOPE (exit 7)
      The token lacks the mem scope, or the id is reserved for people and admins
      (MEMORY.md, a memory/ folder, a top .snp/ folder), or it is under
      projects/<slug>/issues/ or projects/<slug>/<iniciativa>/milestones/: write
      those through snp work / snp state. For anything else, write under
      projects/<slug>/ instead, or ask your admin for a token with the mem
      scope. Nothing was written.
  ILLEGAL (exit 8)
      The write changes a milestone's Status line on a road: sinopia roadmap
      (the first - **Status:** line of a ### M<n> block). Only pending to scoped
      is written this way; move a milestone with snp state transition. Or the
      write moves the roadmap's road off sinopia, changes a road line that is
      neither road: vault nor road: sinopia to anything but road: sinopia, or
      writes such a line where the document did not already carry it. A mem
      write may set an invalid road line to road: sinopia; any other road change
      goes through snpd admin put. Nothing was written.
  DIRTY_PROJECTION (exit 9)
      Your projection (~/.snp/projection/<org>) has local changes, so snp sent
      nothing. The message lists them. Copy out anything you want to keep,
      delete the projection with the command in the message, run snp pull, then
      write again.
  TRANSPORT (exit 10)
      No answer from snpd, or an answer from something else, such as a proxy,
      after 2 retries, so the write may have landed. Run snp mem get on each id
      to check before you write again.
  BUSY (exit 11)
      Another write held the org's lock too long, or the write hit snpd's
      deadline, after 2 retries. Nothing changed; run it again in a few seconds.
  CLIENT_TOO_OLD (exit 12)
      This snp is too old: snpd accepts only a newer one. Install the version
      the message names, then run it again.
  REQUEST_ID_REUSED (exit 13)
      A different write already used this write's request id on snpd. snp makes
      a new id on each run, so run the command again.
  LIMIT (exit 14)
      The token went over its request rate, or this session reached its cap on
      writes. Nothing was written. The message says which: wait and run it
      again, or for the session cap, check for a loop before you set a new
      SNP_SESSION.
  INTERNAL (exit 1)
      The request failed inside snpd in a way it does not expect, after 2
      retries. Run snp mem get on each id to check whether the write landed
      before you run it again; if it fails again, send the message to your
      admin.

Environment:
  SNP_URL: snpd's base URL, such as https://host:8484. A url= line in
  ~/.snp/config works too, and SNP_URL wins over it. With neither, snp uses
  https://snp.tesselabs.com.
  SNP_ORG: your org. Required; an org= line in ~/.snp/config works too, and
  SNP_ORG wins over it.
  SNP_TOKEN: your token, read only from the environment, never from argv or a
  file. Leave it unset in a cloud session whose network secret adds it; snp then
  sends no token of its own.
  SNP_SESSION, or else CLAUDE_CODE_REMOTE_SESSION_ID, or else
  CLAUDE_CODE_SESSION_ID: optional. A claude.ai cloud session sets the second,
  which stays the same across /clear and worker restarts. snp sends the first
  one set with every request, and a write records it on the commit as the
  Snp-Session trailer; a write without a valid one is recorded as undeclared.
  When SNP_ORG is missing, or SNP_URL is not an http or https URL with a host,
  snp exits 5 (INVALID) and sends nothing.

Examples:
  # Create a document that must not exist yet:
  snp mem put --file projects/demo/notes.md=./notes.md \
    --expect-rev projects/demo/notes.md=new --reason "first draft"
  # Replace it, only if nobody changed it since you read it:
  rev=$(snp mem get projects/demo/notes.md --rev-only)
  snp mem put --file projects/demo/notes.md=./notes.md \
    --expect-rev projects/demo/notes.md="$rev"
  # With a rev that is not the current one, nothing is written: exit 3,
  # and stderr says
  #   snp: CONFLICT: projects/demo/notes.md changed since you read it; …
  snp mem put --file projects/demo/notes.md=./notes.md \
    --expect-rev projects/demo/notes.md=0000000000000000000000000000000000000000
```

## snp mem set

```text
Usage:
  snp mem set <id> <key> <value> [--expect-rev <rev>] [--md]

Description:
  Set one frontmatter key of an existing document, as one commit. In the first
  --- block, snpd replaces the first line that starts with <key>: by "<key>:
  <value>", or adds that line before the closing ---. Every other byte stays as
  it was.
  The value goes in as you wrote it, with no YAML quoting; quote it yourself
  when YAML needs it.
  Without --expect-rev, the set applies to the document as it is when the write
  lands, and never conflicts. With it, snpd writes nothing and answers CONFLICT
  unless the document still has that rev.
  To replace a whole document, use snp mem put. To add text to the end of one,
  use snp mem append.

Arguments:
  <id>: the document, such as projects/demo/notes.md. It must exist, and its
  first line must be --- with another --- line closing the frontmatter block.
  <key>: 1 to 64 letters, digits, _, . or -; the first is a letter, a digit or
  _.
  <value>: one line of text. Put -- right before a value that starts with -.
  snp checks every id before it sends anything, and exits 5 (INVALID) unless:
  - it is not empty
  - it is at most 512 bytes
  - it is UTF-8
  - it does not start with /
  - it does not contain ..
  - it has no control character
  - it has no empty or . segment (no // and no /./)
  - it has no .git segment, in any case (.GIT too)
  - it is not _search or _commit (a nested x/_search is fine)
  snpd then applies the rule for writes: agents write only
  projects/<slug>/…/<name>.md or .yaml, with a lowercase slug and letters,
  digits, spaces, dots, dashes or underscores in each name. In any letter case,
  MEMORY.md, a memory/ folder and a top .snp/ folder are refused with
  INSUFFICIENT_SCOPE, and the names CLAUDE.md, AGENTS.md and SKILL.md with
  INVALID. The issue and milestone folders, projects/<slug>/issues/ and
  projects/<slug>/<iniciativa>/milestones/, are refused with INSUFFICIENT_SCOPE
  too: write them through snp work / snp state. An id under a path that is
  already a document is INVALID.

Flags:
  --expect-rev string
      the rev you read
  --md
      print markdown

Output:
  - Default: snpd's JSON on one line,
    {"commit","parent","revs","replayed","overwrote"}. commit is the new commit
    and parent the one before it. revs maps each id to its new rev. overwrote
    maps each id to the rev it replaced, or null when the document is new.
    replayed is true when this request had already landed, so nothing new was
    written.
  - --md: "Wrote commit <commit> on top of <parent>." (or, when replayed, "This
    write had already landed as commit <commit>; nothing new was written."),
    then one "- <id>: rev <rev> (was <old rev>)" line per id, with "(new
    document)" in place of "(was …)" for a new one.
  After the write, snp moves your projection, if you have one, to the new
  commit. If that fails, stderr gets "snp: PROJECTION_STALE: …" and snp still
  exits 0, because the write landed; run snp pull.
  On an error, stderr gets one line, snp: <CODE>: <message>, and stdout gets
  snpd's error JSON when snpd sent one.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value, the wrong number of arguments. snp sent
      nothing; stderr says which, so fix the command and run it again.
  CONFLICT (exit 3)
      You passed --expect-rev, and the document's rev is another one, or there
      is no document. Nothing was written. Read it again with snp mem get and
      retry with its current rev.
  NOT_FOUND (exit 4)
      There is no document at that id. Nothing was written. mem set changes an
      existing document, so create it first with snp mem put.
  INVALID (exit 5)
      The id breaks a rule above, the key or the value is not valid, the
      document has no frontmatter block, --expect-rev is not 40 hex characters,
      or SNP_ORG is missing, or SNP_URL is not an http or https URL with a host.
      Nothing was written; stderr says which, so fix it and run it again.
  UNAUTHENTICATED (exit 6)
      SNP_TOKEN is malformed, unknown, revoked or expired, so snpd refused it.
      Ask your admin for a new token. When SNP_TOKEN is not set, snp sent none:
      set it, or add the snpd network secret to the cloud environment.
  FORBIDDEN (exit 7)
      SNP_TOKEN was issued for another org than SNP_ORG. Set SNP_ORG to the
      token's org, or use a token issued for this one.
  INSUFFICIENT_SCOPE (exit 7)
      The token lacks the mem scope, or the id is reserved for people and admins
      (MEMORY.md, a memory/ folder, a top .snp/ folder), or it is under
      projects/<slug>/issues/ or projects/<slug>/<iniciativa>/milestones/: write
      those through snp work / snp state. For anything else, write under
      projects/<slug>/ instead, or ask your admin for a token with the mem
      scope. Nothing was written.
  ILLEGAL (exit 8)
      The write changes a milestone's Status line on a road: sinopia roadmap
      (the first - **Status:** line of a ### M<n> block). Only pending to scoped
      is written this way; move a milestone with snp state transition. Or the
      write moves the roadmap's road off sinopia, changes a road line that is
      neither road: vault nor road: sinopia to anything but road: sinopia, or
      writes such a line where the document did not already carry it. A mem
      write may set an invalid road line to road: sinopia; any other road change
      goes through snpd admin put. Nothing was written.
  DIRTY_PROJECTION (exit 9)
      Your projection (~/.snp/projection/<org>) has local changes, so snp sent
      nothing. The message lists them. Copy out anything you want to keep,
      delete the projection with the command in the message, run snp pull, then
      write again.
  TRANSPORT (exit 10)
      No answer from snpd, or an answer from something else, such as a proxy,
      after 2 retries, so the write may have landed. Run snp mem get on each id
      to check before you write again.
  BUSY (exit 11)
      Another write held the org's lock too long, or the write hit snpd's
      deadline, after 2 retries. Nothing changed; run it again in a few seconds.
  CLIENT_TOO_OLD (exit 12)
      This snp is too old: snpd accepts only a newer one. Install the version
      the message names, then run it again.
  REQUEST_ID_REUSED (exit 13)
      A different write already used this write's request id on snpd. snp makes
      a new id on each run, so run the command again.
  LIMIT (exit 14)
      The token went over its request rate, or this session reached its cap on
      writes. Nothing was written. The message says which: wait and run it
      again, or for the session cap, check for a loop before you set a new
      SNP_SESSION.
  INTERNAL (exit 1)
      The request failed inside snpd in a way it does not expect, after 2
      retries. Run snp mem get on each id to check whether the write landed
      before you run it again; if it fails again, send the message to your
      admin.

Environment:
  SNP_URL: snpd's base URL, such as https://host:8484. A url= line in
  ~/.snp/config works too, and SNP_URL wins over it. With neither, snp uses
  https://snp.tesselabs.com.
  SNP_ORG: your org. Required; an org= line in ~/.snp/config works too, and
  SNP_ORG wins over it.
  SNP_TOKEN: your token, read only from the environment, never from argv or a
  file. Leave it unset in a cloud session whose network secret adds it; snp then
  sends no token of its own.
  SNP_SESSION, or else CLAUDE_CODE_REMOTE_SESSION_ID, or else
  CLAUDE_CODE_SESSION_ID: optional. A claude.ai cloud session sets the second,
  which stays the same across /clear and worker restarts. snp sends the first
  one set with every request, and a write records it on the commit as the
  Snp-Session trailer; a write without a valid one is recorded as undeclared.
  When SNP_ORG is missing, or SNP_URL is not an http or https URL with a host,
  snp exits 5 (INVALID) and sends nothing.

Examples:
  snp mem set projects/demo/notes.md status done
  # Only if nobody changed the document since you read it:
  rev=$(snp mem get projects/demo/notes.md --rev-only)
  snp mem set projects/demo/notes.md status done --expect-rev "$rev"
  # A document with no frontmatter block: exit 5, and stderr says
  #   snp: INVALID: projects/demo/plain.md has no frontmatter block (a --- …
  snp mem set projects/demo/plain.md status done
```

## snp mem append

```text
Usage:
  snp mem append <id> --file <path>|- [--md]

Description:
  Add text to the end of a document, as one commit. If the document does not end
  with a line break, snpd adds one before your text. If there is no document at
  the id, the text becomes a new document.
  An append takes no --expect-rev and never conflicts: snpd adds the text to the
  document as it is when the write lands.
  To replace a whole document, use snp mem put. To change one frontmatter key,
  use snp mem set.

Arguments:
  <id>: the document, such as projects/demo/notes.md.
  --file is required: a local file of UTF-8 text, or - to read the text from
  stdin. The text must not be empty.
  snp checks every id before it sends anything, and exits 5 (INVALID) unless:
  - it is not empty
  - it is at most 512 bytes
  - it is UTF-8
  - it does not start with /
  - it does not contain ..
  - it has no control character
  - it has no empty or . segment (no // and no /./)
  - it has no .git segment, in any case (.GIT too)
  - it is not _search or _commit (a nested x/_search is fine)
  snpd then applies the rule for writes: agents write only
  projects/<slug>/…/<name>.md or .yaml, with a lowercase slug and letters,
  digits, spaces, dots, dashes or underscores in each name. In any letter case,
  MEMORY.md, a memory/ folder and a top .snp/ folder are refused with
  INSUFFICIENT_SCOPE, and the names CLAUDE.md, AGENTS.md and SKILL.md with
  INVALID. The issue and milestone folders, projects/<slug>/issues/ and
  projects/<slug>/<iniciativa>/milestones/, are refused with INSUFFICIENT_SCOPE
  too: write them through snp work / snp state. An id under a path that is
  already a document is INVALID.

Flags:
  --file string
      <path>, or - for stdin
  --md
      print markdown

Output:
  - Default: snpd's JSON on one line,
    {"commit","parent","revs","replayed","overwrote"}. commit is the new commit
    and parent the one before it. revs maps each id to its new rev. overwrote
    maps each id to the rev it replaced, or null when the document is new.
    replayed is true when this request had already landed, so nothing new was
    written.
  - --md: "Wrote commit <commit> on top of <parent>." (or, when replayed, "This
    write had already landed as commit <commit>; nothing new was written."),
    then one "- <id>: rev <rev> (was <old rev>)" line per id, with "(new
    document)" in place of "(was …)" for a new one.
  After the write, snp moves your projection, if you have one, to the new
  commit. If that fails, stderr gets "snp: PROJECTION_STALE: …" and snp still
  exits 0, because the write landed; run snp pull.
  On an error, stderr gets one line, snp: <CODE>: <message>, and stdout gets
  snpd's error JSON when snpd sent one.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value, no <id>, more than one <id>, or no
      --file. snp sent nothing; stderr says which, so fix the command and run it
      again.
  INVALID (exit 5)
      The id breaks a rule above, the text is empty or not UTF-8, the file
      cannot be read, the text makes the request too big (the whole request is
      at most 1 MiB), or SNP_ORG is missing, or SNP_URL is not an http or https
      URL with a host. Nothing was written; stderr says which, so fix it and run
      it again.
  UNAUTHENTICATED (exit 6)
      SNP_TOKEN is malformed, unknown, revoked or expired, so snpd refused it.
      Ask your admin for a new token. When SNP_TOKEN is not set, snp sent none:
      set it, or add the snpd network secret to the cloud environment.
  FORBIDDEN (exit 7)
      SNP_TOKEN was issued for another org than SNP_ORG. Set SNP_ORG to the
      token's org, or use a token issued for this one.
  INSUFFICIENT_SCOPE (exit 7)
      The token lacks the mem scope, or the id is reserved for people and admins
      (MEMORY.md, a memory/ folder, a top .snp/ folder), or it is under
      projects/<slug>/issues/ or projects/<slug>/<iniciativa>/milestones/: write
      those through snp work / snp state. For anything else, write under
      projects/<slug>/ instead, or ask your admin for a token with the mem
      scope. Nothing was written.
  ILLEGAL (exit 8)
      The write changes a milestone's Status line on a road: sinopia roadmap
      (the first - **Status:** line of a ### M<n> block). Only pending to scoped
      is written this way; move a milestone with snp state transition. Or the
      write moves the roadmap's road off sinopia, changes a road line that is
      neither road: vault nor road: sinopia to anything but road: sinopia, or
      writes such a line where the document did not already carry it. A mem
      write may set an invalid road line to road: sinopia; any other road change
      goes through snpd admin put. Nothing was written.
  DIRTY_PROJECTION (exit 9)
      Your projection (~/.snp/projection/<org>) has local changes, so snp sent
      nothing. The message lists them. Copy out anything you want to keep,
      delete the projection with the command in the message, run snp pull, then
      write again.
  TRANSPORT (exit 10)
      No answer from snpd, or an answer from something else, such as a proxy,
      after 2 retries, so the write may have landed. Run snp mem get on each id
      to check before you write again.
  BUSY (exit 11)
      Another write held the org's lock too long, or the write hit snpd's
      deadline, after 2 retries. Nothing changed; run it again in a few seconds.
  CLIENT_TOO_OLD (exit 12)
      This snp is too old: snpd accepts only a newer one. Install the version
      the message names, then run it again.
  REQUEST_ID_REUSED (exit 13)
      A different write already used this write's request id on snpd. snp makes
      a new id on each run, so run the command again.
  LIMIT (exit 14)
      The token went over its request rate, or this session reached its cap on
      writes. Nothing was written. The message says which: wait and run it
      again, or for the session cap, check for a loop before you set a new
      SNP_SESSION.
  INTERNAL (exit 1)
      The request failed inside snpd in a way it does not expect, after 2
      retries. Run snp mem get on each id to check whether the write landed
      before you run it again; if it fails again, send the message to your
      admin.

Environment:
  SNP_URL: snpd's base URL, such as https://host:8484. A url= line in
  ~/.snp/config works too, and SNP_URL wins over it. With neither, snp uses
  https://snp.tesselabs.com.
  SNP_ORG: your org. Required; an org= line in ~/.snp/config works too, and
  SNP_ORG wins over it.
  SNP_TOKEN: your token, read only from the environment, never from argv or a
  file. Leave it unset in a cloud session whose network secret adds it; snp then
  sends no token of its own.
  SNP_SESSION, or else CLAUDE_CODE_REMOTE_SESSION_ID, or else
  CLAUDE_CODE_SESSION_ID: optional. A claude.ai cloud session sets the second,
  which stays the same across /clear and worker restarts. snp sends the first
  one set with every request, and a write records it on the commit as the
  Snp-Session trailer; a write without a valid one is recorded as undeclared.
  When SNP_ORG is missing, or SNP_URL is not an http or https URL with a host,
  snp exits 5 (INVALID) and sends nothing.

Examples:
  snp mem append projects/demo/notes.md --file ./more.md
  echo "Shipped on 2026-10-17." | snp mem append projects/demo/notes.md --file -
  # Without --file: exit 2, and stderr says
  #   snp: mem append takes <id> --file <path>, or --file - to read stdin
  snp mem append projects/demo/notes.md
```

## snp mem list

```text
Usage:
  snp mem list [--prefix p] [--limit n] [--cursor c] [--md]

Description:
  List the ids of your documents in id order, with each one's rev and size, one
  page at a time. Only documents are listed, not folders.
  --prefix keeps only the ids that start with it, such as projects/demo/. It is
  plain text, not a folder: projects/de matches projects/demo/ and
  projects/design/. It must not start with / or contain .., and is at most 512
  bytes of UTF-8 with no control character.
  To find lines inside documents, use snp mem search. To read one, use snp mem
  get.

Arguments:
  None besides the flags.

Flags:
  --cursor string
      the next value of the previous page
  --limit int
      items per page, 1 to 500 (default 100)
  --md
      print markdown
  --prefix string
      only ids starting with this

Output:
  - Default: snpd's JSON on one line,
    {"items":[{"id","rev","size"}],"at","next"}. rev is the document's 40-hex
    rev, the one --expect-rev takes; size is in bytes; at is the commit the page
    was read at. When nothing matches, items is [] and snp exits 0.
  - --md: one "- <id> (<size> bytes, rev <rev>)" line per document. When nothing
    matches, it prints "No documents match." and exits 0.
  A page holds up to --limit items: 1 to 500, or 100 when you leave it out or
  pass 0. Any other number is INVALID. next is null on the last page. Otherwise,
  pass it to --cursor with the same --prefix to get the next page. Every page
  reads the commit in at, so writes that land between pages do not change the
  listing. With --md, the last line reads More results: run snp …, the command
  that prints the next page.
  On an error, stderr gets one line, snp: <CODE>: <message>, and stdout gets
  snpd's error JSON when snpd sent one.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value, the wrong number of arguments. snp sent
      nothing; stderr says which, so fix the command and run it again.
  INVALID (exit 5)
      The prefix breaks a rule above, --limit is outside 1 to 500, the cursor is
      not one snpd issued or was issued for another prefix, or SNP_ORG is
      missing, or SNP_URL is not an http or https URL with a host. stderr says
      which; fix it, or drop --cursor to start from the first page.
  UNAUTHENTICATED (exit 6)
      SNP_TOKEN is malformed, unknown, revoked or expired, so snpd refused it.
      Ask your admin for a new token. When SNP_TOKEN is not set, snp sent none:
      set it, or add the snpd network secret to the cloud environment.
  FORBIDDEN (exit 7)
      SNP_TOKEN was issued for another org than SNP_ORG. Set SNP_ORG to the
      token's org, or use a token issued for this one.
  INSUFFICIENT_SCOPE (exit 7)
      The token lacks the mem scope. Ask your admin for a token that has it.
  TRANSPORT (exit 10)
      No answer from snpd, or an answer from something else, such as a proxy,
      after 2 retries. Check the network and that SNP_URL points at snpd, then
      run it again.
  CLIENT_TOO_OLD (exit 12)
      This snp is too old: snpd accepts only a newer one. Install the version
      the message names, then run it again.
  LIMIT (exit 14)
      The token went over its request rate. Wait the seconds the message names,
      then run it again.
  INTERNAL (exit 1)
      The request failed inside snpd in a way it does not expect, after 2
      retries. Run it again; if it fails again, send the message to your admin.

Environment:
  SNP_URL: snpd's base URL, such as https://host:8484. A url= line in
  ~/.snp/config works too, and SNP_URL wins over it. With neither, snp uses
  https://snp.tesselabs.com.
  SNP_ORG: your org. Required; an org= line in ~/.snp/config works too, and
  SNP_ORG wins over it.
  SNP_TOKEN: your token, read only from the environment, never from argv or a
  file. Leave it unset in a cloud session whose network secret adds it; snp then
  sends no token of its own.
  SNP_SESSION, or else CLAUDE_CODE_REMOTE_SESSION_ID, or else
  CLAUDE_CODE_SESSION_ID: optional. A claude.ai cloud session sets the second,
  which stays the same across /clear and worker restarts. snp sends the first
  one set with every request, and a write records it on the commit as the
  Snp-Session trailer; a write without a valid one is recorded as undeclared.
  When SNP_ORG is missing, or SNP_URL is not an http or https URL with a host,
  snp exits 5 (INVALID) and sends nothing.

Examples:
  snp mem list --prefix projects/demo/ --md
  snp mem list --limit 500
  # A prefix that starts with /: exit 5, and stderr says
  #   snp: INVALID: the prefix "/projects/" starts with / or contains ..; …
  snp mem list --prefix /projects/
```

## snp mem search

```text
Usage:
  snp mem search <query> [--prefix p] [--limit n] [--cursor c] [--md]

Description:
  Find the lines of your documents that contain <query>. snpd runs git grep -F
  over every document at one commit: the query matches as literal text, not as a
  pattern, and the match is case-sensitive. Spaces, quotes, *, regex syntax, OR
  and -term have no special meaning: the whole query is one literal string.
  There is one hit per matching line, even when the query occurs several times
  on that line. Every line counts, frontmatter included; binary files are
  skipped.
  Results are ordered by id, then line; there is no ranking. Semantic and hybrid
  search are not available yet.
  --prefix keeps only the ids that start with it, such as projects/demo/. It is
  plain text, not a folder: projects/de matches projects/demo/ and
  projects/design/. It must not start with / or contain .., and is at most 512
  bytes of UTF-8 with no control character.
  To list ids instead of lines, use snp mem list. To read a whole document, use
  snp mem get.

Arguments:
  <query>: the text to find, 1 to 512 bytes of UTF-8 with no control character,
  so no tab and no line break. Quote it when it has spaces, and put -- before a
  query that starts with -.

Flags:
  --cursor string
      the next value of the previous page
  --limit int
      items per page, 1 to 500 (default 100)
  --md
      print markdown
  --prefix string
      only ids starting with this

Output:
  - Default: snpd's JSON on one line,
    {"items":[{"id","line","text"}],"at","next"}. "line" counts from 1,
    frontmatter lines included. "text" is the whole line, cut at 1 KiB (1024
    bytes) on a UTF-8 boundary. at is the commit the page was read at. When
    nothing matches, it prints {"items":[],"at":"<commit>","next":null} and
    exits 0.
  - --md: one "- <id>:<line>: <text>" line per match. When nothing matches, it
    prints "No lines match." and exits 0.
  A page holds up to --limit lines: 1 to 500, or 100 when you leave it out or
  pass 0. Any other number is INVALID. next is null on the last page. Otherwise,
  pass it to --cursor with the same query and --prefix to get the next page.
  Every page reads the commit in at, so writes that land between pages do not
  change the results. With --md, the last line reads More results: run snp …,
  the command that prints the next page.
  On an error, stderr gets one line, snp: <CODE>: <message>, and stdout gets
  snpd's error JSON when snpd sent one.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value, the wrong number of arguments. snp sent
      nothing; stderr says which, so fix the command and run it again.
  INVALID (exit 5)
      The query is empty, over 512 bytes, not UTF-8 or holds a control
      character; the prefix breaks a rule above; --limit is outside 1 to 500;
      the cursor is not one snpd issued or was issued for another query or
      prefix; or SNP_ORG is missing, or SNP_URL is not an http or https URL with
      a host. stderr says which; fix it, or drop --cursor to start from the
      first page.
  UNAUTHENTICATED (exit 6)
      SNP_TOKEN is malformed, unknown, revoked or expired, so snpd refused it.
      Ask your admin for a new token. When SNP_TOKEN is not set, snp sent none:
      set it, or add the snpd network secret to the cloud environment.
  FORBIDDEN (exit 7)
      SNP_TOKEN was issued for another org than SNP_ORG. Set SNP_ORG to the
      token's org, or use a token issued for this one.
  INSUFFICIENT_SCOPE (exit 7)
      The token lacks the mem scope. Ask your admin for a token that has it.
  TRANSPORT (exit 10)
      No answer from snpd, or an answer from something else, such as a proxy,
      after 2 retries. Check the network and that SNP_URL points at snpd, then
      run it again.
  CLIENT_TOO_OLD (exit 12)
      This snp is too old: snpd accepts only a newer one. Install the version
      the message names, then run it again.
  LIMIT (exit 14)
      The token went over its request rate. Wait the seconds the message names,
      then run it again.
  INTERNAL (exit 1)
      The request failed inside snpd in a way it does not expect, after 2
      retries. Run it again; if it fails again, send the message to your admin.

Environment:
  SNP_URL: snpd's base URL, such as https://host:8484. A url= line in
  ~/.snp/config works too, and SNP_URL wins over it. With neither, snp uses
  https://snp.tesselabs.com.
  SNP_ORG: your org. Required; an org= line in ~/.snp/config works too, and
  SNP_ORG wins over it.
  SNP_TOKEN: your token, read only from the environment, never from argv or a
  file. Leave it unset in a cloud session whose network secret adds it; snp then
  sends no token of its own.
  SNP_SESSION, or else CLAUDE_CODE_REMOTE_SESSION_ID, or else
  CLAUDE_CODE_SESSION_ID: optional. A claude.ai cloud session sets the second,
  which stays the same across /clear and worker restarts. snp sends the first
  one set with every request, and a write records it on the commit as the
  Snp-Session trailer; a write without a valid one is recorded as undeclared.
  When SNP_ORG is missing, or SNP_URL is not an http or https URL with a host,
  snp exits 5 (INVALID) and sends nothing.

Examples:
  snp mem search "release plan" --prefix projects/demo/
  snp mem search --md --limit 20 "release plan"
  # A query that starts with -:
  snp mem search --md -- "-rc1"
  # --limit over 500: exit 5, and stderr says
  #   snp: INVALID: limit must be a whole number from 1 to 500; …
  snp mem search "release plan" --limit 900
```

## snp state get

```text
Usage:
  snp state get <entity>

Description:
  Read the state of one milestone or one issue as it is at snpd's latest commit,
  with the states it can move to under the org's schema (.snp/schema.yaml).
  A milestone's state is the Status line of its block in the iniciativa's
  roadmap. An issue's state is the status in its file under
  projects/<slug>/issues/.
  To move a milestone, use snp state transition.

Arguments:
  <entity>: the milestone or issue to read, in one of two forms:
  - milestone:<slug>/<iniciativa>/<MID>, such as milestone:sinopia/demo/M1.
  - issue:<slug>/<n>, such as issue:sinopia/12.
  snp checks every part before it sends anything, and exits 5 (INVALID) unless:
  - <slug> matches ^[a-z0-9][a-z0-9-]{0,62}$ and is not memory.
  - <iniciativa> matches the same pattern and is not issues, milestones or
    memory.
  - <MID> matches ^M[0-9]{1,4}[a-z]?$.
  - <n> matches ^[1-9][0-9]{0,8}$.

Output:
  snpd's JSON on one line:
  {"entity","machine","state","record","rev","allowed"}. entity is the ref as
  snpd read it. machine is milestone or issue. state is the current state.
  record is the id of the document that holds it, and rev that document's rev.
  allowed lists the states it can move to, and is [] when there are none.
  On an error, stderr gets one line, snp: <CODE>: <message>, and stdout gets
  snpd's error JSON when snpd sent one.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value, the wrong number of arguments. snp sent
      nothing; stderr says which, so fix the command and run it again.
  NOT_FOUND (exit 4)
      There is no roadmap for the iniciativa, no ### M{n} block in it for that
      milestone, or no issue file for that number. stderr names the document;
      check the ref.
  INVALID (exit 5)
      The ref breaks a rule above, there are two roadmaps for the iniciativa,
      the block has no Status line or one that is not a milestone state, the
      issue file does not parse or its status is not an issue state, or SNP_ORG
      is missing, or SNP_URL is not an http or https URL with a host. stderr
      says which; fix it and run it again.
  INSUFFICIENT_SCOPE (exit 7)
      The token lacks the state scope. Ask your admin for one that has it.
  UNAVAILABLE (exit 15)
      The org has no schema, or one this snpd can't read; stderr says which. Ask
      your admin to run snpd admin schema install (with --force to replace a
      schema snpd can't read).
  UNAUTHENTICATED (exit 6)
      SNP_TOKEN is malformed, unknown, revoked or expired, so snpd refused it.
      Ask your admin for a new token. When SNP_TOKEN is not set, snp sent none:
      set it, or add the snpd network secret to the cloud environment.
  FORBIDDEN (exit 7)
      SNP_TOKEN was issued for another org than SNP_ORG. Set SNP_ORG to the
      token's org, or use a token issued for this one.
  TRANSPORT (exit 10)
      No answer from snpd, or an answer from something else, such as a proxy,
      after 2 retries. Check the network and that SNP_URL points at snpd, then
      run it again.
  CLIENT_TOO_OLD (exit 12)
      This snp is too old: snpd accepts only a newer one. Install the version
      the message names, then run it again.
  LIMIT (exit 14)
      The token went over its request rate. Wait the seconds the message names,
      then run it again.
  INTERNAL (exit 1)
      The request failed inside snpd in a way it does not expect, after 2
      retries. Run it again; if it fails again, send the message to your admin.

Environment:
  SNP_URL: snpd's base URL, such as https://host:8484. A url= line in
  ~/.snp/config works too, and SNP_URL wins over it. With neither, snp uses
  https://snp.tesselabs.com.
  SNP_ORG: your org. Required; an org= line in ~/.snp/config works too, and
  SNP_ORG wins over it.
  SNP_TOKEN: your token, read only from the environment, never from argv or a
  file. Leave it unset in a cloud session whose network secret adds it; snp then
  sends no token of its own.
  SNP_SESSION, or else CLAUDE_CODE_REMOTE_SESSION_ID, or else
  CLAUDE_CODE_SESSION_ID: optional. A claude.ai cloud session sets the second,
  which stays the same across /clear and worker restarts. snp sends the first
  one set with every request, and a write records it on the commit as the
  Snp-Session trailer; a write without a valid one is recorded as undeclared.
  When SNP_ORG is missing, or SNP_URL is not an http or https URL with a host,
  snp exits 5 (INVALID) and sends nothing.

Examples:
  # Read a milestone:
  snp state get milestone:sinopia/demo/M1
  # Read an issue:
  snp state get issue:sinopia/12
  # A ref that breaks a rule: exit 5, and snp sends nothing.
  snp state get milestone:sinopia/demo/M12345
```

## snp state transition

```text
Usage:
  snp state transition <entity> --from <state> --to <state> [--reason text]
      [--plan path]

Description:
  Move one milestone to another state, as one commit. snpd rewrites the Status
  line of its block in the iniciativa's roadmap and the state in the milestone's
  yaml (projects/<slug>/<iniciativa>/milestones/<MID>.yaml), and records the
  edge in the commit. Only an edge of the org schema's milestone machine is
  allowed; snp state get shows the milestone's state now and the states it can
  move to.
  --from is the state you believe the milestone is in. If another writer moved
  it first, nothing is written and you get a CONFLICT naming the state it is in
  now. The roadmap is the record: when the yaml holds another machine state, the
  move is checked against the roadmap and only its state value is rewritten;
  breakdown_key, issues and comments stay. A yaml whose state names anything
  else is refused with INVALID.
  In the default schema, one edge has a guard. A milestone cannot move from
  in-progress to done while any issue its yaml lists is open, that is, not done
  or canceled; snpd refuses the move and names each open issue. An issue that is
  missing or does not parse refuses it too. A milestone with no yaml, or a yaml
  that lists no issues, has none open.
  With --plan, the move from scoped to planned also sets the plan: line of the
  milestone's spec, in the same commit. snpd finds the spec from the - **Spec:**
  link in the milestone's block, and only on a road: sinopia roadmap. snp sends
  only the plan's path.
  When an answer is lost, snp resends the same request id on its own, and snpd
  replays the first result with replayed: true instead of writing twice. To move
  an issue, use snp work transition.

Arguments:
  <entity>: the milestone to move, milestone:<slug>/<iniciativa>/<MID>, such as
  milestone:sinopia/demo/M1. An issue ref is INVALID: issues move through snp
  work transition.
  snp checks the ref before it sends anything, and exits 5 (INVALID) unless:
  - <slug> matches ^[a-z0-9][a-z0-9-]{0,62}$ and is not memory.
  - <iniciativa> matches the same pattern and is not issues, milestones or
    memory.
  - <MID> matches ^M[0-9]{1,4}[a-z]?$.
  --from <state>: the state the milestone is in now.
  --to <state>: the state to move it to.
  --reason <text>: why, recorded on the commit as the Snp-Reason trailer. It
  must be 1 to 200 characters of letters, digits, spaces and . _ : @ / -. Any
  other reason is INVALID and nothing is written; unlike snp mem put, this verb
  does not drop it.
  --plan <path>: the plan, docs/superpowers/plans/<name>.md, whose basename the
  spec records as plan: "[[<name>]]". Only with --from scoped --to planned: with
  any other edge snp exits 5 (INVALID) and sends nothing.

Flags:
  --from <state>
      <state>: the state the milestone is in now (required)
  --plan <path>
      <path>: the plan the milestone's spec records (scoped → planned only)
  --reason <text>
      <text>: why, kept in the commit
  --to <state>
      <state>: the state to move it to (required)

Output:
  snpd's JSON on one line:
  {"commit","parent","revs","replayed","overwrote","entity","from","to"}.
  overwrote maps each id to the rev it replaced, or null when the document is
  new. replayed is true when this request had already landed.
  After the write, snp moves your projection, if you have one, to the new
  commit. If that fails, stderr gets "snp: PROJECTION_STALE: …" and snp still
  exits 0, because the write landed; run snp pull.
  On an error, stderr gets one line, snp: <CODE>: <message>, and stdout gets
  snpd's error JSON when snpd sent one.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value, the wrong number of arguments. snp sent
      nothing; stderr says which, so fix the command and run it again.
  NOT_FOUND (exit 4)
      There is no roadmap for the iniciativa, or no ### M{n} block in it for
      that milestone. Nothing was written.
  INVALID (exit 5)
      The ref is not a valid milestone ref (nothing is sent), or it is an issue
      ref; a state is malformed; the commit trailer for the move would pass the
      200-byte limit, because the ref and the two states are too long together;
      --reason is not 1 to 200 characters of letters, digits, spaces and . _ : @
      / -; there are two roadmaps for the iniciativa; the block has no Status
      line or one that is not a milestone state; --plan with an edge other than
      scoped to planned (nothing is sent); the plan is not
      docs/superpowers/plans/<name>.md; the block has no - **Spec:** link, it
      names no spec, or the spec has no leading frontmatter (stderr names the
      document); or the milestone's yaml or one of its issues cannot be parsed.
      stderr names the document and says to repair it with snpd admin put, which
      your admin runs. Or snpd does not know --plan yet: stderr reads snpd does
      not know --plan: deploy the snpd of this release first (runbook section
      2); snp never retries the move without the plan. Nothing was written.
  ILLEGAL (exit 8)
      The milestone machine has no edge from --from to --to: snpd's JSON then
      carries allowed, the states that can follow, and the message lists them.
      Or, in the default schema, the milestone is moving to done and an issue it
      lists is open: the message names each. With --plan, the roadmap is not on
      road: sinopia: the message names its road. Nothing was written.
  CONFLICT (exit 3)
      The milestone is not in --from. Nothing was written. stderr reads <entity>
      is <current>, expected <from>; read the state again with snp state get and
      retry with it.
  INSUFFICIENT_SCOPE (exit 7)
      The token lacks the state scope. Ask your admin for a token that has it.
      Nothing was written.
  UNAVAILABLE (exit 15)
      The org has no schema, or one this snpd can't read; stderr says which. Ask
      your admin to run snpd admin schema install.
  DIRTY_PROJECTION (exit 9)
      Your projection (~/.snp/projection/<org>) has local changes, so snp sent
      nothing. The message lists them. Copy out anything you want to keep,
      delete the projection with the command in the message, run snp pull, then
      write again.
  UNAUTHENTICATED (exit 6)
      SNP_TOKEN is malformed, unknown, revoked or expired, so snpd refused it.
      Ask your admin for a new token. When SNP_TOKEN is not set, snp sent none:
      set it, or add the snpd network secret to the cloud environment.
  FORBIDDEN (exit 7)
      SNP_TOKEN was issued for another org than SNP_ORG. Set SNP_ORG to the
      token's org, or use a token issued for this one.
  TRANSPORT (exit 10)
      No answer from snpd, or an answer from something else, such as a proxy,
      after 2 retries, so the write may have landed. Run snp mem get on each id
      to check before you write again.
  BUSY (exit 11)
      Another write held the org's lock too long, or the write hit snpd's
      deadline, after 2 retries. Nothing changed; run it again in a few seconds.
  CLIENT_TOO_OLD (exit 12)
      This snp is too old: snpd accepts only a newer one. Install the version
      the message names, then run it again.
  REQUEST_ID_REUSED (exit 13)
      A different write already used this write's request id on snpd. snp makes
      a new id on each run, so run the command again.
  LIMIT (exit 14)
      The token went over its request rate, or this session reached its cap on
      writes. Nothing was written. The message says which: wait and run it
      again, or for the session cap, check for a loop before you set a new
      SNP_SESSION.
  INTERNAL (exit 1)
      The request failed inside snpd in a way it does not expect, after 2
      retries. Run snp mem get on each id to check whether the write landed
      before you run it again; if it fails again, send the message to your
      admin.

Environment:
  SNP_URL: snpd's base URL, such as https://host:8484. A url= line in
  ~/.snp/config works too, and SNP_URL wins over it. With neither, snp uses
  https://snp.tesselabs.com.
  SNP_ORG: your org. Required; an org= line in ~/.snp/config works too, and
  SNP_ORG wins over it.
  SNP_TOKEN: your token, read only from the environment, never from argv or a
  file. Leave it unset in a cloud session whose network secret adds it; snp then
  sends no token of its own.
  SNP_SESSION, or else CLAUDE_CODE_REMOTE_SESSION_ID, or else
  CLAUDE_CODE_SESSION_ID: optional. A claude.ai cloud session sets the second,
  which stays the same across /clear and worker restarts. snp sends the first
  one set with every request, and a write records it on the commit as the
  Snp-Session trailer; a write without a valid one is recorded as undeclared.
  When SNP_ORG is missing, or SNP_URL is not an http or https URL with a host,
  snp exits 5 (INVALID) and sends nothing.

Examples:
  # Plan a scoped milestone:
  snp state transition milestone:sinopia/demo/M1 --from scoped --to planned
  # Plan it and record the plan on its spec, in one commit:
  snp state transition milestone:sinopia/demo/M1 --from scoped --to planned \
    --plan docs/superpowers/plans/2026-11-04-demo-M1-one.md
  # Close it, once its issues are done or canceled:
  snp state transition milestone:sinopia/demo/M1 --from in-progress \
    --to done --reason "all issues merged"
  # The milestone is no longer scoped: exit 3, and stderr says
  #   snp: CONFLICT: milestone:sinopia/demo/M1 is planned, expected scoped; …
  snp state transition milestone:sinopia/demo/M1 --from scoped --to planned
```

## snp work create

```text
Usage:
  snp work create --project <slug> --iniciativa <ini> --milestone <MID> --title
      <text> --body <path|-> [--blocked-by N[,N...]]

Description:
  File one issue into a milestone, as one commit. The issue lands in backlog:
  the commit holds the issue file, the milestone yaml's issues list and the
  project's seq file, and snpd numbers the issue.
  The milestone needs road: sinopia, a breakdown (a breakdown_key in its yaml;
  if it has none, use snp work create_breakdown) and a state of planned or
  in-progress.
  A deferral goes to the project's standing maintenance milestone (for sinopia,
  --iniciativa maintenance --milestone M1), not to the milestone being built: an
  issue created into a milestone being built holds that milestone's done until
  the issue is done or canceled.
  Only a person moves a backlog issue to ready.
  When an answer is lost, snp resends the same request id on its own, and snpd
  replays the first result with replayed: true instead of writing twice. A
  request id reused after 24 hours, or a new run of snp, files a new issue:
  cancel a duplicate with snp work transition N --project S --expect-status
  backlog --to canceled.

Arguments:
  --project <slug>: the project, such as sinopia.
  --iniciativa <ini>: the iniciativa, such as demo.
  --milestone <MID>: the milestone id, such as M1.
  --title <text>: the issue title, one line.
  --body <path|->: the issue body, up to 16 KiB of UTF-8 text, from a file or -
  for stdin.
  --blocked-by N[,N...]: a comma list of existing issue numbers of the project,
  in any milestone. snp sorts it and drops duplicates.

Flags:
  --blocked-by N[,N...]
      N[,N...]: issue numbers this one waits for
  --body <path|->
      <path|->: the issue body, or - for stdin (required)
  --iniciativa <ini>
      <ini>: the iniciativa (required)
  --milestone <MID>
      <MID>: the milestone (required)
  --project <slug>
      <slug>: the project (required)
  --title <text>
      <text>: the issue title (required)

Output:
  snpd's JSON on one line:
  {"commit","parent","revs","replayed","overwrote","number","id"}. number and id
  name the new issue. overwrote maps each id to the rev it replaced, or null
  when the document is new. replayed is true when this request had already
  landed.
  After the write, snp moves your projection, if you have one, to the new
  commit. If that fails, stderr gets "snp: PROJECTION_STALE: …" and snp still
  exits 0, because the write landed; run snp pull.
  On an error, stderr gets one line, snp: <CODE>: <message>, and stdout gets
  snpd's error JSON when snpd sent one.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value, the wrong number of arguments. snp sent
      nothing; stderr says which, so fix the command and run it again.
  NOT_FOUND (exit 4)
      There is no roadmap for the iniciativa. Or the milestone has a breakdown
      but its roadmap has no ### M{n} block. Nothing was written.
  INVALID (exit 5)
      The body file is missing, over 16384 bytes or not UTF-8 text (nothing is
      sent); snpd refuses the title or body under its rules; a --blocked-by
      number has no issue in the project or is not positive; there are more than
      40 of them; the milestone's yaml or the seq file cannot be parsed; or
      SNP_ORG is missing, or SNP_URL is not an http or https URL with a host.
      Nothing was written; stderr says which.
  ILLEGAL (exit 8)
      The road is not sinopia, the milestone has no breakdown, or its state is
      not planned or in-progress. A milestone with no yaml, including one whose
      --milestone is misspelled, reads as having no breakdown; snp state get
      milestone:<slug>/<ini>/<MID> shows whether it exists. Nothing was written;
      stderr names which.
  INSUFFICIENT_SCOPE (exit 7)
      The token lacks the work scope. Ask your admin for a token that has it.
      Nothing was written.
  UNAVAILABLE (exit 15)
      The org has no schema, or one this snpd can't read; stderr says which. Ask
      your admin to run snpd admin schema install.
  DIRTY_PROJECTION (exit 9)
      Your projection (~/.snp/projection/<org>) has local changes, so snp sent
      nothing. The message lists them. Copy out anything you want to keep,
      delete the projection with the command in the message, run snp pull, then
      write again.
  UNAUTHENTICATED (exit 6)
      SNP_TOKEN is malformed, unknown, revoked or expired, so snpd refused it.
      Ask your admin for a new token. When SNP_TOKEN is not set, snp sent none:
      set it, or add the snpd network secret to the cloud environment.
  FORBIDDEN (exit 7)
      SNP_TOKEN was issued for another org than SNP_ORG. Set SNP_ORG to the
      token's org, or use a token issued for this one.
  TRANSPORT (exit 10)
      No answer from snpd, or an answer from something else, such as a proxy,
      after 2 retries, so the write may have landed. Run snp work query
      --project <slug> --iniciativa <ini> --milestone <MID> --status backlog and
      look for an issue with your title before you run it again: a new run files
      a second issue.
  BUSY (exit 11)
      Another write held the org's lock too long, or the write hit snpd's
      deadline, after 2 retries. Nothing changed; run it again in a few seconds.
  CLIENT_TOO_OLD (exit 12)
      This snp is too old: snpd accepts only a newer one. Install the version
      the message names, then run it again.
  REQUEST_ID_REUSED (exit 13)
      A different write already used this write's request id on snpd. snp makes
      a new id on each run, so run the command again.
  LIMIT (exit 14)
      The token went over its request rate, or this session reached its cap on
      writes. Nothing was written. The message says which: wait and run it
      again, or for the session cap, check for a loop before you set a new
      SNP_SESSION.
  INTERNAL (exit 1)
      The request failed inside snpd in a way it does not expect, after 2
      retries, so the write may have landed. Run snp work query --project <slug>
      --iniciativa <ini> --milestone <MID> --status backlog and look for an
      issue with your title before you run it again: a new run files a second
      issue. If it fails again, send the message to your admin.

Environment:
  SNP_URL: snpd's base URL, such as https://host:8484. A url= line in
  ~/.snp/config works too, and SNP_URL wins over it. With neither, snp uses
  https://snp.tesselabs.com.
  SNP_ORG: your org. Required; an org= line in ~/.snp/config works too, and
  SNP_ORG wins over it.
  SNP_TOKEN: your token, read only from the environment, never from argv or a
  file. Leave it unset in a cloud session whose network secret adds it; snp then
  sends no token of its own.
  SNP_SESSION, or else CLAUDE_CODE_REMOTE_SESSION_ID, or else
  CLAUDE_CODE_SESSION_ID: optional. A claude.ai cloud session sets the second,
  which stays the same across /clear and worker restarts. snp sends the first
  one set with every request, and a write records it on the commit as the
  Snp-Session trailer; a write without a valid one is recorded as undeclared.
  When SNP_ORG is missing, or SNP_URL is not an http or https URL with a host,
  snp exits 5 (INVALID) and sends nothing.

Examples:
  # File an issue from a file:
  snp work create --project sinopia --iniciativa demo --milestone M1 \
    --title "Handle empty input" --body ./body.md
  # From stdin, blocked by issues 3 and 5:
  cat body.md | snp work create --project sinopia --iniciativa demo \
    --milestone M1 --title "Retry on 503" --body - --blocked-by 5,3
  # The milestone has no breakdown: exit 8, and stderr says
  #   snp: ILLEGAL: … has no breakdown; …
  snp work create --project sinopia --iniciativa demo --milestone M2 \
    --title "Handle empty input" --body ./body.md
```

## snp work create_breakdown

```text
Usage:
  snp work create_breakdown --file <path|->

Description:
  Create every issue of one milestone as one commit: they all land, or none
  does. snpd numbers the issues and writes their files under
  projects/<slug>/issues/. It does not change the milestone's state: the
  milestone stays planned.
  Only a milestone whose roadmap has road: sinopia, or no road: line when the
  org schema's default_road is sinopia, can be broken down this way, and only
  once: a second breakdown of the same milestone is a CONFLICT.
  Give the breakdown as JSON, in a file or on stdin. breakdown_key names this
  breakdown. When an answer is lost, snp resends the same request id on its own,
  and snpd replays the first result with replayed: true instead of writing
  twice. A new run of snp is a new request: on a milestone that already has a
  breakdown it gets a CONFLICT ending breakdown_key=<key>. Compare that key with
  the breakdown_key in the milestone's yaml, read with snp mem get
  projects/<slug>/<iniciativa>/milestones/<MID>.yaml, and with yours: if it is
  yours, your breakdown already landed, and the yaml lists its issues.

Arguments:
  --file <path|->: the JSON to send, from a local file of UTF-8 text, or - for
  stdin. It is one object with these keys, and no others:
  - project: the project slug, such as sinopia.
  - iniciativa: the iniciativa, such as demo.
  - milestone: the milestone id, such as M1.
  - breakdown_key: <iniciativa>/<MID>/<name>, where <name> is 1 to 64 letters,
    digits, . _ or -.
  - issues: 1 to 40 issues, each {"ref","title","body","blocked_by"}. ref is 1
    to 32 characters of a-z, 0-9 and -, and is how blocked_by names another
    issue of this breakdown. title is one line of 1 to 200 bytes. body is up to
    16 KiB.
  snpd checks the rest of the shape and names the ref and the rule that failed.
  snp checks that the file is JSON with those keys and that it encodes to at
  most 1 MiB, and sends nothing otherwise.

Flags:
  --file <path|->
      <path|->: the breakdown JSON, or - for stdin

Output:
  snpd's JSON on one line: {"commit","parent","revs","replayed","issues"}.
  issues lists {"ref","number","id"} for each issue created, so a later issue
  can name an earlier one by its number. replayed is true when this request had
  already landed.
  After the write, snp moves your projection, if you have one, to the new
  commit. If that fails, stderr gets "snp: PROJECTION_STALE: …" and snp still
  exits 0, because the write landed; run snp pull.
  On an error, stderr gets one line, snp: <CODE>: <message>, and stdout gets
  snpd's error JSON when snpd sent one.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value, the wrong number of arguments. snp sent
      nothing; stderr says which, so fix the command and run it again.
  NOT_FOUND (exit 4)
      There is no roadmap for the iniciativa, or no ### M{n} block in it for
      that milestone. Nothing was written. stderr names the document; check the
      project, iniciativa and milestone.
  INVALID (exit 5)
      The file is missing, not UTF-8 text, not JSON, or has a key not listed
      above; the breakdown encodes to more than 1 MiB; snpd names a ref and the
      rule it breaks; or SNP_ORG is missing, or SNP_URL is not an http or https
      URL with a host. Nothing was written; stderr says which, so fix it and run
      it again. snpd also answers INVALID when the roadmap's road: line is
      neither road: vault nor road: sinopia, when there are two roadmaps for the
      iniciativa or the block has no valid Status line, and when the milestone's
      yaml or the project's seq file cannot be parsed; for those two, stderr
      names the document and says to repair it with snpd admin put, which your
      admin runs.
  ILLEGAL (exit 8)
      The road is not sinopia: the roadmap's road: line, or the org schema's
      default_road when the roadmap has no road: line. Or the milestone is not
      planned. Nothing was written; stderr names the road or state it found.
  CONFLICT (exit 3)
      The milestone already has a breakdown. Nothing was written. stderr names
      breakdown_key=<key>, the key of the existing one; read the milestone's
      yaml with snp mem get projects/<slug>/<iniciativa>/milestones/<MID>.yaml
      to compare it with yours.
  INSUFFICIENT_SCOPE (exit 7)
      The token lacks the work scope. Ask your admin for a token that has it.
      Nothing was written.
  UNAVAILABLE (exit 15)
      The org has no schema, or one this snpd can't read; stderr says which. Ask
      your admin to run snpd admin schema install.
  DIRTY_PROJECTION (exit 9)
      Your projection (~/.snp/projection/<org>) has local changes, so snp sent
      nothing. The message lists them. Copy out anything you want to keep,
      delete the projection with the command in the message, run snp pull, then
      write again.
  UNAUTHENTICATED (exit 6)
      SNP_TOKEN is malformed, unknown, revoked or expired, so snpd refused it.
      Ask your admin for a new token. When SNP_TOKEN is not set, snp sent none:
      set it, or add the snpd network secret to the cloud environment.
  FORBIDDEN (exit 7)
      SNP_TOKEN was issued for another org than SNP_ORG. Set SNP_ORG to the
      token's org, or use a token issued for this one.
  TRANSPORT (exit 10)
      No answer from snpd, or an answer from something else, such as a proxy,
      after 2 retries, so the write may have landed. Run snp mem get on each id
      to check before you write again.
  BUSY (exit 11)
      Another write held the org's lock too long, or the write hit snpd's
      deadline, after 2 retries. Nothing changed; run it again in a few seconds.
  CLIENT_TOO_OLD (exit 12)
      This snp is too old: snpd accepts only a newer one. Install the version
      the message names, then run it again.
  REQUEST_ID_REUSED (exit 13)
      A different write already used this write's request id on snpd. snp makes
      a new id on each run, so run the command again.
  LIMIT (exit 14)
      The token went over its request rate, or this session reached its cap on
      writes. Nothing was written. The message says which: wait and run it
      again, or for the session cap, check for a loop before you set a new
      SNP_SESSION.
  INTERNAL (exit 1)
      The request failed inside snpd in a way it does not expect, after 2
      retries. Run snp mem get on each id to check whether the write landed
      before you run it again; if it fails again, send the message to your
      admin.

Environment:
  SNP_URL: snpd's base URL, such as https://host:8484. A url= line in
  ~/.snp/config works too, and SNP_URL wins over it. With neither, snp uses
  https://snp.tesselabs.com.
  SNP_ORG: your org. Required; an org= line in ~/.snp/config works too, and
  SNP_ORG wins over it.
  SNP_TOKEN: your token, read only from the environment, never from argv or a
  file. Leave it unset in a cloud session whose network secret adds it; snp then
  sends no token of its own.
  SNP_SESSION, or else CLAUDE_CODE_REMOTE_SESSION_ID, or else
  CLAUDE_CODE_SESSION_ID: optional. A claude.ai cloud session sets the second,
  which stays the same across /clear and worker restarts. snp sends the first
  one set with every request, and a write records it on the commit as the
  Snp-Session trailer; a write without a valid one is recorded as undeclared.
  When SNP_ORG is missing, or SNP_URL is not an http or https URL with a host,
  snp exits 5 (INVALID) and sends nothing.

Examples:
  # Create the issues of a planned milestone from a file:
  snp work create_breakdown --file ./breakdown.json
  # Or from stdin:
  cat breakdown.json | snp work create_breakdown --file -
  # A milestone that already has a breakdown: exit 3, and stderr says
  #   snp: CONFLICT: … breakdown_key=demo/M1/first; …
  snp work create_breakdown --file ./breakdown.json
```

## snp work query

```text
Usage:
  snp work query --project <slug> [--iniciativa i] [--milestone M] [--status s]
      [--limit n] [--cursor c] [--md]

Description:
  List the issues of one project as they are at snpd's latest commit, sorted by
  number, one page at a time. You can keep only the issues of one iniciativa, of
  one milestone, or in one status.
  An issue's evidence is a link someone wrote down. Nobody checked it, so --md
  marks it "unverified".
  To read one issue's state and the states it can move to, use snp state get.

Arguments:
  --project <slug>: the project to list. Required; without it snp sends nothing
  and exits 2.
  --iniciativa <name>: keep only the issues of this iniciativa.
  --milestone <MID>: keep only the issues of this milestone, such as M1. It
  needs --iniciativa: a milestone alone is INVALID.
  --status <state>: keep only the issues in this state, such as ready.
  --limit <n>: issues per page, 1 to 200. Leave it out, or pass 0, for 50. Any
  other number is INVALID.
  --cursor <c>: the next value of the previous page, to get the page after it.
  Pass the same filters.
  --md: print markdown instead of JSON.
  There are no positional arguments.

Flags:
  --cursor string
      the next value of the previous page
  --iniciativa string
      only this iniciativa
  --limit int
      issues per page, 1 to 200 (default 50)
  --md
      print markdown
  --milestone string
      only this milestone (needs --iniciativa)
  --project <slug>
      <slug>: the project to list (required)
  --status string
      only this status

Output:
  - Default: snpd's JSON on one line, {"items","next","skipped"}. Each item is
    {"number", "id", "title", "status", "iniciativa", "milestone", "blocked_by",
    "evidence", "evidence_verified", "rev"}. blocked_by lists issue numbers and
    is [] when there are none; items is [] when nothing matches, never null.
    evidence is left out when there is none. next is left out on the last page;
    skipped is left out when every issue file parsed, and otherwise lists the
    ids of the files snpd could not read.
  - --md: one table per milestone, headed "## <iniciativa> <MID>", with the
    columns # | Title | Status | Blocked by | Evidence. Evidence reads "<link>
    (unverified)", and "—" stands for none. A | in a cell is written \|. When
    nothing matches, it prints "No issues." When skipped is not empty, a line
    starts "Warning:" and names the ids; repair those files with snpd admin put.
    When there is a next page, the last line reads "More: --cursor <next>"; pass
    that cursor with the same filters.
  On an error, stderr gets one line, snp: <CODE>: <message>, and stdout gets
  snpd's error JSON when snpd sent one.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value, the wrong number of arguments. snp sent
      nothing; stderr says which, so fix the command and run it again.
  INVALID (exit 5)
      --limit is outside 1 to 200, --milestone has no --iniciativa, the cursor
      is not an issue number, --status is not a state of the org's issue
      machine, a filter is not a valid name, or SNP_ORG is missing, or SNP_URL
      is not an http or https URL with a host. stderr says which; fix it, or
      drop --cursor to start from the first page.
  INSUFFICIENT_SCOPE (exit 7)
      The token lacks the work scope. Ask your admin for a token that has it.
  UNAVAILABLE (exit 15)
      The org has no schema, or one this snpd can't read; stderr says which. Ask
      your admin to run snpd admin schema install.
  UNAUTHENTICATED (exit 6)
      SNP_TOKEN is malformed, unknown, revoked or expired, so snpd refused it.
      Ask your admin for a new token. When SNP_TOKEN is not set, snp sent none:
      set it, or add the snpd network secret to the cloud environment.
  FORBIDDEN (exit 7)
      SNP_TOKEN was issued for another org than SNP_ORG. Set SNP_ORG to the
      token's org, or use a token issued for this one.
  TRANSPORT (exit 10)
      No answer from snpd, or an answer from something else, such as a proxy,
      after 2 retries. Check the network and that SNP_URL points at snpd, then
      run it again.
  CLIENT_TOO_OLD (exit 12)
      This snp is too old: snpd accepts only a newer one. Install the version
      the message names, then run it again.
  LIMIT (exit 14)
      The token went over its request rate. Wait the seconds the message names,
      then run it again.
  INTERNAL (exit 1)
      The request failed inside snpd in a way it does not expect, after 2
      retries. Run it again; if it fails again, send the message to your admin.

Environment:
  SNP_URL: snpd's base URL, such as https://host:8484. A url= line in
  ~/.snp/config works too, and SNP_URL wins over it. With neither, snp uses
  https://snp.tesselabs.com.
  SNP_ORG: your org. Required; an org= line in ~/.snp/config works too, and
  SNP_ORG wins over it.
  SNP_TOKEN: your token, read only from the environment, never from argv or a
  file. Leave it unset in a cloud session whose network secret adds it; snp then
  sends no token of its own.
  SNP_SESSION, or else CLAUDE_CODE_REMOTE_SESSION_ID, or else
  CLAUDE_CODE_SESSION_ID: optional. A claude.ai cloud session sets the second,
  which stays the same across /clear and worker restarts. snp sends the first
  one set with every request, and a write records it on the commit as the
  Snp-Session trailer; a write without a valid one is recorded as undeclared.
  When SNP_ORG is missing, or SNP_URL is not an http or https URL with a host,
  snp exits 5 (INVALID) and sends nothing.

Examples:
  # List a project's issues as tables:
  snp work query --project sinopia --md
  # One milestone's ready issues:
  snp work query --project sinopia --iniciativa x --milestone M1 --status ready
  # The page after a "More: --cursor 50" line:
  snp work query --project sinopia --md --cursor 50
  # A milestone without its iniciativa: exit 5, and stderr says
  #   snp: INVALID: …
  snp work query --project sinopia --milestone M1
```

## snp work transition

```text
Usage:
  snp work transition <n> --project <slug> --expect-status <state> --to <state>
      [--evidence url] [--reason text]

Description:
  Move one issue to another state, as one commit. snpd rewrites the Status line
  of projects/<slug>/issues/<n>.md and records the edge in the commit. Only an
  edge of the org schema's issue machine is allowed; snp work query shows the
  issue's state now.
  --expect-status is the state you believe the issue is in. If another writer
  moved it first, nothing is written and you get a CONFLICT naming the state it
  is in now.
  In the default schema, two edges have guards. Before ready to in-progress,
  every issue in the issue's blocked_by must be done or canceled; one that is
  not, or that snpd cannot read, refuses the move. For in-review to done,
  --evidence is required: a GitHub pull request URL. snpd stores it with
  evidence_verified: false and never opens it. Giving --evidence on any other
  edge is INVALID.
  When an answer is lost, snp resends the same request id on its own, and snpd
  replays the first result with replayed: true instead of writing twice. To move
  a milestone, use snp state transition.

Arguments:
  <n>: the issue number, such as 12.
  --project <slug>: the project the issue belongs to, such as sinopia.
  --expect-status <state>: the state the issue is in now.
  --to <state>: the state to move it to.
  --evidence <url>: for in-review to done, the pull request URL,
  https://github.com/<owner>/<repo>/pull/<number>.
  --reason <text>: why, recorded on the commit as the Snp-Reason trailer. It
  must be 1 to 200 characters of letters, digits, spaces and . _ : @ / -. Any
  other reason is INVALID and nothing is written; unlike snp mem put, this verb
  does not drop it.

Flags:
  --evidence <url>
      <url>: the pull request, for in-review to done
  --expect-status <state>
      <state>: the state the issue is in now (required)
  --project <slug>
      <slug>: the issue's project (required)
  --reason <text>
      <text>: why, kept in the commit
  --to <state>
      <state>: the state to move it to (required)

Output:
  snpd's JSON on one line:
  {"commit","parent","revs","replayed","overwrote","entity","from","to"}.
  overwrote maps each id to the rev it replaced, or null when the document is
  new. replayed is true when this request had already landed.
  After the write, snp moves your projection, if you have one, to the new
  commit. If that fails, stderr gets "snp: PROJECTION_STALE: …" and snp still
  exits 0, because the write landed; run snp pull.
  On an error, stderr gets one line, snp: <CODE>: <message>, and stdout gets
  snpd's error JSON when snpd sent one.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value, the wrong number of arguments. snp sent
      nothing; stderr says which, so fix the command and run it again.
  NOT_FOUND (exit 4)
      The project has no issue with that number. Nothing was written.
  INVALID (exit 5)
      The number or project is not a valid issue ref (nothing is sent); a state
      or --evidence is malformed; --reason is not 1 to 200 characters of
      letters, digits, spaces and . _ : @ / -; --evidence is given on an edge
      other than in-review to done; or a milestone ref was given. snpd also
      answers INVALID when the issue file or one of its blockers cannot be
      parsed; stderr names the document and says to repair it with snpd admin
      put, which your admin runs. Nothing was written.
  ILLEGAL (exit 8)
      The issue machine has no edge from --expect-status to --to: snpd's JSON
      then carries allowed, the states that can follow, and the message lists
      them. Or a blocker is not done or canceled, and the message names each. Or
      in-review to done has no --evidence. Nothing was written.
  CONFLICT (exit 3)
      The issue is not in --expect-status. Nothing was written. stderr reads
      <entity> is <current>, expected <from>; read the state again with snp work
      query and retry with it.
  INSUFFICIENT_SCOPE (exit 7)
      The token lacks the work scope. Ask your admin for a token that has it.
      Nothing was written.
  UNAVAILABLE (exit 15)
      The org has no schema, or one this snpd can't read; stderr says which. Ask
      your admin to run snpd admin schema install.
  DIRTY_PROJECTION (exit 9)
      Your projection (~/.snp/projection/<org>) has local changes, so snp sent
      nothing. The message lists them. Copy out anything you want to keep,
      delete the projection with the command in the message, run snp pull, then
      write again.
  UNAUTHENTICATED (exit 6)
      SNP_TOKEN is malformed, unknown, revoked or expired, so snpd refused it.
      Ask your admin for a new token. When SNP_TOKEN is not set, snp sent none:
      set it, or add the snpd network secret to the cloud environment.
  FORBIDDEN (exit 7)
      SNP_TOKEN was issued for another org than SNP_ORG. Set SNP_ORG to the
      token's org, or use a token issued for this one.
  TRANSPORT (exit 10)
      No answer from snpd, or an answer from something else, such as a proxy,
      after 2 retries, so the write may have landed. Run snp mem get on each id
      to check before you write again.
  BUSY (exit 11)
      Another write held the org's lock too long, or the write hit snpd's
      deadline, after 2 retries. Nothing changed; run it again in a few seconds.
  CLIENT_TOO_OLD (exit 12)
      This snp is too old: snpd accepts only a newer one. Install the version
      the message names, then run it again.
  REQUEST_ID_REUSED (exit 13)
      A different write already used this write's request id on snpd. snp makes
      a new id on each run, so run the command again.
  LIMIT (exit 14)
      The token went over its request rate, or this session reached its cap on
      writes. Nothing was written. The message says which: wait and run it
      again, or for the session cap, check for a loop before you set a new
      SNP_SESSION.
  INTERNAL (exit 1)
      The request failed inside snpd in a way it does not expect, after 2
      retries. Run snp mem get on each id to check whether the write landed
      before you run it again; if it fails again, send the message to your
      admin.

Environment:
  SNP_URL: snpd's base URL, such as https://host:8484. A url= line in
  ~/.snp/config works too, and SNP_URL wins over it. With neither, snp uses
  https://snp.tesselabs.com.
  SNP_ORG: your org. Required; an org= line in ~/.snp/config works too, and
  SNP_ORG wins over it.
  SNP_TOKEN: your token, read only from the environment, never from argv or a
  file. Leave it unset in a cloud session whose network secret adds it; snp then
  sends no token of its own.
  SNP_SESSION, or else CLAUDE_CODE_REMOTE_SESSION_ID, or else
  CLAUDE_CODE_SESSION_ID: optional. A claude.ai cloud session sets the second,
  which stays the same across /clear and worker restarts. snp sends the first
  one set with every request, and a write records it on the commit as the
  Snp-Session trailer; a write without a valid one is recorded as undeclared.
  When SNP_ORG is missing, or SNP_URL is not an http or https URL with a host,
  snp exits 5 (INVALID) and sends nothing.

Examples:
  # Start an issue:
  snp work transition 12 --project sinopia \
    --expect-status ready --to in-progress
  # Close it, with its pull request as evidence:
  snp work transition 12 --project sinopia --expect-status in-review \
    --to done --evidence https://github.com/acme/app/pull/34
  # The issue is no longer ready: exit 3, and stderr says
  #   snp: CONFLICT: issue:sinopia/12 is in-progress, expected ready; …
  snp work transition 12 --project sinopia \
    --expect-status ready --to in-review
```

## snp pull

```text
Usage:
  snp pull [--md]

Description:
  Bring your projection up to snpd's latest commit. Your projection is a
  read-only copy of your org's documents at ~/.snp/projection/<org>: files are
  0444 and folders 0555, and its git data is in ~/.snp/git/<org>.git, outside
  the copy.
  The first pull creates it. Each pull fetches over git from SNP_URL with your
  token, then fast-forwards the copy; it never merges, because the copy holds no
  work of yours. If the copy's folder is missing or empty, pull rebuilds it.
  Before anything else, pull checks the copy for local changes and refuses with
  DIRTY_PROJECTION if it finds any. Edit nothing there: write with snp mem put,
  set or append, which also move an existing projection to the new commit.
  To print where the projection is, use snp root --path.

Arguments:
  None besides the flags.

Flags:
  --md
      print markdown

Output:
  - Default: one line of JSON, {"path","commit","previous","mirror_lag_s"}. path
    is the projection's folder, commit the commit it is at now, and previous the
    commit it was at before, or null on the first pull. mirror_lag_s is how far
    snpd's backup mirror is behind, in seconds (the largest across snpd's orgs;
    0 when snpd runs without a mirror), or null when snpd did not say.
  - --md: "Projection at <path> moved from <previous> to <commit>." (previous is
    "nothing" on the first pull), or "Projection at <path> is already at
    <commit>." when nothing changed; then "Mirror lag: <n> s", or "Mirror lag:
    unknown".
  When the mirror is more than 5 minutes behind, stderr also gets a line that
  starts with "snp: warning: the backup mirror is". Your writes are safe on
  snpd, and pull still exits 0.
  On an error, stderr gets one line, snp: <CODE>: <message>, and stdout gets
  snpd's error JSON when snpd sent one.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value, the wrong number of arguments. snp sent
      nothing; stderr says which, so fix the command and run it again.
  INVALID (exit 5)
      SNP_ORG is missing, or SNP_URL is not an http or https URL with a host, or
      HOME is not an absolute path. snp sent nothing; stderr says which, so fix
      it and run it again.
  DIRTY_PROJECTION (exit 9)
      Your projection has local changes (a changed, deleted or added file), or
      snp could not check it for them. snp sent nothing. The message lists up to
      20 changed files and the command that deletes the projection. Copy the
      files you want to keep outside the projection, run that command, run snp
      pull to rebuild the projection, then write them back with snp mem put. snp
      mem put is refused while the projection has changes, so do not run it
      first.
  UNAUTHENTICATED (exit 6)
      SNP_TOKEN is malformed, unknown, revoked or expired, so snpd refused it.
      Ask your admin for a new token. When SNP_TOKEN is not set, snp sent none:
      set it, or add the snpd network secret to the cloud environment.
  FORBIDDEN (exit 7)
      SNP_TOKEN was issued for another org than SNP_ORG. Set SNP_ORG to the
      token's org, or use a token issued for this one.
  TRANSPORT (exit 10)
      No answer from snpd after 2 retries, an answer from something else, such
      as a proxy, or the git fetch failed. A token without the mem scope fails
      at the fetch too. The projection did not move. Check the network, SNP_URL
      and the token's scopes, then run snp pull again.
  CLIENT_TOO_OLD (exit 12)
      This snp is too old: snpd accepts only a newer one. Install the version
      the message names, then run it again.
  LIMIT (exit 14)
      The token went over its request rate. Wait the seconds the message names,
      then run it again.
  INTERNAL (exit 1)
      git could not update the projection (a full disk, a permission error, or a
      history that no longer fast-forwards after snpd was restored), or snpd
      failed in a way it does not expect, after 2 retries. For a local failure
      the message names the command that deletes the projection and its git
      data: run it, then snp pull to rebuild. Otherwise run it again; if it
      fails again, send the message to your admin.

Environment:
  SNP_URL: snpd's base URL, such as https://host:8484. A url= line in
  ~/.snp/config works too, and SNP_URL wins over it. With neither, snp uses
  https://snp.tesselabs.com.
  SNP_ORG: your org. Required; an org= line in ~/.snp/config works too, and
  SNP_ORG wins over it.
  SNP_TOKEN: your token, read only from the environment, never from argv or a
  file. Leave it unset in a cloud session whose network secret adds it; snp then
  sends no token of its own.
  SNP_SESSION, or else CLAUDE_CODE_REMOTE_SESSION_ID, or else
  CLAUDE_CODE_SESSION_ID: optional. A claude.ai cloud session sets the second,
  which stays the same across /clear and worker restarts. snp sends the first
  one set with every request, and a write records it on the commit as the
  Snp-Session trailer; a write without a valid one is recorded as undeclared.
  When SNP_ORG is missing, or SNP_URL is not an http or https URL with a host,
  snp exits 5 (INVALID) and sends nothing.

Examples:
  snp pull
  snp pull --md
  # Then read the documents as plain files:
  grep -rn "release plan" "$(snp root --path)/projects/demo/"
  # A projection with local changes: exit 9, and stderr says
  #   snp: DIRTY_PROJECTION: the projection at /home/you/.snp/projection/acme …
  snp pull
```

## snp root

```text
Usage:
  snp root [--json] [--md] [--path]

Description:
  Print where your projection is and how it compares with snpd: the commit it is
  at, snpd's latest commit, snpd's build, the oldest snp that snpd accepts, and
  the backup mirror's lag. When projection_commit and server_head differ, run
  snp pull. Your projection is a read-only copy of your org's documents at
  ~/.snp/projection/<org>: files are 0444 and folders 0555, and its git data is
  in ~/.snp/git/<org>.git, outside the copy.
  --path prints only the projection's absolute path and asks snpd nothing. It
  prints the path even before the first snp pull, when nothing is there yet. It
  still reads SNP_URL and SNP_ORG, and still refuses a projection with local
  changes.

Arguments:
  None besides the flags.

Flags:
  --json
      print JSON (the default)
  --md
      print markdown
  --path
      print only the absolute path

Output:
  - Default, or --json: one line of JSON, {"path", "projection_commit",
    "server_head", "server_build", "min_client_version", "mirror_lag_s"}.
    projection_commit is null before the first snp pull. server_head is snpd's
    latest commit for your org. server_build is snpd's version, and
    min_client_version the oldest snp it accepts. mirror_lag_s is how far snpd's
    backup mirror is behind, in seconds (the largest across snpd's orgs; 0 when
    snpd runs without a mirror), or null when snpd did not say.
  - --md: one "<field>: <value>" line per field, in that order, with none for a
    null.
  - --path: only the absolute path, on one line. It wins over --json and --md.
  When the backup mirror has diverged, or is more than 5 minutes behind, stderr
  also gets a line that starts with "snp: warning:" and says what to tell your
  admin. Your writes are safe on snpd, and root still exits 0.
  On an error, stderr gets one line, snp: <CODE>: <message>, and stdout gets
  snpd's error JSON when snpd sent one.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value, the wrong number of arguments. snp sent
      nothing; stderr says which, so fix the command and run it again.
  INVALID (exit 5)
      SNP_ORG is missing, or SNP_URL is not an http or https URL with a host, or
      HOME is not an absolute path. snp sent nothing; stderr says which, so fix
      it and run it again.
  DIRTY_PROJECTION (exit 9)
      Your projection has local changes (a changed, deleted or added file), or
      snp could not check it for them. snp sent nothing. The message lists up to
      20 changed files and the command that deletes the projection. Copy the
      files you want to keep outside the projection, run that command, run snp
      pull to rebuild the projection, then write them back with snp mem put. snp
      mem put is refused while the projection has changes, so do not run it
      first.
  UNAUTHENTICATED (exit 6)
      SNP_TOKEN is malformed, unknown, revoked or expired, so snpd refused it.
      Ask your admin for a new token. When SNP_TOKEN is not set, snp sent none:
      set it, or add the snpd network secret to the cloud environment.
  FORBIDDEN (exit 7)
      SNP_TOKEN was issued for another org than SNP_ORG. Set SNP_ORG to the
      token's org, or use a token issued for this one.
  TRANSPORT (exit 10)
      No answer from snpd after 2 retries, an answer from something else, such
      as a proxy, or git could not read snpd's latest commit. A token without
      the mem scope fails at that last step too. Check the network, SNP_URL and
      the token's scopes, then run it again.
  CLIENT_TOO_OLD (exit 12)
      This snp is too old: snpd accepts only a newer one. Install the version
      the message names, then run it again.
  LIMIT (exit 14)
      The token went over its request rate. Wait the seconds the message names,
      then run it again.
  INTERNAL (exit 1)
      snp could not lock the projection, or snpd failed in a way it does not
      expect, after 2 retries. Run it again; if it fails again, follow the
      message, or send it to your admin.

Environment:
  SNP_URL: snpd's base URL, such as https://host:8484. A url= line in
  ~/.snp/config works too, and SNP_URL wins over it. With neither, snp uses
  https://snp.tesselabs.com.
  SNP_ORG: your org. Required; an org= line in ~/.snp/config works too, and
  SNP_ORG wins over it.
  SNP_TOKEN: your token, read only from the environment, never from argv or a
  file. Leave it unset in a cloud session whose network secret adds it; snp then
  sends no token of its own.
  SNP_SESSION, or else CLAUDE_CODE_REMOTE_SESSION_ID, or else
  CLAUDE_CODE_SESSION_ID: optional. A claude.ai cloud session sets the second,
  which stays the same across /clear and worker restarts. snp sends the first
  one set with every request, and a write records it on the commit as the
  Snp-Session trailer; a write without a valid one is recorded as undeclared.
  When SNP_ORG is missing, or SNP_URL is not an http or https URL with a host,
  snp exits 5 (INVALID) and sends nothing.

Examples:
  snp root
  # Projection commit, snpd's latest commit, build and mirror lag, one per line:
  snp root --md
  # Where the projection is, for scripts and file tools:
  ls "$(snp root --path)/projects/"
  # With an SNP_URL that has no scheme: exit 5, and stderr says
  #   snp: INVALID: SNP_URL is not a valid URL (it must start with http:// …
  SNP_URL=localhost:8484 snp root
```

## snp skills install

```text
Usage:
  snp skills install [--dir DIR | --user [--no-auto-update | --auto-update]]
      [--check]
      (--user writes ~/.claude and ~/.agents/skills; if both exist, Claude Code
      uses the user skill and the project agent)

Description:
  Write the snp-* skills and agents built into this snp into a project's
  .claude/ folder, or with --user into ~/.claude/. With --user the skills also
  go to ~/.agents/skills/, where other agents read them; the agents go only to
  ~/.claude/agents/. Only snp-*/ skill folders and .claude/agents/snp-*.md files
  are touched; every other skill and agent stays as it is. Each entry is
  replaced whole, and snp-* entries this snp no longer ships are removed.
  The target is --dir, or CLAUDE_PROJECT_DIR when you pass neither --dir nor
  --user. In a project, git must ignore every path snp writes or replaces: add
  .claude/skills/snp-* and .claude/agents/snp-* to .gitignore first. In
  ~/.claude and ~/.agents there is no gitignore check, and snp removes only the
  stale entries listed in ~/.snp/skills-installed, which each --user install
  rewrites. A --user install also removes ~/.snp/no-skills, which snp skills
  uninstall --user leaves, so snp update installs the skills again.
  snp refuses a symlink at .claude, .claude/skills, .claude/agents or an snp-*
  entry, and any path that resolves outside the target. With --user, the same
  problem under ~/.agents (a symlink at ~/.agents, ~/.agents/skills or an snp-*
  skill there) skips ~/.agents/skills instead: ~/.claude is still installed,
  stderr gets "snp: skipped <path>: <message>", and the exit code is 0. If a
  write fails, snp puts the old entries back and installs nothing. Two runs on
  one target take turns.
  --check writes nothing in .claude or .agents. It compares the installed snp-*
  files with this snp's, file by file, and exits 1 when they differ.
  With --user on a release build, snp also adds a SessionStart hook to
  ~/.claude/settings.json: the entry <snp path> update --auto, with a timeout of
  10 seconds, so snp updates itself when a session starts. It replaces an older
  snp entry and changes nothing else in the file, though key order and spacing
  may change. A build that is not a release (dev, or a version with +dev) adds
  no hook, and stderr says "snp: auto-update is off for unreleased builds". So
  does a ~/.snp that is not safe (a symlink, not yours, or writable by others):
  the skills are installed, and stderr says what is wrong and how to fix it:
  replace a symlink with a real directory, sudo chown a ~/.snp that is not
  yours, or chmod 700 one others can write. Then run snp skills install --user.
  With --no-auto-update or --auto-update, such a ~/.snp fails the run instead,
  and the message names the same fix and the command to run again with that
  flag.
  --no-auto-update removes the hook, or says on stderr why it could not ("snp:
  could not remove the SessionStart hook from <file>: <reason>"), and creates
  ~/.snp/no-auto-update; while that file is there, a plain --user install leaves
  the hook out. --auto-update deletes the file and adds the hook again. Both
  need --user. If settings.json is not valid JSON, --no-auto-update leaves it as
  it is and installs the skills.

Arguments:
  None besides the flags. Pass --dir or --user, not both. CLAUDE_PROJECT_DIR
  does not count as --dir, so --user wins over it.

Flags:
  --auto-update
      with --user: turn the SessionStart hook back on
  --check
      compare the installed snp-* set with this binary's
  --dir string
      project directory (default $CLAUDE_PROJECT_DIR)
  --no-auto-update
      with --user: remove the SessionStart hook and keep it off
  --user
      install into ~/.claude and ~/.agents/skills

Output:
  - Default: one line of JSON, {"dir","written","removed"}. dir is the target
    folder, absolute. written lists each entry written, such as
    .claude/skills/snp-think, and removed each stale entry removed. When
    ~/.agents was skipped, "skipped" lists why.
  - --check: one line of JSON, {"dir","match","missing","extra","changed"}.
    missing, extra and changed list files under dir, such as
    .claude/skills/snp-think/SKILL.md; match is true when all three are empty. A
    skipped ~/.agents is not compared, and "skipped" lists why. On a mismatch,
    stderr also gets "snp: <n> missing, <n> extra, <n> changed in <dir>; run
    <command>", where <command> is snp skills install with the --dir or --user
    you passed.
  On an error, stderr gets one line, snp: <CODE>: <message>.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value such as --user=maybe, an argument
      besides the flags, --dir together with --user, --no-auto-update together
      with --auto-update, either of them without --user, or either with --check.
      Nothing was written; stderr says which, so fix the command and run it
      again.
  mismatch (exit 1)
      Only with --check: the installed snp-* files differ from this snp's.
      stdout lists them; run snp skills install, with the --dir or --user you
      passed, to bring them in line.
  INVALID (exit 5)
      There is no target (no --dir, --user or CLAUDE_PROJECT_DIR), the target is
      not a folder, a path snp would write is not gitignored or the target is
      not in a git work tree, a symlink or a path outside the target is in the
      way, HOME is not an absolute path, with --no-auto-update or --auto-update
      ~/.snp is not a safe directory, with --user ~/.claude/settings.json is not
      valid JSON or is a symlink that does not resolve or leads to a file you do
      not own (pass --no-auto-update to skip the hook), the path of snp has a
      character outside A-Za-z0-9/._+- (path_unsafe in the message; the shell
      would split it), or a write failed. Nothing was installed unless the
      message says otherwise; stderr says which, so fix it and run it again.

Environment:
  CLAUDE_PROJECT_DIR: the project folder, used when you pass neither --dir nor
  --user. Claude Code sets it for hooks, so a SessionStart hook can run snp
  skills install with no flags.
  HOME: --user installs into $HOME/.claude and $HOME/.agents/skills. It must be
  an absolute path.
  snp skills install reads no SNP_* variables and never calls snpd.

Examples:
  # Into this project's .claude/, once the snp-* entries are gitignored:
  printf '.claude/skills/snp-*\n.claude/agents/snp-*\n' >> .gitignore
  snp skills install --dir .
  # Into ~/.claude and ~/.agents/skills, for every project:
  snp skills install --user
  # The same, without the SessionStart hook that updates snp (and keep it off):
  snp skills install --user --no-auto-update
  # Is the installed set this snp's? Exit 0 if so, 1 if not:
  snp skills install --user --check
  # --dir and --user together: exit 2, and stderr says
  #   snp: skills install takes --dir or --user, not both
  snp skills install --dir . --user
```

## snp skills uninstall

```text
Usage:
  snp skills uninstall --user

Description:
  Remove what snp skills install --user put in ~/.claude and ~/.agents/skills,
  and the SessionStart hook that updates snp. Only --user is supported.
  snp removes only the entries listed in ~/.snp/skills-installed, and only those
  named snp-* in .claude/skills, .claude/agents or .agents/skills. A snp-* skill
  or agent you wrote yourself, and every other skill and agent, stays as it is.
  Any other line in that file is ignored and dropped from that file: snp deletes
  the file when nothing is kept, and when ~/.agents was skipped it keeps only
  the ~/.agents lines.
  The hook is the SessionStart entry whose command is a path to snp followed by
  update --auto. snp removes it wherever the path points, keeps every other hook
  and group, and drops a group it empties. It changes nothing else in
  ~/.claude/settings.json, though key order and spacing may change. If
  settings.json is not valid JSON, or is a symlink that does not resolve, snp
  leaves it as it is, removes the skills, says nothing on stderr, and
  hook_removed is false. If snp cannot read it (it is a folder, say), if it
  links to a file you do not own that holds an snp hook, or if
  hooks.SessionStart is not in the form Claude Code uses, snp leaves it as it
  is, removes the skills and says on stderr "snp: could not remove the
  SessionStart hook from <file>: <reason>". If it holds one, remove the entry by
  hand.
  ~/.snp/no-auto-update stays: it records your choice for a later install. snp
  refuses a symlink at ~/.claude, ~/.claude/skills, ~/.claude/agents or a listed
  entry under ~/.claude, before it removes anything. The same problem under
  ~/.agents skips ~/.agents/skills instead: those lines stay in the file, stderr
  gets "snp: skipped <path>: <message>", and the exit code is 0. Two runs take
  turns, and a second run finds nothing to remove.
  snp also leaves ~/.snp/no-skills. While it is there, snp update still installs
  a newer snp, but no skills and no hook, and says so on stderr. snp skills
  install --user removes it.

Arguments:
  None besides --user, which is required.

Flags:
  --user
      remove from ~/.claude and ~/.agents/skills

Output:
  One line of JSON, {"dir","removed","hook_removed"}. dir is your home folder,
  absolute. removed lists each entry removed, such as .claude/skills/snp-think,
  sorted; it is empty when nothing was installed. hook_removed is true when snp
  removed the SessionStart hook. When ~/.agents was skipped, "skipped" lists
  why.
  On an error, stderr gets one line, snp: <CODE>: <message>.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag, a bad flag value such as --user=maybe, an argument, or no
      --user. Nothing was removed; stderr says which, so fix the command and run
      it again.
  INVALID (exit 5)
      HOME is not an absolute path, a symlink is in the way under ~/.claude,
      ~/.snp is not a folder, or a remove or the settings.json write failed.
      When a remove fails, the message names the entries already removed;
      ~/.snp/skills-installed and the hook stay as they were, and a run again
      finishes the job. When removing or rewriting ~/.snp/skills-installed, or
      writing ~/.snp/no-skills, fails, the entries are already gone: the message
      names them, and the hook stays as it was. When the settings.json write
      fails, the skills are already gone. Any other error removes nothing.
      stderr says which, so fix it and run it again.

Environment:
  HOME: the folder that holds .claude, .agents and .snp. It must be an absolute
  path.
  snp skills uninstall reads no SNP_* variables and never calls snpd.

Examples:
  # Remove the user-level skills, agents and hook:
  snp skills uninstall --user
  # Without --user: exit 2, and stderr says
  #   snp: skills uninstall needs --user
  snp skills uninstall
```

## snp update

```text
Usage:
  snp update [--check | --auto]

Description:
  With no flag, snp update installs the newest snp. It finds the target the way
  --check does (below). When the target is not newer than this snp, it stops
  there with the result. Otherwise it downloads the target's release files from
  SNP_RELEASE_BASE, checks their signatures with the keys built into this snp,
  runs the new binary's snp version, and renames the new binary over this one.
  The old binary stays beside it as snp.prev. Then it runs the new binary's snp
  skills install --user, unless ~/.snp/no-skills exists (snp skills uninstall
  --user leaves it): then it installs no skills, keeps the skills' version in
  ~/.snp/update-state, and stderr says "snp: the skills were not installed,
  because <path> exists; run snp skills install --user to install them". It
  writes ~/.snp/update-state (mode 0600): the release and key sequence numbers
  it has seen, the version and sha256 of snp.prev, the skills' version and the
  outcome. A failed lookup, download, check or install is recorded there too, as
  last. Not recorded: a failure before the lookup, such as unsupported_platform,
  state_unsafe, state_corrupt, another update running, a HOME that is not an
  absolute path, or ~/.snp or its lock that can't be created, an SNP_URL that is
  not a URL, and a failed write of ~/.snp/update-stamp. It writes
  ~/.snp/update-stamp when the result is updated, current, held or skipped, or
  no_target because snpd names no target.
  Before the lookup, snp update reinstalls the skills when ~/.snp/update-state
  exists and does not record this snp's skills: it names another snp, or none,
  as after a failed first update. It skips that while ~/.snp/no-skills exists.
  It also removes the temp files a stopped update left beside the binary. The
  directory of the binary (symlinks resolved) and the binary must be owned by
  you or by root and not writable by group or others, and you must be able to
  write to the directory. snp never uses sudo. Up to the rename, the run gets
  120 seconds; the new snp's skills install then gets up to 30 more.
  --check finds the newest snp and compares it with this one. The target is the
  latest_client in the /healthz of snpd: SNP_URL, else url= in ~/.snp/config,
  else https://snp.tesselabs.com. When there is a target to install, snp sends
  one HEAD to check that its release files are published.
  --check downloads nothing and installs nothing. It creates ~/.snp (mode 0700)
  when it is missing. The only file it writes is ~/.snp/update-stamp. It writes
  ~/.snp/update-stamp only when the result is current, held or skipped, or
  no_target because snpd names no target; never on available, and not when the
  tag is out but its release files are not yet. It also creates
  ~/.snp/update.lock, which keeps two updates from running at once, and never
  writes ~/.snp/update-state.
  --auto is for a Claude Code SessionStart hook. Run alone, as the hook runs it,
  --auto runs the same steps as snp update, prints nothing or one line on
  stdout, writes nothing to stderr, and always exits 0. Anything else on its
  command line is a usage error, reported as one on stderr: an argument or an
  unknown flag exits 2, and --check or --rollback beside --auto exits 5. It
  stops before anything else, writing nothing, when this snp is not a release
  build (a plain vX.Y.Z such as v0.3.1) or the platform is not linux/amd64 or
  linux/arm64. It also says nothing when another update holds the lock; when
  ~/.snp/update-stamp is under 24 hours old, ~/.snp/update-too-old is absent and
  CLAUDE_CODE_REMOTE is unset, in which case it asks snpd nothing; when the
  result is current, held, skipped or no_target, even if ~/.snp/update-stamp
  cannot be written; and when the target's release files are not published yet.
  A CLIENT_TOO_OLD answer from snpd to any snp command leaves
  ~/.snp/update-too-old, so the next --auto checks even with a fresh stamp. An
  update removes it, and so does a run of snp update or snp update --auto that
  finds this snp current and at or above snpd's min_client_version. Up to the
  rename, --auto gets 8 seconds, and snpd's /healthz gets 2 of them, retries
  included; the new snp's skills install then gets up to 1.5 more.
  --rollback puts back snp.prev, the snp the last update replaced. It checks
  that ~/.snp/update-state records snp.prev's sha256, that snp.prev is a regular
  file owned by you or by root, and that its sha256 is the recorded one; then it
  renames snp.prev over this snp. It records this snp's version as skip in
  ~/.snp/update-state, so snp update and --auto leave that version alone and
  move again at the first newer one, and clears the record of snp.prev, so a
  second --rollback is refused. It downloads nothing and does not reinstall the
  skills: the next snp update or --auto does, because the skills still name the
  version rolled back from, unless ~/.snp/no-skills exists.

Arguments:
  None besides the flags. Pass no flag to install, or one of --check, --auto or
  --rollback.

Flags:
  --auto
      hook mode, run alone: at most one line of output, always exit 0
  --check
      report whether a newer snp is available; install nothing
  --rollback
      restore snp.prev, the snp the last update replaced

Output:
  With no flag: one line of JSON, {"result","running","target"}. running is the
  version that was running when the command started. target is the version
  found, or null when there is none. result is one of:
  - updated: target is installed, with its skills unless ~/.snp/no-skills
    exists.
  - current, held, skipped: as for --check below; nothing was downloaded.
  - no_target: snpd names no latest_client, or the tag is out but its release
    files are not yet. Nothing was installed.
  With --check: one line of JSON, {"result","running","target","last"}. running
  is this snp's version, as snp version prints it. target is the version found,
  or null when there is none. last is what ~/.snp/update-state records of the
  last update, {"at","from","to","code"}: when it ran, the snp it ran from, the
  version it aimed at, or empty when it failed before finding one, and updated
  or the code it failed with; or null when nothing is recorded, as before a
  first update or in a new cloud session. result is one of:
  - available: target is newer than this snp, and its release files are
    published.
  - current: this snp is target or newer.
  - held: this snp is target or newer, and snpd's own build is a newer release,
    so snpd holds its clients at target.
  - skipped: target is newer, and ~/.snp/update-state marks it to skip.
  - no_target: there is nothing to install: snpd names no latest_client, or the
    tag is out but its release files are not yet.
  When last records a failure and result is available, the line is still
  printed, the exit is 5 and stderr says snp: INVALID: the last update failed:
  <code>. The last update attempt failed, and target is still to install. With
  any other result, last is shown and the exit is 0.
  With --auto: nothing, or one line. After an update, the line is the hook's
  JSON, {"hookSpecificOutput":{...}}, whose hookEventName is "SessionStart",
  reloadSkills is true and additionalContext is "snp: updated <running> ->
  <target>", in that order. target is the version the signed release names. On a
  failure, the line is snp: update failed: <code>; run snp update, with code one
  of the names below, or internal for any other error. The exit is 0 either way.
  With --rollback: one line of JSON, {"result":"rolled_back","running","from"}.
  running is the version now in place, from is the version rolled back from.
  Without --auto, on an error, stderr gets one line, snp: <CODE>: <message>. A
  failed update, or a check that itself fails, says update failed: <code>, where
  code is one of the names below. A check that exits 5 because last records a
  failure says the last update failed: <code>.

Errors and exit codes:
  usage error (exit 2)
      An unknown flag or an argument besides the flags. snp sent nothing; stderr
      says which, so fix the command and run it again.
  INVALID (exit 5)
      More than one flag (pass one of --check, --auto or --rollback); another
      update is running, still after 10 seconds; SNP_URL (or url= in
      ~/.snp/config) is not an http or https URL with a host; HOME is not an
      absolute path; or update failed: <code>, with code unsupported_platform
      (snp updates itself only on linux/amd64 and linux/arm64), state_unsafe
      (~/.snp is a symlink, not yours, or writable by group or others; or
      update-state, update.lock or update-stamp in it is a symlink),
      state_corrupt (~/.snp/update-state is not valid), or server_bad (snpd's
      /healthz has a latest_client that is not a release tag such as v0.3.1, or
      no valid min_client_version). Fix what the message names, then run it
      again.
  INVALID (exit 5)
      the last update failed: <code>: with --check, the result is available and
      ~/.snp/update-state records that the last update failed with <code>, one
      of the update failure codes on this page. Run snp update to retry it and
      see the error.
  INVALID (exit 5)
      update failed: install_dir_unsafe: the binary or its directory is owned by
      another user, or writable by group or others; or the running binary is
      snp.prev, the copy an update keeps. Fix the owner or run chmod go-w on
      them, or run snp update from snp itself, or re-run the install paste, then
      run it again. update failed: not_writable: you can't write to the binary's
      directory, such as a root-owned /usr/local/bin. Re-run the install paste,
      which installs to ~/.local/bin, then run it again. snp installed nothing
      in either case.
  INVALID (exit 5)
      update failed: prev_unavailable (~/.snp/update-state is missing or records
      no snp.prev, as in a new cloud session, or records this same snp, as when
      another update ran meanwhile), prev_missing (snp.prev is missing, not a
      regular file, or owned by a user other than you or root), or prev_changed
      (snp.prev is not the file the last update kept, or you can't run it). snp
      changed nothing. When prev_changed is because you can't run snp.prev, make
      it executable for you (chmod u+x on it, when it is yours) and run snp
      update --rollback again. Otherwise there is no older snp here to go back
      to: stay on this snp and report the problem; the install paste installs
      the version snpd names, not an older one.
  INVALID (exit 5)
      update failed: <code>, where the release failed a check and snp installed
      nothing: too_large (a file is over its cap: 64 KiB for snp-release and
      snp-keys, 1 KiB for a signature, 128 MiB for the binary), keys_sig
      (snp-keys is not signed by the recovery key built into snp, or does not
      parse), keys_seq (snp-keys is older than one already seen), manifest_sig
      (snp-release is not signed by a trusted release key), manifest_bad
      (snp-release does not parse), version_mismatch (snp-release names another
      version than the target), below_floor (the release is not newer than this
      snp, or is below snpd's min_client_version), replay (its sequence number
      is not above the last one installed), sha_mismatch (the binary is not the
      one snp-release names), or smoke_failed (the new binary did not print its
      version within 2 seconds). Don't work around it: report the code and the
      target, and stay on this snp.
  INVALID (exit 5)
      update failed: rename_failed: snp could not move the new binary (or, with
      --rollback, snp.prev) into place; this snp is still there and still runs.
      Check the space and permissions of its directory, then run it again.
      update failed: skills_failed: the new snp is installed, but its skills
      install failed or took over 30 seconds (1.5 with --auto). Run snp skills
      install --user to see why and to finish.
  TRANSPORT (exit 10)
      update failed: server_unreachable (no answer from snpd, an error from it,
      or an answer from something else, such as a proxy, within 10 seconds, or 2
      with --auto, retries included), fetch_failed (SNP_RELEASE_BASE is not an
      https URL, did not answer, a HEAD or download failed or answered other
      than 200, or it redirected to a URL that is not https), or deadline (the
      update took over 120 seconds, or 8 with --auto; snp installed nothing).
      Check the network, SNP_URL and SNP_RELEASE_BASE, then run it again.
  INTERNAL (exit 1)
      snp could not create ~/.snp, take its lock, read or write
      ~/.snp/update-state (for example, a file owned by root) or write the
      stamp. The message says why; fix it, then run it again. snp <version> is
      installed, but ~/.snp/update-state was not written: the new snp is in
      place and runs, but update-state does not record it; fix what the message
      names. With --rollback: snp <version> is back in place, but
      ~/.snp/update-state was not written, so it still records the update you
      undid and no skip. snp update then refuses the version you left with
      update failed: replay, and, from the first session start after
      ~/.snp/update-stamp is a day old or missing, or from the next one when
      CLAUDE_CODE_REMOTE is set, --auto prints snp: update failed: replay; run
      snp update at every session start, until a newer release is out, which
      they install as usual. Fix what the message names; to stop the line
      sooner, re-run the install paste, which installs the version snpd names,
      that is the version you left while it is still the newest. update failed:
      internal is a bug in snp: report it with what you ran.

Environment:
  SNP_URL: snpd's base URL. A url= line in ~/.snp/config works too, and SNP_URL
  wins over it. With neither, snp update uses https://snp.tesselabs.com.
  SNP_RELEASE_BASE: where snp releases are published, as an https URL. Default
  https://github.com/Tesselabs/snp/releases.
  HOME: snp keeps its update files in $HOME/.snp, and the new snp installs its
  skills under $HOME.
  CLAUDE_CODE_REMOTE: set in a claude.ai cloud session. With it set, --auto asks
  snpd at every session start, even when ~/.snp/update-stamp is under a day old:
  the environment keeps the stamp its first session wrote.
  snp update needs no SNP_ORG or SNP_TOKEN, and sends no token.

Examples:
  snp update
  # Only the result, for a script:
  snp update --check | jq -r .result
  # The SessionStart hook: one line or nothing, exit 0
  snp update --auto
  # Go back to the snp the last update replaced
  snp update --rollback
  # --check and --auto together: exit 5, and stderr says
  #   snp: INVALID: pass one of --check, --auto or --rollback
  snp update --check --auto
```

## snp usage report

```text
Usage:
  snp usage report --since <RFC 3339 time> [--projects-dir <dir>]

Description:
  Count the tokens, human turns and AskUserQuestion prompts that Claude Code
  recorded on this machine, grouped by the skill that was active when each was
  recorded. Give --since, an RFC 3339 time such as 2026-11-07T00:00:00Z.
  snp usage report reads the main transcripts, <dir>/<project>/<session>.jsonl,
  and the subagent transcripts of each session,
  <session>/subagents/agent-*.jsonl. Each usage recorded at or after --since
  goes to the skill active when it was recorded; before any skill is active it
  goes to (none).
  snp usage report makes no network call, never calls snpd, reads no SNP_*
  variable and writes nothing.

Arguments:
  None.

Flags:
  --projects-dir DIR
      read the transcripts under DIR (default $CLAUDE_CONFIG_DIR/projects, else
      $HOME/.claude/projects)
  --since TIME
      count usage at or after TIME, an RFC 3339 time such as
      2026-11-07T00:00:00Z (required)

Output:
  One line of JSON on stdout, then exit 0:
  - since: --since as UTC, in RFC 3339.
  - by_skill: for each skill, input_tokens (an object with total: the input,
    cache creation and cache read tokens) and output_tokens, and main, the same
    two fields counted from main transcripts only.
  - total: every input and output token counted, subagent transcripts included.
  - total_main: the same count for main transcripts only.
  - asks: for each skill, the AskUserQuestion prompts, as answered and
    unanswered.
  - human_turns: the messages a person wrote in main transcripts.
  - sessions: the sessions that have main usage, sorted.
  - files: how many transcript files snp read. A transcript whose last change
    (mtime) is before --since is not read and does not count, even if it holds
    lines from after --since, so old mtimes alone can give the no-usage warning
    below.
  - skipped_lines: how many lines were not JSON or could not be read.
  Two warnings go to stderr and do not change the exit:
  - snp: skipped <k> transcript lines that are not JSON
  - snp: no usage found under <dir>; is this the session's HOME?
  The second one comes when files is 0, by_skill is empty or total_main is 0.
  The JSON line is still printed and the exit is still 0.

Errors and exit codes:
  usage error (exit 2)
      A missing, empty or non-RFC 3339 --since: stderr says snp: INVALID:
      --since must be an RFC 3339 time. Also an unknown flag, a flag with no
      value, or any argument (usage report takes no arguments). snp read
      nothing; fix the command and run it again.
  NOT_FOUND (exit 4)
      The projects directory is missing or is not a directory: stderr says snp:
      NOT_FOUND: no Claude Code transcripts at <dir>. Check HOME or
      CLAUDE_CONFIG_DIR, or pass --projects-dir.
  INTERNAL (exit 1)
      snp cannot find your home directory (and CLAUDE_CONFIG_DIR is unset), or
      cannot read the projects directory. The message says why; fix it, then run
      it again.

Environment:
  CLAUDE_CONFIG_DIR: Claude Code's config directory. When it is set and not
  empty, the default projects directory is $CLAUDE_CONFIG_DIR/projects.
  HOME: with CLAUDE_CONFIG_DIR unset, the default projects directory is
  $HOME/.claude/projects.
  --projects-dir wins over both.
  snp usage report reads no SNP_* variables and never calls snpd.

Examples:
  # Everything since the start of the day:
  SINCE=2026-11-07T00:00:00Z
  snp usage report --since "$SINCE"
  # Read copied transcripts instead of this machine's:
  DIR=/tmp/transcripts
  snp usage report --since "$SINCE" --projects-dir "$DIR"
  # A date without a time is a usage error (exit 2):
  snp usage report --since 2026-11-07
```

## snp review-cap cap

```text
Usage:
  snp review-cap cap --stage S --lens L (--envelope FILE --finding F{k} |
      --severity V --evidence -) [--repo DIR]

Description:
  Cap one review finding's severity to the stage the review is for, and print
  the verdict. A blocker stays a blocker only when its lens bites at --stage or
  an earlier stage, and its Evidence holds a receipt: a command block, or a
  citation that checks out in --repo (without --repo, any citation, unchecked).
  The finding comes in one of two forms. With --severity V --evidence -, the
  severity is V and the Evidence is stdin. With --envelope FILE --finding F{k},
  the finding runs from the first line of FILE that starts with "#### F{k} — "
  up to the next line that starts with "#### ", or to the end of FILE:
  - The finding's first line that starts with "- **Severity:** " holds the
    severity: the text after it up to the first whitespace (so nothing, when
    another space follows), in any case, with *, _ and the backtick stripped
    from its front, and *, _, the backtick, . , ; and : from its end.
  - The Evidence starts after "- **Evidence:** " on that line, or on the next
    line when the line is exactly "- **Evidence:**". It runs up to the finding's
    next line that starts with "- **".
  The stages run solo-internal, external-beta, public. Each lens bites at the
  first stage where its findings harm someone:
  - solo-internal: architecture, ops and devils-advocate.
  - external-beta: product, marketing, ux, security, legal, data-privacy and
    cost.
  The first rule that matches decides, in this order:
  - A finding below blocker keeps its severity: cap_reason none.
  - A blocker whose lens bites at a later stage than --stage becomes high:
    later-stage.
  - A blocker whose Evidence holds a command block stays a blocker: none. A
    command block from devils-advocate does not count.
  - A blocker whose Evidence holds no citation becomes high: no-receipt.
  - A blocker with a citation and no --repo stays a blocker: receipt-unverified.
    With --repo, it stays a blocker (none) when one citation checks out, and
    becomes high (no-receipt) when none does.
  A receipt is a command block or a citation. A command block is a fence line
  (three backticks) followed at once by a line that starts with "$ "; either
  line may start with whitespace. A citation is a word of the Evidence (split at
  spaces, tabs and newlines) shaped path:line[-line][,line[-line]]…, such as
  lib/a.dart:12-20. Before the shape is checked, the characters " ' ( [ and the
  backtick come off the word's front, and " ' ) ] . , ; : and the backtick off
  its end; nothing else is stripped. A citation checks out when path names a
  readable regular file inside --repo and its first range falls within the
  file's lines. A path that starts with / or ~, a path with a .. segment, and a
  path with a symlink anywhere on it, the file itself included, never check out.
  A path with no / must name exactly one regular file of that name in the repo,
  outside every folder named .git, .worktrees or node_modules, the --repo folder
  itself included; the name, read as a shell glob, must also match itself, so a
  name such as z[z] never checks out.
  snp review-cap reads only stdin, the envelope, the --repo tree and the files
  its citations name. It makes no network call, runs no command and writes
  nothing: Evidence is matched as text, never run.

Arguments:
  None.

Flags:
  --envelope FILE
      read the finding from the review envelope FILE; use with --finding
  --evidence -
      read the Evidence from stdin, given as -; use with --severity
  --finding F{k}
      the finding's id F{k} in the envelope, such as F3
  --lens L
      the lens L that raised the finding: product, marketing, ux, architecture,
      security, legal, data-privacy, cost, ops or devils-advocate
  --repo DIR
      the repo DIR the citations point into; without it, no citation is checked
  --severity V
      the finding's severity V: blocker, high, medium or low
  --stage S
      the stage S the review is for: solo-internal, external-beta or public

Output:
  Three lines on stdout, and exit 0 for every verdict, a capped one too:
  severity: blocker|high|medium|low
  bites_at: solo-internal|external-beta
  cap_reason: none|later-stage|no-receipt|receipt-unverified
  severity is the finding's severity after the cap, and bites_at the stage its
  lens bites at. cap_reason says why:
  - none: the severity stands, because it is below blocker, or a receipt checks
    out.
  - later-stage: a blocker became high, because its lens bites at a later stage.
  - no-receipt: a blocker became high, because no receipt checks out.
  - receipt-unverified: a blocker stays one on a citation nothing checked,
    because there was no --repo.

Errors and exit codes:
  usage error (exit 2)
      A missing flag, an unknown argument (a flag cap does not take, or any
      other word), a flag with no value, a bad value, both Evidence forms or
      neither, an envelope snp cannot read, a finding it cannot find there, an
      envelope severity outside blocker, high, medium and low, or a --repo that
      is not a directory or is a symlink. stdout gets nothing, and stderr gets
      snp review-cap: <message> followed by the usage in parentheses. Fix the
      command and run it again.

Environment:
  None. snp review-cap reads no SNP_* variables and never calls snpd.

Examples:
  # An ops blocker whose citation checks out in the repo at .:
  echo 'see lib/a.dart:12-20' | snp review-cap cap --stage solo-internal \
    --lens ops --severity blocker --evidence - --repo .
  # Finding F3 of an envelope, a security blocker at solo-internal, prints
  #   severity: high / bites_at: external-beta / cap_reason: later-stage
  snp review-cap cap --stage solo-internal --lens security \
    --envelope review.md --finding F3
  # Both Evidence forms at once: exit 2, and stderr says
  #   snp review-cap: --envelope/--finding and --severity/--evidence are …
  snp review-cap cap --stage public --lens ops --envelope review.md \
    --finding F1 --severity high --evidence -
```

## snp review-cap carry

```text
Usage:
  snp review-cap carry --spec FILE [--prior FILE]

Description:
  Say whether a re-review carries the prior review's decisions forward, and
  which lenses to reconvene. Give the spec under review as --spec and, for a
  re-review, the prior review note as --prior. Both are absolute paths.
  snp review-cap carry reads these lines:
  - In the spec, under "## Stage": "- **stage:** S", with S one of
    solo-internal, external-beta and public, and "- **live:** text". The ledger
    sits under "## Review decisions": "- YYYY-MM-DD · {review name} · B{k}:
    text".
  - In the prior note, the frontmatter "stage:" line, and under "## Blockers"
    one line per blocker: "- **B{k} — title** (lens, lens) — … **Decision:**
    Accept|Defer|Reject …". "- (none)" and blank lines are skipped.
  A "## " heading inside a fence closes nothing. Under Stage and Review
  decisions a fenced line is an example, not data; under Blockers a fence line
  is outside the grammar (round 1).
  The stage comes from the spec's own "## Stage" (stage_source: spec) and carry
  never writes it back. Without a valid stage line there, the prior's
  frontmatter stage is used (prior-review). Without either, the stage is absent.
  live is the text of the spec's live line, read only when the spec's stage line
  is valid; otherwise it is absent.
  The run is round 1, a full panel with the devil's advocate, when:
  - there is no prior, or it cannot be read;
  - a Blockers line is outside the grammar, repeats an id or names a lens
    outside the roster and devils-advocate;
  - no ledger entry names the prior (its file name without .md) with a B{k} id,
    so no fix pass acted on it.
  Otherwise the run is carry. Each blocker lands in a list, in document order:
  - accepted: every blocker with Decision Accept.
  - decided: Defer and Reject, and an Accept whose id has a ledger entry for
    this prior.
  - undecided: an Accept with no ledger entry, and a decision outside Accept,
    Defer and Reject.
  The reconvene set is the lenses of the accepted and undecided blockers, in
  roster order and without devils-advocate: product, marketing, ux,
  architecture, security, legal, data-privacy, cost, ops. Each lens runs as the
  agent snp-{lens}.
  snp review-cap carry reads only the spec and the prior note. It makes no
  network call, runs no command and writes nothing.

Arguments:
  None.

Flags:
  --prior FILE
      read the prior review note at the absolute path FILE; without it, the run
      is round 1
  --spec FILE
      read the spec under review at the absolute path FILE

Output:
  Ten lines on stdout, then one message line per notice. Exit 0 for every
  outcome, a round 1 too:
  mode: round-1|carry
  prior_review: the prior's file name without .md, or absent
  stage: solo-internal|external-beta|public|absent
  stage_source: spec|prior-review|absent
  live: the spec's live text, or absent
  accepted: B-ids, comma-joined, or -
  decided: B-ids, comma-joined, or -
  undecided: B-ids, comma-joined, or -
  reconvene: lenses in roster order, comma-joined, or -
  reconvene_agents: snp-{lens} for each reconvene lens, or -
  The messages, in this order when they apply:
  - message: No prior review found — … (no --prior, or no readable file there).
    Round 1, with every list -.
  - message: Blockers line N of PRIOR is outside the grammar — … Round 1, with
    every list -.
  - message: No Review decisions entry names PRIOR — no fix pass acted on it; …
    Round 1; the lists stay filled and reconvene is -.
  - message: Stage inherited from PRIOR
  - message: No valid stage line in ## Stage of SPEC
  - message: N accepted blockers have no Review decisions entry — re-opening
    them
  - message: Decision token outside Accept/Defer/Reject — re-opening B-ids
  - message: Reconvene set empty — no lenses convened; readiness carried from
    PRIOR
  A round-1 message ends in: treating as round 1 (full panel, devil's advocate
  included).

Errors and exit codes:
  usage error (exit 2)
      A missing --spec, an unknown argument (a flag carry does not take, or any
      other word), a flag with no value, a --spec or --prior that is not an
      absolute path, or a --spec snp cannot read. stdout gets nothing, and
      stderr gets snp review-cap: <message> followed by the usage in
      parentheses. Fix the command and run it again.

Environment:
  None. snp review-cap reads no SNP_* variables and never calls snpd.

Examples:
  # Round 1: a spec with no prior review. Prints mode: round-1 and a
  # message that no prior review was found:
  SPEC=/work/spec.md
  snp review-cap carry --spec "$SPEC"
  # A re-review. Prints mode: carry when the spec's Review decisions
  # name review-1, with the lenses to reconvene:
  PRIOR=/work/review-1.md
  snp review-cap carry --spec "$SPEC" --prior "$PRIOR"
  # A relative path is a usage error (exit 2):
  snp review-cap carry --spec spec.md
```

## snp review-cap decide

```text
Usage:
  snp review-cap decide --kind blockers|conflicts|single --question FILE
      --labels FILE --result FILE [--prefill FILE]

Description:
  Read the answer to a review question and print one decision line for each
  blocker or conflict it covers. snp review-cap decide never decides for you: an
  answer it cannot read without doubt prints one fallback line, and you ask the
  question again. It never turns an unreadable answer into an Accept.
  Give it four inputs:
  - --kind: blockers, conflicts or single.
  - --question: the exact text of the question that was asked. For blockers and
    conflicts it holds "batch {i} of {b} (".
  - --labels: the option labels of the batch, one per line: "B{k} — title" for
    blockers and single, "C{k} — text" for conflicts. Blank lines are skipped.
    Single takes exactly one label.
  - --result: the raw text of the tool result for that question, as the tool
    returned it.
  Trailing newlines of the question and the result are dropped. The answer is
  the text in the result between the key "{question}"=" and the last " after it.
  If the result starts with "The user doesn't want to proceed with this tool
  use", the question was declined.
  For blockers and conflicts, the answer holds the checked labels first, joined
  by ", " or ",", then typed text. After the checked labels, a separator
  followed by nothing but spaces means there is no typed text. The typed text is
  in quotes only when a label came before it. A comma followed by Defer B,
  Reject B or none starts a new piece of typed text. A piece is one of:
  - a label, the same as a checked one;
  - none, which stands alone;
  - Defer B{k}: reason or Reject B{k}: reason, for blockers only. The reason
    loses its control bytes and its outer spaces. It cannot be empty or hold
    another Defer B{k}: or Reject B{k}:.
  For single, the answer is one of "Accept (must fix)", "Defer (note + proceed)"
  and "Reject (disagree, with reason)", or Defer B{k}: reason or Reject B{k}:
  reason for the one label.
  --prefill takes the output of snp review-cap converge, for blockers only. It
  does nothing unless it holds "line: " lines. When it does, the question must
  hold those lines joined by newlines, byte for byte, and each pre-filled
  blocker of its prefill: lines must appear in them as both of these texts:
  - "B{k} — "
  - " = AC-{id} ("
  A pre-filled blocker cannot also be one of the labels. You may name a
  pre-filled blocker in a Defer or Reject directive.
  snp review-cap decide reads only the files you name. It makes no network call,
  runs no command and writes nothing.

Arguments:
  None.

Flags:
  --kind string
      what the question asks: blockers, conflicts or single
  --labels FILE
      read the option labels of the batch from FILE, one per line
  --prefill FILE
      read the output of review-cap converge from FILE; blockers only
  --question FILE
      read the exact text of the question that was asked from FILE
  --result FILE
      read the raw text of the question's tool result from FILE

Output:
  Lines on stdout, and exit 0 for every outcome, a fallback too. The order is
  the order of the labels, then the pre-filled blockers in prefill: order. There
  are three forms; a run prints decision: lines, or resolution: lines, or a
  single fallback: line.
  decision: B{k} · **Decision:** Accept|Defer|Reject (…), for blockers and
  single:
  - blockers: a checked label is Reject (no reason given — batch {i} of {b}). A
    directive is Defer or Reject with its reason in parentheses. Any other label
    is Accept (batch {i} of {b}). A pre-filled blocker is Accept (covered by
    AC-{id}), unless a directive names it.
  - single: Accept, Defer (no reason given), Reject (no reason given), or the
    directive with its reason.
  resolution: C{k} · accepted|ask-single, for conflicts: accepted for a checked
  label, ask-single for any other.
  fallback: REASON, one line and nothing else. REASON is one of:
  - declined: the question was declined.
  - no-entry: the result holds no answer to the question, or the answer is
    empty.
  - unknown-piece: a piece of the answer is none of the forms above, or none
    comes with another piece.
  - out-of-batch: a directive names a blocker that is neither a label nor
    pre-filled.
  - duplicate: the answer names one blocker or conflict twice.
  - duplicate-labels: two labels share a line or an id.
  - prefill-line-missing: the question lacks the pre-filled lines byte for byte,
    or they lack a pre-filled blocker.

Errors and exit codes:
  usage error (exit 2)
      A missing flag, an unknown argument, a flag with no value (a word that
      starts with -- is taken as a value), a --kind outside blockers, conflicts
      and single, a file snp cannot read, --prefill with another kind, a labels
      line that is not a label of the kind, no labels, more than one label for
      single, a label that is also pre-filled, or a question for blockers or
      conflicts with no "batch {i} of {b} (". stdout gets nothing, and stderr
      gets snp review-cap: <message> followed by the usage in parentheses. Fix
      the command and run it again.

Environment:
  None. snp review-cap reads no SNP_* variables and never calls snpd.

Examples:
  # A batch of two blockers; the answer checked B2, so B2 is rejected:
  snp review-cap decide --kind blockers --question q.txt \
    --labels labels.txt --result result.txt
  #   decision: B1 · **Decision:** Accept (batch 1 of 1)
  #   decision: B2 · **Decision:** Reject (no reason given — batch 1 of 1)
  # A conflict batch:
  snp review-cap decide --kind conflicts --question q.txt \
    --labels labels.txt --result result.txt
  # A declined question prints fallback: declined. A missing --kind is a
  # usage error (exit 2):
  snp review-cap decide --question q.txt --labels labels.txt --result result.txt
```

## snp review-cap converge

```text
Usage:
  snp review-cap converge --spec FILE (--blockers FILE --map FILE | --question
      FILE)

Description:
  Check which blockers a listed acceptance criterion (AC) already covers, and
  render the block that tells you they are pre-accepted. snp review-cap converge
  is the only command that parses that block. snp review-cap decide --prefill
  checks it byte for byte and never parses it.
  Give it --spec and one of two forms:
  - --blockers and --map: check the entries and render the block.
  - --question: check the block already in a question text.
  In the spec, the ACs are the lines under "## Acceptance criteria (testable)",
  up to the next "## " heading. An AC starts at a line like "- [ ] **AC-{id}**
  text", with [ ], [x] or [X] and the bold marks optional. Its id looks like 7,
  A3 or 2b. Its text is that line plus the indented lines after it, up to the
  first line that is neither indented nor blank. Blank lines inside it are
  skipped. Its title is the first bold span of the line.
  The other inputs:
  - --blockers: one line per blocker of the round, "B{k} — title". Blank lines
    are skipped.
  - --map: one line per covered blocker, "B{k} AC-{id} excerpt". The excerpt is
    text copied from the AC. Blank lines are skipped.
  - --question: the question text. The block runs from the line "Pre-accepted
    under --converge (fix covered by a listed AC):" to the first line that
    starts with "To overturn one, type Reject B". Each line between is "B{k} —
    title = AC-{id} (excerpt)", where a trailing … on the excerpt is dropped.
  An entry passes when its AC is listed, its excerpt appears in the AC's text,
  the excerpt is more than the AC's title, and the blocker's title does not name
  the AC. From --map, a title that holds " = AC-" also fails, because the block
  could not be read back.
  snp review-cap converge reads only the files you name. It makes no network
  call, runs no command and writes nothing.

Arguments:
  None.

Flags:
  --blockers FILE
      read the blocker labels of the round from FILE; use with --map
  --map FILE
      read the covered blockers, as B{k} AC-{id} excerpt, from FILE; use with
      --blockers
  --question FILE
      read a question text that may carry the block from FILE
  --spec FILE
      read the spec whose ACs the entries name from FILE

Output:
  Lines on stdout, and exit 0 for every outcome. The prefill: and relay: lines
  come first, one per entry in input order; then counts:, line: and off:.
  prefill: B{k} = AC-{id}, for an entry that passes.
  relay: B{k} · cause, for an entry that fails. The cause is one of:
  - unresolved AC-{id}: the spec lists no such AC.
  - excerpt-mismatch AC-{id}: the excerpt is not in the AC's text, or, from
    --question, it is only whitespace once a trailing … is dropped.
  - excerpt-title-only AC-{id}: the excerpt is part of the AC's title and
    nothing more.
  - names AC-{id}: the blocker's title names the AC.
  - title-ambiguous: the title holds " = AC-" (--map only).
  - duplicate: the blocker appears twice (--question only).
  relay: line {n} · unparseable, for a block line that is not in the form above,
  such as one whose excerpt is an empty "()" (--question only). The header is
  line 1.
  counts: N {n} · P {p} · U {u}, for --map only: the blockers, the entries that
  pass and the rest.
  line: text, the block, as follows. With --map, the block prints only when P is
  at least 1 and U is at least 2: the header, one line "B{k} — title = AC-{id}
  (excerpt)" for each entry that passes, and the line that tells you how to
  overturn one. The title loses its commas and stops at 40 characters, the
  excerpt stops at 48, and each ends in … when cut. With --question, the block
  prints exactly as found.
  off: no-prefill|uncovered-below-2, for --map only, when no line: prints.
  off: no-block, for --question only, when the question holds no block.

Errors and exit codes:
  usage error (exit 2)
      A missing --spec, an unknown argument, a flag with no value, no --map and
      no --question, --question with --map or --blockers, --map without
      --blockers, a file snp cannot read, a --blockers line that is not a label
      or repeats an id, a --map line that is not in the form above, with an
      empty excerpt, with an id missing from --blockers, or that repeats an id.
      stdout gets nothing, and stderr gets snp review-cap: <message> followed by
      the usage in parentheses. Fix the command and run it again.

Environment:
  None. snp review-cap reads no SNP_* variables and never calls snpd.

Examples:
  # Check a map against the spec and render the block:
  snp review-cap converge --spec spec.md --blockers blockers.txt --map map.txt
  #   prefill: B1 = AC-3
  #   counts: N 3 · P 1 · U 2
  #   line: Pre-accepted under --converge (fix covered by a listed AC):
  #   line: B1 — title = AC-3 (excerpt)
  #   line: To overturn one, type Reject B1: {reason} or Defer B1: {reason} …
  # Check the block already in a question:
  snp review-cap converge --spec spec.md --question q.txt
  # A question with no block prints off: no-block. A missing --spec is a
  # usage error (exit 2):
  snp review-cap converge --question q.txt
```
