# Bank of Z — Repository Map

> Rediscovery phase 1 · orientation only · no modernisation proposals.

---

## 1. Programs

### 1a. CICS subsystem (`src/base/cics/cobol/`)

These programs run under IBM CICS as online transaction programs. All carry `CBL CICS('SP,EDF')` and were authored for IBM CICS TS on z/OS.

| Program | One-line purpose | Uses SQL? | Uses DLI? |
|---------|-----------------|-----------|-----------|
| `BNKMENU.cbl` | BMS main menu — validates option and routes to the correct transaction | No | No |
| `BNK1CCS.cbl` | BMS screen driver: Create Customer | No | No |
| `BNK1CAC.cbl` | BMS screen driver: Create Account (validates input, LINKs to CREACC) | No | No |
| `BNK1CCA.cbl` | BMS screen driver: List accounts for a customer | No | No |
| `BNK1CRA.cbl` | BMS screen driver: Credit/Debit cash | No | No |
| `BNK1DCS.cbl` | BMS screen driver: Display/update/delete customer | No | No |
| `BNK1DAC.cbl` | BMS screen driver: Display/delete account | No | No |
| `BNK1UAC.cbl` | BMS screen driver: Update account | No | No |
| `BNK1TFN.cbl` | BMS screen driver: Transfer funds between accounts | No | No |
| `CRECUST.cbl` | Business logic: create customer — enqueues counter, runs async credit-agency calls, writes CUSTOMER + PROCTRAN | Yes | No |
| `CREACC.cbl` | Business logic: create account — enqueues counter, writes ACCOUNT + PROCTRAN | Yes | No |
| `INQCUST.cbl` | Business logic: enquire customer by number, returns CUSTOMER record | Yes | No |
| `INQACC.cbl` | Business logic: enquire account by sort-code + account-number | Yes | Yes |
| `INQACCCU.cbl` | Business logic: enquire all accounts for a customer | Yes | Yes |
| `UPDCUST.cbl` | Business logic: update limited customer fields, no PROCTRAN entry | Yes | Yes |
| `UPDACC.cbl` | Business logic: update account fields (not balance), no PROCTRAN entry | Yes | Yes |
| `DELCUS.cbl` | Business logic: delete customer — deletes all accounts then customer, writes PROCTRAN per deletion | Yes | No |
| `DELACC.cbl` | Business logic: delete single account by customer-number + account-type | Yes | No |
| `DBCRFUN.cbl` | Business logic: debit/credit cash — reads + updates ACCOUNT, writes PROCTRAN | Yes | No |
| `XFRFUN.cbl` | Business logic: transfer funds between accounts — updates both accounts + PROCTRAN | Yes | No |
| `GETSCODE.cbl` | Utility: returns the bank's sort-code (hard-coded in copybook SORTCODE.cpy) | No | No |
| `GETCOMPY.cbl` | Utility: returns the company/bank name via COMMAREA | No | No |
| `BANKDATA.cbl` | Batch-style data initialiser: populates BANKZ.CUSTOMER and BANKZ.ACCOUNT in DB2 from JCL PARM ranges | Yes | No |
| `ABNDPROC.cbl` | Infrastructure: writes abend records to a centralised VSAM KSDS (CF) datastore | No | No |
| `CRDTAGY1.cbl` | Stub: simulated credit agency 1 — random delay + random score (CICS Async API) | No | No |
| `CRDTAGY2.cbl` | Stub: simulated credit agency 2 (same pattern as CRDTAGY1) | No | No |
| `CRDTAGY3.cbl` | Stub: simulated credit agency 3 | No | No |
| `CRDTAGY4.cbl` | Stub: simulated credit agency 4 | No | No |
| `CRDTAGY5.cbl` | Stub: simulated credit agency 5 | No | No |

### 1b. IMS subsystem (`src/base/ims/cobol/` and `src/base/ims/pli/`)

These programs run under IBM IMS TM (message-driven) and are called via PSBs. The IMS database uses HDAM/OSAM datasets.

