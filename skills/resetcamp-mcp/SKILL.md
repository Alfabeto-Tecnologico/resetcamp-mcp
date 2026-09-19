---
name: resetcamp-mcp
description: >-
  Publica e edita conteúdo no ResetCamp pelo MCP do criador: comunidades,
  canais, posts com títulos e negrito, comentários, cursos e mídia por URL
  pública. Use quando o pedido for criar, atualizar, listar ou apagar um post,
  artigo, comentário ou curso na comunidade, ou quando mencionar o MCP do
  ResetCamp, posts_create, posts_update, channels_list ou a ponte mcp-bridge.
---

# MCP do ResetCamp

O agente atua **em nome do criador** cujo token está no servidor. Só vê e escreve nas comunidades desse criador. Não inventa IDs, não cria um segundo post quando o pedido é editar, e não apaga nada que o utilizador não tenha pedido para apagar.

Antes de cada chamada, lê o schema vivo da tool (`GetDynamicTools` / `tools/list`). Os nomes e os campos obrigatórios abaixo são o contrato; o schema ganha se divergir. `additionalProperties` é `false`: um campo a mais faz a chamada falhar.

Se o namespace estiver `error` ou `needsAuth`, autentica com `mcp_auth` e lista as tools de novo. Não contornes o MCP com SQL, server actions ou a API HTTP.

## Antes de escrever

1. Lê a descrição do servidor. Ela diz o nome do criador e os escopos ativos.
2. Se o escopo da tool falta, para e diz qual escopo falta. Não tentes a chamada à mesma.
3. Resolve IDs por tool, nesta ordem:
   - `communities_list` (sem argumentos, ou `includeArchived: true` se a comunidade não aparecer)
   - `channels_list` com o `communityId`, se fores publicar num canal
   - `posts_list` ou `courses_list` se o pedido puder referir-se a algo que já existe
4. `threadCount` em comunidades e canais pode ficar em 0 mesmo com posts. Não uses esse contador para decidir se a comunidade está vazia. Usa `posts_list`.

URL de um post, com o slug da comunidade e o `slug` do canal:

- membros: `/c/{slug}/{channelSlug}/{postId}`
- público (só se `isPublic` foi `true` na criação): `/post/{postId}`

O host é o da API a que o MCP está ligado (`RESETCAMP_API_URL`). Não assumes localhost nem produção.

## Posts

| Tool           | Escopo        | Obrigatório                     | Quando                             |
| -------------- | ------------- | ------------------------------- | ---------------------------------- |
| `posts_list`   | `posts:read`  | nenhum                          | Achar um post antes de criar outro |
| `posts_get`    | `posts:read`  | `postId`                        | Confirmar o que ficou gravado      |
| `posts_create` | `posts:write` | `channelId`, `title`, `content` | Post novo                          |
| `posts_update` | `posts:write` | `postId`                        | Editar o mesmo post                |
| `posts_delete` | `posts:write` | `postId`                        | Só quando o pedido for apagar      |

`isPublic` existe só em `posts_create` e o padrão é `false`. Um artigo que o utilizador quer aberto na internet leva `isPublic: true` na criação. `posts_update` não altera esse flag.

`isPinned` fixa o post no topo do canal. Não fixes sem pedido.

Vídeo do YouTube, Vimeo ou Loom vai em `externalPlatformUrl`. Ficheiro direto (MP4, HLS) vai em `recordingUrl`. Áudio vai em `audioUrl`. Capa vai em `coverImage` com uma URL pública `https`.

### Corpo: Markdown, não texto puro

`content` aceita Markdown ou um JSON TipTap que já seja `{ "type": "doc", ... }`. O servidor grava o documento do editor. Sem isso a página mostra texto puro, sem título nem negrito.

Usa:

- `#` secção (fonte de display, ~30px, peso 700)
- `##` subtítulo (~24px, peso 600)
- `###` terceiro nível (~20px, peso 600)
- `**negrito**` e `*itálico*`
- `` `código` ``
- `[texto](https://...)` — só `http` ou `https`
- listas `-` / `*` e listas numeradas `1.`
- citação `> texto`
- linha em branco entre parágrafos. Linhas seguidas juntam-se com um espaço.

Não mandes HTML solto se puderes escrever Markdown. O HTML aceite é só `h1`–`h3`, `p`, `li`, `blockquote`, `strong`/`b`, `em`/`i`, `code`, `a` e `br`.

Depois de criar ou atualizar, chama `posts_get`. O `content` tem de começar por `{"type":"doc"`. Se pediste títulos, tem de haver nós `heading` com `attrs.level` 1, 2 ou 3. Se o corpo voltar como Markdown cru, o servidor ainda não tem a conversão: não reescrevas o artigo em loop; diz que o deploy dessa versão ainda não está ativo.

Não inventes factos, preços ou citações. Se a fonte não diz, o artigo não diz.

## Comentários

| Tool              | Escopo           | Obrigatório         |
| ----------------- | ---------------- | ------------------- |
| `comments_list`   | `comments:read`  | `postId`            |
| `comments_reply`  | `comments:write` | `postId`, `content` |
| `comments_delete` | `comments:write` | `commentId`         |

Para responder a um comentário, passa `parentMessageId`. Sem ele, a resposta fica no post. `limit` máximo é 100.

## Cursos

Ordem: `courses_create` → `chapters_create` → `lessons_create`. Um curso nasce `Draft`. Só passa a `Published` com `courses_update` quando o pedido for publicar.

- `courses_create` exige `communityId`, `title`, `description`. `price` é em centavos (`9700` = R$ 97,00). `level`: `Beginner`, `Intermediate`, `Advanced`.
- `chapters_create` exige `courseId` e `title`. `position` começa em 1.
- `lessons_create` exige `chapterId` e `title`. Vídeo e miniatura são URL ou chave em `videoKey` / `thumbnailKey`.
- `courses_get` devolve capítulos e aulas. Confirma a árvore depois de criar.

## Mídia

`media_prepare_upload` não envia o ficheiro. Com `externalUrl` pública devolve `status: "ready"` e um `attachmentPayload`. Sem URL devolve `pending_upload`: o binário local não entra no bucket por esta tool. Nesse caso usa uma URL pública, ou diz ao utilizador que o ficheiro tem de ser enviado pelo painel.

- capa do post: `targetContext: "post_cover"` e o `url` devolvido em `coverImage`
- áudio: `post_audio` → `audioUrl`
- vídeo: `post_video` → `recordingUrl` ou `externalPlatformUrl`
- anexo: `post_attachment` e o `attachmentPayload` dentro de `attachments`
- capa de curso: `course_cover` → `fileKey` em `courses_create` / `courses_update`
- vídeo de aula: `lesson_video` → `videoKey`

`mediaType`: `image`, `audio`, `video`, `document`.

## O que não fazer

- Não copies o token `rc_live_...` para o chat, para um commit, ou para um ficheiro do repositório.
- Não chames `posts_delete` ou `comments_delete` para "limpar" um teste sem o utilizador ter pedido.
- Não publiques em produção se o MCP estiver apontado para dev, nem o contrário. O host da API é o ambiente.
- Não trates uma falha de descoberta de tools como "o MCP não tem esta função". Autentica e lista de novo.
