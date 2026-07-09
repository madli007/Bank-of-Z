# Open Questions

> Questions that cannot be answered from source code alone. Require domain expert or production system access.

| # | Question | Evidence | Impact |
|---|----------|----------|--------|
| 1 | `UPDCUST.cbl` header says it accesses "the VSAM datastore" but the CICS setup JCL only creates DB2 tables; no VSAM KSDS DDL or DEFINE CLUSTER is present in the repo. Is customer data stored in VSAM, DB2, or both? | `src/base/cics/cobol/UPDCUST.cbl:14`; `.setup/jcl/cics/Db2-create.jcl` | Critical for data-model mapping |
| 2 | `IBLOGIN.pli` is described as a rewrite of `IBLOGIN1.cbl`. Which one is active in production — the COBOL or the PL/I version? Are both deployed under different transaction codes? | `src/base/ims/pli/IBLOGIN.pli:3`; `src/base/ims/cobol/IBLOGIN1.cbl:4` | Dead-code risk if one is superseded |
| 3 | No automated test suite was found. Do unit tests, integration tests, or z/OS batch test jobs exist outside this repository (e.g., in a separate test dataset or private repo)? | Entire repo search | Affects modernisation confidence |
| 4 | `src/base/cics/copy/WAZI.cpy` is an empty copybook with the comment "THIS IS JCs EDIT". Is this intentionally empty, a placeholder for future content, or dead code that should be removed? | `src/base/cics/copy/WAZI.cpy:8` | Minor — cleanup vs. intentional placeholder |
| 5 | `ABNDPROC.cbl` writes abend records to a "centralised CF (KSDS) datastore". What is the dataset name, and is this still operational in production? | `src/base/cics/cobol/ABNDPROC.cbl:11` | Infrastructure dependency |
| 6 | The sort-code is hard-coded as `987654` in `SORTCODE.cpy`. Is this value the same in production, or is it overridden at deploy time (e.g., via CICS resource definition or JCL symbolic)? | `src/base/cics/copy/SORTCODE.cpy:7` | Hard-coded value is a migration risk |
| 7 | `BANKDATA.cbl` is in `src/base/cics/cobol/` but has no `CBL CICS(...)` directive — it appears to be a standalone DB2 batch initialiser. Is it run as a CICS program, a batch job, or a utility? Does it have accompanying JCL? | `src/base/cics/cobol/BANKDATA.cbl:1` | Classification affects deployment scope |
| 8 | `IBTRAN.cbl` calls Java class `nazare.jmp.controller.InsertHist` via JNI. What JVM configuration is used in the IMS region, and is there a known latency or failure mode for this cross-language call? | `src/base/ims/cobol/IBTRAN.cbl:20`; `src/base/ims/copy/JNI.cpy` | Risk area for re-platforming |
| 9 | Five credit-agency stubs (`CRDTAGY1`–`CRDTAGY5`) simulate random credit scores. In production, are these replaced by real external credit-agency integrations, or are the stubs the production code? | `src/base/cics/cobol/CRDTAGY1.cbl:8` | Determines whether an external API contract exists |
| 10 | The CICS CONTROL table (`BANKZ.CONTROL`) is used for named counters (customer number, account number). What is the current maximum customer/account number in production, and is there any near-overflow concern? | `src/base/cics/copy/CONTROLI.cpy`; `src/base/cics/copy/CONTDB2.cpy` | Data-capacity planning |
