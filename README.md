# Moderated Learning Management System (LMS) - v1.0.0

CS4433 Software Engineering - Final Project, Phase 4 (Implementation).
A C++17 console application for a secondary school: courses, assignments, submissions, marking,
**Head-of-Department moderation before release**, and a full audit log of mark changes.
It works completely offline (embedded SQLite, no network).

## 1. Requirements

| Tool | Version | Notes |
|------|---------|-------|
| C++17 compiler | GCC 9+, Clang 10+, MSVC 2019+ | tested with GCC 13 |
| CMake | 3.16+ | |
| SQLite 3 development files | 3.31+ | `libsqlite3-dev` / `sqlite` (vcpkg) |
| libsodium development files | 1.0.18+ | Argon2id password hashing |
| GoogleTest | any recent | optional: CMake downloads it if not installed |

Ubuntu/Debian: `sudo apt install build-essential cmake libsqlite3-dev libsodium-dev libgtest-dev`
Windows (vcpkg): `vcpkg install sqlite3 libsodium gtest` and pass the vcpkg toolchain file to CMake.

## 2. Build

```bash
git clone <your-repository-url> lms && cd lms
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
```

CMake options: `-DLMS_BUILD_TESTS=OFF` (skip tests), `-DLMS_WARNINGS_AS_ERRORS=ON`, `-DLMS_COVERAGE=ON`.
`schema/schema.sql` is embedded into the program at build time, so the code and the approved
Phase 3 schema cannot drift apart.

## 3. Run

```bash
./build/lms                       # uses ./lms.db (created on first run)
./build/lms --db /srv/school/lms.db
./build/lms --seed-demo --db demo.db && ./build/lms --db demo.db   # sample data for demonstrations
```

* **First run:** on an empty database the program asks you to create the administrator account.
  There are no default passwords.
* `--seed-demo` (empty database only) creates `admin`, `hod.science`, `hod.humanities`, `lect.moyo`,
  `stu.thabo`, `stu.palesa`, `stu.lerato`, all with password `Demo#1234`, plus course SCI101 and one assignment.
* Times are typed and shown in school local time (default UTC+2, Lesotho) and stored in UTC.
  Change with `--utc-offset-minutes <n>`.
* Environment variable `LMS_DB` sets the default database path.

## 4. Test

```bash
ctest --test-dir build --output-on-failure      # or ./build/lms_tests
```

Coverage (GCC): `cmake -S . -B cov -DCMAKE_BUILD_TYPE=Debug -DLMS_COVERAGE=ON && cmake --build cov -j && ./cov/lms_tests && gcovr -r . --object-directory cov --filter 'src/(domain|service|common)/'`

Static analysis: `cppcheck --enable=warning,style,performance,portability --std=c++17 -I src src`

## 5. Source layout (layered architecture, Phase 3 section 3.1)

```
src/common/        DateTime (UTC, portable), Clock (testable time), Validation, Exceptions
src/domain/        User (abstract) + Student/Lecturer/HoD/Admin, UserFactory, entities and rules,
                   IPenaltyStrategy + strategies
src/data/          Repository interfaces (one per aggregate)  -> src/data/sqlite/ SQLite implementations
src/security/      IPasswordHasher + SodiumPasswordHasher (Argon2id)
src/service/       Use-case services, access control, audit (Observer), notifications (Observer)
src/presentation/  Console, role menus, Application loop
src/System.*       Composition root (the only place concrete classes are wired together)
schema/schema.sql  The Phase 3 SQL schema, unchanged
tests/             GoogleTest suite
docs/              Implementation notes, coding standard
```
Dependencies point downward only: presentation -> service -> domain, data interfaces; SQLite code is
reachable only through the repository interfaces.

## 6. Security summary

* Passwords: Argon2id with a random per-password salt (libsodium `crypto_pwhash_str`); only the encoded hash is stored.
* All SQL uses prepared statements with bound parameters; no string-built SQL with user data.
* Role checks in every service method; HoD/lecturer/student ownership checks (department, course, enrolment).
* Generic login error and dummy-hash check for unknown users (no user enumeration).
* Every mark change is written to `audit_entry` in the same transaction as the change.
