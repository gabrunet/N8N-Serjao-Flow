# N8N Serjão Flow

n8n workflow for **Serjão**, the WhatsApp attendant of Soberano Barbearia
(Chatwoot + an AI booking agent).

Node names are kept in Portuguese so they match what you see on the n8n
canvas.

## Files

| File | Description |
|---|---|
| `workflows/serjao.json` | Main workflow, ready to import into n8n |

## Importing

In n8n: **Workflows → ⋯ → Import from File** and pick
`workflows/serjao.json`. You can also copy the file contents and paste them
straight onto the canvas.

## Flow overview

1. **Mensagem recebida** — webhook receiving Chatwoot events.
2. **Info** — normalizes the payload (phone, message, labels, type,
   `isBarber`, etc.).
3. **Mensagem chegando?** — keeps only `incoming` messages that don't carry
   the `agente-off` or `teste` labels.
4. **Tipo de mensagem** — splits text from audio; audio goes through
   download → extract → convert → rename → transcription (Whisper).
5. **Message queue (overlapping messages)** — enqueues into Postgres, waits
   ~16s, reads the queue back and drops overlapping executions before
   clearing it.
6. **Marcar como lidas / auto reaction** — updates `last_seen` and sends a
   reaction based on the message content.
7. **Barber Block → Secretária** — builds the barber-mode block and calls the
   agent (Gemini + Postgres memory + tools via MCP Client, Refletir and
   Escalar humano).
8. **Post-agent handling** — detects `[FORA_DE_ESCOPO]`, drives the counter
   that applies `agente-off` after 3 occurrences, and escalates to a human.
9. **Reply delivery** — validates the output, formats the text for WhatsApp
   and sends it through Chatwoot, toggling the typing/recording status.

## Dependencies

- **n8n credentials:** Chatwoot API, Postgres (Supabase), OpenAI
  (transcription) and Google Gemini (PaLM).
- **Postgres tables:** `n8n_fila_mensagens`, `n8n_historico_mensagens`,
  `n8n_off_topic_counter`.
- **Sub-workflows:** `Escalar Humano` (production) and `Escalar Humano DEV`
  (agent tool).
- **MCP Client:** exposes the booking tools (`list_barbers`, `list_services`,
  `get_available_slots`, `create_booking`, `cancel_booking`,
  `reschedule_booking`, absence management, etc.).

## Masked values

This repository is public, so the identifiers that act as access tokens were
replaced with `00000000-0000-0000-0000-000000000000`. Fill them in with your
own environment's values after importing:

| Node | Field | Placeholder |
|---|---|---|
| **Mensagem recebida** | `path` / `webhookId` | Zeroed UUID — defines the webhook URL Chatwoot calls |
| **MCP Client** | `endpointUrl` | Zeroed UUID at the end of the URL — token for the MCP server holding the booking tools |

Don't commit the real values back while the repository is public.

## Post-import checklist

- Fill in the two masked values from the table above.
- Relink the credentials on each node (the exported IDs belong to the source
  environment).
- Set `telegram_chat_id` on the **Info** node (currently a placeholder).
- Mind the sticky note warning: the calendar e-mails are altered in the
  development environment — review them before promoting to production.
- While testing, lower the **Esperar** node's wait time (defaults to 16s).
