# AGENTS.md — Bank of Z

## What this codebase is

Bank of Z is a demonstration IBM z/OS banking application owned by IBM Corp. (Copyright 2023–2026). It implements a retail bank with customer and account management, cash debit/credit, and fund transfer. Two independent subsystems coexist: a **CICS** online subsystem backed by **DB2** (database `BANKZ`), and an **IMS** online/batch subsystem backed by a separate IMS HDAM/OSAM database and a DB2 `IMSBANK.HISTORY` table. A Node.js **frontend** and a Java/Gradle **REST API** layer (z/OS Connect, OpenBanking-aligned) sit in front of both.

---

## Directory conventions and naming rules

| Path | Contents | Notes |
|------|----------|-------|
| `src/base/cics/cobol/` | CICS COBOL programs | Mixed naming: `BNK1*` = BMS screen drivers; verb-prefixed names (`CRE`, `INQ`, `UPD`, `DEL`, `DBR`, `XFR`, `GET`, `ABND`) = business-logic programs; `CRDTAGY[1-5]` = credit-agency stubs; `BANKDATA` = data initialiser |
| `src/base/cics/copy/` | CICS copybooks (shared record layouts) | Names mirror the program they serve (e.g., `CRECUST.cpy` for `CRECUST.cbl`); `*Z.cpy` variants are Zowe/API-specific COMMAREAs; `*DB2.cpy` are SQL DECLARE TABLE members |
| `src/base/cics/bms/` | BMS map sources | Naming mirrors corresponding BNK1* screen programs |
| `src/base/ims/cobol/` | IMS COBOL programs | `IB*` prefix = IMS bank programs; `LOAD*` = data-load utilities |
| `src/base/ims/copy/` | IMS copybooks | |
| `src/base/ims/pli/` | IMS PL/I programs | `IBLOGIN.pli` is a PL/I rewrite of `IBLOGIN1.cbl` |
| `src/base/ims/DBD/` | IMS Database Definitions (assembler) | One file per IMS database |
| `src/base/ims/PSB/` | IMS Program Specification Blocks (assembler) | One PSB per IMS program (`IB` is the generic PSB) |
| `src/base/ims/LoadData/` | Flat-file test data (`.data`, `.csv`) | Used only by `LOAD*` programs; excluded from DBB build |
| `src/base/ims/java/` | Java package `nazare.jmp.*` | Called by `IBTRAN.cbl` via JNI; Gradle-built |
| `src/base/batch/cobol/` | (none currently) | Placeholder |
| `src/base/batch/pli/` | Batch PL/I programs | `BNKSTMT.pli` = monthly statement |
| `src/base/batch/jcl/` | JCL for batch programs | |
| `src/api/src/main/api/` | OpenAPI 3.0 spec (`openapi.yaml`) | Defines the REST surface |
| `src/api/src/main/zosAssets/` | z/OS Connect provider configs + generated COMMAREA copybooks | The `*/providerFiles/gen/*.cpy` files are **generated artefacts** |
| `src/frontend/` | Node.js SPA (IBM Carbon) | Not a z/OS artefact |
| `.setup/` | Zowe CLI setup/pipeline scripts + JCL | Environment provisioning only; not application logic |
| `docs/modernization/01-rediscovery/` | **Rediscovery outputs** | Agent writes here only |
| `zcodescan/` | IBM Z Code Scan rule overrides | |

**Naming rules observed:**
- Programs ≤ 8 characters (z/OS load-module constraint).
- `BANKZ.*` = CICS subsystem DB2 tables; `IMSBANK.*` = IMS subsystem DB2 table.
- Hard-coded sort-code is `987654` (`src/base/cics/copy/SORTCODE.cpy:7`).
- `CBL CICS('SP,EDF')` on every CICS program; IMS programs use `CBL LIST,MAP,XREF,FLAG(I)`.
- `*Z.cpy` copybook suffix = Zowe/API-facing COMMAREA variant.

---

## Files and folders agents must NOT modify during rediscovery

| Path | Reason |
|------|--------|
| `src/` (entire tree) | Legacy source — read-only during rediscovery |
| `.setup/` | Environment provisioning scripts; changes break z/OS deployments |
| `dbb-app.yaml` | DBB build config; modifying it changes what gets compiled on z/OS |
| `zowe.config.json` | Zowe CLI connection profiles |
| `zcodescan/zcodescan-rules.yaml` | Code-scan rule overrides |
| `docker-compose.yaml` | Local tooling stack |
| `docs/` (except `docs/modernization/`) | Existing documentation |
| `src/api/src/main/zosAssets/*/providerFiles/gen/` | Generated artefacts; do not hand-edit |

**Agent write target:** `docs/modernization/01-rediscovery/` only.

---

## How to run existing builds and tests

### z/OS build (DBB)
The application is built on z/OS using IBM Dependency Based Build (DBB). The `dbb-app.yaml` at the repo root defines compile tasks for COBOL (`src/base/**/cobol/*.cbl`) and PL/I (`src/base/**/*.pli`), plus the OpenAPI spec and frontend.

Local → z/OS pipeline via Zowe CLI:
```sh
# Initial setup (run once)
cd .setup
bash ./setup-local.sh

# Rebuild and redeploy
bash ./pipeline-local.sh
```

These scripts call `setup-remote.sh` / `pipeline-common.sh` on z/OS USS via SSH/Zowe. See `.setup/README` or `docs/installation-and-setup/` for prerequisite Zowe profiles.

### Java (IMS bridge)
```sh
cd src/base/ims/java
./gradlew build
```

### REST API
```sh
cd src/api
./gradlew build
```

### Frontend
```sh
cd src/frontend
npm install
node server.js
```

### No automated test suite was found in the repository.
JCL in `.setup/jcl/` creates and seeds the database but does not run regression tests. Open question logged in `docs/modernization/01-rediscovery/open-questions.md`.
