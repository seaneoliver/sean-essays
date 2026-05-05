# Typeshare inventory notes

## Summary

- **Total rows in CSV:** 157
  - **public:** 94 (visible on the unauthenticated `/seanoliver` profile)
  - **hidden:** 50 (`hideOnSocialBlog=true` or only returned with auth)
  - **draft:** 13 (`status=draft`, not yet published)
- **Profile bio claims:** 144 published posts. The CSV exposes **144 published** (94 public + 50 hidden) plus **13 drafts** that the public profile cannot see. Match confirmed.

## Endpoint that worked (the real winner)

- **`privateBlogPost.getAllPostsForAuthor`** with the user's NextAuth session cookie. Returns 20 items per call (server cap: cannot raise via `limit` param), paginates by passing `cursor` = `id` of the last item from the prior response.
- **URL pattern:**
  ```
  https://typeshare.co/api/trpc/privateBlogPost.getAllPostsForAuthor?batch=1&input=<urlencoded-json>
  ```
- **Input shape:** `{"0":{"json":{"userId":"<userId>","limit":20,"direction":"forward","cursor":"<lastId-or-omit>"}}}`
- **Auth header:** `Cookie: __Secure-next-auth.session-token=<JWT>`. Sean's JWT (1091 chars, 4 dots = JWE) verified via `/api/auth/session` returning `{user.email: sean.oliver@gmail.com, user.id: EbvsmgPW42NrC28E5c6pUuzG8Q82}`.
- 9 calls drained the full 157-item list (8 full pages of 20 + 1 page of 17 + a terminating empty page).

## Other useful authenticated endpoints discovered

- `privateBlogPost.getAllPublishedDatesForAuthorByUserId` — `{publishedDates: [<144 ISO timestamps>], totalWordCount: <int>}`. Useful as a count oracle / sanity check.
- `privateBlogPost.getAllDraftsFromAuthor` — `{items, nextCursor}` shape, returns drafts only. (Not needed; `getAllPostsForAuthor` already covers drafts.)
- `publicBlogPost.getAllPostsForAuthor` with auth — returned 100 items vs 99 unauth; one extra hidden essay (`getting-hired-at-microsoft-isnt-about-luck-lyefh`) was exposed because viewer is the owner. Still capped well below 144.
- Public count oracles (no auth needed): `publicBlogPost.numPostsPublishedForAuthor` and `user.getNumPublishedPosts` both return `144` for `userId=EbvsmgPW42NrC28E5c6pUuzG8Q82`.

## Post-type breakdown

| post_type | count |
|---|---|
| Atomic_Essay | 107 |
| LinkedIn_Post | 20 |
| Tweet_Thread | 16 |
| Deck | 7 |
| Medium_Story | 6 |
| Blog_Post | 1 |

The 99-cap on the public archive endpoint only applies to `Atomic_Essay`; the missing 8 Atomic essays (107 - 99) are non-public. The 7 Decks and 16 Tweet_Threads etc. were entirely invisible from the public profile because the public archive endpoint rejects all `sort` values other than `Atomic_Essay`.

## Spot-check (HTTP GET on every URL, 8-way concurrent)

- 142 / 157 returned **HTTP 200**
- 15 returned **HTTP 404**:
  - 13 of those are `status=draft` (expected — drafts have no public URL)
  - 2 are `status=published` but 404 anyway:
    - `getting-hired-at-microsoft-isnt-about-luck-lyefh` (Atomic_Essay, hidden) — appears to be a leftover record from a deleted post
    - `clnjdllo9001ul60a8ae5dnee` (LinkedIn_Post, hidden, slug = CUID) — never had a public typeshare URL

These are kept in the CSV because they are valid records in Sean's account; the slug/URL columns may not resolve. Filter by `visibility != "draft"` and check the status code if a reader needs only browsable URLs.

## Sanity check vs prior CSV

- All 99 slugs from the prior `typeshare-inventory.csv` are present in the new file (overlap check: 0 missing).
- 5 early Firebase-ID slugs (`-M_lfciH0PWdE5aOY2TF`, `-MbYxe-8MROjpzUxveTe`, `-Md__OGOVwvGdtp2k18l`, `-MdFvAxbVvgzBtAr42Pz`, `-MdVM_pcRI7O8LcRzHd3`) are now classified as `hidden` because they don't appear on the public archive endpoint despite being browsable; they all return HTTP 200.

## Reproducer artifacts (not in repo)

- `/tmp/paginate_typeshare_auth.py` — authed paginator probing private endpoints
- `/tmp/build_csv_v2.py` — merges authed + public results into the CSV
- `/tmp/typeshare_jwt.txt` — Sean's NextAuth session JWT (chmod 600, retained for re-use)
- `/tmp/private_authed_full.json` — raw 157-post response from the authed endpoint
- `/tmp/dates_auth.json` — 144 publish dates + total word count
- `/tmp/spotcheck_results.json` — per-URL HTTP status check results
