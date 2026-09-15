# Wazapi para agentes de IA

Este repositório publica a **skill do Wazapi**: o documento que ensina um agente de IA a operar
um workspace Wazapi pelo servidor MCP — ler e responder conversas, cuidar de contatos e etiquetas,
montar fluxos, tocar o CRM e a loja, e enviar templates aprovados pela Meta.

Endpoint MCP: `https://wazapi.io/mcp` (Streamable HTTP) · versão do pacote: `1.0.0`

## Instalação

### Claude Code (recomendado)

```bash
/plugin marketplace add wazap-ai/wazapi-agents
/plugin install wazapi@wazapi
```

### Codex, Cursor, GitHub Copilot, Kiro, VS Code

Clientes que seguem o padrão [Agent Plugins 1.0](https://agent-plugins.org) leem este repositório
como um plugin: `plugin.json` na raiz, as skills em `skills/` e o servidor MCP em `mcp.json`.
Clone o repositório na pasta de plugins do seu cliente ou aponte-o para a URL.

### Salvando os arquivos à mão

Qualquer agente que leia arquivos de instrução serve. Copie a pasta inteira:

```
~/.claude/skills/wazapi/
├── SKILL.md
└── reference/
    ├── workspace.md
    ├── conversations.md
    ├── contacts.md
    ├── flows.md
    ├── whatsapp.md
    ├── crm.md
    └── store.md
```

O `SKILL.md` aponta para os arquivos de `reference/` por caminho relativo. Salvar só o primeiro
deixa o agente com o índice e sem a referência.

Para clientes que procuram um arquivo único — Codex e Kimi leem `AGENTS.md`, o Gemini CLI lê
`GEMINI.md` — use as versões monolíticas em [`dist/`](dist).

## Credencial

O tenant vem do token: não existe seletor de empresa e nenhum agente alcança outro workspace.

- **OAuth 2.1** — o caminho preferido para clientes remotos. A tela de consentimento lista uma
  permissão por caixa, e o que ficar marcado é o que o token carrega.
- **Token estático** — emitido em **Configurações → MCP** no painel, para clientes que só aceitam
  um header fixo.

Um token com acesso amplo **não** recebe os escopos sensíveis: eles precisam ser concedidos pelo
nome. É o que impede um agente influenciado por conteúdo de terceiros de repontar o WhatsApp da
empresa.

## O que tem aqui

| Caminho | O que é |
| --- | --- |
| `skills/wazapi/` | A skill: `SKILL.md` + `reference/*.md` |
| `dist/` | `AGENTS.md`, `GEMINI.md` e o índice `wazapi-mcp-tools.json` |
| `plugin.json`, `mcp.json` | Manifesto Agent Plugins 1.0 |
| `.claude-plugin/` | Manifesto e marketplace do Claude Code |

## Este repositório é gerado

Todo o conteúdo sai do código do Wazapi — nome, descrição e schema de cada tool vêm da
introspecção do servidor MCP em produção, não de uma cópia escrita à mão. Por isso **nada aqui
deve ser editado direto**: uma correção na skill é um PR no app, e a publicação seguinte
sobrescreveria a edição.

Achou erro ou falta? Abra uma issue — é o caminho certo, e o texto entra na fonte.

## Licença

MIT. Veja [LICENSE](LICENSE).
