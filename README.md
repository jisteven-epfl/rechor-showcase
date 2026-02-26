<div align="center">

# 🚞 ReCHor: Swiss Public Transport Router

> **Demonstration Only:** This repository is intended for project showcasing only. If you are interested in discussing the implementation or technical details, please contact **<example@epfl.com>**.

![Java](https://img.shields.io/badge/Java-17+-blue.svg?style=flat-square)
![Algorithms](https://img.shields.io/badge/Algorithm-CSA-brightgreen.svg?style=flat-square)
![Optimization](https://img.shields.io/badge/Optimization-Pareto__Front-orange.svg?style=flat-square)
![EPFL](https://img.shields.io/badge/School-EPFL-red.svg?style=flat-square)

*An offline, highly-optimized route planning engine for the Swiss public transport network.*

</div>

## 📖 Overview

**ReCHor** (Recherche d'Horaire) is a high-performance routing engine designed for efficient large-scale transit planning. Operating entirely offline, it uses real timetable data to find the best routes between a departure point and a destination at a given date and time.

The system provides results comparable to official platforms like [CFF/SBB](https://www.cff.ch/) or [search.ch](https://search.ch), balancing the trade-offs between journey duration and the number of transfers. It also features utilities for visualizing routes on dynamic maps (`uMap`/`GeoJSON`) and exporting them to personal calendars (`iCalendar` format).

<div align="center">
  <img src="assets/rechor-gui.png" alt="ReCHor GUI" width="800"/>
  <br/>
  <em>ReCHor Graphical User Interface showing Pareto-optimal routes.</em>
</div>

### 📊 Project Statistics

- **Core Implementation**: ~5,000 lines of original Java code (specifically in the `ch.epfl.rechor` packages).
- **Network Scale**: Real-time routing across **30,000+ localized stops** and approximately **3,000,000 daily transport connections** (derived from the full Swiss public transport dataset).
- **Real-time Performance**: Capable of generating an exhaustive Pareto Front for the entire national network in approximately **2-3 seconds** on standard hardware.

### 🏗️ Engineering & OOP Architecture

While constructed according to technical specifications, the project serves as a showcase for sophisticated Object-Oriented patterns used to manage high-density transit data:

- **Interface-Driven Design**: Core entities (Stations, Routes, Trips) are defined as strict interfaces, allowing for seamless swapping between standard and "Buffered" implementations.
- **Flyweight/Buffered Pattern**: To keep the memory footprint minimal while scanning millions of connections, the system uses a custom `StructuredBuffer` architecture. Data is read directly from memory-mapped buffers only when needed, avoiding the overhead of creating millions of individual Java objects.
- **Modern Java Idioms**: Extensively utilizes **Records**, **Sealed Interfaces**, and **Streams** to ensure a robust and type-safe codebase.

---

## 🧠 Core Techniques: Under the Hood

### 1. Multi-Objective Optimization: The Pareto Front

When calculating a route, passengers rarely care about just a single metric; they actively weigh multiple criteria simultaneously:

- **Time** (arriving as early as possible or departing as late as possible)
- **Convenience** (minimizing the number of vehicle changes)

Instead of artificially blending these priorities into a single arbitrary "score," ReCHor uses the foundational concept of the **Pareto Front**.

#### What is a Pareto Front?

<div align="center">
  <img src="assets/pareto-front.png" alt="Pareto Front Illustration" width="400"/>
</div>

In a multi-objective optimization problem, a solution is considered "Pareto-optimal" if it cannot be improved in one criterion without degrading another.
For example, a prospective route arriving at 17:57 with 4 changes is Pareto-optimal if the only way to arrive at 17:57 *with fewer changes* is to depart significantly earlier.

ReCHor maintains an active "Frontier" of these optimal tradeoffs. When you search for a route, it doesn't just return one rigid path; it returns an exhaustive family of optimal journeys. The user can then decide for themselves: *"Do I leave 20 minutes earlier to avoid a 3-train transfer sequence, or do I take the faster route that requires more changes?"*

### 2. The Engine: Connection Scan Algorithm (CSA)

Most classic routing engines rely on mathematical graph-based algorithms like Dijkstra or A*. However, ReCHor moves away from this paradigm for a hardware-efficient technique known as the **Connection Scan Algorithm (CSA)**.

#### How CSA Works

Traditional algorithms build a massive web of nodes (stations) and edges (routes) and "crawl" through them. CSA takes a radically different, simpler, and profoundly faster approach: it treats the entire daily timetable as a single, massive array of **Connections** (direct trips from one stop to the immediate next stop), sorted descendingly by their departure time.

The algorithm traverses every single connection for a given date in a single sweep. When analyzing each connection, the engine recursively evaluates **three fundamental options** for the virtual passenger:

1. **Walk to the Destination:** Can the passenger disembark at the arrival stop and walk to the final destination? (Validated against a pre-computed array of walking distances and footpaths).
2. **Stay in the Vehicle:** Can the passenger stay on board and simply ride to the next stop of the current ongoing trip?
3. **Change Vehicles:** Can the passenger disembark here and board a different vehicle that leaves at a later, compatible time from this exact station?

#### Efficient Implementation (`Router.java`)

In ReCHor's `Router.java` module, this scan must evaluate millions of connections in mere fractions of a second. It does this efficiently because:

- To avoid memory bloat and garbage collector delays in Java, variables describing a state on the Pareto Front (like arrival time, number of route changes, and pointers for journey extraction) are heavily compressed. They are **bit-packed perfectly into 64-bit primitive `long` integers**.
- Re-querying vast blocks of structural timetable properties repeatedly is avoided entirely by feeding instructions to a highly specialized **Cache Memory** stream (`CachedTimeTable`).

### 3. Journey Extraction

Once the Connection Scan Algorithm completes its single sweep, ReCHor possesses the complete Pareto front for the departure station. But the front alone only tells us the abstract metrics (e.g., depart 16:13, arrive 17:28, 3 changes) without tracking the geographic turns.

ReCHor solves this elegantly by using the "payload" bits of our 64-bit integer trackers. When constructing the Pareto frontiers, it leaves behind breadcrumbs—recording precisely which chronological connection index to board and how many stops to ride before the next decision point. A specialized extractor module uses this data to string back together the real-world, turn-by-turn journey for the passenger.

---

## 🛠 Features Summary

- **Lightning-Fast Routing:** Driven by the Connection Scan Algorithm traversing packed primitives backwards in time.
- **Smart Result Generation:** Generates all Pareto-optimal routes (speed vs. convenience).
- **Fully Offline Validation:** Capable of mapping the full Swiss topological and transport network on localized files dynamically mapped to physical memory (`mmap`).
- **Calendar & Map Ready:** Export natively to `iCalendar` (`.ics`) formats and build mapping structures using `GeoJSON` representations on `OpenStreetMap` / `uMap`.
  <p align="center">
    <img src="assets/umap.png" alt="uMap Visualization" width="45%"/>
    &nbsp;&nbsp;
    <img src="assets/icalendar.png" alt="iCalendar Export" width="45%"/>
  </p>
- **Memory Optimized:** Data structures rely heavily on dense 32-bit (24-bit + 8-bit payloads) configurations and 64-bit long primitive representations to avoid object allocation penalties at scale.

### 🔌 Data Source & High Performance I/O

ReCHor does not rely on external APIs; it operates on a substantial **offline binary database** (derived from official GTFS data) provided by EPFL. To handle the scale of millions of entries efficiently, the system implements:

- **Local Binary I/O**: Direct parsing of `.bin` and `.txt` files from a structured timetable directory.
- **Memory-Mapped Files (mmap)**: Leveraging Java's `FileChannel` to project large binary archives into the process virtual address space. This ensures near-instantaneous data access with zero-copy overhead, bypassing traditional database query latency.

## ⚖️ License & Copyright

This project incorporates skeleton code provided by EPFL. This repository is maintained strictly for portfolio demonstration and internship application purposes. In compliance with academic integrity policies, the source code is not publicly available. All rights reserved by the authors and EPFL.

---

> *This project was completed by **[Steven Ji]** and **[Ayoub Ouederni]** as a duo collaboration in the Fall 2024 semester for the EPFL course [Practice of Object-Oriented Programming (CS-108)](https://edu.epfl.ch/coursebook/en/practice-of-object-oriented-programming-CS-108) (9 credits).*
