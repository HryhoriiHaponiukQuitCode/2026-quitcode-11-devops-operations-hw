<!--
Що це: README набору розробника WS11 (ops-kit) — що в наборі, куди кожен файл, як підключити, як перевірити.
Куди: нікуди не копіюється; читайте тут. Файли набору копіюйте за таблицею нижче.
Завдання README домашки: D0, A, B, C, D і бонус треку розробника.
Секрети (лише назви): TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID, VERCEL_AUTOMATION_BYPASS_SECRET; бонус — BACKUP_PASSPHRASE,
SUPABASE_DB_URL, ANTHROPIC_API_KEY або CLAUDE_CODE_OAUTH_TOKEN. SENTRY_AUTH_TOKEN — лише у Vercel чи CI, якщо обрали Sentry.
Джерело: набір ops-kit WS11; факти про сервіси — з їхньої документації, станом на 09.11.2026.
-->

# Набір розробника WS11: експлуатація

## Навіщо в роботі

Продукт уже на проді. Тепер треба, щоб про збій першою дізнавалася людина з команди, а не клієнт, і щоб
інцидент був процедурою, а не панікою. Цей набір — «аптечка» для цього: готові workflow, логер, профіль агента і
шаблони документів. Аптечка не лікує сама: ви кладете її у своє репо, підписуєте своїми назвами й перевіряєте,
що кожен засіб справді спрацьовує.

**Можна брати з посиланням на джерело.** В описі PR напишіть «взято з <посилання на файл набору>». Workflow,
логер, профіль і шаблони — інфраструктура, її брати можна. **Runbook і postmortem — про ваш продукт:** шаблон без
змін нічого не доводить і не зараховується.

## Короткий словник

| Термін | Що це | Побутова аналогія |
|---|---|---|
| **smoke** (димовий тест) | кілька GET-запитів до живого сайту: чи відкривається, чи є сторінка входу, чи відповідає база | увімкнути щойно полагоджений чайник і подивитися, чи не пішов дим |
| **деплой** (deployment) | одна зібрана версія застосунку на Vercel зі своєю адресою | один надрукований тираж книжки |
| **preview / staging** | не-прод деплої: preview — на кожну гілку чи PR, staging — постійна гілка `staging` для приймання | генеральна репетиція перед прем'єрою |
| **захист деплоїв** (Deployment Protection) | Vercel пускає на preview і staging лише після входу у Vercel | двері зі шлагбаумом |
| **обхід захисту** (Protection Bypass for Automation) | секрет, з яким скрипт проходить захист без входу; надсилається заголовком `x-vercel-protection-bypass` | перепустка для кур'єра |
| **алерт** | повідомлення людині, коли перевірка червона | пожежна сигналізація, що дзвонить вам на телефон |
| **cooldown** (пауза для нових версій) | Dependabot не пропонує версію пакета, молодшу за N днів | не їсти пиріг, поки не охолов: шкідливі версії зазвичай знаходять у перші дні |
| **закріплення за SHA** | `uses: дія@<40 символів коміту> # vX.Y.Z` замість `@v4` | номер конкретної партії ліків замість «щось від голови» |
| **структурований журнал** | один рядок JSON на подію з дозволеним переліком полів | бланк з графами замість вільного листа: зайве просто нікуди вписати |
| **MCP** (Model Context Protocol) | спосіб дати агенту інструменти чужого сервісу (Vercel, Supabase) | пульт від чужого телевізора: важливо, які кнопки на ньому працюють |
| **allow / ask / deny** | правила Claude Code: виконувати без питань / спитати вас / заборонити завжди | ключ від архіву, дзвінок охороні, замуровані двері |
| **runbook** | інструкція «що робити, коли…», написана заздалегідь | план евакуації на стіні |
| **postmortem без пошуку винних** (blameless) | розбір інциденту: що сталося, чому, що змінимо — без «хто винен» | розбір польоту в авіації: шукають причину, а не цапа-відбувайла |

## Що в наборі і куди

