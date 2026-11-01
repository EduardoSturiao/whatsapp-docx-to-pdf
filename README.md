# WhatsApp DOCX → PDF

Bot de WhatsApp que recebe um arquivo Word (.doc/.docx) e devolve o mesmo documento convertido em PDF, automaticamente, no mesmo chat.

**Stack:** n8n (orquestração) · Evolution API (WhatsApp) · Gotenberg (conversão via LibreOffice) · Docker Compose

```
WhatsApp ──► Evolution API ──webhook──► n8n ──► Gotenberg
    ▲                                    │
    └──────────── PDF de volta ◄─────────┘
```

## Problema e resultado esperado

Escritórios e pequenos negócios recebem documentos em Word pelo WhatsApp e precisam devolver/arquivar em PDF. O processo manual (baixar, abrir, exportar, reenviar) leva de 2 a 3 minutos por arquivo. Com o bot, leva segundos e não depende de ninguém no computador. Com ~20 arquivos/dia, são **~1 hora economizada por dia**.

---

## 1. Subir os serviços

Pré-requisito: Docker + Docker Compose.

```bash
cp .env.example .env      # edite os valores
docker compose up -d
docker compose ps         # os 4 serviços devem estar "running"
```

| Serviço | Endereço |
|---|---|
| n8n | http://localhost:5678 |
| Evolution API + painel | http://localhost:8080/manager |
| Gotenberg | http://localhost:3000/health |

## 2. Conectar o WhatsApp

1. Abra http://localhost:8080/manager e entre com a `EVOLUTION_API_KEY`.
2. Crie uma instância (ex.: `pdfbot`) do tipo **Baileys**.
3. Clique pra gerar o QR code e escaneie com o WhatsApp (Aparelhos conectados).

> ⚠️ Use um número secundário: API não oficial pode levar bloqueio se abusar do volume.

## 3. Apontar o webhook pro n8n

Durante o desenvolvimento, use a URL de **teste** do n8n (`/webhook-test/`). Quando ativar o fluxo, troque para `/webhook/`.

```bash
curl -X POST http://localhost:8080/webhook/set/pdfbot \
  -H "apikey: SUA_EVOLUTION_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "webhook": {
      "enabled": true,
      "url": "http://n8n:5678/webhook-test/whatsapp-pdf",
      "byEvents": false,
      "base64": false,
      "events": ["MESSAGES_UPSERT"]
    }
  }'
```

A URL usa `n8n` (nome do container) e não `localhost`, porque a chamada sai de dentro da rede do Docker.

## 4. Fluxo no n8n (7 nós)

Dentro do n8n, todas as URLs usam o nome dos containers (`evolution-api`, `gotenberg`).

### 1) Webhook
- Método: `POST` · Path: `whatsapp-pdf`

### 2) Edit Fields (Set) — "Normalizar"
Extrai só o que importa do payload. O documento pode vir em dois lugares (com ou sem legenda):

| Campo | Expressão |
|---|---|
| `instance` | `{{ $json.body.instance }}` |
| `number` | `{{ $json.body.data.key.remoteJid.split('@')[0] }}` |
| `messageId` | `{{ $json.body.data.key.id }}` |
| `fromMe` | `{{ $json.body.data.key.fromMe }}` |
| `doc` | `{{ $json.body.data.message?.documentMessage ?? $json.body.data.message?.documentWithCaptionMessage?.message?.documentMessage }}` |

### 3) IF — "É Word?"
- `fromMe` é `false` **E**
- `{{ $json.doc?.mimetype ?? '' }}` contém `word` (pega .docx e .doc)

Saída *false*: opcional, responder "Me manda um arquivo .docx 😉" com `POST /message/sendText/{instance}`.

### 4) HTTP Request — "Baixar arquivo"
- `POST http://evolution-api:8080/chat/getBase64FromMediaMessage/{{ $json.instance }}`
- Header: `apikey: SUA_EVOLUTION_API_KEY`
- Body JSON:
```json
{ "message": { "key": { "id": "{{ $('Normalizar').item.json.messageId }}" } }, "convertToMp4": false }
```
Retorna o arquivo no campo `base64`.

### 5) Convert to File
- Operação: **Move Base64 String to File**
- Campo de origem: `base64` · Campo binário de saída: `data`
- Opções → File Name: `{{ $('Normalizar').item.json.doc.fileName }}`

> O nome **precisa** terminar em `.docx`/`.doc`: é pela extensão que o Gotenberg sabe o que converter.

### 6) HTTP Request — "Converter (Gotenberg)"
- `POST http://gotenberg:3000/forms/libreoffice/convert`
- Body: **Form-Data** → parâmetro tipo *n8n Binary File*, nome `files`, campo binário `data`
- Opções → Response → Response Format: **File** (saída em `data`)

### 7) Extract From File + HTTP Request — "Devolver PDF"
- Extract From File: **Move File to Base64 String**, de `data` para `pdfBase64`
- `POST http://evolution-api:8080/message/sendMedia/{{ $('Normalizar').item.json.instance }}`
- Header: `apikey: SUA_EVOLUTION_API_KEY`
- Body JSON:
```json
{
  "number": "{{ $('Normalizar').item.json.number }}",
  "mediatype": "document",
  "mimetype": "application/pdf",
  "media": "{{ $json.pdfBase64 }}",
  "fileName": "{{ $('Normalizar').item.json.doc.fileName.replace(/\\.docx?$/i, '') }}.pdf",
  "caption": "Aqui está seu PDF ✅"
}
```

## 5. Testar

1. No n8n, clique em **Listen for test event** no nó Webhook.
2. De outro celular, mande um .docx pro número conectado.
3. Acompanhe cada nó executando e confira o PDF chegando no WhatsApp.
4. Funcionou? Ative o workflow e refaça o passo 3 trocando `/webhook-test/` por `/webhook/`.

## Problemas comuns

| Sintoma | Causa provável |
|---|---|
| Webhook não dispara | URL com `localhost` em vez de `n8n`, ou fluxo inativo usando `/webhook/` |
| Gotenberg retorna erro 400 | Arquivo sem extensão `.docx` no nome |
| Layout do PDF "pulou" | Fonte do documento não existe no container do Gotenberg |
| Evolution reiniciando | Senha do Postgres com caractere especial; veja `docker compose logs evolution-api` |
