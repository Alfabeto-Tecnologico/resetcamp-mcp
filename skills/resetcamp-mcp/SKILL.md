---
name: resetcamp-mcp
description: >-
  Publica e edita conteúdo no ResetCamp pelo MCP do criador: comunidades,
  canais, posts (rascunho ou publicado) com emoji no título e no corpo, negrito,
  links, imagem e vídeo no corpo, anexos (PDF e imagem), traduções em múltiplos
  idiomas (pt-BR, en, es, fr), comentários, cursos e mídia (imagem ou documento
  PDF enviados pelo MCP ou URL pública). Use quando o pedido for criar,
  atualizar, traduzir, publicar, despublicar (voltar a rascunho), listar ou
  apagar um post, rascunho, artigo, comentário ou curso na comunidade, enviar
  uma imagem ou um PDF como anexo, instalar ou ligar o MCP do ResetCamp, ou
  quando mencionarem posts_create, posts_create_draft, posts_publish,
  posts_unpublish, posts_update, media_prepare_upload, channels_list ou a ponte
  mcp-bridge.
---

# MCP do ResetCamp

O agente atua **em nome do criador** cujo token está no servidor. Só vê e escreve nas comunidades desse criador. Não inventa IDs, não cria um segundo post quando o pedido é editar, e não apaga nada que o utilizador não tenha pedido para apagar.

Antes de cada chamada, lê o schema vivo da tool (`GetDynamicTools` / `tools/list`). Os nomes e os campos obrigatórios abaixo são o contrato; o schema ganha se divergir. `additionalProperties` é `false`: um campo a mais faz a chamada falhar.

Se o namespace estiver `error` ou `needsAuth`, autentica com `mcp_auth` e lista as tools de novo. Não contornes o MCP com SQL, server actions ou a API HTTP.

## Instalar o MCP

Faz isto só quando o utilizador pedir para instalar, ligar ou configurar o MCP do ResetCamp, ou quando as tools não estiverem disponíveis. Se um servidor `resetcamp` já responder a `tools/list`, não instales outro.

1. Pede o token de criador. Ele começa por `rc_live_` e só aparece uma vez em Definições → Chaves de API & Agentes MCP. Não inventes um token. Não o commits, não o escrevas no chat por extenso, e não o ponhas num ficheiro versionado.
2. O host é a origem dessa página, por exemplo `https://dev.resetcamp.io`. Não uses localhost salvo se o utilizador estiver a desenvolver o monorepo.
3. Precisas de `bun` no PATH.
4. A ponte é o ficheiro `scripts/mcp-bridge.ts` ao lado deste `SKILL.md`. Se não estiver aí, usa `scripts/mcp-bridge.ts` na raiz do monorepo ResetCamp, se existir. Resolve o caminho absoluto.
5. Na pasta que contém esse `scripts/`, corre `bun install` uma vez, para existir `@modelcontextprotocol/sdk`.
6. Junta o servidor ao `mcp.json` sem apagar os outros. Projeto: `.cursor/mcp.json`. Em todo o Cursor: `~/.cursor/mcp.json`. No Claude Desktop: `~/Library/Application Support/Claude/claude_desktop_config.json` no macOS.

```json
{
  "mcpServers": {
    "resetcamp": {
      "command": "bun",
      "args": ["run", "/CAMINHO/ABSOLUTO/scripts/mcp-bridge.ts"],
      "env": {
        "RESETCAMP_API_URL": "https://dev.resetcamp.io",
        "RESETCAMP_TOKEN": "rc_live_..."
      }
    }
  }
}
```

7. Diz ao utilizador para recarregar os servidores MCP no Cursor. Depois confirma com `tools/list`. Se a descoberta falhar, chama `mcp_auth` e lista de novo.

## Antes de escrever

