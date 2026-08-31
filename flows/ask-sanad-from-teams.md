# Flow: Ask Sanad from Teams

A step by step blueprint for the Power Automate flow that puts Sanad behind a Teams channel.

There is no `.zip` solution export in this repository. This file is the build sheet: follow it in the
Power Automate designer and the flow takes a few minutes to assemble.

**Shape:** an employee posts a question in a Teams channel, the flow asks Sanad through this custom
connector, and the flow replies in the same thread or escalates the question to the trainer.

---

## Before you start

| Requirement | Why |
|---|---|
| The custom connector is imported and a connection exists | The flow calls `AskSanad` on it |
| Sanad is reachable over HTTPS from the gateway machine | Sanad itself binds `127.0.0.1` plain HTTP, so a reverse proxy must terminate TLS |
| A Teams channel dedicated to questions | The trigger fires on every message in it |
| A Teams chat or channel for the trainer | The escalation branch posts there |

If Sanad runs inside the corporate network, install the on-premises data gateway on a machine that can
reach the reverse proxy, share that gateway with Admin permission, and pick it when you create the
connection. The connector declares the gateway option; the gateway itself is infrastructure you install.

---

## Step 1 — Trigger

**Microsoft Teams → When a new channel message is added (V3)**

| Field | Value |
|---|---|
| Team | IT Support |
| Channel | Ask IT |

Every message posted in that channel starts the flow, including the replies this flow posts. Step 2
stops that loop.

---

## Step 2 — Stop the loop

**Control → Condition**

The flow posts its answers with the Flow bot identity, which has no user id. A message written by a
person always has one.

| Left | Operator | Right |
|---|---|---|
| The trigger's **User Id** (from dynamic content) | is not equal to | leave empty |

Put everything below inside the **If yes** branch. Leave **If no** empty, so the flow ends quietly on
its own messages.

---

## Step 3 — Turn the message into plain text

Teams channel messages arrive as HTML. Sanad normalizes Arabic text, not markup.

**Content Conversion → Html to text**

| Field | Value |
|---|---|
| Content | the trigger's **Message body content** |

---

## Step 4 — Ask Sanad

**Sanad Support Bot → Ask Sanad**

| Field | Value |
|---|---|
| Session | the trigger's **User Id** |
| Question | the **plain text** output of Step 3 |

Use the Teams user id as the session, not the message id. Sanad rate limits each session to 20
questions per minute, so a per person session makes the limit act per person. A per message session
would switch that limit off.

The action returns `reply`, `matched`, `suggestions`, `queued`, and `chat_id`.

---

## Step 5 — Branch on the score

**Control → Condition**

Rename the Ask Sanad action to `Ask_Sanad` first so the expressions below resolve.

| Left | Operator | Right |
|---|---|---|
| `coalesce(body('Ask_Sanad')?['matched']?['score'], 0)` | is greater than or equal to | `0.85` |

`matched` is null when nothing matched, so `coalesce` supplies 0 and keeps the comparison valid.
0.85 is the same threshold Sanad uses internally to decide it is sure.

### If yes — Sanad is sure

**Microsoft Teams → Reply with a message in a channel (V2)**

| Field | Value |
|---|---|
| Team | IT Support |
| Channel | Ask IT |
| Message Id | the trigger's **Message Id** |
| Message | the **reply** output of Ask Sanad |

To show which article answered, append `body('Ask_Sanad')?['matched']?['title']` on a second line.

### If no — Sanad is not sure

**Control → Condition** nested inside the **If no** branch.

| Left | Operator | Right |
|---|---|---|
| the **queued** output of Ask Sanad | is equal to | `true` |

#### If yes — Sanad did not know

1. **Reply with a message in a channel (V2)** in the same thread, with the **reply** output. Sanad
   already writes an honest "I do not know, I logged it" message, so post it as is.
2. **Microsoft Teams → Post message in a chat or channel**, addressed to the trainer:

   ```
   New question with no answer.
   From: <the trigger's User display name>
   Question: <the plain text from Step 3>
   ```

#### If no — Sanad offered choices

**Reply with a message in a channel (V2)** in the same thread:

| Field | Value |
|---|---|
| Message | the **reply** output, then `join(body('Ask_Sanad')?['suggestions'], '<br>')` |

`reply` already carries the question "Did you mean one of the following?" in the language of the
asker, so do not write your own.

---

## Step 6 — Optional: collect a vote

Send an adaptive card with a thumbs up and a thumbs down instead of a plain reply, and store
`chat_id` in the card data. In a second flow, trigger on **When someone responds to an adaptive
card**, then call **Sanad Support Bot → Rate an answer**:

| Field | Value |
|---|---|
| Chat id | the `chat_id` carried by the card |
| Vote | `1` for up, `-1` for down |

A down vote pushes the original question back into the trainer queue on the Sanad side. No extra
flow logic is needed for that.

---

## Second flow — morning digest

A short scheduled flow that reports what the bot could not answer.

1. **Schedule → Recurrence**, daily at 08:00.
2. **Sanad Support Bot → Get knowledge base statistics**.
3. **Sanad Support Bot → List questions waiting for a trainer**.
4. **Microsoft Teams → Post message in a chat or channel**, to the trainer:

   ```
   Answers: <answers>   Pending: <queue_pending>
   Instant answer rate over 7 days: <instant_answer_rate>%
   Satisfaction: <overall_satisfaction>%
   ```

5. **Control → Apply to each** over the queue output, listing each `question_raw` with the count of
   its variants. Questions with `priority` equal to 1 asked twice in a row and deserve the top of
   the list.

---

## Failures worth handling

| Response | Meaning | What the flow should do |
|---|---|---|
| 429 with a `reply` field | The session passed 20 questions in one minute | Post the `reply` as is. It is written for the employee, not for a log |
| 401 with an `error` field | The Authorization header is missing or wrong | Fix the connection. The connector sends it, so this means the token changed |
| 400 with an `error` field | The question was empty | Reply asking for the question text |

Sanad returns 429 with a `reply` field and no `error` field, so a generic "read the error property"
handler prints nothing. Read `reply` on that branch.

---

## Why the branch reads `queued` and not only the score

The score alone cannot separate "I do not know" from "did you mean". Both arrive under 0.85. Sanad
already made that decision and reports it in `queued`, so the flow reads the decision rather than
recomputing it. If the thresholds move on the server, the flow keeps working.
