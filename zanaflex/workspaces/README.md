# Zanaflex — n8n Workflows

7 workflows que compõem o backend Zanaflex. Importe **todos** no mesmo projeto n8n.

| Arquivo                                                                | Webhooks expostos                                                                                                                                                                                                       |
| ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Zanaflex-Front.json](Zanaflex-Front.json)                             | `GET /webhook/zanaflex-chat` — serve o HTML do front                                                                                                                                                                    |
| [Zanaflex-Agent-IA.json](Zanaflex-Agent-IA.json)                       | `POST /webhook/zanaflex-AgentRag`                                                                                                                                                                                       |
| [Zanaflex-RAG.json](Zanaflex-RAG.json)                                 | `POST /webhook/zanaflex-index-drive` (upload chat), `POST /webhook/zanaflex-rag-upload` (upload admin/ERP), `POST /webhook/zanaflex-rag-reindex` (reprocessar por `file_id`), trigger `Webhook Setup`/manual para reset |
| [Zanaflex-RAG-Admin.json](Zanaflex-RAG-Admin.json)                     | `POST /webhook/zanaflex-rag-upsert`, `GET /webhook/zanaflex-rag-docs`, `DELETE /webhook/zanaflex-rag-doc-delete`, `POST /webhook/zanaflex-rag-purge-all`                                                                |
| [Zanaflex-Chat-GET-Sessions.json](Zanaflex-Chat-GET-Sessions.json)     | `GET /webhook/zanaflex-sessions?userId=…`                                                                                                                                                                               |
| [Zanaflex-Chat-GET-History.json](Zanaflex-Chat-GET-History.json)       | `GET /webhook/zanaflex-history?sessionId=…&userId=…`                                                                                                                                                                    |
| [Zanaflex-Chat-DELETE-Session.json](Zanaflex-Chat-DELETE-Session.json) | `DELETE /webhook/zanaflex-session?sessionId=…&userId=…`                                                                                                                                                                 |

## Credenciais esperadas

Todos os workflows referenciam:

- **Postgres** → `id: "jFjeYH6Nt3aRNkoM"`, `name: "Supabase_database"` — credencial existente no projeto n8n (a mesma usada pela Aspiramaq/Sameka, apontando para `longflatworm-supabase.cloudfy.live`).
- **OpenAI** → placeholder `REPLACE_ME_OPENAI_CRED` em 4 nós (`Generate Embedding`, `Chat Model`, `Embed Query`). Trocar pelo id real da credencial OpenAI antes de ativar.

## Ordem de import

1. Garanta que as migrations **001 → 013** estão aplicadas no Supabase (ver [../migrations/README.md](../migrations/README.md)).
2. Importe os 7 JSONs na ordem que preferir — eles são independentes entre si.
3. Em `Zanaflex-Agent-IA` e `Zanaflex-RAG`, substitua `REPLACE_ME_OPENAI_CRED` pelo id da sua credencial OpenAI (botão "..." → Edit → escolha a credencial).
4. Ative todos os workflows.

## Pontos de design importantes

### RAG — upsert atômico por documento

`POST /webhook/zanaflex-rag-upsert` recebe:

```json
{
  "file_id": "IT-18.05",
  "title": "Pesagem e Dosagem de Produtos Químicos",
  "code": "IT-18.05",
  "category_code": "IT",
  "url": "https://.../IT-18.05.pdf",
  "mime_type": "application/pdf",
  "content_text": "...texto extraído..."
}
```

O fluxo: **purge_file(file_id) → chunk → embed → insert → upsert_metadata**. Só os chunks **daquele** `file_id` são removidos antes de reinserir — os outros documentos permanecem intactos.

Para PDFs binários, insira um nó `Extract from File` **entre** _Parse Payload_ e _Chunk Content_ e use o output como `content_text`. O fallback de decode ASCII via `content_base64` existe só para emergência.

### Agent — ACL por usuário

A ferramenta `search_knowledge_base` invoca `zanaflex_match_documents_for_user(userId, embedding, ...)` (migration 012). A função aplica a ACL: admin enxerga tudo; demais usuários só veem trechos cujo `category_id` esteja entre `zanaflex_user_allowed_categories_for(userId)`.

A ferramenta `get_document_metadata` também filtra por ACL — se o usuário pedir um código que existe mas para o qual não tem acesso, o agente recebe "Documento não encontrado ou sem acesso" e responde de acordo.

### Memória de chat

O nó `Postgres Chat Memory` grava em `zanaflex_chat_message`. O trigger da migration 010 popula `user_id` extraindo do bloco `[CONTEXTO DO USUÁRIO: ... ID="uuid"]` que o nó `Prepare Input` antepõe na mensagem human. Isso garante que `GET /zanaflex-sessions?userId=...` retorne apenas as sessões daquele usuário.