1. Lê a descrição do servidor. Ela diz o nome do criador, os escopos ativos e a versão ("Versão do servidor: 1.5.0.").
2. Esta skill vale para o servidor 1.1.0 ou mais novo. Rascunho, publicação e envio de imagem só existem se `posts_create_draft`, `posts_publish` e `media_confirm_upload` estiverem no `tools/list`. Se faltarem, o servidor é antigo: diz isso ao utilizador. Não uses `posts_create` no lugar de um rascunho. Voltar um post publicado a rascunho só existe a partir do 1.2.0, com `posts_unpublish` no `tools/list`; num servidor mais antigo diz que não dá, e não apagues o post para "despublicar". O envio de PDF em `post_attachment` e as traduções (`translation` / `translations`) entram no 1.5.0.
3. Se o escopo da tool falta, para e diz qual escopo falta. Não tentes a chamada à mesma.
4. Resolve IDs por tool, nesta ordem:
   - `communities_list` (sem argumentos, ou `includeArchived: true` se a comunidade não aparecer)
   - `channels_list` com o `communityId`, se fores publicar num canal
   - `posts_list` ou `courses_list` se o pedido puder referir-se a algo que já existe
5. `threadCount` em comunidades e canais pode ficar em 0 mesmo com posts. Não uses esse contador para decidir se a comunidade está vazia. Usa `posts_list`.

URL de um post, com o slug da comunidade e o `slug` do canal:

- membros: `/c/{slug}/{channelSlug}/{postId}`
- público (só se o post foi criado com `isPublic: true` por `posts_create`, ou tornado público pela plataforma): `/post/{postId}`

O host é o da API a que o MCP está ligado (`RESETCAMP_API_URL`). Não assumes localhost nem produção.

## Posts

| Tool                 | Escopo        | Obrigatório                                       | Quando                                              |
| -------------------- | ------------- | ------------------------------------------------- | --------------------------------------------------- |
| `posts_list`         | `posts:read`  | nenhum                                            | Achar um post antes de criar outro                  |
| `posts_get`          | `posts:read`  | `postId`                                          | Confirmar o que ficou gravado                       |
| `posts_create`       | `posts:write` | `channelId`, `title`, `content`                   | Post novo, **publicado na hora**                    |
| `posts_create_draft` | `posts:write` | `channelId`, `title`, `content`, `idempotencyKey` | Post novo **em rascunho**, sem publicar             |
| `posts_publish`      | `posts:write` | `postId`                                          | Publicar um rascunho                                |
| `posts_unpublish`    | `posts:write` | `postId`                                          | Voltar um post publicado a rascunho (despublicar)   |
| `posts_update`       | `posts:write` | `postId`                                          | Editar o mesmo post, publicado ou rascunho          |
| `posts_delete`       | `posts:write` | `postId`                                          | Só quando o pedido for apagar                       |

`isPublic` existe em `posts_create` e `posts_create_draft` e o padrão é `false`. Um artigo que o utilizador quer aberto na internet leva `isPublic: true` em `posts_create`, e só numa comunidade que permite posts públicos (nas outras, a chamada dá erro). O rascunho é sempre privado: `posts_create_draft` com `isPublic: true` dá erro. Para abrir na internet um post que nasceu em rascunho, o utilizador torna-o público pela plataforma depois de publicar. `posts_update` não altera esse flag.

`isPinned` fixa o post no topo do canal. Existe em `posts_create` e `posts_update`, não no rascunho. Não fixes sem pedido.

Áudio vai em `audioUrl`. Capa vai em `coverImage` com uma URL pública `https`. Vídeo do YouTube ou do Vimeo vai **no corpo** (ver "Imagem e vídeo no corpo"). `recordingUrl` (ficheiro de vídeo direto, MP4 ou HLS) e `externalPlatformUrl` só aparecem em eventos: num post comum não os uses para o vídeo.

