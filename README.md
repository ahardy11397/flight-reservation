# Flight Reservation CLI (`flight`)

A complete airline seat-reservation CLI built 100% in [Memelang](https://github.com/memelang) (`meme`), demonstrating graduate-level multi-module architecture, generic 2D-list grid algorithms, typed Optional JSON record persistence, deterministic seat allocation heuristics, and cross-compilation across all 6 backend targets (Python, Rust, C++20, Go, C11, and TypeScript).

---

## 1. Architecture & Design

The application is structured into modular Memelang units with a thin CLI router:

- `models.meme`: Defines domain records (`FlightData`) persisted via Memelang's built-in typed JSON serialization.
- `grid_ops.meme`: Implements core seat-grid algorithms on `List<List<String>>`:
  - 2D grid allocation and indexed access
  - Deterministic best-seat selection heuristic
  - Coordinate conversions between alphanumeric seat labels (`"12C"`) and zero-based `(row, col)` matrix indices
  - Orthogonal neighbor occupancy checks
  - Seat count aggregations
- `flight_ops.meme`: Encapsulates I/O and business logic:
  - Flight record persistence via `read json record FlightData ... or else nothing` and `write ... as json`
  - Safe error recovery on corrupt or missing data files (returning `nothing` rather than crashing)
  - 2D grid and FIFO waitlist serialization/deserialization
  - Fixed-point currency formatting (integer cents to half-away-from-zero floating-point dollar display)
- `main.meme`: The thin command-line argument dispatcher:
  - Validates `argument count` and flags
  - Routes subcommands (`add`, `book`, `cancel`, `manifest`, `seatmap`, `waitlist`, `promote`)
  - Ensures clean single-line error reporting with exit code `1`

---

## 2. CLI Usage

### Build the Native Executable

Compile `main.meme` and its dependencies directly into a native C11 binary:

```bash
uv run --project /mnt/ssd/projects/meme meme build main.meme -o flight_bin
```

### Commands

#### 1. Add Flight (`add`)
Creates a new flight with specified dimensions and seat price in cents.
```bash
./flight_bin add <flight-no> <rows> <seat-cols> <price-cents>
# Example:
./flight_bin add FL101 10 6 25000
```
- Seats are generated as rows `1..rows` across columns `A..Z` (for 6 columns: `A..F`).
- Duplicate flight numbers are rejected with exit code `1`.

#### 2. Book Seat (`book`)
Books a seat for a passenger. If seat is omitted, assigns the best available seat.
```bash
./flight_bin book <flight-no> <passenger-name> [seat]
# Automatic best available:
./flight_bin book FL101 "Alice Smith"
# Specific seat assignment:
./flight_bin book FL101 "Bob Jones" 2B
```
- Booking an already occupied seat fails.
- If the flight is full and no seat is specified, rejects booking and instructs the user to join the waitlist.

#### 3. Cancel Seat (`cancel`)
Frees a previously booked seat.
```bash
./flight_bin cancel <flight-no> <seat>
# Example:
./flight_bin cancel FL101 2B
```
- Cancelling an already free seat fails with exit code `1`.

#### 4. Display Seatmap (`seatmap`)
Prints a 2D matrix of the cabin showing free (`.`) and occupied (`X`) seats.
```bash
./flight_bin seatmap <flight-no>
# Example:
./flight_bin seatmap FL101
```
Sample output:
```text
=== Seatmap for Flight FL101 ===
    A B C D E F 
  1 X . X . . X 
  2 . X . . . . 
  3 . . . . . . 
```

#### 5. Manifest & Revenue (`manifest`)
Lists all booked passengers sorted by seat number, along with passenger count and total revenue.
```bash
./flight_bin manifest <flight-no>
# Example:
./flight_bin manifest FL101
```
Sample output:
```text
=== Manifest for Flight FL101 ===
1A: Alice Smith
1C: Charlie Brown
1F: David Miller
2B: Bob Jones
Total passengers: 4
Price per seat: $250.00
Total revenue: $1000.00
```

#### 6. Waitlist Queue (`waitlist`)
Appends a passenger to the flight's strict FIFO waitlist queue.
```bash
./flight_bin waitlist <flight-no> <passenger-name>
# Example:
./flight_bin waitlist FL101 "Eve Adams"
```

#### 7. Promote from Waitlist (`promote`)
Promotes the first passenger from the head of the FIFO waitlist into the best available free seat.
```bash
./flight_bin promote <flight-no>
# Example:
./flight_bin promote FL101
```
- Fails with exit code `1` if the flight is full or if the waitlist is empty.

---

## 3. Seat Selection Rules

When a passenger books without specifying a seat, the system applies a deterministic seat-selection heuristic using the 2D grid:

1. **Row Priority:** Front rows are preferred (Row 1 before Row 2, Row 2 before Row 3, etc.).
2. **Column Priority (6-column layout: A, B, C, D, E, F):**
   - **Windows:** Left window (`A`), then Right window (`F`)
   - **Aisles:** Left aisle (`C`), then Right aisle (`D`)
   - **Middles:** Left middle (`B`), then Right middle (`E`)
   - Order of allocation: `A -> F -> C -> D -> B -> E`
3. **Multi-Row Progression:**
   - 1st passenger gets `1A`
   - 2nd passenger gets `1F`
   - 3rd passenger gets `1C`
   - 4th passenger gets `1D`
   - 5th passenger gets `1B`
   - 6th passenger gets `1E`
   - 7th passenger moves to Row 2: `2A`, then `2F`, etc.

---

## 4. Data Persistence & Fault Tolerance

All flight data is persisted in `data/<flight-no>.json` via Memelang's strongly-typed `read json record` and `write ... as json` facilities:

- Money is strictly stored and calculated as 64-bit integer cents (`price_cents: Int`).
- Currency amounts are formatted as floating-point decimals with half-away-from-zero rounding only at print time (`format <val> with 2 decimals`).
- **Null Safety:** Missing files and corrupt JSON return `Optional<FlightData>` as `nothing`. The application handles `nothing` cleanly without exceptions or runtime tracebacks.

---

## 5. Test Suite

Tests are located in `tests/meme/*.meme` and run using the Memelang test runner (`meme test`).

### Running Tests Across Targets

All test suites verify 100% cleanly across all six backend code generators:

```bash
# Python target
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target python --provider heuristic

# Rust target
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target rust --provider heuristic

# C++20 target
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target cpp --provider heuristic

# Go target
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target go --provider heuristic

# C11 target
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target c --provider heuristic

# TypeScript target
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target ts --provider heuristic
```

### Test Coverage

- `01_flight_add_test.meme`: Flight creation, schema verification, duplicate flight rejection.
- `02_seat_selection_test.meme`: Best-seat determinism asserting exact seats (`1A`, `1F`, `1C`, `1D`, `1B`, `1E`, `2A`, `2F`).
- `03_booking_and_cancel_test.meme`: Specific seat assignment, taken-seat conflict rejection, cancel and rebooking, cancelling free seat rejection.
- `04_waitlist_and_promote_test.meme`: Full flight detection, waitlist offer, strict FIFO ordering, rejection of promotion on full flight, head promotion into newly freed seat, rejection of promotion on empty waitlist.
- `05_seatmap_and_manifest_test.meme`: Seatmap matrix verification, neighbor occupancy detection (left, right, front, back), manifest revenue calculation.
- `06_fault_tolerance_test.meme`: Non-existent flight graceful `nothing` handling, corrupt JSON data tolerance.
