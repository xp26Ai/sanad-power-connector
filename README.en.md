# sanad-power-connector — Sanad inside Power Automate

[العربية](README.md)

**An employee asks in Teams. Sanad's approved answer comes back. No new code.**

A Power Platform custom connector in two files: 4 operations, 12 data definitions, and 52 field names. I took every one of those names from the [Sanad](https://github.com/defeat11/sanad) source. I did not guess any of them.

Sanad is a support bot. It answers from what it was trained on. If it does not know, it says so plainly. It runs on port 4600 and listens on `127.0.0.1` only. This repository is the bridge. It holds an OpenAPI 2.0 definition and a connector properties file. Power Platform imports both as they are.

---

## The problem

Sanad runs inside the network and never reaches the internet. That is on purpose. But employees work in Teams. Nobody opens a second web page to ask how to reset a password.

The usual answer is to write a Teams bot. Then you register it, host it, and keep it alive. That is a lot of machinery to move one line of text.

Power Automate already reaches Teams. It also reaches the internal network through the on-premises data gateway. Only one piece was missing. Power Platform needed a definition to talk to Sanad.

---

## How it works

The connector has four operations. All four are real routes in Sanad:

| Operation | Request | Where it lives in Sanad | What comes back |
|---|---|---|---|
| Ask Sanad | `POST /api/chat` | `routes/chat.js:38` | The reply, the matched answer, and its score |
| Rate an answer | `POST /api/feedback` | `routes/chat.js:59` | Confirmation that the vote was saved |
| List pending questions | `GET /api/queue` | `routes/train.js:261` | Questions waiting for a trainer |
| Get statistics | `GET /api/stats` | `routes/train.js:621` | Counters for the answer base |

`Ask Sanad` is the one that matters. You send `session` and `text`. You get back five fields: `reply`, `matched`, `suggestions`, `queued`, and `chat_id`.

The last three carry Sanad's own decision. They are not my reading of it:

| What returns | What it means | Where it is decided |
|---|---|---|
| `matched.score` at 0.85 or above | An approved answer, sent back at once | `T_AUTO` in `lib/engine.js` |
| `suggestions` is not empty | A partial match, so Sanad offers three titles | `T_BRAIN` in `lib/engine.js` |
| `queued` is `true` | Sanad does not know, so the question went to the trainer queue | `lib/engine.js` |

Sanad limits each `session` to 20 questions per minute. So the flow puts the Teams user id in that field. This turns the limit into a per person limit.

When the limit is hit, Sanad returns 429. The body carries only a `reply` field and no `error` field. The connector documents that different shape. It is the shape the server really sends.

The full flow, from a Teams message to the reply: [`flows/ask-sanad-from-teams.md`](flows/ask-sanad-from-teams.md).

---

## The key design decision

**The problem:** in `server.js` the chat router is mounted before the training router. Both sit on `/api`. The first router serves a request to `/api/chat`. It never reaches the token middleware inside the second one. So two operations need a token and two do not.

**The decision:** one security definition applied to all four operations, not to two.

**The cost:** the connection screen asks for a token. It asks even for a flow that only sends questions.

**The benefit:** one connection serves every flow. And if the middleware later moves in front of `/api/chat`, no existing flow breaks.

Auth is an API key in the `Authorization` header. It has the exact shape Sanad reads: the word `Bearer`, a space, then the value of `SANAD_TOKEN`.

---

## Importing it

This definition is **ready to import**. I built it against an API that really runs. I did not publish it to a Microsoft 365 tenant, and nothing here claims otherwise.

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

Sanad speaks plain HTTP, and custom connectors need HTTPS. So a reverse proxy has to terminate TLS in front of it.

`apiProperties.json` declares the on-premises data gateway option through a `gatewaySetting` parameter. That parameter is what shows the choice on the connection screen. You install the gateway and share it with Admin permission outside this repository.

### Proof

```
$ python -m json.tool apiDefinition.swagger.json > /dev/null && echo ok
ok
$ python -c "import json; d=json.load(open('apiDefinition.swagger.json',encoding='utf-8')); print(len(d['paths']), len(d['definitions']))"
4 12
```

I also searched `routes/` and `lib/` in the Sanad repository. I looked for each of the 52 field names under `definitions`. I found all 52. Zero invented names.

---

## Why I built it

Sanad answers inside the network, and employees ask their questions in Teams.

Closing that gap took only a definition: 0 new lines of code in Sanad.

---

MIT — © 2026 Osaid Ahmad
