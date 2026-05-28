# Customizações do Fork — Tecnova (Nova)

Este documento registra todas as modificações feitas nos arquivos do upstream
(danny-avila/LibreChat) para o fork da Tecnova. Consulte este arquivo ao
resolver conflitos de merge com o upstream.

---

## Regra geral

Toda customização segue o padrão: **absorver a lógica nova do upstream +
reintroduzir nossa modificação**. Nunca descartar a versão do upstream.

---

## 1. File picker — suporte a Excel e documentos

**Problema resolvido:** o diálogo de upload só mostrava PDF e imagens para o
endpoint OpenAI, ignorando os `supportedMimeTypes` configurados no yaml.

### `packages/data-provider/src/file-config.ts`

**O que fizemos:** adicionamos a constante `openaiDocumentExtensions` logo após
`bedrockDocumentExtensions`.

```ts
/** File extensions accepted by OpenAI/standard document uploads (for input accept attributes) */
export const openaiDocumentExtensions =
  '.pdf,.csv,.doc,.docx,.xls,.xlsx,.pptx,.txt,application/pdf,text/csv,application/vnd.ms-excel,application/vnd.openxmlformats-officedocument.spreadsheetml.sheet,application/vnd.openxmlformats-officedocument.wordprocessingml.document,application/vnd.openxmlformats-officedocument.presentationml.presentation,text/plain';
```

**Como resolver conflito:** manter a versão do upstream + reinserir o bloco
acima logo após o export de `bedrockDocumentExtensions`.

---

### `client/src/components/Chat/Input/Files/AttachFileMenu.tsx`

**O que fizemos:** substituímos os `accept` hardcoded dos casos `document`,
`image_document` e `image_document_video_audio` para usar `openaiDocumentExtensions`.