| Program | Language | One-line purpose |
|---------|----------|-----------------|
| `IBLOGIN1.cbl` | COBOL | IMS MPP: authenticate customer (password check), set login-status flag in IMS DB |
| `IBLOGOUT.cbl` | COBOL | IMS MPP: log customer out, clear login-status flag |
| `IBGCUDAT.cbl` | COBOL | IMS MPP: get customer data by customer-ID from IMS DB |
| `IBSCUDAT.cbl` | COBOL | IMS MPP: set/update customer data in IMS DB |
| `IBACSUM.cbl` | COBOL | IMS MPP: account summary — retrieves balance from IMS + DB2 HISTORY table |
| `IBTRAN.cbl` | COBOL | IMS MPP: execute a financial transaction (debit/credit), calls Java class `InsertHist` via JNI to write to DB2 HISTORY |
| `IBLOGIN.pli` | PL/I | IMS MPP: PL/I rewrite of IBLOGIN1 (described as a rewrite in header `src/base/ims/pli/IBLOGIN.pli:3`) |

#### IMS data-load programs (`src/base/ims/cobol/`)

All five are batch-load utilities that read fixed-format flat files and insert records into IMS databases:

| Program | Source DD | Loads |
|---------|-----------|-------|
| `LOADACCT.cbl` | `ACCTIN` | ACCOUNT IMS segments |
| `LOADCUST.cbl` | `CUSTIN` | CUSTOMER IMS segments |
| `LOADCUSA.cbl` | `CUSAIN` | CUSTOMER account-association segments |
| `LOADHIST.cbl` | `HISTIN` | HISTORY records |
| `LOADTSTA.cbl` | `TSTAIN` | TSTAT (transaction-status) records |

### 1c. Batch subsystem (`src/base/batch/`)

| Program | Language | One-line purpose |
|---------|----------|-----------------|
| `BNKSTMT.pli` (`src/base/batch/pli/`) | PL/I | Monthly account-statement report — reads BANKZ.ACCOUNT, BANKZ.CUSTOMER, BANKZ.PROCTRAN; writes to SYSPRINT |

JCL: `src/base/batch/jcl/BNKSTMT.jcl` — runs BNKSTMT via DSN SYSTEM(DBD1), passes sort-code and YYYYMM parms.

---

## 2. Copybooks

### CICS copybooks (`src/base/cics/copy/` — 42 files)

