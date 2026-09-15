---
type: llm
weight: 1
---

A resposta acerta quando:

- Escolhe enviar um **template aprovado** (`send_template_message`) e **não** `send_text_message`,
  porque a janela de texto livre de 24h nunca abriu — o contato nunca escreveu.
- Manda descobrir o template antes, com `list_whatsapp_templates`, em vez de inventar um nome.
- Explica que `parameters` preenche as variáveis do corpo por posição.

A resposta erra quando recomenda `send_text_message` para esse caso, inventa um nome de template
como se já existisse, ou afirma que dá para mandar texto livre a quem nunca escreveu.

Crédito extra, não obrigatório: menciona que um template MARKETING não cria contato a partir do
telefone, ou que o envio passa por uma política da empresa (opt-out, limite de repetição).
