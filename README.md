# ReCHor

**Journey planner for Swiss public transport** — a Java / JavaFX desktop application that,
for a given date, time and pair of stops, computes every optimal journey across the Swiss
network, then displays and exports them.

Semester project for **CS-108 — Practice of object-oriented programming** (EPFL), done in
a pair.

> `Java 22` · `JavaFX 21` · 44 classes · real CFF timetables (~258 MB of binary data)

---

## Features

- **Stop search with forgiving autocomplete**: accent- and case-insensitive, ranked by
  relevance (position in the word, start/end of word), with support for stop aliases and
  multi-word queries.
- **Every optimal journey** between two stops for a given date, including walking
  transfers and vehicle changes.
- **Journey summary** (departure, arrival, duration, vehicles used) and a **detailed view**
  of each journey: intermediate stops, tracks/platforms, walking legs.
- **Exports**: a journey to **iCalendar** (`.ics`, importable into a calendar) and to
  **GeoJSON** (map route).
- Responsive UI: heavy computations run off the JavaFX thread, with a progress indicator.

## Architecture

The code is split into four layers, from lowest to highest level:

| Package | Role |
|---|---|
| `ch.epfl.rechor` | Cross-cutting building blocks: `Preconditions`, packed values (`Bits32_24_8`, `PackedRange`), formatting (`FormatterFr`), iCalendar (`IcalBuilder`) and JSON (`Json`) builders, stop search index (`StopIndex`). |
| `ch.epfl.rechor.timetable` | Timetable model: `TimeTable`, `Stations`, `Platforms`, `Routes`, `Trips`, `Connections`, `Transfers`, `StationAliases`, plus a cache (`CachedTimeTable`). |
| `ch.epfl.rechor.timetable.mapped` | "Flat" implementation: timetables read directly from memory-mapped binary files, with no object deserialisation. |
| `ch.epfl.rechor.journey` | Routing engine: `Router`, `Profile`, `ParetoFront`, `PackedCriteria`, `JourneyExtractor`, then exports (`JourneyIcalConverter`, `JourneyGeoJsonConverter`). |
| `ch.epfl.rechor.gui` | JavaFX interface: `QueryUI` (form), `StopField` (autocomplete), `SummaryUI` (list), `DetailUI` (details), `Main`. |

### The algorithmic core

Routing relies on the **CSA** (*Connection Scan Algorithm*): the day's connections are
scanned **once, in reverse chronological order**, which avoids building a graph and makes
the search very fast.

The optimisation is **multi-criteria**: the goal is not just the fastest journey but every
non-dominated one — a slower trip with fewer changes is still relevant. These optima are
kept in **Pareto fronts** (`ParetoFront`), where each criterion (arrival time, number of
changes, departure time) is packed into a single `long` (`PackedCriteria`) to cut
allocations drastically.

### The binary format

Timetables are not loaded into memory as objects: they are read on the fly from
memory-mapped `.bin` files, through accessors that decode bit fields (`Bits32_24_8`, for
instance, packs a 24-bit index and an 8-bit value into a single `int`). This is what makes
it possible to handle hundreds of thousands of connections without exhausting memory.

## Requirements

- **JDK 22** or later
- **JavaFX SDK 21** (not included — [download here](https://gluonhq.com/products/javafx/))

## Running

### From IntelliJ IDEA

1. Open the project folder.
2. Add the JavaFX SDK 21 as a library (*File → Project Structure → Libraries*), pointing
   to its `lib/` folder.
3. Run the class `ch.epfl.rechor.gui.Main`.

### From the command line

```bash
JFX=/path/to/javafx-sdk-21/lib

javac -cp "$JFX/*" -d out $(find src/ch/epfl/rechor -name '*.java')
java  -cp "out:resources:$JFX/*" ch.epfl.rechor.gui.Main
```

> When JavaFX is loaded from the *classpath*, the JVM prints an "unnamed module"
> warning: it is harmless.

## Repository layout

```
src/ch/epfl/rechor/   source code (44 classes)
test/                 JUnit test suite
resources/            vehicle icons (PNG) and style sheets (CSS)
timetable/            binary timetables — one day per folder (2025-05-26 → 06-01)
```

## Data

The timetables cover a **one-week demonstration period** (26 May → 1 June 2025). A search
for a date outside this range simply returns an empty list of journeys.

## Context

Developed for EPFL's CS-108 course. The assignment, split into about a dozen weekly stages,
specified the public signature of every class and the output formats (iCalendar, GeoJSON);
the internal implementation was free.
