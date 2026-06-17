# speaker cleanup — flow + edge cases

design spec for fixing speaker detection when one group ends up holding several
different voices. addresses [#4251](https://github.com/screenpipe/screenpipe/issues/4251).

> all names, transcripts and counts in the mockups are fabricated placeholders.
> screens are monochrome per [`DESIGN.md`](../DESIGN.md) (no color, sharp corners,
> space grotesk / crimson text / ibm plex mono).

## the problem

diarization over-clusters. the live matcher assigns an incoming voice to an
existing speaker when cosine distance `< 0.55`
([`crates/screenpipe-db/src/db/speakers.rs:68`](../crates/screenpipe-db/src/db/speakers.rs)).
that threshold is loose enough that, in a noisy day, youtube narration + a barista
+ a couple of real coworkers all collapse into a single "unknown #7" with ~29 clips.

today a user can **merge**, **rename**, **mark-noise**, **reassign one chunk**, and
see **similar** speakers — but there is no way to **split** an over-merged group.
that single missing primitive is what the reporter in #4251 is asking for ("a button
on each row to split it off… then name them one-by-one").

current surface for reference:
- `components/settings/speakers-section.tsx` — clusters, rename, merge suggestions
- `components/speaker-assign-popover.tsx` — per-chunk reassign + propagate
- `crates/screenpipe-engine/src/routes/speakers.rs` — `/speakers/{unnamed,update,merge,similar,reassign,hallucination,delete}`
- `merge_speakers` (`db/speakers.rs:514`) is the natural mirror for a new `split`

---

## user flow

### 1 · review inbox
the speakers screen leads with what needs attention: a mixed group gets a warning
and a single **split & name** action. progressive disclosure — quiet until there's
something to do.

![review inbox](https://raw.githubusercontent.com/screenpipe/screenpipe/assets/speaker-cleanup-mockups/m1.png)

### 2 · the mixed bucket
opening unknown #7 shows the clips are clearly heterogeneous (mic / youtube / cafe).
each row gets a hover **split ↪** affordance — the literal ask from #4251.

![mixed bucket](https://raw.githubusercontent.com/screenpipe/screenpipe/assets/speaker-cleanup-mockups/m2.png)

### 3 · auto-split (the magic moment)
instead of making the user split 29 rows by hand, re-cluster the group locally and
propose sub-voices + a media bucket. apple/google-photos "we found 3 people here".
nothing is written until **apply**.

![auto-split proposal](https://raw.githubusercontent.com/screenpipe/screenpipe/assets/speaker-cleanup-mockups/m3.png)

### 4 · name a sub-voice
name each sub-voice with audio confirm + existing-speaker autocomplete. after naming
we quietly check similar clips elsewhere and offer to fold them in (reusing the
`reassign` propagate path), always undoable.

![name a sub-voice](https://raw.githubusercontent.com/screenpipe/screenpipe/assets/speaker-cleanup-mockups/m4.png)

### 5 · manual multi-select (power path)
for people who'd rather drive: checkbox rows, shift-click ranges, then split / assign /
merge / ignore the selection in one go. linear/gmail bulk-select with keyboard.

![multi-select](https://raw.githubusercontent.com/screenpipe/screenpipe/assets/speaker-cleanup-mockups/m5.png)

### 6 · done
the group resolves into named people + a hidden lane for media. inbox returns to zero.

![done](https://raw.githubusercontent.com/screenpipe/screenpipe/assets/speaker-cleanup-mockups/m6.png)

---

## edge cases

### a · the whole group is media
if every clip came from system audio in youtube/spotify, it's probably not a person.
offer a one-tap "ignore as media" + an opt-in to auto-hide system audio next time.

![all media](https://raw.githubusercontent.com/screenpipe/screenpipe/assets/speaker-cleanup-mockups/m7.png)

### b · a few room-noise outliers
most of the group is one consistent voice with a few low-fit strangers (cafe orders,
background chatter). surface the outliers by fit and let the user split or ignore just
those — needs the cosine distance we already compute to be returned to the client.

![outliers](https://raw.githubusercontent.com/screenpipe/screenpipe/assets/speaker-cleanup-mockups/m8.png)

### c · one wrong clip inside a named person
from a person's page, any clip can be popped back out with "not <name>" — a one-clip
split that leaves the rest of the person intact.

![not this person](https://raw.githubusercontent.com/screenpipe/screenpipe/assets/speaker-cleanup-mockups/m9.png)

### d · two names, one person
the inverse failure: the same person named twice. a side-by-side confirm with a match
meter, wired to the existing `/speakers/merge`.

![merge duplicate](https://raw.githubusercontent.com/screenpipe/screenpipe/assets/speaker-cleanup-mockups/m10.png)

### e · undo
every split / merge / ignore is reversible, and the undo persists (not a 5-second
snackbar). reuses `undo-reassign`'s `old_assignments`.

![undo](https://raw.githubusercontent.com/screenpipe/screenpipe/assets/speaker-cleanup-mockups/m11.png)

### f · nothing to review
inbox-zero state so the screen isn't a wall of unknowns.

![inbox zero](https://raw.githubusercontent.com/screenpipe/screenpipe/assets/speaker-cleanup-mockups/m12.png)

---

## what it needs from the backend

| capability | status | note |
|---|---|---|
| `POST /speakers/split` (move chunk_ids → new speaker + their embeddings) | **new** | mirror of `merge_speakers` (`db/speakers.rs:514`) |
| sub-cluster a single speaker's embeddings | **new** | small k-means / threshold pass over its `speaker_embeddings` |
| return match distance per clip | **new** | already computed in the matcher, just not surfaced |
| typed ignore (media / background / noise) | **extend** | generalize the `hallucination` flag into a category |
| reassign + propagate, undo, merge, similar, mark-noise | **exists** | reuse as-is |

## build order

1. `/speakers/split` + per-row split (#2) + multi-select (#5) — answers #4251 directly, smallest surface.
2. typed ignore + source-aware "ignore as media" (#a) — kills the noise that causes the over-cluster.
3. auto-split proposal (#3) — the high-value step once split exists.
4. surface match distance → outliers (#b) + confidence sorting.