| Copybook | Purpose |
|----------|---------|
| `CUSTOMER.cpy` | 03-level CUSTOMER-RECORD layout (eyecatcher `CUST`, key = SORTCODE+NUMBER, includes 88-levels for status) |
| `ACCOUNT.cpy` | 03-level ACCOUNT-DATA layout (eyecatcher `ACCT`, key = SORT-CODE+NUMBER) |
| `PROCTRAN.cpy` | 03-level PROC-TRAN-DATA layout (eyecatcher `PRTR`, includes 88-levels for transaction types) |
| `CUSTDB2.cpy` | SQL DECLARE TABLE for DB2 BANKZ.CUSTOMER |
| `ACCDB2.cpy` | SQL DECLARE TABLE for DB2 BANKZ.ACCOUNT |
| `CONTDB2.cpy` | SQL DECLARE TABLE for DB2 BANKZ.CONTROL |
| `CONTROLI.cpy` | Working-storage layout of CONTROL record (customer count, last-customer, account count, last-account) |
| `ACCTCTRL.cpy` | ACCOUNT-CONTROL-RECORD layout (eyecatcher `CTRL`) |
| `CUSTCTRL.cpy` | CUSTOMER-CONTROL-RECORD layout |
| `SORTCODE.cpy` | Single 77-level: `SORTCODE PIC 9(6) VALUE 987654` — the hard-coded sort-code |
| `ABNDINFO.cpy` | ABND-VSAM-KEY + abend metadata layout used by ABNDPROC |
| `BANKMAP.cpy` | BMS-related bank map layout |
| `BNK1DDM.cpy` | BMS data definitions for BNK1 screens |
| `CUSTMAP.cpy` | BMS customer-screen map layout |
| `INQACC.cpy` | INQACC COMMAREA layout |
| `INQACCCU.cpy` | INQACCCU COMMAREA layout |
| `INQACCZ.cpy` | Zowe/API variant of account inquiry COMMAREA |
| `INQACCCZ.cpy` | Zowe/API variant of accounts-by-customer COMMAREA |
| `INQCUSTZ.cpy` | Zowe/API variant of customer inquiry COMMAREA |
| `CRECUST.cpy` | CRECUST COMMAREA layout |
| `CREACC.cpy` | CREACC COMMAREA layout |
| `UPDACC.cpy` | UPDACC COMMAREA layout |
| `UPDCUST.cpy` | UPDCUST COMMAREA layout |
| `DELCUS.cpy` | DELCUS COMMAREA layout |
| `DELACC.cpy` | DELACC COMMAREA layout |
| `DELACCZ.cpy` | Zowe/API variant of DELACC COMMAREA |
| `DBCRFUN.cpy` | (implied) DBCRFUN COMMAREA layout |
| `XFRFUN.cpy` | XFRFUN COMMAREA layout |
| `PAYDBCR.cpy` | Pay debit/credit COMMAREA layout |
| `PAYDBCRZ.cpy` | Zowe/API variant of PAYDBCR |
| `GETSCODE.cpy` | GETSCODE response layout |
| `GETCOMPY.cpy` | GETCOMPY response layout |
| `NEWCUSNO.cpy` | New-customer-number response layout |
| `NEWACCNO.cpy` | New-account-number response layout |
| `PROCDB2.cpy` | PROCTRAN DB2 SQL DECLARE TABLE |
| `PROCISRT.cpy` | PROCTRAN insert working-storage layout |
| `RESPSTR.cpy` | Common response-string layout |
| `STCUSTNO.cpy` | Starting customer number layout |
| `WAZI.cpy` | **Empty placeholder** (`THIS IS AN EMPTY COPYBOOK` — `src/base/cics/copy/WAZI.cpy:8`) |

### IMS copybooks (`src/base/ims/copy/` — 4 files)

| Copybook | Purpose |
|----------|---------|
| `IBTRAN.cpy` | INPUT-AREA / OUTPUT-AREA IMS message layout for IBTRAN |
| `IBGHIST.cpy` | INPUT-AREA / OUTPUT-AREA for history enquiry (IBACSUM) |
| `IBSHIST.cpy` | History-related layout (set-history variant) |
| `JNI.cpy` | COBOL JNI (Java Native Interface) interop declarations for calling Java classes from IMS COBOL |

---

## 3. BMS Maps (`src/base/cics/bms/` — 10 files)

| Map | Likely screen |
|-----|--------------|
| `BNK1MAI.bms` | Main menu |
| `BNK1CCM.bms` | Create customer |
| `BNK1CAM.bms` | Create account |
| `BNK1CDM.bms` | Customer detail / delete |
| `BNK1DAM.bms` | Account detail / delete |
| `BNK1UAM.bms` | Update account |
| `BNK1TFM.bms` | Transfer funds |
| `BNK1CCS.bms`-equivalent → `BNK1B2M.bms` | Secondary BNK1 screen |
| `BNK1DCM.bms` | Display customer |
| `BNK1ACC.bms` | Account list |

---

## 4. JCL

| File | Purpose |
|------|---------|
| `.setup/jcl/cics/Db2-create.jcl` | DDL: CREATE DATABASE BANKZ; creates tables ACCOUNT, PROCTRAN, CONTROL, CUSTOMER |
| `.setup/jcl/cics/Db2-drop.jcl` | Drops BANKZ tables |
| `.setup/jcl/cics/Db2-insert.jcl` | Seeds initial data into BANKZ tables |
| `.setup/jcl/cics/Db2-grant.jcl` | GRANTs on BANKZ tables |
| `.setup/jcl/cics/Db2-bind.jcl` | Binds DB2 plans for CICS programs |
| `.setup/jcl/ims/Db2-create.jcl` | DDL: CREATE DATABASE IMSBANK; creates HISTORY table |
| `.setup/jcl/ims/Db2-drop.jcl` | Drops IMSBANK tables |
| `src/base/batch/jcl/BNKSTMT.jcl` | Runs PL/I batch statement program via DSN SYSTEM(DBD1) |

