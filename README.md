# ResetCamp MCP

Skill para agentes publicarem e editarem conteúdo no ResetCamp pelo MCP do criador: comunidades, canais, posts em rascunho ou publicados (emoji, negrito, links, imagem e vídeo no corpo), comentários, cursos e mídia (imagem enviada pelo MCP ou por URL pública). Precisa do servidor MCP 1.1.0 ou mais novo.

```bash
npx skills add Alfabeto-Tecnologico/resetcamp-mcp
```

Depois pede ao agente para instalar o MCP do ResetCamp. A skill traz a ponte e os passos para gravar o `.cursor/mcp.json`. O token `rc_live_...` cria-se em Definições → Chaves de API & Agentes MCP e não vai neste repositório.

O agente passa a actuar em nome do criador cujo token está no servidor MCP. A skill não inclui chaves, tokens nem dados de uma comunidade.

Página: https://skills.sh/Alfabeto-Tecnologico/resetcamp-mcp
