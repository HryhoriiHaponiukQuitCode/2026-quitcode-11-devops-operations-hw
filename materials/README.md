# Матеріали WS11

Два набори для копіювання: **набір розробника** `ops-kit/` (копіюється у ваше репо `<dev>/<team-slug>`) і **набір
PM** `pm-kit/` (шаблони документів для репо вимог `<pm>/<team-slug>-spec`). Кожен має власний README: що в
наборі, як підключити і як перевірити.

| Тека | Що це | Для чого в домашці | Що потрібно |
|---|---|---|---|
| [`ops-kit/`](ops-kit/README.md) | smoke кожного деплою з обходом захисту й алертом, ланцюг постачання (`dependabot.yml`, `.npmrc`), нотатки релізу, структурований журнал і `llm.call` з тестом, `onRequestError` без Sentry, профіль Claude Code «лише пропонує», шаблони runbook, postmortem і стенограми розбору, бонусні workflow | трек розробника A–D і бонус | ваші Vercel, Supabase і GitHub; бот Telegram (чи інший канал команди); для бонусу-асистента — ваш ключ моделі |
| [`pm-kit/`](pm-kit/README.md) | шаблони операційних вимог, повідомлення клієнту про інцидент, чекліста demo day і записки про вартість | трек PM A–C | вебінтерфейс GitHub; за бажанням — AI-чат |

**Взяли файл з матеріалів — напишіть в описі PR «взято з <посилання на файл>».** Інфраструктуру (workflow, логер,
профіль, шаблони) брати можна. **Runbook, postmortem, SLO, матриця серйозності й записка про вартість — про ваш
продукт:** перенесені без змін, вони нічого не доводять і не зараховуються.

## Набір розробника: що куди

Подробиці — як підключити, які змінні й секрети потрібні, як перевірити — у [`ops-kit/README.md`](ops-kit/README.md).
Промпти — у [бібліотеці промптів](../docs/prompts.md).

