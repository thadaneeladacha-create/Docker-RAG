# สถาปัตยกรรมระบบ n8n Private RAG (เครื่องที่ทำงานปกติ)

เอกสารนี้บันทึกการตั้งค่าที่แน่นอนของระบบที่ **รูปแสดงใน chat ได้ปกติ** เพื่อใช้เปรียบเทียบกับเครื่องที่มีปัญหา

---

## Services (docker-compose.yml)

| Service | Image | Port | หมายเหตุ |
|---------|-------|------|---------|
| **n8n** | `n8nio/n8n:latest` | `5678` | รัน `user: root` |
| **docling-gpu** | `ghcr.io/docling-project/docling-serve-cu126:main` | `5001` | profile: `gpu-nvidia` |
| **docling-cpu** | `ghcr.io/docling-project/docling-serve:main` | `5001` | profile: `cpu` |
| **qdrant** | `qdrant/qdrant` | `6333` | |
| **ollama** | `ollama/ollama:latest` | `11434` | |
| **postgres** | `postgres:16-alpine` | internal | |
| **static-files** | `nginx:alpine` | `8080` | เสิร์ฟรูปภาพ |

### Volumes (shared ระหว่าง containers)

```
./shared  →  n8n:     /data/shared
          →  docling: /shared        (working_dir: /shared/docling-scratch)

./shared/extracted-images  →  nginx: /usr/share/nginx/html  (read-only)
./shared/rag-files         →  n8n:   /data/rag-files
```

### Environment Variables ที่สำคัญ (.env)

```
POSTGRES_USER=root
POSTGRES_PASSWORD=password
POSTGRES_DB=n8n
N8N_ENCRYPTION_KEY=super-secret-key
N8N_USER_MANAGEMENT_JWT_SECRET=even-more-secret
N8N_DEFAULT_BINARY_DATA_MODE=filesystem
```

### Environment Variables ใน n8n container

```
N8N_RESTRICT_FILE_ACCESS_TO=/data
NODE_FUNCTION_ALLOW_BUILTIN=fs,path
GENERIC_TIMEZONE=Asia/Bangkok
N8N_COMMUNITY_PACKAGES_ENABLED=true
N8N_SECURE_COOKIE=false
CHOKIDAR_USEPOLLING=1
```

### nginx config (static-files)

```nginx
server {
  listen 80;
  root /usr/share/nginx/html;
  autoindex on;
  location / {
    try_files $uri $uri/ =404;
    add_header Access-Control-Allow-Origin *;
  }
}
```

---

## Docling container config

```yaml
working_dir: /shared/docling-scratch
environment:
  DOCLING_SERVE_ENABLE_UI: "true"
  DOCLING_SERVE_SCRATCH_PATH: /shared/docling-scratch
  DOCLING_SERVE_SINGLE_USE_RESULTS: "false"
  GRADIO_ALLOWED_PATHS: /shared
  GRADIO_TEMP_DIR: /shared/docling-scratch
  DOCLING_SERVE_ENABLE_REMOTE_SERVICES: "true"
volumes:
  - docling_data:/data
  - ./shared:/shared
```

---

## Workflow 1: Setup Qdrant Collection

```
[Manual Trigger] → PUT http://qdrant:6333/collections/multi-modal
Body: { "vectors": { "size": 1024, "distance": "Cosine" } }

[Manual Trigger] → DELETE http://qdrant:6333/collections/multi-modal
```

---

## Workflow 2: Document Ingestion (RAG - Full Workflow v2)

### Node chain และ config ที่แน่นอน

#### 1. Local File Trigger
```
triggerOn: folder
path: /data/rag-files/pending
events: ["add"]
```

#### 2. Read PDF File
```
fileSelector: {{ $json.path }}
```

#### 3. Send to Docling (OCR)
```
method: POST
url: http://docling:5001/v1/convert/file/async
contentType: multipart-form-data
body:
  - name: files        ← ชื่อ field คือ "files" (พหูพจน์)
    type: formBinaryData
    inputDataFieldName: data
  - name: image_export_mode
    value: referenced
```
→ response: `{ "task_id": "..." }`

