# Design Document

**Author:** BADREDDINE ELFAJI

**Video Overview:** [Link]

**Project name:** Opening Repertoire & Match Tracker

**Github Link:** https://github.com/Badr-Eddine-Elfaji/Chess_Repertoire_DB_Design

---

## Scope

### Purpose

This database is designed to log online chess matches, track rating fluctuations, and analyze performance across specific opening systems (such as the London System and the Italian Game). It provides a structured way to evaluate training progress and identify which opening variations yield the highest win rates.

### In Scope

| Category  | Description                                                                                               |
|-----------|-----------------------------------------------------------------------------------------------------------|
| Players   | Usernames and current ratings across platforms (e.g., Lichess)                                            |
| Matches   | Individual game records including date, opponent, result, and raw PGN (Portable Game Notation)            |
| Openings  | A dictionary of chess openings and variations tied to their ECO (Encyclopedia of Chess Openings) codes    |
### Out of Scope

- Move-by-move engine evaluations (e.g., Stockfish centipawn loss per move), as this requires substantial storage and dynamic computation.
- Tournament scheduling or live matchmaking functionality.
- External training module data (e.g., spaced repetition data from platforms like Listudy).

---

## Functional Requirements

### Supported Operations

- Insert new match records quickly after a game concludes.
- Query overall win/loss/draw ratios for a specific opening.
- Update player ratings to reflect current skill levels.
- Retrieve all games played against a specific opponent or on a specific date.

### Unsupported Operations

- Automatic real-time game data ingestion via an API natively within SQLite. Data ingestion must be handled by an external script (e.g., Python) before insertion into the database.
- Reconstructing the visual 2D board state directly from SQL queries.

---

## Representation

### This schema targets MySQL 8.0.

### Entities

Three entities are represented in this database:

1. `players`
2. `openings`
3. `matches`

### Attributes

#### `players`

| Column      | Type                          | Constraints         |
|-------------|-------------------------------|---------------------|
| player_id   | INT UNSIGNED AUTO_INCREMENT   | PRIMARY KEY         |
| username    | VARCHAR(50)                   | UNIQUE, NOT NULL    |
| rating      | SMALLINT UNSIGNED             | NOT NULL            |

#### `openings`

| Column      | Type                          | Constraints         |
|-------------|-------------------------------|---------------------|
| opening_id  | INT UNSIGNED AUTO_INCREMENT   | PRIMARY KEY         |
| eco_code    | CHAR(3)                       | NOT NULL            |
| name        | VARCHAR(100)                  | NOT NULL            |
| variation   | VARCHAR(150)                  | NOT NULL            |

#### `matches`

| Column           | Type                                    | Constraints             |
|------------------|-----------------------------------------|-------------------------|
| match_id         | INT UNSIGNED AUTO_INCREMENT             | PRIMARY KEY             |
| white_player_id  | INT UNSIGNED                            | FOREIGN KEY, NOT NULL   |
| black_player_id  | INT UNSIGNED                            | FOREIGN KEY, NOT NULL   |
| opening_id       | INT UNSIGNED                            | FOREIGN KEY, NOT NULL   |
| platform         | ENUM('lichess', 'chess.com')            | NOT NULL                |
| result           | ENUM('1-0', '0-1', '1/2-1/2')           | NOT NULL                |
| date_played      | DATETIME                                |                         |
| pgn              | TEXT                                    |                         |

### Type Justifications

- **INT / SMALLINT UNSIGNED:** Memory-efficient for strictly positive values such as IDs and chess ratings.
- **CHAR(3):** Matches the fixed length of standard ECO codes exactly.
- **ENUM:** Appropriate for categorical fields with a strictly defined set of valid options (platforms, results).

### Constraint Justifications

- **UNIQUE:** Prevents duplicate usernames in the `players` table.
- **ENUM validation:** Replaces manual CHECK constraints, enforcing data integrity at the schema level.
- **Foreign Keys:** Maintains referential integrity, ensuring every match references a valid player and opening.

---

## Relationships

- **Players to Matches:** One-to-many. A player may appear in multiple match records as either the white or black side.
- **Openings to Matches:** One-to-many. A single opening can be associated with many match records.

![Database ER Diagram](chess_ERD.png)

---

## Optimizations

| Optimization          | Detail                                                                               | Rationale                                             |
|-----------------------|--------------------------------------------------------------------------------------|-------------------------------------------------------|
| Primary key indexes   | Auto-created by MySQL on all primary key columns                                     | Enables fast row lookups across all three tables      |
| Secondary index       | `openings.name`                                                                      | Speeds up querying rows based on the name of opening  |
| Secondary index       | `matches.platform`                                                                   | Speeds up platform-specific queries                   |
| Secondary index       | `matches.date_played`                                                                | SPeeds up to query based on date ranges               |
| View                  | `opening_win_rates` pre-aggregates wins, losses, and draws per opening variation     | Reduces query complexity for performance analysis     |

---

## Limitations

### Current Design Limitations

- PGN data is stored as a single text block. Querying specific move orders requires inefficient string matching rather than relational logic.

### Representational Limitations

- Complex 2D positional board states (such as piece pins or board geometry) cannot be efficiently queried. Because each game is stored as a monolithic PGN string, retrieving specific board setups or deep move sequences requires highly inefficient string-matching, which is not well-suited to standard relational database design.

