# sanad-power-connector — Sanad inside Power Automate

[العربية](README.md)

**An employee's question in Teams goes in, Sanad's approved answer comes out, with no new code.**

A Power Platform custom connector in two files: 4 operations, 12 data definitions, and 52 field names. Every one of those names was taken from the [Sanad](https://github.com/defeat11/sanad) source, not guessed.

Sanad is a support bot that answers from what it was trained on, or says plainly that it does not know. It runs on port 4600 and listens on `127.0.0.1` only. This repository is the bridge: an OpenAPI 2.0 definition and a connector properties file that Power Platform imports as they are.

---

## The problem

Sanad runs inside the network and never reaches the internet. That is deliberate. Meanwhile employees live in Teams, and nobody opens a second web page to ask how to reset a password.

The usual answer is to write a Teams bot, then register it, host it, and keep it alive. That is a lot of machinery to move one string of text.

Power Automate already reaches Teams, and already reaches the internal network through the on-premises data gateway. Only one piece was missing: a definition telling Power Platform how to talk to Sanad.

---

## How it works

The connector exposes four operations. All four are real routes in Sanad:

| Operation | Request | Where it lives in Sanad | What comes back |
|---|---|---|---|
| Ask Sanad | `POST /api/chat` | `routes/chat.js:38` | The reply, the matched answer, and its score |
| Rate an answer | `POST /api/feedback` | `routes/chat.js:59` | Confirmation that the vote was stored |
| List pending questions | `GET /api/queue` | `routes/train.js:261` | Questions waiting for a trainer |
| Get statistics | `GET /api/stats` | `routes/train.js:621` | Counters for the answer base |

`Ask Sanad` is the one that matters. Send `session` and `text`, get back five fields: `reply`, `matched`, `suggestions`, `queued`, and `chat_id`.

The last three carry Sanad's own decision rather than an interpretation of it:

| What returns | What it means | Where it is decided |
|---|---|---|
| `matched.score` at 0.85 or above | An approved answer, returned immediately | `T_AUTO` in `lib/engine.js` |
| `suggestions` is not empty | A middling match, so three titles are offered | `T_BRAIN` in `lib/engine.js` |
| `queued` is `true` | Sanad does not know, and the question entered the trainer queue | `lib/engine.js` |

Sanad limits each `session` to 20 questions per minute. The flow therefore passes the Teams user id into that field, which turns the limit into a per person limit.

When the limit is hit, Sanad returns 429 carrying only a `reply` field and no `error` field. The connector documents that different shape because it is the shape the server actually sends.

The full flow, from a Teams message to the reply: [`flows/ask-sanad-from-teams.md`](flows/ask-sanad-from-teams.md).

---

## The key design decision

**The problem:** in `server.js` the chat router is mounted before the training router, and both sit on `/api`. A request to `/api/chat` is served by the first one and never reaches the token middleware inside the second. So two operations need no token and two do.

**The decision:** one security definition applied to all four operations, not to two.

**The cost:** the connection screen asks for a token even from a flow that only asks questions.

**The benefit:** one connection serves every flow. And if the middleware later moves in front of `/api/chat`, no existing flow breaks.

Authentication is an API key in the `Authorization` header, in the exact shape Sanad reads: the word `Bearer`, a space, then the value of `SANAD_TOKEN`.

---

## Importing it

This is a definition that is **ready to import**, built against an API that genuinely runs. It has not been published to a Microsoft 365 tenant, and nothing here claims otherwise.

From the portal:

1. Power Automate → Custom connectors → New → Import an OpenAPI file.
2. Pick `apiDefinition.swagger.json`.
3. Change `host` from `sanad.contoso.local` to your Sanad host name.
4. On the security tab choose API Key, header `Authorization`.
5. Create the connection and paste `Bearer`, a space, then the token.

Or from the command line:

```bash
paconn create --api-def apiDefinition.swagger.json --api-prop apiProperties.json
```

Sanad speaks plain HTTP, and custom connectors require HTTPS, so a reverse proxy has to terminate TLS in front of it.

`apiProperties.json` declares the on-premises data gateway option through a `gatewaySetting` parameter, which is what surfaces the choice on the connection screen. Installing the gateway and sharing it with Admin permission happens outside this repository.

### Proof

```
$ python -m json.tool apiDefinition.swagger.json > /dev/null && echo ok
ok
$ python -c "import json; d=json.load(open('apiDefinition.swagger.json',encoding='utf-8')); print(len(d['paths']), len(d['definitions']))"
4 12
```

I also searched `routes/` and `lib/` in the Sanad repository for each of the 52 field names under `definitions`. All 52 were found. Zero invented names.

---

## Why I built it

Sanad answers inside the network, and employees ask their questions in Teams.

Closing that gap took a definition and nothing else: 0 new lines of code in Sanad.

---

MIT — © 2026 Osaid Ahmad
