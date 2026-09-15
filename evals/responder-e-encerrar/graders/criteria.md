---
type: llm
weight: 1
---

A resposta acerta quando cita os efeitos colaterais que a assinatura das tools não revela:

- Enviar por `send_text_message` **atribui a conversa ao dono do token** e muda o status para
  `pending`.
- O envio **encerra o Agente de IA** que estiver atendendo aquela conversa: a sessão sai como
  handoff e o bot não responde a próxima mensagem do cliente.
- Ao encerrar com `update_conversation_status`, a empresa **pode exigir a etapa do CRM**
  (`crmStageUuid`); a recusa lista as etapas permitidas, e etapa de perda pede motivo.

A resposta erra quando descreve os dois passos como operações sem efeito colateral, ou afirma que
encerrar a conversa é sempre aceito sem informação adicional.

Crédito extra, não obrigatório: lembra de conferir o `ok` do retorno (uma recusa não vira erro de
tool) ou de confirmar o texto com a pessoa antes de enviar.