O recorte da capa vai em `coverImageCropData` (servidor 1.4.0 ou mais novo): `{ x, y, width, height, zoom, aspectRatio }`, o mesmo formato que o seletor de capa da plataforma guarda. `x`, `y`, `width` e `height` são percentagens da imagem; `zoom` (1 = sem zoom) e `aspectRatio` (largura/altura da área mostrada, 16/9 se faltar) são opcionais. Só vale junto com uma capa. **Sem recorte, a plataforma mostra a capa numa faixa de altura fixa e corta-lhe o alto e o baixo.** Para mostrar a capa inteira, manda `{ "x": 0, "y": 0, "width": 100, "height": 100, "zoom": 1, "aspectRatio": <largura/altura da imagem> }` (uma imagem 1600×900 tem `aspectRatio` `1.7778`).

### Rascunho: `posts_create_draft` e `posts_publish`

Quando o utilizador pede um rascunho, ou quer rever antes de publicar:

1. Chama `posts_create_draft`. Nunca uses `posts_create` para isto: ele publica na hora.
2. Guarda o `postId` da resposta. O link do post já existe.
3. Edita com `posts_update` até o utilizador aprovar.
4. Só quando ele disser para publicar, chama `posts_publish` com o `postId`.

`idempotencyKey` é obrigatória: um texto de 1 a 200 caracteres que identifica **este** rascunho (por exemplo `ep25-post-comunidade`). Usa uma chave estável por rascunho e uma chave nova para cada rascunho novo.

- Se a resposta se perder, repete a chamada com a **mesma chave e o mesmo conteúdo**. Volta o mesmo post, com `replayed: true`. Não nasce um segundo post.
- A mesma chave com conteúdo diferente dá erro e cita o post. Edita esse post com `posts_update`.
- Se o post dessa chave foi apagado, o erro pede outra chave.
- Se a chave já foi usada noutra comunidade (com o token travado noutra), o erro também pede outra chave.

`posts_publish` passa `Draft` a `Active`: o post entra na contagem do canal e da comunidade. A data do post (`createdAt`) passa a ser a da publicação, para ele aparecer no topo do feed. Repetir num post já publicado não muda nada (`published: false`): só renova o cache da página. Usa isso se a página não abrir logo a seguir a publicar.

### Despublicar: `posts_unpublish`

Quando o utilizador pede para despublicar, tirar do ar, esconder para corrigir ou voltar um post a rascunho, chama `posts_unpublish` com o `postId`. Nunca apagues o post para isso.

- `Active` passa a `Draft`: o post sai do feed, do canal, da página pública e da IA da comunidade e sai da contagem. Vira um rascunho **privado** do autor (`isPublic` passa a `false`). Comentários, curtidas, anexos e o link ficam. Ninguém é notificado.
- Corrige com `posts_update` e publica de novo com `posts_publish`. Ele volta com a **data original** (não sobe ao topo do feed) e sem pontos de novo. Se era público, o utilizador torna-o público outra vez pela plataforma.
- Repetir num post que já é rascunho não muda nada (`unpublished: false`).
- Recusa, com a razão no erro: post de **outro autor** (viraria rascunho privado dele), post **trancado** pela moderação e **evento**. Conta ao utilizador a razão; não tentes contornar.
- Só despublica quando o utilizador pedir.

O autor também tem os dois botões na plataforma: **Publicar** no aviso de rascunho da página do post e **Voltar para rascunho** no menu do post.

**O rascunho só é visível ao autor**, na plataforma e pelo MCP. `posts_get`, `posts_update`, `posts_publish`, `posts_unpublish` e `posts_delete` de um rascunho de outro autor respondem "não encontrado". Ninguém mais o abre pelo link.

`posts_get` e `posts_list` devolvem `status` (`Active`, `Draft` ou `Locked`). `posts_list` sem filtro traz os publicados e os teus rascunhos. Para listar só rascunhos usa `status: "Draft"`. Post apagado nunca aparece.

### Editar: `posts_update` é PATCH

Campo ausente **mantém** o valor. Só manda o que muda.

