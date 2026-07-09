# Integrations Inventory

> Every place this system talks to the outside world.
> Direction: **IN** = system receives data, **OUT** = system sends data, **BOTH** = read and write.
> Load-bearing = production-critical. Legacy-only = test fixtures / migration utilities only.

---

## 1. DB2 Databases

### 1.1 CICS Subsystem — Database `BANKZ` (DB2 subsystem `DBD1`)

DB2 subsystem name is hard-coded as `DBD1` in JCL (`.setup/jcl/cics/Db2-create.jcl:9`, `.setup/jcl/cics/Db2-bind.jcl:23`).  
All programs use static SQL; DBRM library is `BANKZ.V0R1M0.DBRM`; package `BANKZPACK`; plan `BANKZPLN`.

| Table | Columns (key fields) | Direction | Programs | Load-bearing? |
|-------|----------------------|-----------|----------|---------------|
| `BANKZ.ACCOUNT` | ACCOUNT_SORTCODE (PK), ACCOUNT_NUMBER (PK), ACCOUNT_CUSTOMER_NUMBER, ACCOUNT_TYPE, ACCOUNT_INTEREST_RATE, ACCOUNT_OPENED, ACCOUNT_OVERDRAFT_LIMIT, ACCOUNT_LAST/NEXT_STATEMENT, ACCOUNT_AVAILABLE/ACTUAL_BALANCE | BOTH | CREACC (INSERT), DBCRFUN (SELECT, UPDATE), DELACC (SELECT, DELETE), INQACC (cursor SELECT), INQACCCU (cursor SELECT), UPDACC (SELECT, UPDATE), XFRFUN (SELECT×2, UPDATE×2), BANKDATA (INSERT/seed), BNKSTMT (cursor SELECT) | Yes |
| `BANKZ.CUSTOMER` | CUSTOMER_SORTCODE (PK), CUSTOMER_NUMBER (PK), name/address/phone/DOB/status/credit_score | BOTH | CRECUST (INSERT), DELCUS (SELECT, DELETE), INQCUST (SELECT), UPDCUST (SELECT, UPDATE), BANKDATA (INSERT/seed), BNKSTMT (SELECT) | Yes |
| `BANKZ.PROCTRAN` | PROCTRAN_SORTCODE (PK), PROCTRAN_NUMBER (PK), DATE, TIME, REF, TYPE, DESC, AMOUNT | BOTH | CREACC (INSERT), DBCRFUN (INSERT), DELACC (INSERT), DELCUS (INSERT×multiple), XFRFUN (INSERT×2), BNKSTMT (cursor SELECT) | Yes |
| `BANKZ.CONTROL` | CONTROL_NAME (PK), CONTROL_VALUE_NUM, CONTROL_VALUE_STR | BOTH | CREACC (SELECT/UPDATE for account-number counter), CRECUST (SELECT/UPDATE for customer-number counter), BANKDATA (INSERT/init) | Yes |

Schema DDL: `.setup/jcl/cics/Db2-create.jcl`  
Copybook declarations: `src/base/cics/copy/ACCDB2.cpy`, `src/base/cics/copy/CUSTDB2.cpy`, `src/base/cics/copy/PROCDB2.cpy`, `src/base/cics/copy/CONTDB2.cpy`

> **Note:** `CONTDB2.cpy:7` declares the table as `STTESTER.CONTROL`, but the DDL creates `BANKZ.CONTROL`. See open-question #14.

---

### 1.2 IMS Subsystem — Database `IMSBANK` (DB2 subsystem `DBD1`)

| Table | Columns (key fields) | Direction | Programs / Classes | Load-bearing? |
|-------|----------------------|-----------|-------------------|---------------|
| `IMSBANK.HISTORY` | TXID (BIGINT, UNIQUE), TIMESTMP, TRANSTYP, AMOUNT, REFTXID, ACCID, BALANCE | BOTH | `nazare.jmp.history.TransactionService.saveTransactionDetail()` via `InsertHist.insertHist()` via `IBTRAN.cbl` JNI (INSERT); `TransactionService.getTransactionDetail()` (SELECT); `IMSBankHistory.java` main() test harness (SELECT) | Yes (INSERT path); test-harness SELECT in `IMSBankHistory.java` is legacy-only |

