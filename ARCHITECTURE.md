# สถาปัตยกรรมระบบ n8n Private RAG

## ภาพรวมระบบ

ระบบ RAG (Retrieval-Augmented Generation) แบบ private ที่รันทั้งหมดบน Docker ประกอบด้วยบริการหลัก 6 ตัว:

| Service | Image | Port | หน้าที่ |
|---------|-------|------|---------|
| **n8n** | `n8nio/n8n:latest` | `5678` | Workflow automation / UI หลัก |
| **Docling** | `ghcr.io/docling-project/docling-serve` | `5001` | แปลง PDF → Markdown + ดึงรูปภาพ |
| **Qdrant** | `qdrant/qdrant` | `6333` | Vector database (เก็บ embedding) |
| **Ollama** | `ollama/ollama:latest` | `11434` | LLM (`llama3.2`) + Embedding (`qwen3-embedding:0.6b`) |
| **PostgreSQL** | `postgres:16-alpine` | internal | ฐานข้อมูล n8n |
| **nginx (static-files)** | `nginx:alpine` | `8080` | เสิร์ฟรูปภาพที่ดึงจาก PDF |

---

## โครงสร้าง Directory ที่สำคัญ

```
n8n-private-rag/
├── docker-compose.yml
├── .env
├── shared/                          # shared volume ระหว่าง docling กับ n8n
│   ├── docling-scratch/             # Docling ใช้เป็น working dir / บันทึกรูปชั่วคราว
│   ├── extracted-images/            # nginx เสิร์ฟรูปจากโฟลเดอร์นี้ที่ port 8080
│   └── rag-files/
│       ├── pending/                 # วาง PDF ที่ต้องการ ingest ไว้ที่นี่
│       └── processed/              # PDF ถูกย้ายมาที่นี่หลัง ingest เสร็จ
├── n8n/demo-data/workflows/         # workflow ที่ import อัตโนมัติตอน container ขึ้นครั้งแรก
└── workflows/                       # workflow files ทั้งหมด (version ต่าง ๆ)
    ├── 1_setup_qdrant.json
    ├── 2_document_ingestion.json
    ├── 3_rag_chat.json
    ├── RAG - 3. AI Chat (Private RAG) (1).json   # เวอร์ชันเก่า
    └── RAG - Full Workflow v2.json                 # เวอร์ชันล่าสุด (แนะนำใช้)
```

---

## Workflow 1: Setup Qdrant Collection

สร้าง/ลบ collection ใน Qdrant ด้วยมือ

```
[Manual Trigger] → PUT http://qdrant:6333/collections/multi-modal
                   { "vectors": { "size": 1024, "distance": "Cosine" } }

[Manual Trigger] → DELETE http://qdrant:6333/collections/multi-modal
```

- **Collection name**: `multi-modal`
- **Vector size**: `1024` dimensions
- **Distance metric**: `Cosine`

---

## Workflow 2: Document Ingestion (PDF → Docling → Qdrant)

### ขั้นตอนการทำงาน

```
[วาง PDF ใน pending/]
       ↓
[Local File Trigger] — detect ไฟล์ใหม่ใน /data/rag-files/pending/
       ↓
[Read PDF File] — อ่านไฟล์ PDF เป็น binary
       ↓
[Send to Docling (OCR)] — POST http://docling:5001/v1/convert/file/async
   - field: files (binary PDF)
   - field: image_export_mode = "referenced"
   → ได้ task_id กลับมา
       ↓
[Wait 5s] → [Check Docling Status] — GET /v1/status/poll/{task_id}
       ↓
[Check Status] — Switch node ตรวจ task_status:
   - "success"  → ต่อไป
   - "pending"  → วนกลับ Wait 5s
   - "started"  → วนกลับ Wait 5s
   - อื่น ๆ    → Stop and Error
       ↓ (success)
[HTTP Request] — GET http://docling:5001/v1/result/{task_id}
   → ได้ JSON ที่มี document.md_content (Markdown พร้อมอ้างอิงรูปภาพ)
       ↓
   ┌──────────────────────────────────┐
   │                                  │
[Transform Image Paths]        [Code in JavaScript]
   แปลง path รูปใน Markdown         ดึงชื่อไฟล์รูปทั้งหมด
   เป็น URL http://localhost:8080/    ออกมาเป็น array
       ↓                                  ↓
[Insert to Qdrant]              [Split Out] — แยกเป็นรายรูป
   + Document Loader                       ↓
   + Text Splitter (Markdown)      [Execute Command] — mv รูปจาก
   + Ollama Embeddings               docling-scratch/ → extracted-images/
       ↓                                  ↓
[Move PDF to Processed]          [Limit (1)] → [Move to Processed]
```