- `""` (texto vazio) remove: `coverImage`, `audioUrl`, `recordingUrl`, `externalPlatformUrl`. `null` também é aceite. `coverImageCropData` remove-se só com `null`.
- `coverImageCropData` ausente mantém o recorte guardado; um objeto substitui-o (o post tem de ter capa, a atual ou a que vai na mesma chamada). Trocar a `coverImage` por outra URL **sem** mandar o recorte apaga o recorte antigo, porque era da imagem anterior. Remover a capa apaga-o também, e mandar um recorte na mesma chamada que remove a capa dá erro.
- `[]` limpa: `tags`, `attachments`.
- `translation: null` desativa a tradução do post e apaga as traduções guardadas; em `translations`, `{ "<locale>": null }` apaga apenas a tradução daquele idioma.
- Não há `status`: para publicar usa `posts_publish`; para voltar a rascunho, `posts_unpublish`.
- Manda sempre `expectedUpdatedAt` com o `updatedAt` que o `posts_get` devolveu, tal e qual. Se alguém mexeu no post na plataforma depois dessa leitura, a chamada falha ("o post mudou durante a edição") e nada é gravado: lê de novo com `posts_get` e refaz a edição. **Sem `expectedUpdatedAt` não há essa conferência**: a edição grava por cima da mudança.
- Cada `posts_update` muda o `updatedAt`. Antes da edição seguinte, lê de novo com `posts_get`.

### Traduções: `translation` e `translations`

Os idiomas suportados pela plataforma são `pt-BR`, `en`, `es` e `fr`. `posts_create`, `posts_create_draft` e `posts_update` aceitam (servidor 1.5.0 ou mais novo):

- `translation`: configuração de tradução do post:
  - `sourceLocale` (obrigatório quando `translation` é enviado): idioma original do post (`pt-BR`, `en`, `es` ou `fr`).
  - `followPlatformLocales` (opcional, padrão `true`): quando `true`, acompanha todos os idiomas da plataforma exceto `sourceLocale` e `excludedLocales`. Quando `false`, traduz apenas para `selectedLocales`.
  - `excludedLocales` (opcional): idiomas excluídos quando `followPlatformLocales: true`.
  - `selectedLocales` (opcional): idiomas-alvo quando `followPlatformLocales: false`.
- `translations`: mapa de traduções prontas `{ "<locale>": { "title": "...", "content": "..." } }` (ou `null` em `posts_update` para remover um idioma).
  - `content` aceita Markdown ou JSON TipTap, igual ao corpo principal do post.
  - Não envies `sourceLocale` dentro de `translations` (o idioma original já é o `title` e `content` do próprio post).
  - Se enviares `translations` num post que ainda não tem `sourceLocale`, manda também `translation.sourceLocale` na mesma chamada.
  - Toda tradução enviada em `translations` é gravada com `status: "ready"` e **nunca é sobrescrita** pelos workers de tradução por IA.
  - Quando o post é `Active` (em `posts_create`, `posts_update` ou ao publicar um rascunho com `posts_publish`), os idiomas-alvo configurados em `translation` que **não** foram enviados em `translations` são agendados automaticamente para tradução por IA em segundo plano. Num `Draft`, as traduções manuais ficam guardadas imediatamente e a IA só é disparada quando o rascunho for publicado.
- `posts_get` devolve `translation` (`{ sourceLocale, followPlatformLocales, excludedLocales, selectedLocales }`) e `translations` (lista ordenada por idioma com `{ locale, title, content, status, errorMessage, updatedAt }`).

### Ícones: no título e no corpo

Não há campo de ícone. O emoji é um caractere Unicode no texto, nos dois sítios:

- **título** — `title` é texto simples, sem Markdown. `🚀 Bem-vindos à Comunidade` aparece com o foguete no cabeçalho do post.
- **corpo** — o mesmo caractere no Markdown, no início da linha ou no meio da frase. `🤖 **Inteligência Artificial**` grava o emoji e o negrito.

Não uses shortcodes (`:rocket:`). O servidor não os converte.

### Corpo: Markdown, não texto puro