Schema DDL: `.setup/jcl/ims/Db2-create.jcl`  
JDBC URL pattern (Type 4 remote): `jdbc:db2://${db2Hostname}:${db2Port}/${db2Location}` (SSL enabled) — `src/base/ims/java/src/main/java/nazare/jmp/controller/IMSBankHistory.java:70`  
JDBC URL pattern (Type 2 local): `jdbc:db2:${DB2_LOCATION}` default `DBD1LOC` — `src/base/ims/java/src/main/java/nazare/jmp/history/TransactionService.java:34`  
DB2 driver: `com.ibm.db2:jcc:12.1.4.0` — `src/base/ims/java/build.gradle:36`

---

## 2. IMS Hierarchical Databases (HDAM/OSAM)

All databases use HDAM/OSAM access method. DBDs are assembled from `src/base/ims/DBD/`. PCB names are declared in PSBs under `src/base/ims/PSB/`. Programs call `CBLTDLI` (COBOL) or `PLITDLI` (PL/I).

| IMS Database (DBD NAME) | DDname (DD1=) | Root Segment | Direction | Programs (function codes) | Load-bearing? |
|-------------------------|--------------|--------------|-----------|---------------------------|---------------|
| `ACCOUNT` | `ACCOUNT` | ACCOUNT (25 bytes: ACCID, ACCTYPE, BALANCE, LASTTXID) | BOTH | IBACSUM (GHU), IBTRAN (GHU, REPL), LOADACCT (ISRT) | Yes; LOADACCT is legacy-only |
| `CUSTACCS` | `CUSTACCS` | CUSTACCS (CUSTID, ACCID, ACCNUM) | BOTH | IBACSUM (GHU, GHN), IBTRAN (GHU), LOADCUSA (ISRT) | Yes; LOADCUSA is legacy-only |
| `CUSTOMER` | `CUSTOMER` | CUSTOMER segment | BOTH | IBGCUDAT (GU), IBLOGIN1 (GHU, REPL), IBLOGOUT (GHU, REPL), IBSCUDAT (GHU, REPL), LOADCUST (ISRT) | Yes; LOADCUST is legacy-only |
| `HISTORY` | `HISTORY` | HISTORY-SEG (TXID, TIMESTMP, TRANSTYP, AMOUNT, REFTXID, ACCID, BALANCE) | BOTH | IBTRAN (ISRT via DBPCB3), LOADHIST (ISRT) | Yes; LOADHIST is legacy-only |
| `TSTAT` | `TSTAT` | TSTAT-SEG | OUT (ISRT) | LOADTSTA (ISRT) | Legacy-only (load utility) |
| `ACCTYPE` | `ACCTYPE` | — | — | No online program found | Legacy-only |
| `CUSTTYPE` | `CUSTTYPE` | — | — | No online program found | Legacy-only |
| `TSTATTYP` | `TSTATTYP` | — | — | No online program found | Legacy-only |
| `TTYPE` | `TTYPE` | — | — | No online program found | Legacy-only |

IMS I/O PCB (`IOPCB`) is used by all IMS programs for reading input messages (GU) and writing output messages (ISRT to LTERMPCB).  
IMS HLQ: `BANKZ.IMS2` — `.setup/config/config.yaml:126`; IMS system HLQ: `IMSV15`; datastore name: `IMS2`.  
RECON datasets: `BANKZ.IMS2.RECON1`, `BANKZ.IMS2.RECON2`, `BANKZ.IMS2.RECON3` — `.setup/config/config.yaml:140–142`.

---

## 3. CICS Files (VSAM)

| File Name | Dataset | Type | Record Layout | Direction | Programs | Load-bearing? |
|-----------|---------|------|---------------|-----------|----------|---------------|
| `ABNDFILE` | `BANKZ.V0R1M0.ABNDFILE` | VSAM KSDS | 681-byte variable; key length 12; defined in `src/base/cics/copy/ABNDINFO.cpy` | OUT (WRITE only) | ABNDPROC (EXEC CICS WRITE FILE line 142) | Yes — every CICS business program links ABNDPROC on abend |

CICS resource definition: `.setup/zconfig/bank-of-z-definitions.yaml:5–42`  
ABNDPROC is the sole writer; no program performs READ/REWRITE/DELETE on ABNDFILE in the source tree.

---