### รายละเอียด Transform Image Paths (จุดสำคัญ)

```javascript
// Markdown จาก Docling มีรูปแบบ:
// ![Image](image_000001_abc123.png)

markdownContent = markdownContent.replace(
  /!\[([^\]]*)\]\(([^)]+)\)/g,
  (match, alt, imgPath) => {
    const filename = imgPath.split('/').pop();
    return `![${alt}](http://localhost:8080/${filename})`;
    //                   ^^^^^^^^^^^^^^^^
    //         URL ที่ใช้เสิร์ฟรูปผ่าน nginx (port 8080)
  }
);
```

**⚠️ ปัญหาหลัก**: URL `http://localhost:8080/` หมายถึงเครื่องที่รัน browser ไม่ใช่เครื่องที่รัน Docker  
ถ้าเข้า n8n chat จากเครื่องอื่นในเครือข่าย รูปจะโหลดไม่ได้เพราะ browser จะพยายามโหลดจาก `localhost` ของตัวเอง

### Text Chunking Settings

| Parameter | Value |
|-----------|-------|
| Chunk size | 1500 characters |
| Chunk overlap | 150–300 characters |
| Split mode | Markdown |
| Embedding model | `qwen3-embedding:0.6b` via Ollama |

### Metadata ที่เก็บใน Qdrant

```json
{
  "source": "/data/rag-files/pending/filename.pdf",
  "filename": "filename.pdf",
  "processed_at": "2026-06-22T..."
}
```

---

## Workflow 3: AI Chat (RAG)

### ขั้นตอนการทำงาน

```
[User พิมพ์ใน Chat UI] — http://localhost:5678/webhook/rag-chat-webhook
       ↓
[AI Agent] (llama3.2, temperature=0.1)
   System prompt: "ให้ใช้ tool search_documents ค้นหาข้อมูลก่อนตอบ
                   หากพบรูปภาพให้แสดงในรูปแบบ Markdown: ![คำอธิบาย](URL)"
       ↓
   agent เรียก tool: search_documents
       ↓
[Qdrant Search Tool] — semantic search ใน collection "multi-modal"
   - topK: 5 (ดึง chunk ที่เกี่ยวข้องมา 5 อัน)
   - ใช้ Ollama Embeddings (qwen3-embedding:0.6b) แปลง query เป็น vector
       ↓
   ได้ Markdown chunks กลับมา (อาจมี ![alt](http://localhost:8080/image.png))
       ↓
[AI Agent สร้างคำตอบ]
   - ถ้ามีรูปใน chunk → คำตอบจะมี Markdown image syntax
   - n8n Chat UI render Markdown → แสดงรูปจาก URL
       ↓
[Window Buffer Memory] — จำบทสนทนา 10 turns ล่าสุด
```

---

## การไหลของรูปภาพ (Image Pipeline) แบบละเอียด

```
PDF มีรูปภาพ
    │
    ↓ POST /v1/convert/file/async (image_export_mode=referenced)
  Docling (container)
    │  สร้างไฟล์รูปที่:
    │  /shared/docling-scratch/image_000001_<hash>.png
    │
    ↓ Docling return: md_content มี "![Image](image_000001_<hash>.png)"
  n8n Code Node "Transform Image Paths"
    │  แปลงเป็น: "![Image](http://localhost:8080/image_000001_<hash>.png)"
    │
    ↓ Execute Command: mv /data/shared/docling-scratch/*.png /data/shared/extracted-images/
  ไฟล์รูปย้ายไปที่:
    ./shared/extracted-images/image_000001_<hash>.png
    │
    ↓ nginx container mount: ./shared/extracted-images → /usr/share/nginx/html
  nginx เสิร์ฟที่: http://<HOST>:8080/image_000001_<hash>.png
    │
    ↓ Markdown text (พร้อม URL) ถูก embed และเก็บใน Qdrant
  
[เมื่อ chat]
  Qdrant คืน chunk ที่มี: "![Image](http://localhost:8080/image_000001_<hash>.png)"
    │
    ↓ AI Agent ส่งกลับใน response
  n8n Chat UI render: <img src="http://localhost:8080/image_000001_<hash>.png">
    │
    ↓ Browser โหลดรูปจาก localhost:8080
  ✅ เครื่องที่รัน Docker เอง: โหลดได้
  ❌ เครื่องอื่น: โหลดไม่ได้ (localhost ชี้ไปที่ตัวเอง)
```