`content` aceita Markdown ou um JSON TipTap que já seja `{ "type": "doc", ... }`. O servidor grava o documento do editor. Sem isso a página mostra texto puro, sem título nem negrito.

Usa:

- `#` secção (fonte de display, ~30px, peso 700)
- `##` subtítulo (~24px, peso 600)
- `###` terceiro nível (~20px, peso 600)
- `**negrito**` e `*itálico*`
- `` `código` ``
- `[texto](https://...)` — o texto visível é o que está entre colchetes. Só `http` ou `https`
- uma URL crua `https://...` também vira link clicável, com o próprio endereço como texto. O ponto final da frase fica de fora
- listas `-` / `*` e listas numeradas `1.`
- citação `> texto`
- linha em branco entre parágrafos. Linhas seguidas juntam-se com um espaço.

### Imagem e vídeo no corpo

- **imagem**: uma linha só com `![descrição](https://url-da-imagem)` vira um bloco de imagem. A URL tem de ser `https` completa.
- **vídeo**: uma linha só com `![](https://www.youtube.com/watch?v=...)` ou `![](https://vimeo.com/...)` vira o vídeo embutido. É assim que um post comum leva vídeo.
- uma imagem no meio de um parágrafo quebra o parágrafo em dois, com a imagem no meio. Põe cada imagem na sua linha.
- para uma imagem do computador do utilizador, envia primeiro (ver "Mídia: enviar uma imagem") e usa o `publicUrl` devolvido.

Não mandes HTML solto se puderes escrever Markdown. O HTML aceite é só `h1`–`h3`, `p`, `li`, `blockquote`, `strong`/`b`, `em`/`i`, `code`, `a` e `br`.

Depois de criar ou atualizar, chama `posts_get`. O `content` tem de começar por `{"type":"doc"`. Se pediste títulos, tem de haver nós `heading` com `attrs.level` 1, 2 ou 3. Se o corpo voltar como Markdown cru, o servidor ainda não tem a conversão: não reescrevas o artigo em loop; diz que o deploy dessa versão ainda não está ativo.

Não inventes factos, preços ou citações. Se a fonte não diz, o artigo não diz.

## Comentários

| Tool              | Escopo           | Obrigatório                  |
| ----------------- | ---------------- | ---------------------------- |
| `comments_list`   | `comments:read`  | `postId`                     |
| `comments_reply`  | `comments:write` | `postId`, `content`          |
| `comments_update` | `comments:write` | `commentId`, `content`       |
| `comments_delete` | `comments:write` | `commentId` (`confirm?: false` para prévia) |

Para responder a um comentário, passa `parentMessageId`. Sem ele, a resposta fica no post. `limit` máximo é 100.

## Cursos

Ordem: `courses_create` → `chapters_create` → `lessons_create`. Um curso nasce `Draft`. Só passa a `Published` com `courses_update` quando o pedido for publicar. Antes de publicar, usa `course_diagnostic_360` para validar capítulos vazios e aulas sem vídeo.

- `courses_create` exige `communityId`, `title`, `description`. `price` é em centavos (`9700` = R$ 97,00). `level`: `Beginner`, `Intermediate`, `Advanced`.
- `chapters_create` exige `courseId` e `title`. `position` começa em 1. `chapters_update` e `chapters_delete` (`confirm: false` para prévia) gerem os capítulos.
- `lessons_create` exige `chapterId` e `title`. Vídeo e miniatura são URL ou chave em `videoKey` / `thumbnailKey`. `lessons_update` e `lessons_delete` (`confirm: false` para prévia) gerem as aulas.
- `courses_get` devolve capítulos e aulas. Confirma a árvore depois de criar.

## Diagnósticos 360°, Eventos, Membros/Moderação, Planos e Resources (Onda 1 & Onda 2)

- **Resources (`resources/read`)**:
  - `resetcamp://me/context`: lê num único passo o utilizador, todas as comunidades e os seus canais, `defaultCommunityId` e cotas de IA/storage. Prefere isto a chamar `communities_list` + `channels_list` separados quando precisares do panorama inicial.
  - `resetcamp://catalog/domains`: catálogo dos 9 domínios e tools.
  - Templates: `resetcamp://community/{slugOrId}`, `resetcamp://post/{postId}`, `resetcamp://course/{courseId}`.
