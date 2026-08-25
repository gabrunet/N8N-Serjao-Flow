# N8N Serjão Flow

Workflow do n8n do **Serjão**, atendente WhatsApp da Soberano Barbearia
(Chatwoot + agente de IA com agendamento).

## Arquivos

| Arquivo | Descrição |
|---|---|
| `workflows/serjao.json` | Workflow principal, pronto para importar no n8n |

## Como importar

No n8n: **Workflows → ⋯ → Import from File** e selecione
`workflows/serjao.json`. Também é possível copiar o conteúdo do arquivo e
colar direto no canvas.

## Visão geral do fluxo

1. **Mensagem recebida** — webhook que recebe os eventos do Chatwoot.
2. **Info** — normaliza o payload (telefone, mensagem, etiquetas, tipo,
   `isBarber`, etc.).
3. **Mensagem chegando?** — filtra apenas mensagens `incoming` sem as
   etiquetas `agente-off` e `teste`.
4. **Tipo de mensagem** — separa texto de áudio; o áudio passa por
   download → extract → convert → rename → transcrição (Whisper).
5. **Fila de mensagens encavaladas** — enfileira no Postgres, espera ~16s,
   busca a fila e descarta execuções encavaladas antes de limpar a fila.
6. **Marcar como lidas / reação automática** — atualiza o `last_seen` e
   envia uma reação conforme o conteúdo da mensagem.
7. **Barber Block → Secretária** — monta o bloco de modo barbeiro e chama o
   agente (Gemini + memória Postgres + tools via MCP Client, Refletir e
   Escalar humano).
8. **Tratamentos pós-agente** — detecta `[FORA_DE_ESCOPO]`, controla o
   contador que aplica `agente-off` após 3 ocorrências e escala para humano.
9. **Envio da resposta** — valida a saída, formata o texto para WhatsApp e
   envia pelo Chatwoot, alternando o status de digitando/gravando.

## Dependências

- **Credenciais n8n:** Chatwoot API, Postgres (Supabase), OpenAI (transcrição)
  e Google Gemini (PaLM).
- **Tabelas Postgres:** `n8n_fila_mensagens`, `n8n_historico_mensagens`,
  `n8n_off_topic_counter`.
- **Sub-workflows:** `Escalar Humano` (produção) e `Escalar Humano DEV`
  (tool do agente).
- **MCP Client:** expõe as tools de booking (`list_barbers`, `list_services`,
  `get_available_slots`, `create_booking`, `cancel_booking`,
  `reschedule_booking`, gestão de ausências, etc.).

## Valores mascarados

Este repositório é público, então os identificadores que funcionam como token
de acesso foram substituídos por `00000000-0000-0000-0000-000000000000`.
Preencha com os valores reais do seu ambiente após importar:

| Nó | Campo | Placeholder |
|---|---|---|
| **Mensagem recebida** | `path` / `webhookId` | UUID zerado — define a URL do webhook que o Chatwoot chama |
| **MCP Client** | `endpointUrl` | UUID zerado no final da URL — token do servidor MCP com as tools de booking |

Não comite os valores reais de volta enquanto o repositório for público.

## Ajustes após importar

- Preencher os dois valores mascarados da tabela acima.
- Revincular as credenciais aos nós (os IDs exportados são do ambiente de
  origem).
- Preencher `telegram_chat_id` no nó **Info** (está como placeholder).
- Conferir o aviso do sticky note: os e-mails das agendas estão alterados no
  ambiente de desenvolvimento — revisar antes de subir para produção.
- Ao testar, reduzir o tempo do nó **Esperar** (padrão 16s).