---

## สาเหตุที่เครื่องอื่นดูรูปไม่ได้

| จุด | ปัญหา |
|-----|-------|
| **URL ที่ embed ใน Qdrant** | `http://localhost:8080/...` hardcoded ตอน ingest |
| **localhost** | หมายถึงเครื่องที่รัน browser ไม่ใช่ Docker host |
| **เครื่องอื่น** | ไม่มี nginx ที่ port 8080 → 404 / connection refused |

### วิธีแก้

**Option A**: แทน `localhost` ด้วย IP จริงของเครื่องที่รัน Docker ใน Code Node "Transform Image Paths":

```javascript
// เปลี่ยนจาก:
return `![${alt}](http://localhost:8080/${filename})`;

// เป็น (ใส่ IP จริงของ Docker host):
return `![${alt}](http://192.168.x.x:8080/${filename})`;
```

**Option B**: ใช้ environment variable หรือ n8n variable เก็บ host URL แล้วอ้างอิงใน Code Node

**Option C**: ถ้าต้องการ re-index ด้วย IP ใหม่ → ลบ collection แล้ว ingest PDF ใหม่อีกครั้ง (เพราะ URL ถูก embed ไปแล้วใน Qdrant)

---

## Models ที่ใช้

| ประเภท | Model | ที่ใช้ |
|--------|-------|--------|
| Chat LLM | `llama3.2` | ตอบคำถามใน chat |
| Embedding | `qwen3-embedding:0.6b` | แปลง text เป็น vector (1024 dim) |

---

## Workflow Versions

| ไฟล์ | Docling API | Status field | หมายเหตุ |
|------|------------|--------------|---------|
| `2_document_ingestion.json` | `/v1/convert/source` (sync) | `$json.status` | เวอร์ชันแรก |
| `RAG - Full Workflow v2.json` | `/v1/convert/file/async` + `/v1/result/{id}` | `$json.task_status` | เวอร์ชันล่าสุด ✅ แนะนำ |
| `RAG - 3. AI Chat (Private RAG) (1).json` | — | — | เวอร์ชันเก่า ใช้ `qwen3:embedding` (ผิด) |

---

## Services ที่เข้าถึงได้จากเครื่อง Host

| URL | บริการ |
|-----|--------|
| `http://localhost:5678` | n8n UI + Chat |
| `http://localhost:5001` | Docling UI (Gradio) |
| `http://localhost:6333` | Qdrant API + Dashboard |
| `http://localhost:8080` | nginx static files (รูปภาพจาก PDF) |
| `http://localhost:11434` | Ollama API |

---

## สรุป Flow ทั้งหมด

```
[วาง PDF]
    ↓
pending/ → n8n detects → อ่าน PDF → ส่ง Docling → รอ → ดึงผล
    ↓
Docling แปลง PDF → Markdown + รูป .png ใน docling-scratch/
    ↓
n8n แปลง path รูปเป็น http://localhost:8080/... ใน Markdown
    ↓
n8n ย้ายรูป → extracted-images/ (nginx เสิร์ฟ)
    ↓
n8n split Markdown → chunks → embed (qwen3) → เก็บใน Qdrant
    ↓
n8n ย้าย PDF → processed/

[ถามใน Chat]
    ↓
User → n8n Chat → AI Agent (llama3.2)
    ↓
Agent ค้นหา Qdrant (top-5 chunks)
    ↓
chunks มี text + URL รูป (http://localhost:8080/...)
    ↓
Agent สร้างคำตอบ + แนบ Markdown image
    ↓
Chat UI render รูปจาก URL
    ↓ ✅ เครื่องเดียวกัน: เห็นรูป
    ↓ ❌ เครื่องอื่น: รูปไม่แสดง (localhost ผิดเครื่อง)
```