- **Visões 360° e Preflights**:
  - `community_overview_360`: resumo executivo da comunidade, membros, posts sem resposta, rascunhos, cursos e `recommendedActions`.
  - `post_engagement_360`: métricas do post, status de traduções e comentários de alunos que ainda não tiveram resposta do criador.
  - `course_diagnostic_360`: árvore completa do curso + preflight de publicação (`readyToPublish`, `blockers`, `warnings`).
  - `member_profile_360`: perfil do aluno, membresia, pontos/ranking no Leaderboard e histórico de moderação.
- **Canais, Eventos, Membros/Moderação e Planos**:
  - Canais: `channels_create`, `channels_update`.
  - Eventos/Lives: `events_list`, `events_get`, `events_create`, `events_update`.
  - Membros e Moderação: `members_list`, `moderation_reports_list`, `moderation_resolve_report` (`confirm: false` para prévia), `waitlist_list`, `invite_links_get_or_create`.
  - Planos e Assinaturas: `plans_list`, `plans_check_buyability`, `subscriptions_summary_360`, `coupons_list`.
  - Telemetria: `mcp_telemetry_get`.

## Mídia

`media_prepare_upload` tem dois modos.

**Modo de envio (`PUT` assinado)** — quando **não** há `externalUrl` e o contexto é `post_cover`, `post_image` ou `post_attachment`: reserva um upload no S3 com `upload.url`, `intentId`, `fileId`, `key`, `publicUrl` e (para `post_attachment`) `attachmentPayload`.

**Com `externalUrl` pública**, ou noutro `targetContext` (`post_audio`, `post_video`, `course_cover`, `lesson_video`): não envia ficheiro nenhum. Com `externalUrl`, devolve `status: "ready"` e `attachmentPayload`. Sem URL nesses outros contextos, devolve `pending_upload`: o binário não entra no bucket por esta tool.

- capa do post: `post_cover` e o `publicUrl` devolvido em `coverImage`
- imagem no corpo: `post_image` e o `publicUrl` em `![descrição](url)`
- anexo de post (PDF ou imagem): `post_attachment` e o `attachmentPayload` dentro de `attachments` em `posts_create`, `posts_create_draft` ou `posts_update`
- áudio: `post_audio` → `audioUrl`
- vídeo do YouTube ou do Vimeo: não passa por aqui, vai no corpo como `![](url do vídeo)`
- vídeo (ficheiro direto): `post_video` → `recordingUrl` (só aparece em eventos, como `externalPlatformUrl`)
- capa de curso: `course_cover` → `fileKey` em `courses_create` / `courses_update`
- vídeo de aula: `lesson_video` → `videoKey`

`mediaType`: `image`, `audio`, `video`, `document`.

### Mídia: enviar uma imagem (`post_cover`, `post_image`, `post_attachment`)

Para a capa de um post (`post_cover`), uma imagem no corpo (`post_image`) ou uma imagem em anexo (`post_attachment`) **sem** `externalUrl`, o MCP envia o ficheiro por uma URL assinada. Serve para PNG, JPEG, WebP e GIF, até 10 MB.

1. Chama `media_prepare_upload` com `targetContext` (`post_cover`, `post_image` ou `post_attachment`), `fileName`, `mediaType: "image"`, `mimeType` (`image/png`, `image/jpeg`, `image/webp` ou `image/gif`), `fileSize` (os bytes exatos do ficheiro) e uma `idempotencyKey` estável por ficheiro (por exemplo `ep25-capa`).
2. Lê o `status` da resposta:
   - `awaiting_upload`: faz `PUT` do ficheiro em `upload.url`, com os `upload.headers` e um corpo de **exatamente** `fileSize` bytes. A URL vale 360 segundos. `HTTP 412` quer dizer que o ficheiro já está lá: segue em frente. Depois chama `media_confirm_upload` com `intentId` e `fileId`.
   - `confirmed`: já foi enviado. Não há nada a enviar.
   - `attached`: o ficheiro dessa chave já está ligado a um post (`threadId` na resposta). Reutiliza só nesse post. Se esse post foi apagado, o erro pede outra `idempotencyKey`.
