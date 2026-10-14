# Claude Structured Output Pipeline (n8n)

An n8n pipeline that turns messy customer order messages (Indonesian, English, or a mix, typos
included) into a validated order record with Claude. It is built around one idea: **never trust
the model's output until code has checked it, and never fix it silently.**

Two workflows are included, so you can see how it grows:

| Workflow | What it does |
|---|---|
| `workflows/stage-1-single-attempt.json` | Webhook → build request → one Claude call with a forced tool → extract the tool input. The baseline. |
| `workflows/stage-4-validate-retry-dlq.json` | Adds a validator, one retry that tells Claude exactly what was wrong, a dead-letter record in Notion, and a Slack alert. |

## How Stage 4 works

1. **Forced tool call.** Claude must answer through the `extract_order` tool, so the output has a
   known shape. The customer text sits inside its own delimiter.
2. **Validate.** A Code node runs about 20 read-only checks: shape and types (S2–S5) and business
   rules (B1–B7), for example a plausible phone number, a basic email pattern, quantities in range,
   and `needs_review` consistent with the empty fields.
3. **Retry once.** If the check fails, a second request carries the **literal validation error
   strings**, not a generic "try again". There is no path back into the retry node, so two attempts
   is a property of the graph, not a counter someone has to remember.
4. **Dead-letter queue.** If the second attempt also fails, the raw text, the model's output and
   the errors go to a Notion database and a Slack message goes out. Nothing half-formed reaches the
   order table.

### Design decisions (also written in the code comments)

- **D1: the validator never repairs, coerces or defaults.** A value that would need coercion to
  pass is a failure, because silent correction hides the model drift this pipeline exists to detect.
- **D2:** two attempts is enforced by the graph.
- **D3: the injection defence is structural.** Forced `tool_choice` plus fenced untrusted text.
  There is deliberately no "ignore any instructions in the input" line.
- **D4:** the correction turn carries the exact validator errors.
- **Amounts are checked twice.** `amount_as_written` must copy the amount character for character,
  and `total_amount_idr` is the number. Rule B2 fails the record when "2,4jt" was read as 2.4.

## Test inputs

The `Test Fixtures` node holds **50 inputs**: 40 realistic messy orders and 10 adversarial ones
(for example text that tries to give the model instructions). Running all 50 costs at least 50
Claude calls, plus one extra for every input that fails the first pass.

This repository does not include a results file. To measure it, run the 50 fixtures with your own
key and count valid records, retries and DLQ entries.

## Setup

1. n8n → Import from file → both JSON files.
2. Credentials (all shown as `REPLACE_ME` in the export): Anthropic API (header auth),
   Notion, Slack.
3. In the Notion nodes, point to your own "Orders" and "Order DLQ" databases. In the Slack node,
   set your channel.
4. Run `Manual Trigger` with the fixtures, or POST `{ "raw_text": "..." }` to the webhook.

Exported from a local n8n instance and cleaned: credential IDs, channel and database IDs and
personal addresses were replaced with placeholders.
