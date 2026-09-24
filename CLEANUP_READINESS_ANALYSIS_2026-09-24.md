# รายงานวิเคราะห์ความพร้อม Cleanup — 24 กันยายน 2026

## ขอบเขตและข้อจำกัด

การตรวจนี้ใช้ไฟล์ใน repository เท่านั้น: โค้ด production ล่าสุดใน
`1_โค้ดทุกไฟล์_GAS`, workbook snapshot
`3_XLSX_โครงสร้าง/Phaopanya_Master_Data_23_09_2026.xlsx`, แผนปฏิบัติการ และ
Cleanup Suite rev.2. จึงยืนยันได้ว่า **ไฟล์ที่ส่งมาสอดคล้องกันหรือไม่** แต่ไม่
อาจยืนยันสถานะ Apps Script หรือ Google Sheet ที่กำลังรันจริงได้โดยไม่มีการเข้าถึง
Spreadsheet ID นั้น.

## คำตัดสิน

**อนุมัติให้ทำ Step 00--04 และ dry-run ได้; ห้าม Apply หรือ Promote** จนกว่าจะ
ปิด Implementation Gate ในแผน. Cleanup Suite rev.2 ลดความเสี่ยงเชิงข้อมูลจากชุด
แรกอย่างมีนัยสำคัญ แต่ยังไม่มี row-level `CLEANUP_AUDIT`, allow-list ของ
Spreadsheet ID, หรือ atomic audit-before-mutation ตามที่แผนบังคับ.

## ผลตรวจข้อมูล snapshot

| รายการ | ผลจาก workbook 23/09/2026 | ข้อสรุป |
|---|---:|---|
| จำนวนชีต | 10 | ครอบคลุม MASTER, index, geo dictionary และชีตงาน |
| `MASTER_PLACE` | 11,961 data rows | ตรง baseline ในแผน |
| `MD_ID` / `MATCH_KEY` ซ้ำ | 0 / 0 | ผ่าน identity gate |
| `STATUS` | `ACTIVE` 11,961 rows | ไม่มีสถานะอื่นใน snapshot |
| `PROVINCE` และ `AMPHOE` ว่าง | 1,702 / 1,702 | เป็นกลุ่มเป้าหมายของ Phase 3 |
| `AMPHOE != Amphoe_Khet` เมื่อทั้งคู่ไม่ว่าง | 383 | ตรง baseline Phase 2 |
| `PROVINCE != Changwat` เมื่อทั้งคู่ไม่ว่าง | 40 | ตรง baseline review |
| Postal (`Rahatpraisanee`) | 11,961 ค่า เป็นตัวเลขแบบ Excel เช่น `10130.0` | ตรวจรูปแบบด้วย numeric/string conversion ไม่ใช่ wildcard text |
| Artifact ท้าย MASTER | `AB=าา`, `AC=383`, `AD=กด`, และ header ว่างต่อท้าย | ต้องบันทึก baseline แล้วลบเฉพาะ AB:AD ที่ยืนยันก่อนเริ่ม |

การเปรียบเทียบ N/O กับ V/W ข้างต้นไม่นับกรณีที่ N/O ว่าง; หากนับ blank เป็น
ความต่างจะได้ 2,085 และ 1,742 ตามลำดับ ซึ่งเป็นเหตุให้ต้องระบุเงื่อนไขนี้ชัดเจน
ใน test/acceptance criteria.

## ผลตรวจชุดโค้ด

### Production baseline

* มี GAS ปัจจุบัน 10 ไฟล์: `00_*.gs` ถึง `06_*.gs`, `99_SelfTest.gs` และ
  `Service_SCG.gs`.
* โค้ด baseline ยังไม่มีสัญลักษณ์จาก patch ที่ต้องติดตั้ง: `SCRIPT_VERSION`,
  `pickGeoMatcher_`, `testPostalFormat_` และ FIX-A ที่ `geoExtractEn_`.
  จึงต้องถือว่า patch ทั้งสี่ชิ้น **ยังไม่ได้ apply ใน source package นี้**.
* `cleanThai` baseline มีอยู่แล้ว แต่ไม่มีหลักฐานในไฟล์ว่าเป็นรุ่น patch สำหรับ
  duplicate-prefix/phone; ห้ามทำ Phase 1b โดยยังไม่ replace ตาม patch contract.

### Cleanup Suite ที่ควรใช้

มีชุดที่ชื่อใกล้เคียงกันสองตำแหน่ง แต่ไม่เท่ากันทุกไฟล์. ชุดใน
`Phaopanya_GAS_Cleanup_Suite_rev2_24-09/.../Phaopanya_GAS_Cleanup_Suite_2026-09-23`
เป็น revision ใหม่กว่าและควรเป็นชุดเดียวที่ใช้ตรวจ/ติดตั้ง เพราะเพิ่มไฟล์
`59_CleanupPhase1b.gs` และเปลี่ยน Phase 1 เป็น 1a/1b.

สิ่งที่ rev.2 ทำได้ดี:

1. Phase 1a ไม่แก้ `MATCH_KEY`; Phase 1b จึงย้าย key พร้อมซ่อม
   `SYS_MASTER_IDX` และลบ ghost เป็นงานท้ายสุด.
2. Phase 2 เพิ่ม `NV_CONFLICT`, `KNN_THIN`, `KNN_FAR`, median-distance limit
   และ pilot 10 rows ก่อน apply ทั้งชุด.