JSON не дозволяє коментарів, тому заголовки двох JSON-файлів — у розділі [нижче](#заголовки-json-файлів). Решта
файлів починається з коментаря: що це, куди, яке завдання, які секрети, джерело.

| Файл набору | Куди у вашому репо | Завдання | Секрети й змінні (лише назви) | Промпт |
|---|---|---|---|---|
| `.github/workflows/smoke.yml` | `.github/workflows/smoke.yml` (замінює файл з D0) | A, B, D | `VERCEL_AUTOMATION_BYPASS_SECRET`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`; змінна репо `PROD_URL` | [R04], [R05] |
| `scripts/smoke.mjs` | `scripts/smoke.mjs` (замінює файл з D0; команди й коди виходу ті самі) | A | `VERCEL_AUTOMATION_BYPASS_SECRET` (необов'язковий) | [R04] |
| `.github/dependabot.yml` | `.github/dependabot.yml` | D | — | [R02] |
| `npmrc.add` | **дописати** рядки в кінець `.npmrc` у корені (немає — створіть) | D | — | [R02] |
| `.github/release.yml` | `.github/release.yml` | A | — | [R13] |
| `src/lib/log.ts` | `src/lib/log.ts` | B | — | [R07] |
| `src/lib/ai/llm-log.ts` | `src/lib/ai/llm-log.ts` (поруч з AI-функцією WS10) | B | — | [R07] |
| `tests/llm-log.test.ts` | `tests/llm-log.test.ts` | B | — | [R07] |
| `src/instrumentation.ts.example` | `src/instrumentation.ts` — **лише без Sentry** | B | `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` як змінні **Vercel**; `APP_ENV` (Vercel, лише гілка `staging`) | [R06] |
| `.claude/settings.incident.json` | правила — у `.claude/settings.json`; файл цілком — `.claude/settings.incident.json` (див. [профіль](#профіль-лише-пропонує-c)) | C | — | [R08] |
| `mcp.json.example` | сервери — у `.mcp.json`; або файл цілком — `.mcp.incident.json` | C | — (вхід через OAuth); `SUPABASE_PROJECT_REF` — не секрет | [R08] |
| `docs/runbook.template.md` | `docs/runbook.md` | C, бонус | — | [R11] |
| `docs/incidents/postmortem.template.md` | `docs/incidents/<YYYY-MM-DD>-<slug>.md` | C | — | [R10] |
| `docs/incidents/rca-transcript.template.md` | `docs/incidents/<YYYY-MM-DD>-<slug>-rca.md` | C | — | [R09] |
| `bonus/backup.yml` | `.github/workflows/backup.yml` | бонус | секрети середовища `production`: `BACKUP_PASSPHRASE`, `SUPABASE_DB_URL` | [R17] |
| `bonus/restore-drill.md` | не копіюється: інструкція; результат — розділ «Відновлення» в `docs/runbook.md` | бонус | ті самі | [R17] |
| `bonus/oncall-assistant.yml` | `.github/workflows/oncall-assistant.yml` | бонус | один з двох, ваш: `ANTHROPIC_API_KEY` або `CLAUDE_CODE_OAUTH_TOKEN` | [R18] |

Де живуть значення (назви розділів звірте в інтерфейсі): GitHub → Settings → Secrets and variables → Actions
(секрети й змінна `PROD_URL`); Vercel → Settings → Environment Variables (`APP_ENV`, `TELEGRAM_*` для
`instrumentation.ts`, `SENTRY_AUTH_TOKEN`). У файлах,
журналах прогонів, тексті алерту, на знімках і в чаті з агентом — **лише назви**.

### Заголовки JSON-файлів

**`.claude/settings.incident.json`**

- **Що це:** профіль Claude Code «лише пропонує» для розбору інциденту. `defaultMode: "plan"` (агент читає й
  пропонує план, файлів не редагує), `disableBypassPermissionsMode: "disable"` (режим «без питань» вимкнено).
  `allow` — читання (файли, `git log/show/diff/status`, `gh run/pr/issue view`, `vercel logs/inspect`, інструменти
  читання MCP Vercel і Supabase, **кожен повною назвою**). `ask` — `mcp__vercel__filter_project_envs` (уміє
  розшифровувати значення) і `mcp__supabase__execute_sql` (довільний SQL). `deny` — `Edit`, `Write`,
  `NotebookEdit`; читання `.env*` і `accounts.*`; команди `vercel`/`supabase`, що змінюють деплой, змінні чи базу;
  `git push`, `gh pr merge`, `gh workflow run`, `gh secret`, `gh variable`, `gh release`; усі інструменти запису й
  розкриття секретів MCP Vercel і Supabase **повними назвами**.
- **Куди:** див. [профіль](#профіль-лише-пропонує-c). **Завдання:** C. **Секрети:** не потребує.
- **Джерело:** документація Claude Code «Settings» і «Identity and Access Management» (правила, порядок
  deny → ask → allow, `--settings`, `--permission-mode`); переліки інструментів — код Supabase MCP і перелік
  інструментів Vercel MCP, станом на 09.11.2026. Набір інструментів серверів змінюється — звіряйте з `/mcp`.

**`mcp.json.example`**

- **Що це:** два сервери MCP без жодного токена. Vercel — `https://mcp.vercel.com`, вхід через OAuth у `/mcp`.
  Supabase — `https://mcp.supabase.com/mcp?project_ref=…&read_only=true&features=database,debugging,docs`:
  `project_ref` обмежує одним проєктом, `read_only=true` — запити лише на читання, `features` — лише групи
  інструментів бази, журналів і документації.
- **Куди:** сервери — у `.mcp.json` вашого репо (поруч з наявними), або файл цілком — `.mcp.incident.json` для
  окремого запуску. **Завдання:** C. **Секрети:** немає. `${SUPABASE_PROJECT_REF}` — змінна середовища; простіше
  вписати ref прямо: це не секрет (він є в публічній адресі `NEXT_PUBLIC_SUPABASE_URL`).
- **Джерело:** документація Supabase «MCP server» і Vercel «Vercel MCP», станом на 09.11.2026.

## Порядок: D → A → B → C → бонус

Чому так: D (ланцюг постачання) — дрібні файли, що одразу дають перший PR Dependabot з `next` ≥ 16.4.5 і закривають
D0. A дає smoke кожного деплою — без нього B не має чого алертити. B — сигнали, без них дріл у C нічого не
помітить. C — дріл, який перевіряє все попереднє разом. Бонус — після DoD.

Кожен крок — окрема гілка й PR з зеленим CI. Перевірки шаблону перед кожним PR:

```powershell
npm test; if ($?) { npm run typecheck }; if ($?) { npm run lint }
```

### D. Ланцюг постачання

**Навіщо:** найчастіший шлях шкідливого коду у ваш прод — не ваш код, а свіжа версія чужого пакета чи дії GitHub.

1. **`.github/dependabot.yml`** — копія. `npm` і `github-actions` щотижня, `cooldown` (7 днів, мажорні — 30),
   групи (мінорні й патчі `npm` — одним PR, усі дії — одним PR), для `npm` — не більше 5 відкритих PR. Безпекові оновлення
   Dependabot `cooldown` не чекають — це окремий перемикач: Settings → Code security → Dependabot security updates
   (назву звірте в інтерфейсі).
2. **`npmrc.add`** — **дописати** в `.npmrc`. Вимога до версії npm і що робити з терміновим оновленням — у
   коментарі файлу. Коротко: перевірте `npm -v`; на старому npm параметр мовчки не діє.
3. **SHA замість тегів** у ваших власних workflow: `uses: <дія>@<повний SHA> # v<версія>`. Готові SHA — у
   `smoke.yml` набору. SHA для іншої дії: `gh api repos/<власник>/<дія>/commits/<тег> --jq .sha`.
4. **Push protection і CodeQL default setup** — у налаштуваннях репо (Settings → Code security; назви звірте в
   інтерфейсі). Доказ — знімок налаштувань або перший прогін CodeQL.

**Як перевірити:**

- Insights → Dependency graph → Dependabot (назву звірте в інтерфейсі) показує обидві екосистеми без помилки; за кілька хвилин — перший PR.
- `npm install` локально не ставить версію, молодшу за 7 днів (на npm, що підтримує параметр).
- Знайти незакріплені дії — команда має нічого не надрукувати:

  ```bash
  grep -rn "uses:" .github/workflows | grep -vE "@[0-9a-f]{40} # v"
  ```

### A. Smoke кожного деплою й реліз

**Навіщо:** прод-smoke з D0 перевіряє лише `main`. Staging і preview теж треба перевіряти — інакше реліз-кандидат,
який приймає PM, до PM ніхто не перевіряє.

1. **`scripts/smoke.mjs`** — заміна D0-версії. Нове: якщо задано `VERCEL_AUTOMATION_BYPASS_SECRET`, кожен запит
   несе заголовок обходу; значення скрипт не друкує (у журналі лише `"bypass": true`).
2. **`.github/workflows/smoke.yml`** — заміна D0-версії. Job `prod` працює як раніше; новий job `deploy` бере
   адресу з події Vercel (`client_payload.url`) для кожного не-прод деплою. Обидва при збої шлють алерт (крок B).
3. **Vercel → Settings → Deployment Protection** (назви звірте в інтерфейсі):
   - Vercel Authentication — **Standard Protection**: preview і staging закриті, прод відкритий. Режим «усі деплої»
     закрив би й прод, а курс читає прод `/api/health` без входу.
   - **Protection Bypass for Automation** — створити секрет; значення — у GitHub Secrets як
     `VERCEL_AUTOMATION_BYPASS_SECRET`. Перегенерували секрет — оновіть його в GitHub і зробіть Redeploy (старі
     деплої можуть не прийняти новий секрет; перевірте в документації Vercel).
4. **Staging** (промпт [R03]): гілка `staging`, у Vercel — змінна `APP_ENV=staging` лише для гілки `staging`.
5. **`.github/release.yml`** — категорії нотаток релізу за мітками PR. Тег `v1.0.0` — після приймання PM.

**Важливо про тригери.** `repository_dispatch`, `status` і `schedule` GitHub запускає лише з файлу на гілці за
замовчуванням. До злиття перевіряйте вручну: Actions → smoke → Run workflow.

**Як перевірити:**

- Синтаксис і швидкий червоний прогін локально (код виходу 1 — правильно, адреса недоступна):

  ```powershell
  node --check scripts/smoke.mjs
  node scripts/smoke.mjs https://example.invalid; echo "exit=$LASTEXITCODE"
  ```

- Прод, як у D0: `node scripts/smoke.mjs <прод-URL> <SHA main>` → код 0.
- Після злиття: push у `staging` → Actions → прогін `smoke` з подією `repository_dispatch`, job `deploy`, зелений.
  У журналі — `"target": "deploy"` і `"bypass": true`, значення секрету немає.
- Без секрету job `deploy` пише попередження «Немає секрету VERCEL_AUTOMATION_BYPASS_SECRET». Якщо захист
  увімкнено — job ще й червоний (стіна входу Vercel замість `/api/health`), так і має бути. Якщо захист вимкнено —
  job зелений, але staging відкритий усім: це варто записати в runbook.
- `release.yml`: Releases → Draft a new release → оберіть тег → «Generate release notes» (назву кнопки звірте в
  інтерфейсі) — PR розкладено за категоріями з файлу, PR з міткою `skip-release-notes` немає.

### B. Сигнали

**Навіщо:** червоний прогін, якого ніхто не бачить, нічим не кращий за відсутній. Помилку сервера без трекера
помічає клієнт, а не ви. А журнал AI-функції без структури або мовчить, або пише текст користувача.

**Алерт у Telegram (крок у `smoke.yml`).**

1. Створіть бота через @BotFather (назву команди звірте в самому BotFather) → токен у GitHub Secrets як
   `TELEGRAM_BOT_TOKEN`.
2. Додайте бота в чат команди, де є люди; id чату дізнайтеся методом `getUpdates` з документації Telegram Bot API →
   у GitHub Secrets як `TELEGRAM_CHAT_ID`. Значення ніде не записуйте.
3. Крок алерту `if: failure()` передає значення лише через `env:`, без `set -x`. Немає секретів — крок пропускається
   з попередженням, smoke не ламається.

**Навмисно червоний прогін** (доказ для B): Actions → smoke → Run workflow → поле `url` = `https://example.invalid`.
Job `deploy` падає, у чаті — «ЧЕРВОНИЙ SMOKE» з репо, job, середовищем, хостом і посиланням на прогін. Знімок
повідомлення — з замазаним chat id.

**Трекер помилок сервера — одне з двох:**

- **Sentry:** майстер `npx @sentry/wizard@latest -i nextjs`; `SENTRY_AUTH_TOKEN` — лише у Vercel чи CI. Збір
  вхідних і вихідних даних моделі вимкніть (README, B).
- **Без Sentry:** `src/instrumentation.ts.example` → `src/instrumentation.ts`. `onRequestError` шле в той самий чат
  маршрут, метод, назву помилки, digest, середовище, деплой і до 5 верхніх рядків стеку (лише рядки `at …`: функція,
  файл, рядок коду). Ніколи — заголовки, тіло, рядок запиту чи текст помилки (`error.message`). Змінні `TELEGRAM_BOT_TOKEN` і `TELEGRAM_CHAT_ID` — у **Vercel** (Production і Preview), бо код працює на
  Vercel, а не в GitHub Actions.

**Як перевірити трекер:** тимчасовий маршрут, що кидає помилку, на preview чи staging → повідомлення в чаті з
маршрутом і верхніми рядками стеку → повний стек, якщо треба, — у Vercel Runtime Logs (на Hobby журнал живе
годину — дивіться одразу) → маршрут приберіть або закрийте роллю адміністратора.

**Журнал `llm.call`.**

- `src/lib/log.ts` — загальний логер: пише лише поля з дозволеного переліку (схеми), кожне за правилом (короткий
  токен, шлях, ціле число, `true/false`, значення з переліку). Усе інше відкидає. Рядки довші за 120 символів
  обрізає.
- `src/lib/ai/llm-log.ts` — схема події `llm.call`: `event`, `ts`, `requestId`, `route`, `model`, `fallbackUsed`,
  `state` (`ok | needs_info | refused | unavailable`), `latencyMs`, `inputTokens`, `outputTokens`, `costUsd`
  (необов'язкове), `promptVersion`, `errorKind`. Поля для тексту користувача, промпту чи відповіді **немає**.
- `tests/llm-log.test.ts` — 9 тестів, зокрема: зайві поля (`text`, `prompt`, `email`, `headers`…) у рядок не
  потрапляють; текст користувача, підсунутий у дозволене поле, відкидається.

#### llm.call у коді AI-функції WS10

Якщо ваша функція побудована на наборі WS10 (`proposeDraft` у `src/lib/ai/draft.ts`), зміни такі. Перевірено на
копії шаблону з набором WS10: тести, typecheck і lint зелені.

`src/lib/ai/draft.ts`:

```ts
import { isFallback, logLlmCall } from "./llm-log";
import { DRAFT_INSTRUCTIONS, DRAFT_PROMPT_VERSION, wrapDocument } from "./prompts/draft";

export type DraftOptions = {
  // …наявні поля…
  /** WS11: куди писати llm.call. Немає — журналу немає (так працюють golden-тести з WS10). */
  log?: { requestId: string; route: string; primaryModel: string };
};

// у proposeDraft, перед try:
const started = performance.now();

// після успішного generateText, перед return:
if (opts.log) {
  const cost = (result.providerMetadata?.openrouter as { usage?: { cost?: unknown } } | undefined)?.usage?.cost;
  logLlmCall({
    requestId: opts.log.requestId,
    route: opts.log.route,
    model: result.response.modelId,
    fallbackUsed: isFallback(result.response.modelId, opts.log.primaryModel),
    state: result.output.status, // ok | needs_info | refused
    latencyMs: performance.now() - started,
    inputTokens: result.usage.inputTokens,
    outputTokens: result.usage.outputTokens,
    costUsd: typeof cost === "number" ? cost : undefined,
    promptVersion: DRAFT_PROMPT_VERSION,
  });
}

// у catch:
const failure = classifyError(error);
if (opts.log) {
  logLlmCall({
    requestId: opts.log.requestId,
    route: opts.log.route,
    model: opts.log.primaryModel,
    fallbackUsed: false,
    state: "unavailable",
    latencyMs: performance.now() - started,
    promptVersion: DRAFT_PROMPT_VERSION,
    errorKind: failure.reason, // КОД причини; failure.message (текст постачальника) у журнал не йде
  });
}
return failure;
```

`src/app/ai-draft/actions.ts` (чи де ви викликаєте `proposeDraft`):

```ts
import { PRODUCT_MODELS } from "@/lib/ai/models";

const result = await proposeDraft(input, model, {
  log: { requestId: crypto.randomUUID(), route: "/ai-draft", primaryModel: PRODUCT_MODELS[0] },
});
```

Чому `isFallback`: OpenRouter відповідає назвою моделі, яка справді відповіла; якщо вона інша, ніж перша в
переліку, — спрацював запасний варіант (fallback). Суфікс `:free` при порівнянні ігнорується.

**Як перевірити журнал:** `npm test` — 9 тестів `llm-log` зелені. Потім один виклик AI-функції на preview → Vercel
Runtime Logs → рядок `{"event":"llm.call",…}` з токенами й станом, без тексту, який ви вводили.

### C. Інцидент-дріл

**Навіщо:** інцидент — процедура, а не паніка. Ви проходите її один раз на staging, коли нічого не горить, і
лишаєте в репо все, щоб її повторив будь-хто.

#### Профіль «лише пропонує» (C)

**Чому так суворо.** MCP Vercel має ті самі права, що ваш акаунт: відкат, Promote, деплой, зміна й читання
значень змінних. Документованого режиму «лише читання» в нього немає (станом на 09.11.2026). Claude Code
перевіряє правила в порядку **deny → ask → allow**: заборона сильніша за дозвіл. Тому інструменти запису забороняємо **кожен повною назвою**, а
читання дозволяємо теж повною назвою, не шаблоном: шаблон `mcp__vercel__get_*` пропустив би й
`get_project_env`, який повертає розшифровані значення. А `mcp__vercel__*` у deny заборонив би й читання журналів.

Рубрика дивиться на правила в `.claude/settings.json`. Зробіть так:

1. **Постійні правила — у `.claude/settings.json`** (вони не заважають щоденній роботі). Перенесіть з
   `settings.incident.json`: увесь `ask`; з `allow` — правила `mcp__…`; з `deny` — усі `mcp__…`, `Read(.env…)`,
   `Read(…accounts…)`, за бажанням `gh secret` і `gh variable`. Ваші наявні правила збережіть.
   **Не переносьте** в `.claude/settings.json`: `defaultMode: "plan"`; заборони `Edit`, `Write`, `NotebookEdit`,
   `git push`, `gh pr merge`, `gh workflow run`, `gh release`; усі заборони команд `vercel …` і `supabase …`. Вони
   зупинили б звичайну розробку: `supabase migration new`, `supabase db reset --local`, `vercel env pull`. Ці
   заборони діють лише в сесії розбору (крок 3).
2. **Файл цілком** — `.claude/settings.incident.json` у вашому репо (той самий шлях, що в наборі). Сервери MCP —
   у `.mcp.json` (додайте `vercel` і `supabase` з `mcp.json.example` поруч з наявними) або цілим файлом
   `.mcp.incident.json`.
3. **Запуск сесії розбору** з кореня репо:

   ```powershell
   claude --settings .claude/settings.incident.json --mcp-config .mcp.incident.json --strict-mcp-config --permission-mode plan
   ```

   `--settings` додає профіль поверх `.claude/settings.json` (заборони обох файлів діють разом).
   `--strict-mcp-config` — лише сервери з `.mcp.incident.json`, без інших. Якщо сервери у `.mcp.json`, приберіть
   `--mcp-config` і `--strict-mcp-config`. Далі в сесії — `/mcp` → вхід у Vercel (OAuth) і Supabase.
4. **Звірте назви.** Префікс `mcp__vercel__` — це ім'я сервера у вашому файлі MCP; назвали інакше — правила не
   спрацюють. Точні назви інструментів подивіться в `/mcp` і порівняйте з `deny`: набір інструментів сервера
   змінюється. Нового інструмента запису немає в `deny` — додайте.

**Про MCP детальніше.**

- **Vercel:** токенів у файлі немає, вхід — OAuth у `/mcp`. Щоб обмежити одним проєктом, адресу проєкту дає
  `vercel mcp --project` ([гайд інциденту](../../docs/incident-guide.md)).
- **Supabase** (лише якщо гіпотеза справді про базу): staging і прод ділять базу, тож це підключення до **прод-бази**.
  `read_only=true` — запити від користувача Postgres лише на читання; `features=database,debugging,docs` — без
  інструментів функцій, сховища й гілок. `execute_sql` — в `ask`: ви бачите запит до виконання.
- **`ask`** — `filter_project_envs` і `execute_sql`: Claude Code спитає вас. На `filter_project_envs` у дрілі відповідь
  «ні»: назви змінних агенту даєте ви (R09, крок 3). На `execute_sql` — «так», лише якщо бачите запит і розумієте навіщо.

**Як перевірити профіль** (у сесії з кроку 3):

- «Покажи значення змінної середовища `APP_ENV` на staging» → відмова або питання, а не значення.
- «Зроби відкат staging на попередній деплой» → відмова (`request_rollback` у `deny`).
- «Покажи останні деплої й помилки staging» → працює без питань.
- Відповіді збережіть у стенограму — це доказ для C.

**Шаблони документів.**

- `docs/incidents/rca-transcript.template.md` → `docs/incidents/<YYYY-MM-DD>-<slug>-rca.md`: заголовок (дата,
  модель, профіль, що агент міг і чого не міг) + дослівна стенограма через `/export`. Переказ не зараховується.
- `docs/incidents/postmortem.template.md` → `docs/incidents/<YYYY-MM-DD>-<slug>.md`: хронологія з часом, вплив
  мовою користувача, першопричина з доказом на кожне твердження, «що НЕ було причиною», дії з власником.
- `docs/runbook.template.md` → `docs/runbook.md`: червоний smoke, відкат і Promote (пастка Hobby), база на паузі,
  ротація ключа, журнали й скільки вони живуть, чого ніколи не робити.

**Як перевірити документи:** у них немає жодного рядка «Приклад:» чи `<…>` з шаблону; кожне твердження
postmortem має посилання на прогін, деплой, коміт чи рядок стенограми; runbook відповідає на «червоний smoke о
2-й ночі» без питань до автора.

### Бонус (+10) — одне з двох

**Резервна копія і тренування відновлення** — `bonus/backup.yml` + `bonus/restore-drill.md` (там покроково).
Безкоштовний Supabase резервних копій не дає. Workflow щоночі й вручну робить дамп закріпленою версією CLI через
**session pooler**, шифрує `gpg` (AES-256), кладе лише зашифрований файл артефактом на 7 днів і відновлює його в
локальну базу CI з порівнянням кількості рядків. SQL і дані в журнал не потрапляють. Секрети — у середовищі GitHub
`production`: `BACKUP_PASSPHRASE`, `SUPABASE_DB_URL`. Відновлення схеми `auth` не відрепетирувано: якщо тренування
падає на ній, запустіть зі `scope=public` (подробиці в `restore-drill.md`).

**Асистент чергового** — `bonus/oncall-assistant.yml`. **Необов'язковий і платний:** кожен червоний smoke = один
запуск агента (до 4 ходів) за ваш ключ — `ANTHROPIC_API_KEY` (оплата за використання) або
`CLAUDE_CODE_OAUTH_TOKEN` (підписка, лише особисте використання). Безкоштовного шляху немає. Фактичну вартість
візьміть з Console → Usage (назву розділу звірте в інтерфейсі) і запишіть у файл здачі. Агент лише читає журнал і код (`Read`, `Grep`, `Glob`); issue з
міткою «чернетка агента, не перевірено» створює окремий job без агента.

**Як перевірити бонус:** Actions → backup → Run workflow → обидва jobs зелені, у Summary — таблиця рядків
«дамп = відновлено», артефакт лише `*.gpg`. Для асистента — навмисно червоний smoke (як у B) → за кілька хвилин
issue з міткою «чернетка агента, не перевірено».

## Що в наборі не перевірено

Чесно, щоб ви знали, де дивитися уважніше (станом на 09.11.2026):

- `backup.yml` цілком на справжньому проєкті Supabase не запускався; відновлення схеми `auth` не відрепетирувано.
- Чи враховує Dependabot `min-release-age` з `.npmrc` під час оновлення lockfile.
- Логін бота, який запускає smoke після деплою (`allowed_bots` в асистенті), — звірте у своєму журналі.
- Назви розділів інтерфейсів Vercel і GitHub — звірте в інтерфейсі.
- Назви інструментів Vercel MCP можуть відрізнятися на вашому підключенні — звірте з `/mcp`.

## Промпти

Бібліотека промптів: що дати агенту й що перевірити в результаті —
[R02] ланцюг постачання · [R03] staging · [R04] smoke кожного деплою · [R05] алерт · [R06] трекер помилок ·
[R07] журнал `llm.call` · [R08] профіль «лише пропонує» · [R09] розбір інциденту агентом · [R10] blameless
postmortem · [R11] runbook · [R12] дія з postmortem → перевірка в CI · [R13] реліз · [R17] бонус: копія бази й
відновлення · [R18] бонус: асистент чергового.

[R02]: https://github.com/koldovsky/2026-quitcode-11-devops-operations-hw/blob/main/docs/prompts.md#r02
[R03]: https://github.com/koldovsky/2026-quitcode-11-devops-operations-hw/blob/main/docs/prompts.md#r03
[R04]: https://github.com/koldovsky/2026-quitcode-11-devops-operations-hw/blob/main/docs/prompts.md#r04
[R05]: https://github.com/koldovsky/2026-quitcode-11-devops-operations-hw/blob/main/docs/prompts.md#r05
[R06]: https://github.com/koldovsky/2026-quitcode-11-devops-operations-hw/blob/main/docs/prompts.md#r06
[R07]: https://github.com/koldovsky/2026-quitcode-11-devops-operations-hw/blob/main/docs/prompts.md#r07
[R08]: https://github.com/koldovsky/2026-quitcode-11-devops-operations-hw/blob/main/docs/prompts.md#r08
[R09]: https://github.com/koldovsky/2026-quitcode-11-devops-operations-hw/blob/main/docs/prompts.md#r09
[R10]: https://github.com/koldovsky/2026-quitcode-11-devops-operations-hw/blob/main/docs/prompts.md#r10
[R11]: https://github.com/koldovsky/2026-quitcode-11-devops-operations-hw/blob/main/docs/prompts.md#r11
[R12]: https://github.com/koldovsky/2026-quitcode-11-devops-operations-hw/blob/main/docs/prompts.md#r12
[R13]: https://github.com/koldovsky/2026-quitcode-11-devops-operations-hw/blob/main/docs/prompts.md#r13
[R17]: https://github.com/koldovsky/2026-quitcode-11-devops-operations-hw/blob/main/docs/prompts.md#r17
[R18]: https://github.com/koldovsky/2026-quitcode-11-devops-operations-hw/blob/main/docs/prompts.md#r18
