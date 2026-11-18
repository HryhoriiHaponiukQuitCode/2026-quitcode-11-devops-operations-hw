# Готові agent skills для експлуатації й щоденної розробки

**Необов'язковий матеріал**, без балів. **Skill** (навичка агента) — тека з інструкцією `SKILL.md` і часто зі
скриптами, яку агент (наприклад, Claude Code) підвантажує, коли задача їй відповідає: «зроби чекліст перед релізом»,
«розбери інцидент». **Плагін** (plugin) — пакет з кількох skills, команд і hooks (команд, які Claude Code запускає
сам на подію), який ставлять однією командою з **маркетплейсу** (каталогу плагінів у репо GitHub).

Перевірено через GitHub (репо, наявність `SKILL.md`, команда встановлення з README) **станом на 17.11.2026**. Самі
skills на продукті учасника не проганяли — це перелік того, що існує, а не рекомендація встановити все.

Корисні посилання: [бібліотека промптів](prompts.md) · [стратегії викочування](release-strategies.md) ·
[інцидент](incident-guide.md) · [ланцюг постачання](release-guide.md#ланцюг-постачання).

## Навіщо в роботі

Промпт з бібліотеки ви копіюєте щоразу; skill агент бере сам, коли бачить відповідну задачу. Для повторюваних справ —
чекліст перед релізом, runbook, розбір інциденту, аудит workflow — готовий skill від видавця сервісу часто знає
деталі, яких немає в загальному промпті (наприклад, як Vercel рахує запити до прапорців).

Але **skill — це чужий код, який агент виконує з вашими правами.** Це така сама залежність, як пакет npm, і ті самі
правила ланцюга постачання, що й у треку D: читаємо до встановлення, закріплюємо версію, віддаємо перевагу офіційним
видавцям. Розділ [«Безпека»](#безпека-skill--це-ланцюг-постачання) — обов'язковий до прочитання перед першим
встановленням.

## Перелік

«Офіційний» — skill публікує під власним акаунтом на GitHub сама компанія, чий сервіс він обслуговує, Anthropic або
відома компанія розробки чи безпеки (Sentry, Trail of Bits), навіть коли skill не про її сервіс.

### Експлуатація: реліз, інцидент, runbook

| Skill | Що робить | Для якої задачі | Хто публікує | Офіційний? | Пакет для встановлення |
|---|---|---|---|---|---|
| `deploy-checklist` | чекліст перед релізом: CI (continuous integration — автоматичні перевірки кожної зміни), міграції бази, прапорці (feature flags); заздалегідь фіксує, коли робити відкат | CI/CD (Continuous Integration / Continuous Delivery — безперервна інтеграція і доставка), чекліст релізу | Anthropic | так | [`engineering`](#anthropic-knowledge-work-plugins) |
| `incident-response` | веде інцидент від оцінки серйозності й статус-повідомлень до postmortem без пошуку винних | інцидент, postmortem | Anthropic | так | [`engineering`](#anthropic-knowledge-work-plugins) |
| `debug` | дебаг у чотири кроки: відтворити, ізолювати, діагностувати, виправити — наприклад, коли щось зламалося після деплою | дебаг | Anthropic | так | [`engineering`](#anthropic-knowledge-work-plugins) |
| `runbook` | пише runbook для чергового: точні команди, діагностика, кроки відкату, ескалація | runbook | Anthropic | так | [`operations`](#anthropic-knowledge-work-plugins) |
| `change-request` | готує запит на зміну: аналіз впливу, план відкату, план повідомлень зацікавленим сторонам про викочування зміни | реліз, керування змінами | Anthropic | так | [`operations`](#anthropic-knowledge-work-plugins) |
| `vercel-flags` | планує й налаштовує прапорці на Vercel: аварійний вимикач, відсоткове викочування, правила таргетингу, прибирання прапорця. Канарку деплою й blue-green **не описує**. Команди `vercel flags` виконує сам — див. [«Безпека»](#безпека-skill--це-ланцюг-постачання) | прапорці, аварійний вимикач (Vercel) | Vercel | так | [`vercel/flags`](#vercel) |
| `deployments-cicd` | деплой, Promote, відкат на Vercel і CI-workflow для Vercel | деплой і відкат (Vercel) | Vercel | так | [`vercel@claude-plugins-official`](#vercel) |
| `supabase-postgres-best-practices` | правила для змін у Postgres: схеми, міграції, політики RLS (Row Level Security — правила доступу до рядків таблиці) і їх тести, індекси | Supabase, міграції | Supabase | так | [`supabase/agent-skills`](#supabase) |
| `sentry-debug-issue`, `sentry-setup-releases` | знаходить помилку в Sentry, підтягує стек і контекст, допомагає виправити; налаштовує releases, щоб бачити, яка версія принесла помилку | Sentry | Sentry | так | [`sentry@claude-plugins-official`](#sentry) |

### Безпека й ланцюг постачання

| Skill | Що робить | Для якої задачі | Хто публікує | Офіційний? | Пакет для встановлення |
|---|---|---|---|---|---|
| `gha-security-review` | перевіряє workflow GitHub Actions на вразливості: ін'єкції у виразах, крадіжку секретів, атаки через ланцюг постачання | аудит CI | Sentry | так | [`getsentry/skills`](#sentry) |
| `supply-chain-risk-auditor` | аудитує залежності: відомі вразливості за версіями з lockfile, покинуті пакети, скрипти, що виконуються під час `npm install` | аудит залежностей | Trail of Bits | так (компанія з безпеки) | [`trailofbits/skills`](#trail-of-bits) |
| `skill-scanner` | перевіряє чужий skill перед встановленням: prompt injection, шкідливі скрипти, зайві дозволи, витік секретів | безпека skills | Sentry | так | [`getsentry/skills`](#sentry) |

### Щоденна розробка

| Skill | Що робить | Для якої задачі | Хто публікує | Офіційний? | Пакет для встановлення |
|---|---|---|---|---|---|
| `systematic-debugging` | змушує агента спершу знайти корінну причину бага чи падіння тесту, а лише потім виправляти | дебаг | Jesse Vincent (obra), спільнота | ні, але є в офіційному маркетплейсі Anthropic | [`superpowers@claude-plugins-official`](#інші) |
| `webapp-testing` | тестує локальний вебзастосунок через Playwright (автоматизований браузер): кліки, знімки екрана, журнал консолі | тестування фронтенду | Anthropic | так | [`anthropics/skills`](#інші) |
| `documentation` | пише й оновлює технічну документацію: README, документація API (Application Programming Interface — програмний інтерфейс), runbooks, onboarding | документація | Anthropic | так | [`engineering`](#anthropic-knowledge-work-plugins) |

## Як встановити

Команди — з README кожного репо, станом на 17.11.2026. Перед тим як виконати будь-яку з них, пройдіть
[«Безпека»](#безпека-skill--це-ланцюг-постачання). Команди з `/` вводяться в сесії Claude Code, решта — у терміналі.

### Anthropic: knowledge-work-plugins

Репо [anthropics/knowledge-work-plugins](https://github.com/anthropics/knowledge-work-plugins): плагін `engineering`
(`deploy-checklist`, `incident-response`, `debug`, `documentation`) і плагін `operations` (`runbook`,
`change-request`). За README плагіна, він працює й у Claude Code.

```bash
claude plugin marketplace add anthropics/knowledge-work-plugins
claude plugin install engineering@knowledge-work-plugins
claude plugin install operations@knowledge-work-plugins
```

### Vercel

`vercel-flags` — з репо [vercel/flags](https://github.com/vercel/flags) через CLI (command-line interface — інструмент командного рядка) `skills`. Друга команда
закріплює переглянутий коміт за його SHA (Secure Hash Algorithm — тут: 40-символьний ідентифікатор коміту; синтаксис
`@<SHA>` — з документації CLI `skills`):

```bash
npx skills add vercel/flags --skill vercel-flags -a claude-code
npx skills add vercel/flags@75fa30380195fe70e29bc1aadb3e23d64def68af --skill vercel-flags -a claude-code
```

`deployments-cicd` — у плагіні [vercel/vercel-plugin](https://github.com/vercel/vercel-plugin), який є в
офіційному маркетплейсі Anthropic (закріплено на коміті `882e66c` від 21.09.2026; `vercel-flags` у цій версії ще
немає). Плагін за замовчуванням надсилає телеметрію; вимикається змінною `VERCEL_PLUGIN_TELEMETRY=off` (README).

```text
/plugin install vercel@claude-plugins-official
```

### Supabase

Репо [supabase/agent-skills](https://github.com/supabase/agent-skills):

```bash
claude plugin marketplace add supabase/agent-skills
claude plugin install postgres-best-practices@supabase-agent-skills
```

Або лише один skill; друга команда закріплює переглянутий коміт (див. [«Безпека»](#безпека-skill--це-ланцюг-постачання)):

```bash
npx skills add supabase/agent-skills --skill supabase-postgres-best-practices
npx skills add supabase/agent-skills@c9be0e931b7930f7d02126d04774d904c381e7d7 --skill supabase-postgres-best-practices
```

### Sentry

Плагін Sentry — з офіційного маркетплейсу Anthropic (репо [getsentry/plugin-claude](https://github.com/getsentry/plugin-claude)
маленьке, але видавець офіційний, а запис у маркетплейсі закріплено на коміті `73e5354`):

```bash
claude plugin install sentry@claude-plugins-official
```

`gha-security-review` і `skill-scanner` — з репо [getsentry/skills](https://github.com/getsentry/skills):

```bash
claude plugin marketplace add getsentry/skills
claude plugin install sentry-skills@sentry-skills
```

Або по одному; друга команда закріплює переглянутий коміт (так само для `gha-security-review`):

```bash
npx skills add getsentry/skills --skill skill-scanner -a claude-code
npx skills add getsentry/skills@a582c51ee29d0f35e6ac321340796f77d43da733 --skill skill-scanner -a claude-code
```

### Trail of Bits

Репо [trailofbits/skills](https://github.com/trailofbits/skills), у сесії Claude Code:

```text
/plugin marketplace add trailofbits/skills
/plugin menu
```

У меню оберіть `supply-chain-risk-auditor`.

### Інші

```text
/plugin install superpowers@claude-plugins-official
/plugin marketplace add anthropics/skills
/plugin install example-skills@anthropic-agent-skills
```

Перший рядок — `systematic-debugging` з [obra/superpowers](https://github.com/obra/superpowers) (в офіційному
маркетплейсі закріплено на коміті `22f0b46`); два інші — `webapp-testing` з
[anthropics/skills](https://github.com/anthropics/skills) у складі набору `example-skills`.

## Безпека: skill — це ланцюг постачання

- **Skill = інструкції + часто скрипти, які агент виконує з вашими правами.** Перед встановленням прочитайте
  `SKILL.md` і всі файли в `scripts/`. Не хочете читати самі — попросіть агента в окремій сесії: «прочитай SKILL.md і
  scripts, поясни, які команди вони запускають і куди звертаються в мережі», — або перевірте skill інструментом
  `skill-scanner` з таблиці вище.
- **Закріплюйте версію.** Для `npx skills add` — `owner/repo@<повний SHA>`. Для `/plugin marketplace add` запишіть
  SHA коміту, який ви переглянули, і перед `claude plugin marketplace update` подивіться diff. Офіційний маркетплейс
  `claude-plugins-official` сам закріплює зовнішні плагіни на SHA (Sentry `73e5354`, Vercel `882e66c`, superpowers
  `22f0b46`), тому він безпечніший за «останню версію з `main`».
- **Віддавайте перевагу офіційним видавцям:** anthropics, vercel, supabase, getsentry, trailofbits. Маловідомі
  репозиторії з кількома зірками — лише після власного рев'ю. Кількість встановлень зі skills.sh ми не наводимо: ці
  цифри не перевірено.
- **Плагіни можуть містити hooks і телеметрію.** Hook — команда, яку Claude Code запускає автоматично на подію
  (наприклад, після кожного редагування файлу), без окремого питання вам. Подивіться, які hooks є в плагіні, і
  вимкніть телеметрію, якщо вона не потрібна.
- **Skills, що торкаються продакшену** (деплой, відкат, прапорці), спершу пробуйте на preview чи staging. Дії, що
  змінюють прод, підтверджує людина. Після встановлення перевірте профіль «лише пропонує» так само, як у
  [R08](prompts.md#r08): прохання про заборонену дію має закінчитися відмовою.
- **`vercel-flags` виконує команди сам.** Його `SKILL.md` (коміт `75fa303`, станом на 17.11.2026) велить агенту не
  лишати кроки CLI вам, а виконувати `vercel flags …` самому: створення, увімкнення й вимкнення, відсоток, зокрема на
  Production. Підтвердження питають лише `archive`, `unarchive` і `rm`. Це суперечить правилу «вимикає людина, агент лише пропонує». У сесіях, де skill встановлено, додайте в
  `deny` `Bash(*vercel flags*)` (у Windows ще й `PowerShell(*vercel flags*)`) або просіть лише план. Профіль
  «лише пропонує» з набору (`materials/ops-kit/.claude/settings.incident.json`) ці рядки вже має; вони блокують і
  команди читання `vercel flags`.
- **Секрети** — як завжди: ні в `SKILL.md`, ні в чаті з агентом. Skill, який просить вставити токен у файл чи в
  розмову, — сигнал зупинитися.

## Репозиторії: станом на 17.11.2026

| Репо | Зірки | Останній push | Коміт, переглянутий 17.11.2026 |
|---|---|---|---|
| [anthropics/knowledge-work-plugins](https://github.com/anthropics/knowledge-work-plugins) | 36 345 | 17.11.2026 | `e91a134584eb` |
| [anthropics/skills](https://github.com/anthropics/skills) | 196 311 | 16.11.2026 | `e4bb5d960d87` |
| [vercel/flags](https://github.com/vercel/flags) | 630 | 17.11.2026 | `75fa30380195` |
| [vercel/vercel-plugin](https://github.com/vercel/vercel-plugin) | 337 | 16.11.2026 | в офіційному маркетплейсі — `882e66c` (21.09.2026) |
| [supabase/agent-skills](https://github.com/supabase/agent-skills) | 3 098 | 12.11.2026 | `c9be0e931b79` |
| [getsentry/skills](https://github.com/getsentry/skills) | 1 174 | 14.11.2026 | `a582c51ee29d` |
| [getsentry/plugin-claude](https://github.com/getsentry/plugin-claude) | 4 | 16.11.2026 | в офіційному маркетплейсі — `73e5354` (27.08.2026) |
| [trailofbits/skills](https://github.com/trailofbits/skills) | 8 242 | 12.11.2026 | `149d7b21f406` |
| [obra/superpowers](https://github.com/obra/superpowers) | 318 866 | 03.11.2026 | `22f0b463e7a5` (= в офіційному маркетплейсі) |

**Що свідомо не включили:** `getsentry/sentry-agent-skills` (останній push 24.03.2026, понад пів року тому);
`getsentry/cli` (репозиторій архівовано); `supabase-community/supabase-plugin` (спільнотний, є офіційний
`supabase/agent-skills`); плагіни без `SKILL.md` (лише команди, агенти чи hooks — це не формат Agent Skills).