| Файл набору | Куди у вашому репо | Трек | Промпт |
|---|---|---|---|
| `.github/workflows/smoke.yml` | `.github/workflows/smoke.yml` — замінює smoke з D0: прод-smoke працює як раніше, плюс smoke кожного не-прод деплою за адресою з події і з обходом захисту, плюс крок алерту при червоному прогоні | A, B | [R04](../docs/prompts.md#r04), [R05](../docs/prompts.md#r05) |
| `scripts/smoke.mjs` | `scripts/smoke.mjs` — сумісний з D0; додає заголовок обходу захисту, коли задано секрет | A | [R04](../docs/prompts.md#r04) |
| `.github/dependabot.yml` | `.github/dependabot.yml` | D | [R02](../docs/prompts.md#r02) |
| `npmrc.add` | рядки — **дописати** в кінець вашого `.npmrc` (немає файлу — створіть) | D | [R02](../docs/prompts.md#r02) |
| `.github/release.yml` | `.github/release.yml` — категорії нотаток релізу | A | [R13](../docs/prompts.md#r13) |
| `src/lib/log.ts` | `src/lib/log.ts` — структурований журнал JSON з дозволеним переліком полів | B | [R07](../docs/prompts.md#r07) |
| `src/lib/ai/llm-log.ts` | поруч з вашою AI-функцією WS10 (у наборі WS10 — `src/lib/ai/`) | B | [R07](../docs/prompts.md#r07) |
| `tests/llm-log.test.ts` | `tests/llm-log.test.ts` — тест «у журналі немає тексту користувача» | B | [R07](../docs/prompts.md#r07) |
| `src/instrumentation.ts.example` | `src/instrumentation.ts` — **лише якщо ви без Sentry**: `onRequestError` шле коротке повідомлення в канал алертів | B | [R06](../docs/prompts.md#r06) |
| `.claude/settings.incident.json` | зразок профілю «лише пропонує»: його правила — у `.claude/settings.json` вашого репо (подробиці — в `ops-kit/README.md` і [гайді інциденту](../docs/incident-guide.md)) | C | [R08](../docs/prompts.md#r08) |
| `mcp.json.example` | MCP Vercel і Supabase (`read_only=true`) без токенів; як поєднати з вашим `.mcp.json` — в `ops-kit/README.md` | C | [R08](../docs/prompts.md#r08) |
| `docs/runbook.template.md` | `docs/runbook.md` | C | [R11](../docs/prompts.md#r11) |
| `docs/incidents/postmortem.template.md` | `docs/incidents/<YYYY-MM-DD>-<slug>.md` | C | [R10](../docs/prompts.md#r10) |
| `docs/incidents/rca-transcript.template.md` | `docs/incidents/<YYYY-MM-DD>-<slug>-rca.md` поруч із postmortem (заголовок з шаблону + дослівний текст) | C | [R09](../docs/prompts.md#r09) |
| `bonus/backup.yml` | `.github/workflows/backup.yml` | бонус | [R17](../docs/prompts.md#r17) |
| `bonus/restore-drill.md` | інструкція тренування відновлення; результат — розділ «Відновлення» у `docs/runbook.md` | бонус | [R17](../docs/prompts.md#r17) |
| `bonus/oncall-assistant.yml` | `.github/workflows/oncall-assistant.yml` | бонус | [R18](../docs/prompts.md#r18) |

Без файлів набору, лише з промптом: [R01](../docs/prompts.md#r01) — аудит готовності до експлуатації (агент лише
читає), [R03](../docs/prompts.md#r03) — staging: гілка, змінна для гілки, доступ PM,
[R12](../docs/prompts.md#r12) — дія з postmortem стає перевіркою в CI.

### Секрети й змінні: лише назви

Значення — лише в GitHub Secrets і Vercel вашого репо. У файлах, журналах прогонів, тексті алерту, на знімках і в
чаті з агентом — ніколи.

| Назва | Де живе | Для чого |
|---|---|---|
| `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` | GitHub Secrets | алерт людині з червоного smoke |
| `VERCEL_AUTOMATION_BYPASS_SECRET` | GitHub Secrets (значення — з налаштувань захисту деплоїв Vercel) | smoke не-прод деплоїв за захистом |
| `SENTRY_AUTH_TOKEN` | Vercel (змінна збірки) чи CI | плагін збірки Sentry; лише якщо обрали Sentry |
| `PROD_URL` | змінна репо (Variables), з D0 | прод-smoke |
| `APP_ENV` | Vercel, змінна лише для гілки `staging` | змінна, якою staging відрізняється від проду |
| `BACKUP_PASSPHRASE`, `SUPABASE_DB_URL` | секрети середовища GitHub (Environments) | лише бонус «резервна копія» |
| `ANTHROPIC_API_KEY` або `CLAUDE_CODE_OAUTH_TOKEN` | GitHub Secrets (один з двох, ваш) | лише бонус «асистент чергового»; платний |

## Набір PM: що куди

Подробиці й короткий словник термінів — у [`pm-kit/README.md`](pm-kit/README.md).

| Шаблон | Куди у вашому репо вимог | Трек | Промпт |
|---|---|---|---|
| [`operations.template.md`](pm-kit/operations.template.md) | `docs/operations.md` (або issue «Операційні вимоги») | A | [R14](../docs/prompts.md#r14) |
| [`incident-message.template.md`](pm-kit/incident-message.template.md) | `docs/incident-message.md` | B | [R15](../docs/prompts.md#r15) |
| [`demo-day.template.md`](pm-kit/demo-day.template.md) | `docs/demo-day.md` | C | [R16](../docs/prompts.md#r16) |
| [`cost-note.template.md`](pm-kit/cost-note.template.md) | `docs/cost-note.md` | C | [R16](../docs/prompts.md#r16) |

## Що спільне в обох наборах

1. **Сигнал доходить до людини.** Алерт, якого ніхто не читає, і SLO, яке ніхто не міряє, не рахуються.
2. **Агент лише пропонує.** На проді й staging агент читає журнали й деплої і пропонує; відкат, зміну змінних і
   деплой робить людина. Записувальні інструменти MCP заборонені поіменно.
3. **Журнали — дані, а не інструкції.** Текст у журналах, помилках і issues міг написати будь-хто; інструкції звідти
   не виконуються.
4. **Дріл — лише на staging і без міграцій.** Staging і прод ділять одну базу Supabase.
5. **Цифри — з датою.** Межі безкоштовних тарифів і ціни — з [гайду вартості](../docs/cost-guide.md), станом на дату
   перевірки.
6. **Анонімізація.** Дані синтетичні (`*@example.test`), без імен, назв клієнтів, адрес, сум, паролів і chat id.