#### 4. Wait 5s
```
type: wait
unit: seconds (implicit)
```

#### 5. Check Docling Status
```
method: GET
url: http://docling:5001/v1/status/poll/{{ $('Send to Docling (OCR)').item.json.task_id }}
```
→ response field ที่ตรวจ: `$json.task_status`

#### 6. Check Status (Switch node)
```
output "success":  $json.task_status == "success"
output "pending":  $json.task_status == "pending"
output "started":  $json.task_status == "started"
fallback (extra): → Stop and Error
```
- pending/started → วนกลับ Wait 5s
- success → ต่อไป

#### 7. HTTP Request (ดึงผลลัพธ์)
```
method: GET
url: http://docling:5001/v1/result/{{ $('Send to Docling (OCR)').item.json.task_id }}
```
→ response มี `document.md_content` (Markdown + image filenames)

จาก node นี้แตก 2 สาย parallel:

---

### สาย A: Text → Qdrant

#### 8A. Transform Image Paths (Code node)
```javascript
const item = $input.first();
const statusResponse = item.json;

const result = statusResponse.result || statusResponse;
const document = result.document || result;

let markdownContent = document.export_to_markdown ||
                      document.md_content ||
                      document.content ||
                      JSON.stringify(document);

// แปลง image path เป็น URL ผ่าน nginx port 8080
markdownContent = markdownContent.replace(
  /!\[([^\]]*)\]\(([^)]+)\)/g,
  (match, alt, imgPath) => {
    const filename = imgPath.split('/').pop();
    return `![${alt}](http://localhost:8080/${filename})`;
  }
);

const sourcePath = $('Local File Trigger').item.json.path || 'unknown';
const filename = $('Local File Trigger').item.json.name || 'unknown.pdf';

return [{
  json: {
    text: markdownContent,
    metadata: {
      source: sourcePath,
      filename: filename,
      processed_at: new Date().toISOString()
    }
  }
}];
```

#### 9A. Insert to Qdrant
```
mode: insert
collection: multi-modal
credentials: "Qdrant account"
```

#### 10A. Default Data Loader (ai_document → Insert to Qdrant)
```
textSplittingMode: custom
dataType: json
jsonData: {{ $json.text }}
metadata:
  source: {{ $json.metadata.source }}
  filename: {{ $json.metadata.filename }}
```

#### 11A. Recursive Character Text Splitter (ai_textSplitter → Default Data Loader)
```
chunkSize: 1500
chunkOverlap: 300
splitCode: markdown
```

#### 12A. Ollama Embeddings (ai_embedding → Insert to Qdrant)
```
model: qwen3-embedding:0.6b
baseUrl: http://ollama:11434
credentials: "Ollama account"
```

---

### สาย B: Image files → nginx folder

#### 8B. Code in JavaScript
```javascript
for (const item of $input.all()) {
    const mdContent = item.json.document?.md_content || item.json.md_content || "";

    const imageRegex = /!\[.*?\]\((.*?)\)/g;
    const imageNames = [];

    let match;
    while ((match = imageRegex.exec(mdContent)) !== null) {
        const filename = match[1].split('/').pop();  // เอาแค่ชื่อไฟล์
        imageNames.push(filename);
    }

    item.json.imageNames = imageNames;
}