## 4. CICS Named Counter Service (NCS)

Named counters are used to generate unique customer and account numbers. They are managed via EXEC CICS commands against the CICS Named Counter pool.

| Counter Name | Suffix | Purpose | Direction | Programs | Load-bearing? |
|-------------|--------|---------|-----------|----------|---------------|
| `BANKZACCT` + sort-code | 6-char sort-code appended | Account number sequence | BOTH | CREACC (ENQ, GET/INC, DEQ — lines ~188–420), INQACC (read-only query — lines ~142–160) | Yes |
| `BANKZCUST` + sort-code | 6-char sort-code appended | Customer number sequence | BOTH | CRECUST (ENQ, GET/INC, DEQ — lines ~174–500), BANKDATA (init seed) | Yes |

References: `src/base/cics/cobol/CREACC.cbl:188`, `src/base/cics/cobol/CRECUST.cbl:174`, `src/base/cics/cobol/INQACC.cbl:142`

---

## 5. CICS Transactions and Internal Linkage

### 5.1 CICS Transaction IDs (hardcoded)

| TRANSID | Entry Program | Description | Defined in |
|---------|--------------|-------------|------------|
| `OMEN` | BNKMENU | Main menu | `src/base/cics/cobol/BNKMENU.cbl` |
| `OCAC` | BNK1CAC | Create account screen | `src/base/cics/cobol/BNK1CAC.cbl` |
| `OCCS` | BNK1CCS | Create customer screen | `src/base/cics/cobol/BNK1CCS.cbl` |
| `OCRA` | BNK1CRA | Debit/credit screen | `src/base/cics/cobol/BNK1CRA.cbl` |
| `ODAC` | BNK1DAC | Display/delete account screen | `src/base/cics/cobol/BNK1DAC.cbl` |
| `ODCS` | BNK1DCS | Display/update/delete customer screen | `src/base/cics/cobol/BNK1DCS.cbl` |
| `OTFN` | BNK1TFN | Transfer funds screen | `src/base/cics/cobol/BNK1TFN.cbl` |
| `OUAC` | BNK1UAC | Update account screen | `src/base/cics/cobol/BNK1UAC.cbl` |
| `OCR1`–`OCR5` | CRDTAGY1–CRDTAGY5 | Async credit-agency checks | `src/base/cics/cobol/CRECUST.cbl:627` |

### 5.2 EXEC CICS LINK (synchronous program-to-program)

| Caller | Callee | Purpose | Source line |
|--------|--------|---------|-------------|
| BNK1CAC | CREACC | Create account | `src/base/cics/cobol/BNK1CAC.cbl` |
| BNK1CCA | INQACCCU | Inquire accounts for customer | `src/base/cics/cobol/BNK1CCA.cbl` |
| BNK1CCS | CRECUST | Create customer | `src/base/cics/cobol/BNK1CCS.cbl` |
| BNK1CRA | DBCRFUN | Debit/credit function | `src/base/cics/cobol/BNK1CRA.cbl` |
| BNK1DAC | INQACC | Inquire account | `src/base/cics/cobol/BNK1DAC.cbl` |
| BNK1DAC | DELACC | Delete account | `src/base/cics/cobol/BNK1DAC.cbl` |
| BNK1DCS | INQCUST | Inquire customer | `src/base/cics/cobol/BNK1DCS.cbl` |
| BNK1DCS | DELCUS | Delete customer | `src/base/cics/cobol/BNK1DCS.cbl` |
| BNK1DCS | UPDCUST | Update customer | `src/base/cics/cobol/BNK1DCS.cbl` |
| BNK1TFN | XFRFUN | Transfer funds | `src/base/cics/cobol/BNK1TFN.cbl` |
| BNK1UAC | UPDACC | Update account | `src/base/cics/cobol/BNK1UAC.cbl` |
| CREACC | INQCUST | Validate customer exists | `src/base/cics/cobol/CREACC.cbl` |
| CREACC | INQACCCU | Check existing accounts | `src/base/cics/cobol/CREACC.cbl` |
| CRECUST | INQCUST | Check customer does not already exist | `src/base/cics/cobol/CRECUST.cbl` |
| DELCUS | INQCUST | Verify customer | `src/base/cics/cobol/DELCUS.cbl` |
| DELCUS | INQACCCU | Get accounts to delete | `src/base/cics/cobol/DELCUS.cbl` |
| DELCUS | DELACC | Delete each account | `src/base/cics/cobol/DELCUS.cbl` |
| INQACCCU | INQCUST | Validate customer | `src/base/cics/cobol/INQACCCU.cbl` |
| All business programs | ABNDPROC | Centralised abend handler | e.g. `src/base/cics/cobol/CREACC.cbl` |

