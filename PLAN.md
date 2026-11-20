# PLAN — flight-reservation: a Memelang graduate-level demo

Build a complete airline seat-reservation CLI entirely in Memelang. Beta
demonstration program: what a CS graduate would build solo for a small
charter airline.

Language docs: /mnt/ssd/projects/meme/docs/WRITING_GUIDE.md and
docs/TUTORIAL.md (read both first). Compiler commands via
`uv run --project /mnt/ssd/projects/meme meme ...` from this repo.

## Ground rules (non-negotiable)

- Work ONLY in this repo. NO git commits.
- 100% Memelang; money in integer cents; floats only via
  `format <x> with 2 decimals` at print time.
- JSON persistence in `data/` via Optional record readers/writers; missing/
  corrupt file = clean Optional handling, never a traceback.
- Tests never assert exact wall-clock values (fixed epochs in test data).
- Everything must parse with `--provider heuristic`.
- Test suites run in the FOREGROUND only.

## Domain and CLI (thin argv-router main + logic modules; unknown command →
usage + exit 1; every error = one stderr line + exit 1)

    flight add <flight-no> <rows> <seat-cols> <price-cents>
               # e.g. flight add FL101 10 6 25000  -> seats A..F x rows 1..10
    flight book <flight-no> <passenger-name> [seat]   # seat like 12C; no seat
               # = best available (front row, window A/F preferred, then aisle
               # C/D, then middle B/E); fills deterministically
    flight cancel <flight-no> <seat>
    flight manifest <flight-no>       # occupied seats, sorted, + revenue
    flight seatmap <flight-no>        # 2D grid printout, . free, X taken
    flight waitlist <flight-no> <passenger-name>   # FIFO queue
    flight promote <flight-no>        # fill first free seat from waitlist head

Business rules: booking a taken seat fails; cancelling a free seat fails;
when a flight is full, `book` WITHOUT a seat offers waitlist (rejection
message tells the user); `promote` on a full flight fails; waitlist is
strict FIFO; revenue = booked seats × price, reported per flight.
Seat grid is a `List<List<String>>` (passenger name or empty marker) — the
2D-list feature is the centerpiece; use it for seatmap, best-seat search,
and neighbor checks.

## Tests — tests/meme/*.meme, `meme test`

At minimum: add (ok, duplicate flight rejected); best-seat determinism
(first booking gets 1A, second 1F, third 1C — assert exact seats); specific
seat booking + taken-seat rejection; cancel + rebook; full-flight waitlist
+ promote FIFO order; seatmap rendering matches bookings; manifest revenue
math; unknown flight; corrupt data tolerance. Fixed data, unique scratch
files per test, cleaned up.

## Acceptance ("done means")

1. `meme test` python target — 0 failed.
2. Same `--target rust cpp go c ts` — ALL SIX green.
3. `meme build main.meme -o flight_bin` — end-to-end demo (add flight, 8
   bookings incl. specific seats, cancel, waitlist, promote, seatmap,
   manifest) with transcript in your final summary.
4. README.md: CLI usage, seat-selection rules, data files, tests.
5. Final message: files, test count per target, demo transcript,
   deviations with reasons.
