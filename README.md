# WhatsApp DOCX → PDF

Bot de WhatsApp que recebe um arquivo Word (.doc/.docx) e devolve o mesmo documento convertido em PDF, automaticamente, no mesmo chat.

**Stack:** n8n (orquestração) · Evolution API (WhatsApp) · Gotenberg (conversão via LibreOffice) · PostgreSQL · Docker Compose

**Custo:** zero. Tudo é open source e roda localmente.

> **Status:** testado de ponta a ponta com WhatsApp real (recebimento do .docx → conversão → envio do PDF).

```
WhatsApp ──► Evolution API ──webhook──► n8n ──► Gotenberg
    ▲                                    │
    └──────────── PDF de volta ◄─────────┘
```

## O problema

Escritórios e pequenos negócios recebem documentos em Word pelo WhatsApp e precisam devolver ou arquivar em PDF. O processo manual (baixar, abrir no Word, exportar, reenviar) leva de 2 a 3 minutos por arquivo e depende de alguém no computador. O bot faz isso em segundos, a qualquer hora.

*Estimativa:* com ~20 arquivos/dia, são cerca de **1 hora economizada por dia**.

---

## Como o fluxo funciona

| # | Nó | O que faz |
|---|---|---|
| 1 | **Webhook** | Recebe o aviso da Evolution API quando chega uma mensagem |
| 2 | **Normalizar** | Extrai só os campos úteis (instância, número, ID da mensagem, dados do documento). Trata documentos enviados com e sem legenda |
| 3 | **É Word?** | Deixa passar só arquivos Word enviados por outra pessoa (`fromMe = false`), evitando loop com as próprias respostas |
| 4 | **Baixar Arquivo** | Pede à Evolution o conteúdo do arquivo (em base64) |
| 5 | **Montar Arquivo** | Converte o base64 em arquivo binário, com nome e extensão |
| 6 | **Converter PDF** | Envia o arquivo ao Gotenberg e recebe o PDF |
| 7 | **PDF para Base64** | Converte o PDF de volta em base64 |
| 8 | **Enviar PDF** | Manda o PDF pelo WhatsApp para quem enviou o Word |

**Decisões de design:**
- **Normalização no início:** se a integração com WhatsApp mudar (ex.: migração para a API oficial), só o nó *Normalizar* e os de entrada/saída precisam mudar. O resto do fluxo continua igual.
- **Chave da API em credencial do n8n**, nunca no corpo dos nós, então o workflow exportado pode ser publicado sem expor segredos.
- **Redis desligado** na Evolution: o cache local basta para esse volume e economiza memória.

---

## Como rodar

### 1. Pré-requisitos
- Docker Desktop (no Windows, com WSL 2)
- ~3 GB de RAM livres para os containers

### 2. Configurar as senhas

Copie o modelo e troque os valores (use só letras e números):

```bash
cp .env.example .env
```

No PowerShell, para gerar senhas aleatórias direto no arquivo:

```powershell
$chars = (48..57) + (65..90) + (97..122)
$apiKey = -join ($chars | Get-Random -Count 32 | ForEach-Object {[char]$_})
$dbPass = -join ($chars | Get-Random -Count 24 | ForEach-Object {[char]$_})
"EVOLUTION_API_KEY=$apiKey`nPOSTGRES_PASSWORD=$dbPass" | Set-Content .env -Encoding ascii
```

### 3. Subir os serviços

```bash
docker compose up -d
docker compose ps    # os 4 serviços devem estar "Up"
```

| Serviço | Endereço |
|---|---|
| n8n | http://localhost:5678 |
| Evolution API (painel) | http://localhost:8080/manager |
| Gotenberg | http://localhost:3000/health |

### 4. Testar o conversor sozinho

```bash
curl -X POST http://localhost:3000/forms/libreoffice/convert -F "files=@seu-arquivo.docx" -o saida.pdf
```

### 5. Importar o workflow no n8n
1. Crie sua conta local em http://localhost:5678
2. Crie uma credencial **Header Auth** chamada `Evolution API`, com **Name** `apikey` e **Value** igual à `EVOLUTION_API_KEY` do `.env`
3. Novo workflow → **⋯** → **Import from File** → `workflow/whatsapp-docx-to-pdf.json`
4. Nos nós **Baixar Arquivo** e **Enviar PDF**, selecione a credencial `Evolution API`

### 6. Testar sem WhatsApp

O nó Webhook já vem com uma mensagem simulada fixada (a mesma de `sample-payload.json`). Para disparar manualmente:

1. No nó Webhook, clique em **Listen for test event**
2. Rode:
```bash
curl -X POST http://localhost:5678/webhook-test/whatsapp-pdf -H "Content-Type: application/json" --data "@sample-payload.json"
```

### 7. Conectar o WhatsApp

1. Em http://localhost:8080/manager, entre com a `EVOLUTION_API_KEY`
2. Crie uma instância chamada `pdfbot` e escaneie o QR code (WhatsApp → Aparelhos conectados)
3. Aponte o webhook da instância para o n8n:

```bash
curl -X POST http://localhost:8080/webhook/set/pdfbot \
  -H "apikey: SUA_EVOLUTION_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "webhook": {
      "enabled": true,
      "url": "http://n8n:5678/webhook/whatsapp-pdf",
      "byEvents": false,
      "base64": false,
      "events": ["MESSAGES_UPSERT"]
    }
  }'
```

> A URL usa `n8n` (nome do container) e não `localhost`, porque a chamada parte de dentro da rede do Docker.

4. No n8n, desafixe os dados do Webhook e **ative** o workflow
5. Envie um .docx para o número conectado e receba o PDF de volta

> ⚠️ A Evolution API não é oficial. Use um número secundário: volume alto ou envio em massa podem levar a bloqueio.

---

## Como escalar

Este projeto foi pensado para portfólio e baixo volume. Para produção com volume alto:

**WhatsApp Cloud API (oficial) no lugar da Evolution API**
- Sem risco de bloqueio por integração não oficial, com suporte e throughput muito maior
- Exige app no Meta for Developers, conta Business e webhook público com HTTPS (ex.: Cloudflare Tunnel)
- Muda só a entrada e a saída: o download de mídia vira duas chamadas (ID → URL → arquivo) e o envio exige upload prévio do PDF. O n8n já tem nós nativos para a Cloud API
- Custo: mensagens recebidas são gratuitas. Desde out/2026, respostas dentro da janela de 24h têm franquia de 1.000 mensagens/mês por número e são cobradas por mensagem acima disso

**Infraestrutura**
- n8n em **queue mode** (Redis + workers) para processar várias conversões em paralelo
- Múltiplas réplicas do Gotenberg atrás de um balanceador
- Hospedagem 24/7 (ex.: tier gratuito da Oracle Cloud) em vez da máquina local

---

## Problemas comuns

| Sintoma | Causa provável |
|---|---|
| Webhook não dispara | URL com `localhost` em vez de `n8n`, ou workflow inativo usando `/webhook/` |
| Gotenberg: `no form file found for extensions` | O arquivo chegou sem `.docx` no nome. Confira o *File Name* no nó Montar Arquivo |
| Expressão aparece como texto literal | Campo em modo **Fixed**. Campos com `{{ }}` precisam estar em **Expression** |
| `Can't get data for expression` | Os nós anteriores ainda não foram executados nesta sessão. Execute o fluxo até ali |
| Layout do PDF diferente do Word | Fonte do documento não existe no container do Gotenberg |
| Evolution reiniciando | Senha do Postgres com caractere especial. Veja `docker compose logs evolution-api` |