### 5.3 EXEC CICS RUN TRANSID (asynchronous — credit check)

`CRECUST.cbl:687` issues `EXEC CICS RUN TRANSID(OCR1…OCR5) CHANNEL(CIPCREDCHANN)`.  
Five child transactions run in parallel, each reading container `CIPA`–`CIPE` from channel `CIPCREDCHANN` and writing a credit score back into the same container. `CRECUST` then collects results with `EXEC CICS FETCH CHILD`.

| Channel | Containers | Child TRANSIDs | Programs |
|---------|------------|----------------|----------|
| `CIPCREDCHANN` | CIPA, CIPB, CIPC, CIPD, CIPE | OCR1, OCR2, OCR3, OCR4, OCR5 | CRDTAGY1–CRDTAGY5 |

References: `src/base/cics/cobol/CRECUST.cbl:615`, `src/base/cics/cobol/CRDTAGY1.cbl` – `CRDTAGY5.cbl`

### 5.4 BMS Maps

| Mapset | Map | Screen Driver |
|--------|-----|---------------|
| BNK1MAI | BNK1ME | BNKMENU |
| BNK1CAM | BNK1CA | BNK1CAC |
| BNK1ACC | BNK1ACC | BNK1CCA |
| BNK1CCM | BNK1CC | BNK1CCS |
| BNK1CDM | BNK1CD | BNK1CRA |
| BNK1DAM | BNK1DA | BNK1DAC |
| BNK1DCM | BNK1DC | BNK1DCS |
| BNK1TFM | BNK1TF | BNK1TFN |
| BNK1UAM | BNK1UA | BNK1UAC |

All maps are in `src/base/cics/bms/`.

---

## 6. Batch Files (Sequential Datasets)

### 6.1 BNKSTMT Batch Job

JCL: `src/base/batch/jcl/BNKSTMT.jcl`  
Trigger: manual execution (`JOB CLASS=A`); no scheduler found in repository.  
DB2 plan: `BANKZPLN`; load library: `BANKZ.V0R1M0.LOAD`; subsystem: `DBD1`.

| DDname | Direction | Content | Notes |
|--------|-----------|---------|-------|
| `DATECARD` | IN | 80-byte fixed; first 6 chars = `YYYYMM` reporting month | `BNKSTMT.pli:133`; if absent, defaults to current month |
| `SORTCODE` | IN | 80-byte fixed; first 6 chars = 6-digit sort code | `BNKSTMT.pli:130`; if absent, defaults to `123456` (hard-coded fallback) |
| `SYSPRINT` | OUT | Report output — formatted monthly account statements | `BNKSTMT.jcl:14` |

Program: `src/base/batch/pli/BNKSTMT.pli` — reads `BANKZ.ACCOUNT` and `BANKZ.PROCTRAN` via DB2 cursors; reads `BANKZ.CUSTOMER` for name/address.

### 6.2 IMS Data-Load Flat Files (legacy-only)

Load programs read flat files via COBOL `SELECT … ASSIGN TO <ddname>`. These programs are load/init utilities not part of the online transaction flow.

| Program | DDname | Direction | Content |
|---------|--------|-----------|---------|
| LOADACCT | `ACCTIN` | IN | Account records for IMS ACCOUNT segment | 
| LOADCUSA | `CUSAIN` | IN | Customer-account link records for IMS CUSTACCS segment |
| LOADCUST | `CUSTIN` | IN | Customer records for IMS CUSTOMER segment |
| LOADHIST | `HISTIN` | IN | History records for IMS HISTORY segment |
| LOADTSTA | `TSTAIN` | IN | Transaction-status records for IMS TSTAT segment |

Sample data files: `src/base/ims/LoadData/` (excluded from DBB build). All load programs output to IMS databases via `CBLTDLI ISRT`.

---

## 7. z/OS Connect REST API Layer

