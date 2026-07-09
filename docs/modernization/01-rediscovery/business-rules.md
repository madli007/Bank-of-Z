# Business Rules

> Rediscovery outputs. Rules below describe observed behavior only.

## CICS financial operations (`DBCRFUN`, `XFRFUN`)

- `DBCRFUN` starts each request as failure (`COMM-SUCCESS='N'`, `COMM-FAIL-CODE='0'`) and overwrites incoming sort-code with the hard-coded bank sort-code `987654`. This means account lookup is always constrained to that sort-code. Sources: `src/base/cics/cobol/DBCRFUN.cbl:201`, `src/base/cics/cobol/DBCRFUN.cbl:202`, `src/base/cics/cobol/DBCRFUN.cbl:211`, `src/base/cics/cobol/DBCRFUN.cbl:212`, `src/base/cics/copy/SORTCODE.cpy:7`.
  Worked example: input `COMM-ACCNO=00001234`, `COMM-SORTC=123456`, `COMM-AMT=50.00` -> program changes sort-code to `987654` before `SELECT ACCOUNT`, so it searches `987654/00001234`.

- `DBCRFUN` maps account lookup failures as: not found (`SQLCODE=+100`) -> `COMM-FAIL-CODE='1'`; any other SQL error -> `COMM-FAIL-CODE='2'`; both keep `COMM-SUCCESS='N'`. Sources: `src/base/cics/cobol/DBCRFUN.cbl:246`, `src/base/cics/cobol/DBCRFUN.cbl:281`, `src/base/cics/cobol/DBCRFUN.cbl:283`, `src/base/cics/cobol/DBCRFUN.cbl:284`, `src/base/cics/cobol/DBCRFUN.cbl:286`.
  Worked example: if `ACCOUNT` row for `987654/00001234` does not exist, output includes `COMM-SUCCESS='N'`, `COMM-FAIL-CODE='1'`.

- In `DBCRFUN`, payment-origin debits from `MORTGAGE` or `LOAN` accounts are rejected with `COMM-FAIL-CODE='4'`. The condition is account type plus `COMM-FACILTYPE=496`. Sources: `src/base/cics/cobol/DBCRFUN.cbl:330`, `src/base/cics/cobol/DBCRFUN.cbl:331`, `src/base/cics/cobol/DBCRFUN.cbl:333`, `src/base/cics/cobol/DBCRFUN.cbl:335`.
  Worked example: `COMM-AMT=-25.00`, account type `LOAN`, `COMM-FACILTYPE=496` -> rejected (`COMM-SUCCESS='N'`, `COMM-FAIL-CODE='4'`) before balance update.

- In `DBCRFUN`, payment-origin debits enforce non-negative available funds: if `(ACCOUNT_AVAILABLE_BALANCE + COMM-AMT) < 0` and `COMM-FACILTYPE=496`, request fails with `COMM-FAIL-CODE='3'`. Sources: `src/base/cics/cobol/DBCRFUN.cbl:344`, `src/base/cics/cobol/DBCRFUN.cbl:347`.
  Worked example: available `100.00`, `COMM-AMT=-150.00`, `COMM-FACILTYPE=496` -> computed difference `-50.00`; output `COMM-SUCCESS='N'`, `COMM-FAIL-CODE='3'`.

- `DBCRFUN` repeats the `MORTGAGE/LOAN + COMM-FACILTYPE=496` block outside the debit-only branch, so both debit and credit payment-origin requests for those account types are rejected with fail code `4`. Sources: `src/base/cics/cobol/DBCRFUN.cbl:307`, `src/base/cics/cobol/DBCRFUN.cbl:368`, `src/base/cics/cobol/DBCRFUN.cbl:371`, `src/base/cics/cobol/DBCRFUN.cbl:373`.
  Worked example: `COMM-AMT=+75.00`, account type `MORTGAGE`, `COMM-FACILTYPE=496` -> rejected with `COMM-FAIL-CODE='4'`.

- For accepted `DBCRFUN` requests, both available and actual balances are adjusted by adding `COMM-AMT` (negative values reduce balances, positive values increase them), then written back to `ACCOUNT`. Sources: `src/base/cics/cobol/DBCRFUN.cbl:382`, `src/base/cics/cobol/DBCRFUN.cbl:384`, `src/base/cics/cobol/DBCRFUN.cbl:393`, `src/base/cics/cobol/DBCRFUN.cbl:415`, `src/base/cics/cobol/DBCRFUN.cbl:416`.
  Worked example: available `1000.00`, actual `900.00`, `COMM-AMT=-120.00` -> updated to available `880.00`, actual `780.00`.