3. รายงาน Phase 2 มี `MAPS_URL` เพื่อ review พิกัด และ config เก็บ index sheet
   ชัดเจน.
4. มี backup tab, full-file Drive backup, formula guard และ `LockService`.

## ช่องว่างที่ยัง block การ Apply

| ลำดับ | หลักฐานจากโค้ด rev.2 | ผลกระทบ | สิ่งที่ต้องทำก่อน Apply |
|---:|---|---|---|
| 1 | มีเพียง `CLEANUP_LOG` 4 คอลัมน์ (`เวลา`, `เฟส`, `สถานะ`, `รายละเอียด`) | ตามรอยได้ระดับ run ไม่ใช่ cell | สร้าง `CLEANUP_AUDIT` ด้วย 13 headers ตามแผน |
| 2 | ไม่มี config/guard สำหรับ allow-list Spreadsheet ID | เมนู Apply ใช้กับไฟล์ผิดหรือ production ได้ | default-deny และ allow เฉพาะ controlled-copy ID |
| 3 | Phase 1a/2/3 ใช้ `setValues` เขียนทั้งคอลัมน์; 1b/1c มี `deleteRows` | ไม่มี old/new record ที่ผูก `MD_ID`, และ rollback ราย row ไม่ได้ | build audit records ก่อน mutation; abort ถ้า audit write ล้มเหลว |
| 4 | restore เขียนเฉพาะขอบเขต backup; คอลัมน์ที่เพิ่มภายหลังยังคงอยู่ | restore ไม่เท่ากับ baseline แบบ full-schema | กำหนด/ทดสอบ restore contract และ verify data + schema |
| 5 | backup Drive URL ถูก log แต่ไม่ถูกบังคับบันทึกใน `CLEANUP_STATUS` ก่อน Apply | approval evidence ไม่ครบ | บันทึก controlled-copy ID/URL, backup URL และ approver ใน status |

## ลำดับดำเนินการที่ปลอดภัย

1. สร้าง **controlled cleanup copy** จาก production; ห้ามทำใน production.
2. บันทึก Spreadsheet ID และ URL ของ copy, ทำ Drive backup และเก็บ URL.
3. รัน Phase 0, export/เก็บ baseline metrics, และยืนยันว่า AB:AD เป็น test artifact
   ก่อนลบ. อย่าลบ header ว่าง/คอลัมน์อื่นโดยอัตโนมัติ.
4. ติดตั้ง Cleanup Suite **rev.2** 10 ไฟล์ และ patch SelfTest, SCRIPT_VERSION,
   FIX-A; เก็บ patch `cleanThai` ไว้คู่กับ Phase 1b ตาม contract.
5. ทำ dry-run ของ Phase 2 → pilot 10 rows → review `P2_FIX`/`MAPS_URL`; ทำ
   Phase 1a/3/4/5 เป็น dry-run ตามแผนเท่านั้น.
6. Implement และ integration-test gate ทั้งหมด: audit count = allowed-field diff,
   MD_ID/row count/protected fields, rollback-to-baseline, allow-list deny test.
7. หลัง reviewer อนุมัติ dry-run และ evidence ครบ จึงอนุญาต controlled-copy
   Apply ทีละ phase; Phase 1b ต้องทำ patch `cleanThai` วันเดียวกัน แล้วรัน
   Self-Test.

## เกณฑ์ตรวจรับที่ต้องเก็บเป็นหลักฐาน

* `MD_ID` ไม่เปลี่ยนและไม่มี duplicate; `MATCH_KEY` ไม่มี duplicate หลัง 1b.
* จำนวน audit record เท่ากับจำนวน cell/row mutation ที่อนุญาต และทุก record มี
  `RUN_ID`, operator, reason, evidence, confidence และ rollback status.
* Diff หลังแต่ละ phase จำกัดเฉพาะ field ที่แผนอนุญาต: 1a (ข้อความและ audit
  columns), 2 (`U/V/W/X/AA`), 3 (`N/O`), 1b/1c (key/index/merge ที่ได้รับ
  confirm).
* Full rollback จาก `BK_MASTER_*`/backup copy คืนทั้งค่า, row count, ID, header
  schema และ metrics เท่ากับ baseline.

## วิธีทำซ้ำการตรวจใน repository

```bash
# syntax ของ GAS ทุกไฟล์ (ส่งผ่าน stdin เพราะ Node ไม่รู้จักนามสกุล .gs)
while IFS= read -r -d '' f; do node --check < "$f"; done < <(find \
  1_โค้ดทุกไฟล์_GAS \
  Phaopanya_GAS_Cleanup_Suite_rev2_24-09/Phaopanya_GAS_Cleanup_Suite_2026-09-23/GAS \
  -name '*.gs' -print0)

# ตรวจว่าแผนระบุ gate และหา audit/guard/mutation ใน suite
rg -n -i 'CLEANUP_AUDIT|ALLOW.?LIST|SPREADSHEET.?ID|setValues|deleteRows' \
  ข้อมูลวิเคราะห์_GAS_Cleanup_2026-09-23/CLEANUP_EXECUTION_PLAN.md \
  Phaopanya_GAS_Cleanup_Suite_rev2_24-09/Phaopanya_GAS_Cleanup_Suite_2026-09-23/GAS
```
