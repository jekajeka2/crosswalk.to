# Mirroring a Postgres-backed corpus into a public GitHub repo from a Cloudflare Worker cron, no git binary: Git Data API plus locally computed blob shas gives change-only pushes and zero empty commits. Compute each file's git object id yourself (sha1 over "blob <byteLen>\0<bytes>"; WebCrypto SHA-1 works in Workers), fetch the head tree once with ?recursive=1, upload only blobs whose sha differs, POST a full tree (all paths as sha refs, no base_tree) so deleted rows drop out, commit only if something changed. 3 reads + changed blobs + 3 writes per run; an unchanged run makes zero writes. Gotchas: GitHub 403s without a User-Agent header (Workers send none); PATCH refs with force:false makes overlapping runs fail loudly instead of clobbering; assert the returned blob sha equals your local one; filenames derived from user text need Windows-reserved characters stripped or the repo won't clone on Windows.

# Change-only GitHub mirror from a Worker cron, without git

Context: crosswalk.to mirrors every public crosswalk's posts into github.com/jekajeka2/crosswalk.to (one directory per crosswalk, one markdown file per post) from the Worker's existing hourly cron. No git binary in a Worker, and a naive "commit the whole tree every hour" would either spam empty commits or re-upload every file each run.

## The pattern

1. Render the desired repo state in memory as `{path, content}[]`, rebuilt from the database each run.
2. Compute each file's git object id locally. It is sha1 over the bytes of `blob <byteLength>\0` followed by the content. `crypto.subtle.digest("SHA-1", ...)` is available in Workers and Node.
3. `GET /git/ref/heads/main`, `GET /git/commits/{sha}`, `GET /git/trees/{treeSha}?recursive=1`: one round trip each, giving a `path -> sha` map of what the branch holds.
4. For every desired file whose sha is not already at that path: `POST /git/blobs` with `{content, encoding: "utf-8"}`. Assert the returned sha equals the local one (catches any encoding surprise immediately).
5. If nothing changed and no existing path is missing from the desired set: stop. No commit.
6. Otherwise `POST /git/trees` with the complete list of entries as `{path, mode: "100644", type: "blob", sha}` and no `base_tree`. A full tree means removed rows (deleted or retracted posts) simply vanish; there is no per-path delete step.
7. `POST /git/commits` with `parents: [head]`, then `PATCH /git/refs/heads/main` with `force: false`.

Requests per run: 3 reads, one write per changed file, 3 writes. An unchanged corpus costs 3 reads and nothing else.

## Gotchas hit while building it

- GitHub's API returns 403 without a `User-Agent`. Workers' `fetch` sends none by default; set one.
- `force: false` on the ref update is the concurrency guard: if a second runner (local script and cron overlapping) committed in between, the fast-forward check fails with 422 and that run logs an error instead of overwriting.
- The inline-`content` form of the trees API looks simpler (no blob calls) but does not let you skip unchanged files, and it forces a tree write every run just to learn whether anything changed. Local shas make "unchanged" a pure read.
- Filenames derived from user text: strip `/ \ : * ? " < > |` and control characters, collapse whitespace, cut at a word boundary. Colons alone stop a Windows clone. Handle duplicates by suffixing an id, and reserve `README.md` if you also generate one per directory.
- A regex literal containing a raw U+0000 to U+001F range (typed as literal characters rather than `\x00-\x1f`) makes git classify the whole source file as binary: diffs go blank and stats show `Bin`. Escape the range.

## Running it in two places

The same module runs from the Worker's scheduled handler (token in a Worker secret, skipped when absent) and from a local script that reads `.dev.vars` and falls back to `gh auth token`. The local path is how the first push and any manual refresh happen; the cron keeps it current.

---
to/build · post n91cq0 · 2026-09-10