- `DBCRFUN` writes one `PROCTRAN` row after account update. Type/description mapping is: debit counter -> `DEB` + `COUNTER WTHDRW`; debit payment -> `PDR` + first 14 chars of `COMM-ORIGIN`; credit counter -> `CRE` + `COUNTER RECVED`; credit payment -> `PCR` + first 14 chars of `COMM-ORIGIN`. Amount is stored as `COMM-AMT`. Sources: `src/base/cics/cobol/DBCRFUN.cbl:491`, `src/base/cics/cobol/DBCRFUN.cbl:493`, `src/base/cics/cobol/DBCRFUN.cbl:498`, `src/base/cics/cobol/DBCRFUN.cbl:499`, `src/base/cics/cobol/DBCRFUN.cbl:500`, `src/base/cics/cobol/DBCRFUN.cbl:506`, `src/base/cics/cobol/DBCRFUN.cbl:511`, `src/base/cics/cobol/DBCRFUN.cbl:512`, `src/base/cics/cobol/DBCRFUN.cbl:513`, `src/base/cics/cobol/DBCRFUN.cbl:519`, `src/base/cics/copy/PROCTRAN.cpy:34`, `src/base/cics/copy/PROCTRAN.cpy:35`, `src/base/cics/copy/PROCTRAN.cpy:45`, `src/base/cics/copy/PROCTRAN.cpy:46`.
  Worked example: `COMM-AMT=-20.00`, `COMM-FACILTYPE=496`, `COMM-ORIGIN='WEBAPP01USER0001...'` -> inserted type `PDR`, description `WEBAPP01USER00`, amount `-20.00`.

- If `DBCRFUN` cannot insert into `PROCTRAN`, it attempts `SYNCPOINT ROLLBACK`; if rollback also fails, it logs to `ABNDPROC` and abends with `HROL`. It then sets `COMM-SUCCESS='N'` and moves literal `'02'` into `COMM-FAIL-CODE` (field is `PIC X`, so only one character is retained). Sources: `src/base/cics/cobol/DBCRFUN.cbl:564`, `src/base/cics/cobol/DBCRFUN.cbl:620`, `src/base/cics/cobol/DBCRFUN.cbl:633`, `src/base/cics/cobol/DBCRFUN.cbl:636`, `src/base/cics/cobol/DBCRFUN.cbl:637`, `src/base/cics/cobol/DBCRFUN.cbl:185`, `src/base/cics/copy/PAYDBCR.cpy:20`.
  Worked example: account update succeeds but `INSERT PROCTRAN` fails -> transaction rolls back; return is failure with single-character fail-code storage.

- `XFRFUN` normalizes both from/to sort-codes to hard-coded `987654` and rejects non-positive transfer amounts (`COMM-AMT <= 0`) with `COMM-FAIL-CODE='4'`. Sources: `src/base/cics/cobol/XFRFUN.cbl:281`, `src/base/cics/cobol/XFRFUN.cbl:289`, `src/base/cics/cobol/XFRFUN.cbl:291`, `src/base/cics/copy/SORTCODE.cpy:7`.
  Worked example: input `COMM-FSCODE=111111`, `COMM-TSCODE=222222`, `COMM-AMT=0.00` -> both sort-codes reset to `987654`; request rejected with fail code `4`.

- `XFRFUN` does not allow same-account transfer (`FROM` account+sort equals `TO` account+sort). This path links to `ABNDPROC` and issues `ABCODE('SAME')` abend rather than returning a normal fail-code response. Sources: `src/base/cics/cobol/XFRFUN.cbl:316`, `src/base/cics/cobol/XFRFUN.cbl:364`, `src/base/cics/cobol/XFRFUN.cbl:371`, `src/base/cics/cobol/XFRFUN.cbl:257`.
  Worked example: from `987654/00000001` to `987654/00000001` -> abend `SAME`.

- `XFRFUN` chooses update order by account number: if `FROM < TO`, it updates FROM first then TO; otherwise TO first then FROM. This enforces a deterministic lock order. Sources: `src/base/cics/cobol/XFRFUN.cbl:378`, `src/base/cics/cobol/XFRFUN.cbl:389`, `src/base/cics/cobol/XFRFUN.cbl:398`, `src/base/cics/cobol/XFRFUN.cbl:600`, `src/base/cics/cobol/XFRFUN.cbl:610`.
  Worked example: transfer from `00001000` to `00002000` -> debit `00001000` first, then credit `00002000`.