---

## 5. IMS DBDs and PSBs (`src/base/ims/DBD/` and `src/base/ims/PSB/`)

### DBDs (9 files — HDAM/OSAM access)

| DBD | Description |
|-----|-------------|
| `CUSTOMER.asm` | Customer master segment (key=CUSTID 4 bytes, 279-byte segment) |
| `ACCOUNT.asm` | Account segment (key=ACCID 8 bytes LONG, 25-byte segment) |
| `CUSTACCS.asm` | Customer-to-accounts relationship |
| `CUSTTYPE.asm` | Customer type code table |
| `ACCTYPE.asm` | Account type code table |
| `HISTORY.asm` | Transaction history |
| `TSTAT.asm` | Transaction status |
| `TSTATTYP.asm` | Transaction status type |
| `TTYPE.asm` | Transaction type |

### PSBs (8 files)

`IB.asm`, `IBACSUM.asm`, `IBGCUDAT.asm`, `IBLOAD.asm`, `IBLOGIN.asm`, `IBLOGOUT.asm`, `IBSCUDAT.asm`, `IBTRAN.asm` — each PSB defines the PCBs (program communication blocks) for the IMS programs of the same name.

---

## 6. DB2 Schema (inferred from DDL)

### BANKZ database (CICS subsystem)

| Table | Key columns | Notes |
|-------|-------------|-------|
| `BANKZ.ACCOUNT` | `ACCOUNT_SORTCODE` + `ACCOUNT_NUMBER` (unique index) | Also indexed on `(SORTCODE, CUSTOMER_NUMBER)` |
| `BANKZ.CUSTOMER` | `CUSTOMER_SORTCODE` + `CUSTOMER_NUMBER` (unique index) | Also indexed on `(LAST_NAME, FIRST_NAME)` |
| `BANKZ.PROCTRAN` | `PROCTRAN_SORTCODE` + `PROCTRAN_NUMBER` | Audit log of processed transactions |
| `BANKZ.CONTROL` | `CONTROL_NAME` (unique index) | Named counter table (customer count, account count, etc.) |

### IMSBANK database (IMS subsystem)

| Table | Key | Notes |
|-------|-----|-------|
| `IMSBANK.HISTORY` | `TXID` BIGINT UNIQUE | Written by Java class `InsertHist` via JNI from IBTRAN |

---

## 7. API Layer (`src/api/`)

- **OpenAPI spec**: `src/api/src/main/api/openapi.yaml` — OpenBanking-compliant REST API (customers, accounts, balances, transactions). OAuth2-secured.
- **zosAssets**: Per-program COMMAREA copybooks auto-generated for Zowe/z/OS Connect integration:  
  `CRECUST`, `DBCRFUN`, `IBACSUM`, `IBGCUDAT`, `IBSCUDAT`, `IBTRAN`, `INQACC`, `INQACCCU`, `INQCUST`, `UPDCUST`.
  Generated `.cpy` files live under `*/providerFiles/gen/` and should be treated as derived artefacts.
- **Java/Gradle build**: `src/api/build.gradle` — builds the z/OS Connect API JAR.

---

## 8. Frontend (`src/frontend/`)

Node.js + IBM Carbon Design System SPA. HTML pages map 1:1 to API operations. Not a legacy COBOL concern — only relevant as the API consumer.

---

## 9. IMS Java bridge (`src/base/ims/java/`)

A Java package (`nazare.jmp.*`) that runs on z/OS under IMS Java. Key class: `InsertHist` — called by `IBTRAN.cbl` via JNI (copybook `JNI.cpy`) to insert rows into `IMSBANK.HISTORY`. `IMSBankHistory.java` queries HISTORY over JDBC. This is an unusual hybrid: COBOL calling Java via JNI inside an IMS region.

---

## 10. Build and CI tooling