```ts
// Importar junto com bedrockDocumentExtensions:
import {
  openaiDocumentExtensions,
  bedrockDocumentExtensions,
  // ...demais imports
} from 'librechat-data-provider';

// Casos no handleUploadClick:
} else if (fileType === 'document') {
  inputRef.current.accept = openaiDocumentExtensions;
} else if (fileType === 'image_document') {
  inputRef.current.accept = `image/*,.heif,.heic,${openaiDocumentExtensions}`;
} else if (fileType === 'image_document_video_audio') {
  inputRef.current.accept = `image/*,.heif,.heic,${openaiDocumentExtensions},video/*,audio/*`;
```

**Como resolver conflito:** manter a lógica nova do upstream no
`handleUploadClick` + garantir que os três casos acima usam
`openaiDocumentExtensions` em vez de `.pdf,application/pdf` hardcoded.

---

## 2. Extração de texto para documentos no chat regular

**Problema resolvido:** arquivos Excel, Word, CSV e TXT enviados no chat
regular (não agentes) não tinham o texto extraído no upload — iam direto
para armazenamento binário. Na hora de enviar à OpenAI, a Responses API
(GPT-5.4) rejeitava por não suportar esses MIME types como `file_data`.
O texto extraído já funcionava para agentes (via `EToolResources.context`)
mas estava ausente para uploads de chat regular.

### `api/server/services/Files/process.js`

**O que fizemos:** estendemos a condição do bloco `EToolResources.context`
para também rotear uploads de chat regular quando:
- É um `messageAttachment` sem `tool_resource` explícito
- O arquivo não é imagem e não é PDF
- O MIME type está em `documentParserMimeTypes` (Excel, Word, CSV, TXT, etc.)

```js
} else if (
  tool_resource === EToolResources.context ||
  (messageAttachment &&
    !tool_resource &&
    !isImage &&
    file.mimetype !== 'application/pdf' &&
    (documentParserMimeTypes.some((regex) => regex.test(file.mimetype)) ||
      textMimeTypes.test(file.mimetype)))
) {
```

`textMimeTypes` deve ser importado junto com `documentParserMimeTypes` no topo do arquivo:

```js
  textMimeTypes,
  documentParserMimeTypes,
} = require('librechat-data-provider');
```

PDF continua indo pelo caminho binário (funciona nativamente na Responses API).
Imagens continuam pelo caminho de imagem. Apenas documentos de texto
(Excel → CSV, Word → texto, etc.) são roteados para extração.

**Como resolver conflito:** manter a lógica nova do upstream na condição
`else if` + reinserir a segunda parte do `||` conforme acima.

---

## 3. Workflows GitNexus — silenciados no fork

**Problema resolvido:** os workflows do GitNexus (infraestrutura interna do
upstream no DigitalOcean) disparavam no fork da Tecnova a cada sync,
falhando por falta de credenciais e gerando notificações de e-mail.

**Guard adicionado em todos os jobs dos 4 workflows:**

```yaml
if: github.repository == 'danny-avila/LibreChat' && (
  # ... condição original do upstream ...
)
```

### Arquivos afetados

| Arquivo | Job | Posição do guard |
|---|---|---|
| `.github/workflows/gitnexus-deploy.yml` | `build-image` | Envolvendo condição existente com `&&` |
| `.github/workflows/gitnexus-deploy.yml` | `deploy` | Novo `if:` adicionado |
| `.github/workflows/gitnexus-cleanup-pr.yml` | `cleanup` | Prefixando condição existente com `&&` |
| `.github/workflows/gitnexus-index.yml` | `index` | Envolvendo condição existente com `&&` |
| `.github/workflows/gitnexus-pr-command.yml` | `dispatch` | Prefixando condição existente com `&&` |

**Como resolver conflito (padrão para todos):**

1. Pegar a versão nova do upstream para o bloco `if:`
2. Envolver com o guard do repositório:

```yaml
if: |
  github.repository == 'danny-avila/LibreChat' && (
    # colar aqui a condição nova do upstream
  )
```

---

## 4. Workflows de publish NPM — guard no job `publish-npm`

**Problema resolvido:** em maio/2026 o upstream refatorou os workflows de
publish (`client.yml`, `data-provider.yml`, `data-schemas.yml`) separando o
job único em dois: `pack` (build + empacotamento) e `publish-npm` (publicação
no npm). Nosso guard que estava no job antigo passou a conflitar.

**Guard atual nos 3 arquivos (job `publish-npm`):**

```yaml
# client.yml e data-schemas.yml
  publish-npm:
    needs: pack
    if: github.repository == 'danny-avila/LibreChat' && github.ref == 'refs/heads/main' && needs.pack.outputs.skip != 'true'

# data-provider.yml
  publish-npm:
    needs: pack
    if: github.repository == 'danny-avila/LibreChat' && github.ref == 'refs/heads/main'
```

O job `pack` não precisa de guard — ele só builda e empacota, sem publicar.

**Como resolver conflito:**

1. Aceitar a estrutura nova do upstream para o arquivo inteiro
2. Garantir que o job `publish-npm` tenha `github.repository == 'danny-avila/LibreChat' &&`
   prefixando a condição `if:` existente

O `sync-fork.yml` restaura o guard automaticamente via `sed` caso uma
sincronização futura o remova.

---

## 5. Agentes com GPT-5.x — ativação automática da Responses API

**Problema resolvido:** agentes criados com modelos `gpt-5.4` (ou qualquer
`gpt-5.x` versionado) iniciavam mas não produziam nenhuma saída de texto.
A função `getOpenAILLMConfig` só ativava `useResponsesApi: true` quando a
busca na web estava habilitada — nunca por detecção do modelo.

### `packages/api/src/endpoints/openai/llm.ts`

**O que fizemos:** adicionamos detecção automática de modelos `gpt-[5-9].x`
antes do bloco de web search (linha ~476), ativando `useResponsesApi = true`
quando o modelo é versionado (ex.: `gpt-5.4`, `gpt-5.4-mini`) e o flag
ainda não foi configurado explicitamente.

```ts
if (!useOpenRouter && llmConfig.useResponsesApi == null && llmConfig.model) {
  if (/\bgpt-[5-9]\.\d/i.test(llmConfig.model as string)) {
    llmConfig.useResponsesApi = true;
  }
}
```

O regex `/\bgpt-[5-9]\.\d/i` exclui intencionalmente `gpt-5` (base), que é
tratado como modelo de raciocínio separado pelo upstream.

**Como resolver conflito:** manter a versão do upstream + reinserir o bloco
acima imediatamente antes de `if (useOpenRouter && enableWebSearch)`.

---

## Configuração da Nova (fora do repositório)

O `librechat.yaml` com as configurações da Nova (modelSpecs, fileConfig,
interface, termos de uso, etc.) está no **volume do FileBrowser no Railway**,
não versionado neste repositório. Portanto, não há risco de conflito de merge
para essas configurações.