- In `XFRFUN`, FROM-account handling is: `SELECT` not found -> fail code `1`; other read/update SQL failures -> fail code `3`; successful FROM update subtracts transfer amount from both balances. Sources: `src/base/cics/cobol/XFRFUN.cbl:967`, `src/base/cics/cobol/XFRFUN.cbl:968`, `src/base/cics/cobol/XFRFUN.cbl:970`, `src/base/cics/cobol/XFRFUN.cbl:986`, `src/base/cics/cobol/XFRFUN.cbl:989`, `src/base/cics/cobol/XFRFUN.cbl:1017`.
  Worked example: FROM available `500.00`, actual `500.00`, `COMM-AMT=75.00` -> FROM becomes `425.00/425.00`.

- In `XFRFUN`, TO-account handling is: not found (`+100`) -> fail code `2` and immediate rollback to avoid partial transfer; other SQL failures -> fail code `3`. On `SQLCODE=-911` with `SQLERRD(3)=13172872`, program retries up to 5 times (`DB2-DEADLOCK-RETRY < 6`) with rollback and 1-second delay, then restarts section `UPDATE-ACCOUNT-DB2`. Sources: `src/base/cics/cobol/XFRFUN.cbl:1102`, `src/base/cics/cobol/XFRFUN.cbl:1103`, `src/base/cics/cobol/XFRFUN.cbl:1111`, `src/base/cics/cobol/XFRFUN.cbl:1188`, `src/base/cics/cobol/XFRFUN.cbl:1197`, `src/base/cics/cobol/XFRFUN.cbl:1199`, `src/base/cics/cobol/XFRFUN.cbl:1201`, `src/base/cics/cobol/XFRFUN.cbl:1203`, `src/base/cics/cobol/XFRFUN.cbl:1282`.
  Worked example: TO row lock conflict (`-911`, reason `13172872`) on first attempt -> rollback, wait 1 second, retry transfer flow; after 5 retries, falls into abend path.

- `XFRFUN` explicitly states no overdraft-limit check, and debit logic subtracts from FROM balances without testing available-balance floor. Sources: `src/base/cics/cobol/XFRFUN.cbl:21`, `src/base/cics/cobol/XFRFUN.cbl:986`, `src/base/cics/cobol/XFRFUN.cbl:989`.
  Worked example: FROM available `50.00`, `COMM-AMT=100.00` can produce `-50.00` available balance if SQL updates succeed.

- After both account updates, `XFRFUN` writes one `PROCTRAN` record for the FROM account: type is set via `PROC-TY-TRANSFER` (`'TFR'`), description is a packed transfer descriptor with header `TRANSFER` plus TO sort-code and TO account, and amount is transfer amount. Sources: `src/base/cics/cobol/XFRFUN.cbl:1582`, `src/base/cics/cobol/XFRFUN.cbl:1603`, `src/base/cics/cobol/XFRFUN.cbl:1607`, `src/base/cics/cobol/XFRFUN.cbl:1609`, `src/base/cics/cobol/XFRFUN.cbl:1610`, `src/base/cics/cobol/XFRFUN.cbl:1612`, `src/base/cics/copy/PROCTRAN.cpy:47`, `src/base/cics/copy/PROCTRAN.cpy:51`, `src/base/cics/copy/PROCTRAN.cpy:52`.
  Worked example: transfer `25.00` from `...0001` to `...0002` -> one `PROCTRAN` row on FROM account with type `TFR`, descriptor encoding destination `98765400000002`.

- If `XFRFUN` cannot insert into `PROCTRAN`, it logs to `ABNDPROC` and abends with code `WPCD` (it does not attempt a local fail-code return on that path). Sources: `src/base/cics/cobol/XFRFUN.cbl:1701`, `src/base/cics/cobol/XFRFUN.cbl:1717`, `src/base/cics/cobol/XFRFUN.cbl:257`.

- Both programs include storm-drain diagnostics for DB2 connection-loss SQLCODE 923 by setting a condition string and emitting a diagnostic message. Sources: `src/base/cics/cobol/DBCRFUN.cbl:678`, `src/base/cics/cobol/DBCRFUN.cbl:686`, `src/base/cics/cobol/XFRFUN.cbl:1746`, `src/base/cics/cobol/XFRFUN.cbl:1747`, `src/base/cics/cobol/XFRFUN.cbl:1756`.