3. `media_confirm_upload` devolve `confirmed` com `key`, `publicUrl` (e `attachmentPayload` quando `post_attachment`), ou `pending` com `retryAfterMs`. Em `pending`, espera esse tempo e chama de novo.
4. Usa o `publicUrl`: em `coverImage`, ou no corpo como `![descrição](publicUrl)` (ou `attachmentPayload` em `attachments` se for `post_attachment`).

### Mídia: enviar um documento PDF como anexo (`post_attachment`)

Para anexar um documento PDF a um post (`targetContext: "post_attachment"`, `mediaType: "document"`) **sem** `externalUrl` (servidor 1.5.0 ou mais novo):

1. Chama `media_prepare_upload` com:
   - `targetContext: "post_attachment"`
   - `mediaType: "document"`
   - `mimeType: "application/pdf"`
   - `fileName`: nome do ficheiro (ex.: `"guia-como-escolher-banco-de-dados.pdf"`)
   - `fileSize`: tamanho exato em bytes, até **25 MB** (`26_214_400` bytes)
   - `idempotencyKey`: chave estável por ficheiro (ex.: `"epzz-guia-pt-pdf"`)
2. Se `status` for `"awaiting_upload"`, faz `PUT` do binário em `upload.url` com os `upload.headers` (`Content-Type: application/pdf` etc.) e exatamente `fileSize` bytes (`HTTP 200` ou `HTTP 412` se já enviado).
3. Chama `media_confirm_upload` com `intentId` e `fileId` até receber `status: "confirmed"`.
4. Para documentos de anexo, `publicUrl` é `null` (os anexos não têm URL pública permanente: na plataforma, o download passa pela rota autenticada `/api/threads/{threadId}/attachments/{attachmentId}` e exige membro logado da comunidade).
5. Passa o `attachmentPayload` devolvido (`{ id, name, type: "document", url: "<s3Key>", size, mimeType: "application/pdf" }`) dentro do array `attachments` de `posts_create`, `posts_create_draft` ou `posts_update`.

Se a resposta do `PUT` ou do `media_prepare_upload` se perder, repete `media_prepare_upload` com a **mesma** `idempotencyKey` e o mesmo ficheiro. Volta a mesma reserva, com URL nova. Outro ficheiro na mesma chave é recusado. Se o servidor disser que o ficheiro enviado não bate com o tamanho ou o tipo reservado, reserva de novo com **outra** `idempotencyKey`.

A imagem ou o documento só fica ligado ao post quando o post é gravado (`posts_create`, `posts_create_draft` ou `posts_update`) com o `publicUrl` no `coverImage`/corpo ou com o `attachmentPayload` em `attachments`. Usa só ficheiros enviados por este mesmo criador.

## O que não fazer

- Não copies o token `rc_live_...` para o chat, para um commit, ou para um ficheiro do repositório.
- Não uses `posts_create` quando o utilizador quer um rascunho. Usa `posts_create_draft` e, depois do aval dele, `posts_publish`.
- Não chames `posts_publish` sem o utilizador ter pedido para publicar, nem `posts_unpublish` sem ele ter pedido para despublicar.
- Não reutilizes uma `idempotencyKey` de rascunho ou de upload para outro conteúdo.
- Não chames `posts_delete` ou `comments_delete` para "limpar" um teste sem o utilizador ter pedido.
- Não publiques em produção se o MCP estiver apontado para dev, nem o contrário. O host da API é o ambiente.
- Não trates uma falha de descoberta de tools como "o MCP não tem esta função". Autentica e lista de novo.