### 7.1 Server Configuration

| Component | Value | Source |
|-----------|-------|--------|
| z/OS Connect version | 3.0 (`zosConnect-3.0` feature) | `src/api/src/main/liberty/config/server.xml:3` |
| HTTP port | 9080 | `src/api/src/main/liberty/config/http-endpoint.xml:5` |
| HTTPS port | 9443 | `.setup/config/config.yaml:109` |
| Frontend Liberty server HTTP port | 9081 | `.setup/config/config.yaml:114` |

### 7.2 CICS IPIC Connection

| Attribute | Value | Source |
|-----------|-------|--------|
| Connection ID | `bankzCicsConnection` | `src/api/src/main/liberty/config/cics.xml:7` |
| Host | `${CICS_HOST}` (env variable) | `src/api/src/main/liberty/config/cics.xml:7` |
| Port | `${CICS_PORT}` (env variable) | `src/api/src/main/liberty/config/cics.xml:7` |
| Auth | `${CICS_USER}` / `${CICS_PASSWORD}` | `src/api/src/main/liberty/config/cics.xml:9` |
| Default in config | `localhost:4320` | `.setup/config/config.yaml:104–107` |

CICS programs exposed via this connection:

| z/OS Asset | CICS Program | TRANSID | Type |
|------------|-------------|---------|------|
| CRECUST | CRECUST | OMEN | cicsCommarea-1.0 |
| DBCRFUN | DBCRFUN | (EIB_ONLY) | cicsCommarea-1.0 |
| INQACC | INQACC | (EIB_ONLY) | cicsCommarea-1.0 |
| INQACCCU | INQACCCU | (EIB_ONLY) | cicsCommarea-1.0 |
| INQCUST | INQCUST | (EIB_ONLY) | cicsCommarea-1.0 |
| UPDCUST | UPDCUST | (EIB_ONLY) | cicsCommarea-1.0 |

### 7.3 IMS Connect Connection

| Attribute | Value | Source |
|-----------|-------|--------|
| Connection ID | `imsConn` | `src/api/src/main/liberty/config/ims.xml:7` |
| Host | `${IMS_HOST}` (env variable) | `src/api/src/main/liberty/config/ims.xml:10` |
| Port | `${IMS_PORT}` (env variable) | `src/api/src/main/liberty/config/ims.xml:10` |
| IMS Datastore | `${IMS_DATASTORE}` (env variable) | `src/api/src/main/liberty/config/ims.xml:7` |
| Auth | `${IMS_USER}` / `${IMS_PASSWORD}` | `src/api/src/main/liberty/config/ims.xml:12` |
| Default in config | `localhost:9977` | `.setup/config/config.yaml:120–121` |

IMS programs exposed via this connection:

| z/OS Asset | IMS Transaction Code | Type |
|------------|---------------------|------|
| IBACSUM | IBACSUM | imsTransaction-1.0 |
| IBGCUDAT | IBGCUDAT | imsTransaction-1.0 |
| IBSCUDAT | IBSCUDAT | imsTransaction-1.0 |
| IBTRAN | IBTRAN | imsTransaction-1.0 |

### 7.4 OpenAPI Routes (`src/api/src/main/api/openapi.yaml`)

CICS-backed routes:

| Method | Path | Backend program |
|--------|------|----------------|
| POST | `/customers` | CRECUST |
| GET | `/customers/{customerId}` | INQCUST |
| GET | `/customers/{customerId}/accounts` | INQACCCU |
| GET | `/accounts` | INQACC |
| GET | `/accounts/{accountId}` | INQACC |
| GET | `/accounts/{accountId}/balances` | INQACC |
| POST | `/accounts/{accountId}/deposit` | DBCRFUN |
| GET | `/accounts/{accountId}/transactions` | INQACC |
| GET | `/accounts/{accountId}/transactions/{transactionId}` | INQACC |
| PUT | `/accounts/{accountId}` | UPDCUST |

IMS-backed routes:

| Method | Path | Backend program |
|--------|------|----------------|
| GET | `/ims/customers/{customerId}` | IBGCUDAT |
| GET | `/ims/customers/{customerId}/accounts` | IBACSUM |
| GET | `/ims/accounts/{accountId}` | IBACSUM |
| GET | `/ims/accounts/{accountId}/balances` | IBACSUM |
| POST | `/ims/accounts/{customerId}/{accountId}/deposit` | IBTRAN |

