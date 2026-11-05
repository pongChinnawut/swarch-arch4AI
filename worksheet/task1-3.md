# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** ชินนวุฒิ แก้วฉิม

**Student ID / รหัสนิสิต:** 6870063021

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 493 | 599 | 93.5% | 100% |
| t0-small | 10 | 21 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

> Large model เพิ่ม accuracy จาก 72.5% เป็น 93.5% แต่ p50 เพิ่มจาก 10 เป็น 493 ms และต้องจ่ายค่า large model ทุกรีวิว เทียบกับ small ที่ไม่เรียก large model เลย

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 26.4 | 238 | 443 | 20 | 0 | 92.8% |
| t1-after | 9.5 | 393 | 1009 | 0 | 25 | 92.6% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

> p50 คำนวณเฉพาะ request ที่สำเร็จ จึงไม่รวม 20 requests ที่ล้มเหลว และ request ที่ค้างยังทำให้โหลดที่เดินหน้าต่อได้ลดลงด้วย ต้องดู errors และเวลา run ประกอบ: t1-before มี 20 errors และใช้เวลา 26.4 วินาที เทียบกับ baseline ที่ไม่มี errors และใช้ 10.0 วินาที

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

> Request ที่ค้างยังถือ connection ใน connection pool, socket และหน่วยความจำของ coroutine/request ทั้งใน review service และ gateway เมื่อสะสมจะลดทรัพยากรที่เหลือให้ request ใหม่และอาจทำให้ต้องรอหรือ timeout ต่อเนื่อง แม้ async จะไม่ได้บล็อกทั้ง event loop ก็ตาม

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

> เก็บ pending review อย่างคงทน แล้วให้ worker retry และติด label ภายหลัง โดยไม่ทำรีวิวหาย ใช้ T4 asynchronous/batch ผ่าน queue หรือการอ่านรายการ pending เป็นระยะ พร้อมป้องกันการประมวลผลซ้ำ

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

> Timeout 0.1 วินาทีสั้นกว่าระยะทำงานปกติของ large model ประมาณ 0.2 วินาที จึงทำให้รีวิวปกติจำนวนมากกลายเป็น pending ส่วน 10 วินาทีถือทรัพยากรนาน ผิด QR2 และอาจชน timeout 10 วินาทีของ client จนโพสต์รีวิวล้มเหลว

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 5 | 545 | 91.5% | 38% |
| t2-0.7 | 0.7 | 372 | 583 | 93.0% | 53% |
| t2-0.8 | 0.8 | 479 | 707 | 93.5% | 64% |
| t2-0.9 | 0.9 | 475 | 593 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

> เลือก threshold 0.6 สำหรับ Scenario A เพราะ t2-0.6 มี accuracy 91.5% ผ่าน QR3 และเรียก large model 38% หรือประมาณ 15.2 calls/s ที่ 40 รีวิว/s ซึ่งไม่เกิน capacity 20 calls/s ต้องยอมเสีย accuracy 2 จุดเปอร์เซ็นต์เทียบกับ t0-large และต้องตรวจ precision ของ negative alerts เพิ่มก่อนใช้งานจริง

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

> ที่ threshold 0.6 รีวิวส่วนใหญ่ตอบด้วย small model จึงลด p50 จาก 493 เหลือ 5 ms แต่กลุ่มที่ช้าที่สุดยังต้องรอ large model และคิว ทำให้ p95 ลดเพียงจาก 599 เหลือ 545 ms การลดเวลาของกลุ่มที่เร็วอยู่แล้วไม่ได้กำจัด tail latency ของกลุ่มที่ต้อง escalate

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

> เมื่อเจอประโยคปฏิเสธหรือคำประชดนอกข้อมูลฝึก small model อาจมั่นใจสูงแต่ทำนายผิด เช่น อ่าน “not bad” ว่าเป็น negative ทำให้ cascade ไม่ส่งต่อไป large model เจ้าของร้านอาจได้รับ alert ผิดและเสียความเชื่อถือในระบบ

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 5 | 586 | 91.5% | 31% | 36% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

> รีวิวที่มีข้อความซ้ำกับรายการที่ยังอยู่ใน cache ได้ประโยชน์ รวมถึงข้อความที่ต่างกันเฉพาะตัวพิมพ์หรือช่องว่าง ข้อความใหม่ที่ไม่เคยมี key อยู่ใน cache จะ miss และรีวิวที่มีความหมายเหมือนกันแต่ใช้คำต่างกันก็ไม่ hit ด้วย cache แบบนี้

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

> ต้อง invalidate cache ของโมเดลเก่าเพื่อไม่ให้ใช้ผลทำนายเก่าต่อ ออกแบบ key ให้มี model version รวมถึงเวอร์ชัน normalization และ configuration ที่เปลี่ยนผลทำนาย เช่น strategy/threshold แล้วแยก namespace หรือ clear cache เมื่อเปลี่ยนเวอร์ชัน

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

> จากการวัดพบว่า p95 ที่ดูดีขึ้นอาจซ่อน requests ที่ล้มเหลว: t1-before มี p95 443 ms แต่ errors 20 ขณะที่ t1-after ไม่มี errors แต่ p95 ประมาณ 1,009 ms จึงต้องอ่าน latency คู่กับ errors, pending และขอบเขตข้อมูลที่ใช้คำนวณเสมอ
