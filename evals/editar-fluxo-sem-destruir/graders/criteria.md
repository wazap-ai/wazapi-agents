---
type: llm
weight: 1
---

A resposta acerta quando:

- Avisa que `update_flow_graph` **substitui o grafo inteiro** — nós e arestas ausentes do payload
  são apagados — e que por isso é preciso buscar o grafo atual com `get_flow` e devolvê-lo
  completo, com a alteração aplicada.
- Passa por `validate_flow_graph` antes de `update_flow_graph`.
- Descobre a forma do bloco em vez de inventar campos, com `get_flow_block_schema` (e/ou
  `list_flow_block_types`).

A resposta erra quando trata o update como patch, sugere enviar só o nó novo, ou pula a validação.