### 7.5 OAuth2 Endpoints (OpenAPI spec)

| Endpoint | URL | Notes |
|----------|-----|-------|
| Authorization | `https://auth.bankofz.example.com/oauth/authorize` | `src/api/src/main/api/openapi.yaml:627` |
| Token | `https://auth.bankofz.example.com/oauth/token` | `src/api/src/main/api/openapi.yaml:628` |

`bankofz.example.com` is an `example.com` domain — almost certainly a placeholder. See open-questions.

---

## 8. Node.js Frontend

| Item | Value | Direction | Source |
|------|-------|-----------|--------|
| Frontend HTTP port | `process.env.PORT` or `3001` | IN (browser) | `src/frontend/server.js:11` |
| API proxy target | `process.env.API_BASE_URL` or `http://localhost:9080` | OUT | `src/frontend/server.js:12` |
| Proxied URL prefixes | `/api/`, `/ims/`, `/customers`, `/accounts` | OUT | `src/frontend/server.js:38` |
| z/OS Liberty direct URL | `http://<hostname>:9080/api` (when port ≠ 3001) | OUT | `src/frontend/config.js:17` |
| Hard-coded sort-code default | `987654` | — | `src/frontend/config.js:21` |

Carbon Web Components CDN: `https://1.www.s81c.com/common/carbon/web-components/version/v2.47.0/` — referenced in HTML pages (e.g. `src/frontend/customer-delete.html:63`). This is a runtime dependency on IBM's CDN.

---

## 9. Java IMS Bridge (`nazare.jmp.*`)

Path: `src/base/ims/java/`  
Built with Gradle. Dependency on `com.ibm.db2:jcc:12.1.4.0` (`src/base/ims/java/build.gradle:36`) and `com.ibm.jzos` (JZOS field access).

| Class | Method | Direction | External System | Notes |
|-------|--------|-----------|-----------------|-------|
| `nazare.jmp.controller.InsertHist` | `insertHist(ByteBuffer)` | OUT | `IMSBANK.HISTORY` via DB2 JDBC | Called from `IBTRAN.cbl` via JNI (`src/base/ims/cobol/IBTRAN.cbl:21,490`) |
| `nazare.jmp.history.TransactionService` | `saveTransactionDetail()` | OUT | `IMSBANK.HISTORY` INSERT | `src/base/ims/java/src/main/java/nazare/jmp/history/TransactionService.java:140` |
| `nazare.jmp.history.TransactionService` | `getTransactionDetail()` | IN | `IMSBANK.HISTORY` SELECT | `src/base/ims/java/src/main/java/nazare/jmp/history/TransactionService.java:47` |
| `nazare.jmp.controller.IMSBankHistory` | `main()` | IN | `IMSBANK.HISTORY` SELECT (SSL/Type 4) | **Legacy-only test harness**; not called by any program |

DB2 connection:
- Production (Type 2 local): `jdbc:db2:DBD1LOC` (default; override via `-Ddb2.location`) — `TransactionService.java:27`
- Test harness (Type 4 remote + SSL): `jdbc:db2://${db2Hostname}:${db2Port}/${db2Location}` — `IMSBankHistory.java:70`

JNI entry point: `IBTRAN.cbl` calls `FindClass`, `GetStaticMethodId`, `NewDirectByteBuffer`, `CallStaticVoidMethod`, `DeleteLocalRef` — `src/base/ims/cobol/IBTRAN.cbl:509–560`. JNI environment populated by IMS Java region at program initialisation.

---

## 10. Hard-coded Values

