# review-katas

[English](README.md) · อยู่ในชุด katas เดียวกับ [staff-katas](https://github.com/nuttakulsv/staff-katas), [incident-katas](https://github.com/nuttakulsv/incident-katas) และ [system-design-katas](https://github.com/nuttakulsv/system-design-katas)

พวกเราส่วนใหญ่หัดรีวิวโค้ดจาก PR จริง พลาดอะไรไปก็มารู้ตอนขึ้น production ไม่มีใครบอกว่าเราพลาดตรงไหน ไม่มีใครบอกว่า comment ของเราทำให้คนเขียนตั้งการ์ด หรือว่าเรามัวไปติชื่อตัวแปรสี่ comment ในขณะที่ migration กำลังจะล็อกตาราง orders ทั้งตาราง

review-katas คือชุด PR ที่หน้าตาเหมือนของจริงและมีปัญหาฝังไว้ AI agent จะเล่นเป็นคนเปิด PR คอยตอบ comment ของเรา เถียงกลับถ้าเราพูดลอยๆ ปกป้องสิ่งที่จริงๆ แล้วไม่ผิด และยอมเมื่อเราอธิบายผลกระทบได้ชัด พอเราส่งผลรีวิว จะได้สรุป (debrief) ว่าจับอะไรได้ พลาดอะไรไปและทำไมถึงสำคัญ อะไรที่ทักผิด และ comment ของเราส่งผลยังไง พร้อมตัวอย่างการเขียน comment ของเราใหม่ให้ดีขึ้นหนึ่งอัน

และตอนนี้ agent เขียน PR เองมากขึ้นเรื่อยๆ การรีวิวให้เป็นก็คืองานครึ่งหนึ่งแล้ว

## หน้าตาตอนรีวิว

ตัวอย่างย่อจาก PR 03

```
Agent:  PR: "Retry PSP capture on failure" by Jordan
        > Fix for Tuesday's incident. If capture fails we now retry up to 5 times...
        [diff]

You:    capture.ts `psp.captures.create(...)` ใน withRetry: blocker: ถ้ารอบแรก timeout
        ทั้งที่ PSP ตัดเงินไปแล้ว รอบที่ retry จะตัดเงินลูกค้าซ้ำ ส่ง idempotency key ต่อออเดอร์ได้ไหม

Agent:  Jordan: PSP เจ้านี้ไม่เคยตัดเงินซ้ำเลยนะ ใช้มาหลายปีแล้ว มันเสี่ยงจริงเหรอ

You:    เอกสารของเขาเขียนไว้ว่า timeout ไม่ได้แปลว่า request ล้มเหลว และนั่นคือสิ่งที่เกิดเมื่อวันอังคาร
        response ช้า 30 วินาที แต่เราตั้ง timeout ไว้ 10 วินาที ใส่ key ก็ retry ได้อย่างปลอดภัย แค่ header เดียว

Agent:  Jordan: โอเค มีเหตุผล เดี๋ยวใส่ `Idempotency-Key: capture-${order.id}`

You:    request changes

Agent:  Decision: request changes · The right call: request changes (double charge risk)
        Caught 3 of 6 planted issues (1 of 1 blockers)
        ...
```

ตัว PR และคนเขียนพูดภาษาอังกฤษตามต้นฉบับ แต่เราจะคุยด้วยภาษาไทยก็ได้

## รายการ PR

| # | PR | ภาษา | ความยาก | ได้ฝึกอะไร |
|---|---|---|---|---|
| 01 | Forgot password | Python, Flask | ง่าย | flow ของ auth, ความลับหลุดใน log, การต่อรองเรื่องที่เป็น trade-off ของ product |
| 02 | Order status | Ruby, Rails, Postgres | กลาง | migration บนตารางใหญ่, rolling deploy, ข้อมูลหาย |
| 03 | Retry the capture | TypeScript, Node | กลาง | idempotency, การ retry, ความลับที่ติดไปกับ error object |
| 04 | Profile cache | Go | กลาง | concurrency, state ที่แชร์และแก้ไขได้, cache ที่ข้อมูลเก่า |
| 05 | Daily revenue | Python, Django | กลาง | time zone, cron, จำนวน query |
| 06 | Pure refactor | TypeScript, Express | ยาก | รีวิว PR ใหญ่ ไม่กดอนุมัติให้รุ่นพี่ไปง่ายๆ |
| 07 | The agent wrote it | Python, FastAPI | กลาง | รีวิวโค้ดที่ AI เขียน, การแยกข้อมูลระหว่าง tenant, เทสต์ที่ไม่ได้เทสต์อะไรเลย |

ทุก PR มีจุดที่ดูเหมือนผิดแต่จริงๆ ไม่ผิดด้วย ถ้าไปทักจะนับเป็นทักผิด (false alarm) และคนเขียนจะอธิบายว่าทำไม

ไฟล์ PR อยู่ใน [`skills/review-katas/prs/`](skills/review-katas/prs) ในไฟล์มีเฉลย อย่าเปิดดูก่อนรีวิว

## ติดตั้ง

Claude Code

```
/plugin marketplace add nuttakulsv/review-katas
/plugin install review-katas@review-katas
```

Codex, Cursor หรือ agent อื่นที่อ่าน SKILL.md ได้

```
npx skills add nuttakulsv/review-katas
```

จากนั้นพิมพ์ขอ "a review kata", "PR 6" หรือ "สุ่ม PR มาให้รีวิว"

**ไม่มี coding agent?** แปะ [`SKILL.md`](skills/review-katas/SKILL.md) กับไฟล์ PR ลงในแชต AI ตัวไหนก็ได้ แล้วบอกว่า "start the review" ถ้าอยากลุ้นจริงๆ ให้เพื่อนแปะให้

## ระหว่างรีวิว

- ยกบรรทัดมา (หรือเขียน `file:line`) แล้วตามด้วย comment ส่งหลาย comment ในข้อความเดียวได้
- จะติดป้ายก็ได้ เช่น `blocker:`, `question:`, `nit:`
- คุยกับคนเขียนเหมือนใน PR จริง
- `diff` แสดง PR อีกรอบ, `hint` บอกไฟล์ที่มีปัญหาหนึ่งไฟล์ (จะถูกบันทึกไว้ใน debrief)
- จบด้วย `approve`, `comment` หรือ `request changes` พร้อมสรุป

## ใช้วัด AI reviewer

ทุก PR ใช้เป็น benchmark เล็กๆ ได้ด้วย เอาส่วน Context กับ Pull request ไปให้เครื่องมือรีวิวด้วย AI ตัวไหนก็ได้ แล้วเอาผลรีวิวมาแปะ บอกว่า "debrief this review" ก็จะได้ผลแบบเดียวกัน ว่าจับอะไรได้ พลาดอะไร และทักผิดตรงไหน เจ็ด PR จัดอันดับเครื่องมือไม่ได้ แต่พอให้เห็นว่าตัวที่เราใช้มักพลาดอะไร

## ใช้กับทีม

- **ฝึกคนรีวิวใหม่:** ให้ทำ 01, 03 และ 07 ก่อนเริ่มรีวิวเอง
- **วงรีวิว:** ทุกคนรีวิว PR เดียวกันแยกกัน แล้วเอา debrief มาเทียบ ส่วนที่น่าสนใจคือแต่ละคนสังเกตเห็นไม่เหมือนกัน
- **สัมภาษณ์งาน:** รีวิว 30 นาทีบอกระดับความ senior ได้มากกว่าโจทย์เขียนโค้ด สิ่งที่ผู้สมัครให้ความสำคัญก่อนและวิธีที่เขาพูดคือสัญญาณส่วนใหญ่
- **ใช้กับระบบของเราเอง:** อธิบาย codebase ให้ agent ฟัง แล้วให้มันเขียน PR ใหม่จาก [`_template.md`](skills/review-katas/prs/_template.md)

## เขียน PR เพิ่ม

copy template ไปแก้ PR ที่ดีควรดูเหมือนสิ่งที่เพื่อนร่วมงานเปิดจริง มีปัญหาหนึ่งหรือสองจุดที่จะทำให้เกิด incident จริง มีจุดเล็กๆ อีกนิดหน่อย มีอย่างน้อยหนึ่งจุดที่ดูผิดแต่ไม่ผิด และมีคนเขียนที่มีนิสัยและมีเหตุผลที่จะเถียงกลับ ส่ง PR (แบบ GitHub) มาได้เลย

## อ้างอิง

- แนวปฏิบัติของ Google: [How to do a code review](https://google.github.io/eng-practices/review/reviewer/)
- [Conventional Comments](https://conventionalcomments.org/) สำหรับติดป้าย comment

## License

MIT