### Privacidade dos endpoints de chat

Todos os 3 endpoints Chat-\* exigem `userId` e filtram por ele no SQL. Mesmo que um usuário descubra um `sessionId` alheio, a query retorna 0 linhas (cláusula `AND user_id = ...`).

## Como atualizar um IT já indexado

Basta enviar o **mesmo** `file_id` no webhook `zanaflex-rag-upsert`. A função `zanaflex_rag_purge_file` apaga só os chunks antigos daquele `file_id` e os novos chunks substituem. Nenhum outro documento é tocado. Tempo de reindexação ≈ proporcional ao tamanho daquele 1 PDF, não ao tamanho do RAG inteiro.

## RAG — entry-points unificados

O workflow `Zanaflex-RAG.json` concentra **3 webhooks de ingestão** que compartilham o mesmo pipeline downstream (extração → chunk → embed → upsert):

### 1. `POST /webhook/zanaflex-index-drive` — upload via chat

Usado quando o usuário anexa arquivo a uma sessão de chat. Mantém o comportamento original: prefixa o nome no Drive com a data (`21-05-2026_arquivo.pdf`), grava `session_id`, e o arquivo + vetores são apagados após 24 h pelo trigger `24h Trigger (Session)1`.

Multipart: `data` (binário) + body com `sessionId`, `category_id`, `category_code`, `drive_folder_id`, `code`.

### 2. `POST /webhook/zanaflex-rag-upload` — upload permanente (admin/ERP)

Usado pelo botão **Adicionar Documento** do front-end e pelo ERP da Zanaflex.

Multipart:

- `data` (arquivo) — binário do documento. O **nome do arquivo** é a chave de dedupe.
- `categoria` (string) — `category_code` (ex: `IT`, `NR`). A categoria precisa ter `drive_folder_id` configurado.

Fluxo:

1. `Lookup Upload Category` resolve `drive_folder_id` + `category_id`.
2. `Search Drive by Filename` busca arquivo com nome exato na pasta.
3. **Achou** → `Drive: Update File` mantém o mesmo `fileId` e troca só o conteúdo. **Não achou** → `Drive: Upload File (no prefix)` cria sem prefixar data.
4. `Set Upload Item Shape` normaliza para o shape do Loop Over Items.
5. Pipeline padrão: extrai texto (PDF/XLSX/CSV/Google Doc) com fallback OCR via Azure Responses → chunk + embed → insert no `zanaflex_documents` (apagando antes os chunks daquele `file_id`) → upsert em `zanaflex_document_metadata` (`source = 'admin'`).

Exemplo:

```bash
curl -X POST "https://longflatworm-n8n.cloudfy.live/webhook/zanaflex-rag-upload" \
  -F "data=@./IT-06.06.pdf" \
  -F "categoria=IT"
```

### 3. `POST /webhook/zanaflex-rag-reindex` — reprocessar 1/N/todos

Re-executa o pipeline RAG **sem** re-upload no Drive. Útil para refazer chunks após mudança no prompt/extrator.

JSON body (aceita qualquer um dos formatos):

```json
{ "file_id": "1AbCDef…" }
{ "file_ids": ["1AbC…", "2XyZ…"] }
{ "ids": ["1AbC…"] }
```

Fluxo:

1. `Normalize Reindex IDs` extrai e sanitiza o array.
2. `Fetch Reindex Metadata` busca `id`, `mimeType`, `webViewLink`, `category_id`, `category_code` em `zanaflex_document_metadata` + `zanaflex_document_categories`.
3. Responde imediatamente `{ status: 'queued', reindex_count, ids }` (HTTP 200) — o processamento continua em background.
4. Mesmo pipeline downstream: re-baixa do Drive pelo `file_id`, re-extrai, re-chunka, re-embede, substitui chunks no `zanaflex_documents`, atualiza `last_indexed_at`.

Exemplo:

```bash
curl -X POST "https://longflatworm-n8n.cloudfy.live/webhook/zanaflex-rag-reindex" \
  -H "Content-Type: application/json" \
  -d '{"file_ids":["1AbCDef","2XyZ"]}'
```

### Reset / Database Setup

O trigger manual `When clicking 'Execute workflow'` (ou webhook interno `Webhook Setup`) varre **todas** as categorias com `drive_folder_id`, lista cada pasta do Drive, anexa o contexto da categoria via `Attach Cat to Drive Files`, e dispara o pipeline para cada arquivo. Útil após reset completo da tabela vetorial.
