# sanad-power-connector — سَنَد داخل Power Automate

[English](README.en.md)

**الموظف يسأل في Teams — ويجيه رد سَنَد المعتمد، بدون ولا سطر كود جديد.**

Custom Connector لـ Power Platform، ملفين بس: 4 عمليات، و 12 تعريف بيانات، و 52 اسم حقل. كل اسم منها منسوخ حرفياً من كود [سَنَد](https://github.com/defeat11/sanad)، مو تخمين.

سَنَد بوت دعم فني ما يخترع إجابات: يرد من اللي تدرّب عليه، أو يقول «ما أعرف». يشتغل على المنفذ 4600، ويستمع على `127.0.0.1` بس. هذا المستودع هو الوصلة: ملف OpenAPI 2.0 وملف خصائص الـ connector، يستوردهم Power Platform على طول.

---

## المشكلة

سَنَد يشتغل جوّا الشبكة وما يطلع للإنترنت. قرار أمني مقصود. الموظف طول يومه في Teams، وما راح يفتح صفحة ويب ثانية عشان يسأل عن كلمة المرور.

الحل المعتاد إنك تكتب بوت Teams، وتسجّله، وتنشره، وتصونه. شغل كثير عشان تمرّر نص.

Power Automate يوصل لـ Teams أصلاً، ويوصل للشبكة الداخلية عبر الـ data gateway. الناقص قطعة وحدة: تعريف يشرح لـ Power Platform كيف يكلّم سَنَد.

---

## كيف يشتغل

الـ connector يكشف أربع عمليات، كلها مسارات موجودة فعلاً في سَنَد:

| العملية | الطلب | مصدرها في سَنَد | ماذا ترجع |
|---|---|---|---|
| Ask Sanad | `POST /api/chat` | `routes/chat.js:38` | الرد، والإجابة اللي طابقت، ودرجتها |
| Rate an answer | `POST /api/feedback` | `routes/chat.js:59` | تأكيد إن التصويت انسجّل |
| List pending questions | `GET /api/queue` | `routes/train.js:261` | الأسئلة اللي تنتظر المدرّب |
| Get statistics | `GET /api/stats` | `routes/train.js:621` | عدّادات قاعدة الإجابات |

`Ask Sanad` هي الأساس. ترسل `session` و `text`، ويجيك خمسة حقول: `reply` و `matched` و `suggestions` و `queued` و `chat_id`.

الحقول الثلاثة الأخيرة تعطيك قرار سَنَد نفسه، مو تفسير له:

| ما يرجع | معناه | مصدره |
|---|---|---|
| `matched.score` من 0.85 فأعلى | إجابة معتمدة، يرد فيها على طول | `T_AUTO` في `lib/engine.js` |
| `suggestions` ليست فارغة | المطابقة متوسطة، فيعرض ثلاثة عناوين | `T_BRAIN` في `lib/engine.js` |
| `queued` تساوي `true` | سَنَد ما يعرف، والسؤال راح لطابور المدرّب | `lib/engine.js` |

سَنَد يحدّ كل `session` بـ 20 سؤالاً في الدقيقة. عشان كذا الـ flow يمرّر معرّف مستخدم Teams في هذا الحقل، فيصير الحدّ لكل شخص لحاله.

لو تجاوزت الحدّ، سَنَد يرجّع رمز 429 ومعه حقل `reply` بس، بدون حقل `error`. الـ connector يوثّق هذا الشكل المختلف لأنه الشكل الحقيقي.

مخطط الـ flow كامل من Teams لين الرد: [`flows/ask-sanad-from-teams.md`](flows/ask-sanad-from-teams.md).

---

## أهم قرار في التصميم

**المشكلة:** في `server.js` الـ chat router مركّب قبل الـ train router، وكلاهما على `/api`. فطلب `/api/chat` يوقف عند الأول وما يوصل للـ auth middleware اللي في الثاني. النتيجة: عمليتان بدون token، وعمليتان بـ token.

**القرار:** تعريف أمان واحد ينطبق على العمليات الأربع كلها، مو على اثنتين بس.

**التكلفة:** شاشة الاتصال تطلب token حتى من flow ما يسوي غير سؤال.

**العائد:** اتصال واحد يكفي كل الـ flows. ولو انتقل الـ middleware قدّام `/api/chat` بعدين، ما ينكسر أي flow شغّال.

التوثيق عبارة عن API key في هيدر `Authorization`، بالشكل اللي يقرأه سَنَد فعلاً: كلمة `Bearer`، بعدها مسافة، بعدها قيمة `SANAD_TOKEN`.

---

## الاستيراد

هذا تعريف **جاهز للاستيراد**، مبني على API شغّال فعلاً. ما نُشر في tenant لـ Microsoft 365، وما فيه ادعاء غير كذا.

من الواجهة:

1. Power Automate ← Custom connectors ← New ← Import an OpenAPI file.
2. اختر `apiDefinition.swagger.json`.
3. غيّر `host` من `sanad.contoso.local` لاسم مضيف سَنَد عندك.
4. في تبويب الأمان اختر API Key، والهيدر `Authorization`.
5. أنشئ الاتصال، وألصق `Bearer` بعدها مسافة بعدها الـ token.

أو من سطر الأوامر:

```bash
paconn create --api-def apiDefinition.swagger.json --api-prop apiProperties.json
```

سَنَد يستمع HTTP بس، والـ custom connectors تبي HTTPS. عشان كذا تحتاج reverse proxy ينهي TLS قدّامه.

ملف `apiProperties.json` يعلن خيار الـ on-premises gateway عبر معامل نوعه `gatewaySetting`، فيطلع الخيار في شاشة الاتصال. الـ gateway نفسها تتثبّت وتتشارك بصلاحية Admin خارج هذا المستودع.

### إثبات

```
$ python -m json.tool apiDefinition.swagger.json > /dev/null && echo ok
ok
$ python -c "import json; d=json.load(open('apiDefinition.swagger.json',encoding='utf-8')); print(len(d['paths']), len(d['definitions']))"
4 12
```

الـ 52 اسم حقل اللي في `definitions` دوّرت عليها وحدة وحدة في `routes/` و `lib/` بمستودع سَنَد. كلها موجودة. صفر اسم مخترع.

---

## لماذا بنيته

سَنَد يجاوب جوّا الشبكة، والموظفين يسألون في Teams.

سدّ الفجوة احتاج تعريف بس: 0 سطر كود جديد في سَنَد.

---

MIT — © 2026 Osaid Ahmad