| File | Purpose |
|------|---------|
| `dbb-app.yaml` | IBM DBB (Dependency Based Build) config: COBOL + PL/I compile/link, impact analysis, source dirs |
| `zapp.yaml` | IBM Z Open Editor ZAPP (property groups, remote dataset locations) |
| `zowe.config.json` | Zowe CLI team config (connection profiles) |
| `.setup/pipeline-common.sh` | Remote pipeline: rebuild and redeploy on z/OS USS |
| `.setup/setup-common.sh` | Remote setup: initial environment provisioning |
| `zcodescan/zcodescan-rules.yaml` | IBM Z Code Scan rule overrides |
| `renovate.json` | Renovate dependency-update bot config |
| `docker-compose.yaml` | Local tooling containers (not z/OS runtime) |

---

## 11. Unusual / noteworthy observations

| Observation | Location |
|-------------|----------|
| `WAZI.cpy` is an **empty copybook** (8-line placeholder, no data definitions) | `src/base/cics/copy/WAZI.cpy:8` |
| Sort-code is **hard-coded** as literal `987654` in a copybook, not a parameter | `src/base/cics/copy/SORTCODE.cpy:7` |
| IMS COBOL (`IBTRAN.cbl`) calls a **Java class via JNI** — unusual hybrid pattern | `src/base/ims/cobol/IBTRAN.cbl:20`, `src/base/ims/copy/JNI.cpy` |
| `IBLOGIN.pli` is documented as a **PL/I rewrite** of `IBLOGIN1.cbl`; both exist | `src/base/ims/pli/IBLOGIN.pli:3` and `src/base/ims/cobol/IBLOGIN1.cbl` |
| `UPDCUST.cbl` mentions accessing "VSAM datastore" in its header comment, but `UPDACC.cbl` says "DB2 datastore"; the CICS setup JCL only creates DB2 tables | `src/base/cics/cobol/UPDCUST.cbl:14`, `.setup/jcl/cics/Db2-create.jcl` |
| Generated COMMAREA copybooks in `src/api/src/main/zosAssets/*/providerFiles/gen/` are derived artefacts | e.g., `src/api/src/main/zosAssets/CRECUST/providerFiles/gen/CRECUST_request_0.cpy` |
| `BANKDATA.cbl` carries `CBL SQL` but no `CBL CICS(...)` — it appears to be a standalone DB2 batch program, not a CICS program, despite living in `src/base/cics/cobol/` | `src/base/cics/cobol/BANKDATA.cbl:1` |
| `IBTRAN.cbl` is declared `PROGRAM-ID. "IBTRAN" recursive` — unusual for batch/online COBOL | `src/base/ims/cobol/IBTRAN.cbl:3` |
| IMS load-data flat files (`.data`, `.csv`) live alongside DBD/PSB sources | `src/base/ims/LoadData/` |

---

## Suggested reading order

| Priority | Subsystem | Reason |
|----------|-----------|--------|
| **First** | CICS core programs: `CRECUST` → `INQCUST` → `UPDCUST` → `DELCUS` → `CREACC` → `INQACC` → `UPDACC` → `DELACC` | Highest business density; PROCTRAN write pattern recurs everywhere; copybooks `CUSTOMER.cpy`, `ACCOUNT.cpy`, `PROCTRAN.cpy` are the master record layouts |
| **Second** | CICS financial operations: `DBCRFUN`, `XFRFUN` | Short programs but encode overdraft / transfer rules; feed directly into PROCTRAN |
| **Third** | BMS screen drivers: `BNKMENU` + all `BNK1*` | Presentation layer only; understand last to avoid noise when extracting business rules |
| **Fourth** | IMS core transactions: `IBLOGIN1`/`IBLOGIN.pli`, `IBLOGOUT`, `IBGCUDAT`, `IBSCUDAT`, `IBACSUM`, `IBTRAN` | Separate subsystem with its own DB; the COBOL–Java JNI bridge in IBTRAN needs a specialist |
| **Worst** | `ABNDPROC`, `GETSCODE`, `GETCOMPY`, all `CRDTAGY*` stubs | Infrastructure / stubs; little business logic but referenced by many programs |
| **Last** | IMS load programs (`LOAD*`), API layer (`src/api/`), frontend | Data setup utilities and derived/generated artefacts; read only if data-model questions remain |