| Value | Meaning | Source |
|-------|---------|--------|
| `987654` | Bank sort code | `src/base/cics/copy/SORTCODE.cpy:7` |
| `'CICS Bank Sample Application'` | Company name returned by GETCOMPY | `src/base/cics/cobol/GETCOMPY.cbl` |
| `DBD1` | DB2 subsystem ID (CICS JCL) | `.setup/jcl/cics/Db2-bind.jcl:23`, `.setup/jcl/cics/Db2-create.jcl:9` |
| `DBD1LOC` | DB2 location (IMS Java, Type 2) | `src/base/ims/java/src/main/java/nazare/jmp/history/TransactionService.java:27` |
| `BANKZPLN` | DB2 plan name (batch BNKSTMT) | `src/base/batch/jcl/BNKSTMT.jcl:17` |
| `BANKZPACK` | DB2 package name | `.setup/jcl/cics/Db2-bind.jcl:25` |
| `123456` | Default sort-code fallback in BNKSTMT | `src/base/batch/pli/BNKSTMT.pli:256` — different from `987654` in CICS; see open-questions |
| `IMS2` | IMS datastore / SSID | `.setup/config/config.yaml:124` |
| `BANKZGRP` | CICS resource group | `.setup/zconfig/bank-of-z-definitions.yaml:4` |

---

## 11. Build-time Dataset Dependencies (DB2 / Compiler Load Libraries)

These are compile/bind-time dependencies, not runtime integrations. Listed for completeness; values are from `.setup/build/datasets.yaml`.

| Variable | Dataset | Purpose |
|----------|---------|---------|
| `SDSNLOAD` | `DB2V13.SDSNLOAD` | DB2 precompiler / binder |
| `SDSNEXIT` | `DB2V13.SDSNEXIT` | DB2 exit routines |
| `SIGYCOMP` | `IGY.V6R5M0.SIGYCOMP` | COBOL compiler |
| `IBMZPLI` | `PLI.V6R2M0.SIBMZCMP` | PL/I compiler |
| `SDFHLOAD` | `CICSTS63.CICS.SDFHLOAD` | CICS load library |
| `SDFHCOB` | `CICSTS63.CICS.SDFHCOB` | CICS COBOL stubs |
| `SDFHPL1` | `DFH.V6R2M24P.CICS.SDFHPL1` | CICS PL/I stubs |
| `SCSQCOBC` | `CSQ.V9R1M0.SCSQCOBC` | IBM MQ COBOL library (referenced; **no MQ calls found in source**) |
| `SCSQPLIC` | `CSQ.V9R3M0.SCSQPLIC` | IBM MQ PL/I library (referenced; **no MQ calls found in source**) |

> MQ libraries appear in `datasets.yaml` but no `MQOPEN`, `MQGET`, or `MQPUT` calls are present in any COBOL or PL/I source file. This may indicate a future integration, a removed feature, or a template artefact. Logged as open-question.

---

## 12. Partner / Third-party Systems

| System | Reference | Purpose | Load-bearing? |
|--------|-----------|---------|---------------|
| Credit agencies (×5) | `CRDTAGY1.cbl`–`CRDTAGY5.cbl` | Simulate external credit-score lookups via CICS async channel | **Unknown** — stubs only; see open-question #9 |
| IBM Carbon Web Components CDN (`1.www.s81c.com`) | `src/frontend/customer-delete.html:63` (and other HTML pages) | Frontend UI components served from IBM CDN | Yes (runtime browser dependency) |
| OAuth2 identity provider (`auth.bankofz.example.com`) | `src/api/src/main/api/openapi.yaml:627` | Token issuance and authorisation | **Unknown** — `example.com` placeholder; see open-questions |

---

## 13. Batch Schedules / JCL Job Triggers

| Job | JCL | Trigger | Notes |
|-----|-----|---------|-------|
| `BNKSTMT` | `src/base/batch/jcl/BNKSTMT.jcl` | Manual (`JOB CLASS=A`); no scheduler artefact found | Monthly statement generation |
| `DB2BIND` | `.setup/jcl/cics/Db2-bind.jcl` | Manual / pipeline step | CICS DB2 package bind |
| `DB2CREAT` | `.setup/jcl/cics/Db2-create.jcl` | Manual / one-time setup | Create BANKZ DB2 database and tables |
| `DB2DROP` | `.setup/jcl/cics/Db2-drop.jcl` | Manual | Drop BANKZ tables |
| `DB2CRE` | `.setup/jcl/ims/Db2-create.jcl` | Manual / one-time setup | Create IMSBANK DB2 database and HISTORY table |
| IMS bind jobs | `.setup/jcl/ims/Db2-bind.j2` | Pipeline step | IMS DB2 plan bind (plans IBTRAN, IBGHIST, IBSHIST) |

No cron, TWS, or JES scheduler artefacts found in the repository. Batch is assumed to be submitted manually or by an external scheduler not captured here.