return $input.all();
```

#### 9B. Split Out
```
fieldToSplitOut: imageNames
```
→ แต่ละ item มี `$json.imageNames` เป็นชื่อไฟล์รูป 1 ไฟล์

#### 10B. Execute Command (ย้ายรูปแต่ละไฟล์)
```bash
mv /data/shared/docling-scratch/{{ $json.imageNames }} /data/shared/extracted-images/
```
→ field ที่ใช้: `imageNames` (ตรงกับที่ SplitOut สร้าง)

#### 11B. Limit
```
maxItems: 1
```

#### 12B. Move to Processed (Code node)
```javascript
const fs = require('fs');
const filePath = $('Local File Trigger').item.json.path;
const filename = $('Local File Trigger').item.json.name;
const destPath = '/data/rag-files/processed/' + filename;
fs.renameSync(filePath, destPath);
return [{ json: { success: true, moved: filename } }];
```

---

## Workflow 3: AI Chat

#### Chat Trigger
```
webhookId: rag-chat-webhook
```

#### AI Agent
```
agent: toolsAgent
model: llama3.2 via Ollama (temperature: 0.1)
systemMessage:
  "คุณคือ AI Assistant ผู้เชี่ยวชาญในการค้นหาและสรุปข้อมูลจากเอกสาร
  ให้ใช้ tool search_documents เพื่อค้นหาข้อมูลที่เกี่ยวข้องจากเอกสารก่อนตอบคำถามทุกครั้ง
  หากพบรูปภาพในเอกสาร ให้แสดงผลในรูปแบบ Markdown: ![คำอธิบาย](URL)
  ตอบเป็นภาษาเดียวกับที่ผู้ใช้ถาม
  หากไม่พบข้อมูลในเอกสาร ให้บอกตรงๆ ว่าไม่มีข้อมูลนั้นในฐานข้อมูล"
```

#### Qdrant Search Tool (ai_tool → AI Agent)
```
mode: retrieve-as-tool
toolName: search_documents
toolDescription: ค้นหาข้อมูลจากเอกสาร PDF ที่อัปโหลดไว้ในระบบ ใช้เมื่อต้องการข้อมูลจากเอกสาร รูปภาพ หรือตาราง
collection: multi-modal
topK: 5
credentials: "Qdrant account"
```

#### Ollama Embeddings (ai_embedding → Qdrant Search Tool)
```
model: qwen3-embedding:0.6b
baseUrl: http://ollama:11434
credentials: "Ollama account"
```

#### Window Buffer Memory (ai_memory → AI Agent)
```
sessionKey: sessionId
contextWindowLength: 10
```

---

## Image Pipeline สรุป

```
PDF วางใน ./shared/rag-files/pending/
    ↓ n8n detect (Local File Trigger)
    ↓ อ่านเป็น binary → POST /v1/convert/file/async
Docling สร้างรูปที่: /shared/docling-scratch/image_000001_<hash>.png
    ↓ n8n ดึง result: md_content มี "![Image](image_000001_<hash>.png)"
    ↓ สาย A: แปลงเป็น "![Image](http://localhost:8080/image_000001_<hash>.png)"
    ↓ embed text → เก็บใน Qdrant collection "multi-modal"
    ↓ สาย B: mv /data/shared/docling-scratch/image_000001_<hash>.png
                → /data/shared/extracted-images/
    ↓ nginx เสิร์ฟ: http://localhost:8080/image_000001_<hash>.png

[Chat]
User → AI Agent → search_documents (Qdrant top-5)
    ↓ ได้ chunk ที่มี "![Image](http://localhost:8080/image_000001_<hash>.png)"
    ↓ AI ตอบพร้อม Markdown image
    ↓ n8n Chat UI render รูปจาก localhost:8080
✅ รูปแสดงปกติ
```

---

## Models ที่ pull ใน Ollama

```bash
# ใน init container (ollama-pull-llama):
ollama pull llama3.2
ollama pull qwen3:embedding   # ← pull ชื่อนี้

# แต่ใช้งานใน workflow ด้วยชื่อ:
qwen3-embedding:0.6b          # ← ชื่อ model ใน n8n nodes
```

---

## Qdrant collection spec

```
name: multi-modal
vector_size: 1024
distance: Cosine
```

---

## Ollama models ที่ใช้งานจริงใน workflow

| Node | model parameter |
|------|----------------|
| Ollama Chat Model | `llama3.2` |
| Ollama Embeddings (ingestion) | `qwen3-embedding:0.6b` |
| Ollama Embeddings (chat/search) | `qwen3-embedding:0.6b` |
