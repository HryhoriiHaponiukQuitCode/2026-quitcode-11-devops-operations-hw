# Середовища, реліз і ланцюг постачання: як зміна безпечно доходить до проду

**Середовище** (environment) — окрема копія застосунку зі своєю адресою і своїми налаштуваннями, де зміну можна
подивитися до того, як її побачать користувачі. **Реліз** (release) — домовленість «ось ця версія готова», з номером і
списком змін. **Ланцюг постачання** (supply chain) — усе чуже, з чого складається ваш продукт: пакети npm, дії GitHub
Actions, інструменти в CI. Цей гайд — про те, як на безкоштовних Vercel Hobby, Supabase Free і GitHub Free провести
зміну через перевірене середовище, поставити тег `v1.0.0` і не впустити в репо шкідливий пакет.

Корисні посилання: [README, трек розробника](../README.md#трек-розробника--100--10-балів) ·
[покроково](walkthrough-dev.md) · [промпти R01–R04, R13](prompts.md#r01) ·
[сигнали й алерт](signals-guide.md) · [інцидент і runbook](incident-guide.md) ·
[ліміти й вартість](cost-guide.md) · [набір `ops-kit`](../materials/ops-kit/README.md).

Факти про тарифи й інтерфейси — **станом на 09.11.2026**, з посиланням на джерело. Назви кнопок у Vercel і GitHub
змінюються частіше, ніж можливості: якщо пункту меню не знайшли — шукайте за змістом і перевірте в інтерфейсі.

## Чотири середовища

Аналогія — театр. **Local** — репетиція вдома перед дзеркалом. **Preview** — прогін однієї сцени: кожен PR отримує
власну адресу. **Staging** — генеральна репетиція в костюмах на справжній сцені: увесь реліз-кандидат цілком, і його
приймає PM. **Production** — прем'єра з глядачами.

| Середовище | Звідки береться | Адреса | Хто дивиться | База Supabase |
|---|---|---|---|---|
| **Local** | `npm run dev` на вашому комп'ютері | `http://localhost:3000` | ви | ваш проєкт (як у `.env.local`) |
| **Preview** | кожен push у гілку чи PR | своя на кожен деплой | ви, CodeRabbit, PM за посиланням | **та сама, що на проді** |
| **Staging** | гілка `staging` | стабільна адреса гілки | PM приймає реліз-кандидат | **та сама, що на проді** |
| **Production** | гілка `main` | прод-URL з першого рядка README | користувачі, клієнт, demo day | ваш проєкт |

Чим staging відрізняється від preview: адреса **не змінюється** від деплою до деплою (PM зберігає її в закладках),
у staging є **власні змінні середовища**, і туди потрапляє лише те, що вже злито в `main`. Preview — чернетка одного
PR; staging — уся наступна версія продукту перед виходом.

## Staging на Vercel Hobby

Vercel має окрему можливість «Custom Environments» (іменоване середовище `staging` поруч із Preview і Production), але
вона **лише на Pro та Enterprise**. На Hobby Vercel офіційно описує інший шлях — «Preview branch for staging»,
доступний на всіх планах ([Vercel: Environments](https://vercel.com/docs/deployments/environments)):

**staging на Hobby = гілка `staging` + адреса цієї гілки + змінні середовища лише для цієї гілки.**

1. **Гілка.** Створіть `staging` від `main`: у вебінтерфейсі GitHub (Branches → New branch) або
   `git switch -c staging main` і `git push -u origin staging`. Розробки в `staging` немає: вона **отримує зміни з
   `main`** через PR `main → staging` чи merge. Інакше PM прийме не те, що піде на прод.
2. **Адреса гілки.** Кожна гілка вже має стабільну адресу виду `<проєкт>-git-<гілка>-<scope>.vercel.app`, яка завжди
   показує останній деплой гілки ([Vercel: Generated URLs](https://vercel.com/docs/deployments/generated-urls)).
   Скопіюйте справжню адресу з Vercel → Deployments (фільтр за гілкою `staging`): довгі назви обрізаються до 63
   символів, тож не збирайте її вручну. Власний домен необов'язковий. Якщо він у вас є, його можна прив'язати до гілки:
   Settings → Domains → Edit → Git Branch = `staging`
   ([Vercel: домен для гілки](https://vercel.com/docs/domains/working-with-domains/assign-domain-to-a-git-branch)).
   Новий домен спершу прив'язується до гілки за замовчуванням, тому його треба відредагувати.
3. **Змінна лише для гілки.** Settings → Environment Variables → нова змінна, середовище **Preview**, гілка
   **`staging`**. Наприклад, `APP_ENV=staging`. Така змінна перекриває однойменну змінну Preview лише для цієї гілки.
   З терміналу (Vercel CLI ≥ 64.7.0): `vercel env add APP_ENV --environment preview --git-branch staging`
   ([Vercel CLI: env](https://vercel.com/docs/cli/env)).
4. **Передеплой.** Змінні застосовуються **лише до нових деплоїв**
   ([Vercel: Environment Variables](https://vercel.com/docs/environment-variables)). Після зміни змінної зробіть
   новий push у `staging` чи Redeploy.
5. **Покажіть, що це staging.** На staging системна змінна `VERCEL_ENV` все одно дорівнює `preview`, тому D0
   `/api/health` сам staging не відрізнить. Додайте назву середовища з `APP_ENV` у відповідь `/api/health` (лише
   назву, це не секрет) чи в підвал сторінки. Рубрика A приймає обидва варіанти. Промпт —
   [R03 «Staging: гілка, змінна для гілки, доступ PM»](prompts.md#r03).

Що **не** зараховується як staging: просто preview з PR; змінна, однакова для всіх середовищ; окрема розробка в
`staging` повз `main`.

**Ключ OpenRouter на staging.** Якщо у WS10 ви додали `OPENROUTER_API_KEY` у Preview, staging його успадкує. Тоді
staging витрачає ті самі 50 безкоштовних запитів на добу, що й прод і ваш eval: додаткові ключі ліміт не збільшують.
Плануйте UAT PM на staging і живий eval на різні UTC-доби. Детальніше — у [гайді вартості](cost-guide.md).

## Спільна база: правила staging

Staging і прод ходять в **одну базу Supabase**. Так домовились у WS6: другий безкоштовний проєкт Supabase часто вже
зайнятий двійником WS9, а Branching (окрема база на гілку) — лише на Pro. Сам Supabase радить для staging окремий
проєкт ([Supabase: Managing environments](https://supabase.com/docs/guides/deployment/managing-environments)). Ми
свідомо обираємо компроміс: нуль гривень і нуль нових слотів, але **кожна дія на staging — це дія з бойовими
даними**. Звідси правила:

| Правило | Чому |
|---|---|
| **Лише тестові акаунти** з `accounts.zip`, жодних справжніх людей і даних | staging пише в ті самі таблиці, що й прод |
| Тестові записи — з помітною міткою, наприклад `[staging]` у назві | щоб їх легко знайти й прибрати й не показати клієнту випадково |
| **Жодних руйнівних дій:** видалення чужих записів, масових операцій, «очистити все» | відновити нічим: на Free немає резервних копій |
| **Міграції — лише ті, що додають** (нова таблиця, нова колонка, що може бути порожньою, новий індекс) | код на staging і код на проді мають однаково працювати з тією самою схемою |
| Міграції — лише через `main` і job `db-deploy`, ніколи вручну з `staging` | інакше staging змінить схему проду для всіх |
| **Інцидент-дріл — без міграцій:** ламаємо змінну середовища чи код | база, прод і чужі дані не страждають |
| Ніякого `supabase migration down` і `db reset --linked` | обидві команди знищують дані: `migration down` не «відкочує крок», а скидає схему й накочує міграції заново |
| Supabase CLI — закріплена в шаблоні версія (`npx supabase` з проєкту), не `npx supabase@latest` | CLI 3.0.0 (02.11.2026) за замовчуванням спрямовує команди бази в **локальну** базу ([Supabase CLI v3](https://supabase.com/changelog/supabase-cli-v3)) |
| Секретний ключ Supabase (`sb_secret_…`) — ніколи в змінних Preview | preview збирається з коду будь-якої гілки; RLS захищає дані, лише поки працює публічний ключ |

### Expand/contract: як змінити схему, не зламавши ні staging, ні прод

**Expand/contract** («розширити, потім звузити») — спосіб змінювати схему так, щоб і старий, і новий код працювали з
базою в кожен момент. Аналогія — ремонт мосту: спершу будують об'їзд, переводять на нього рух, і лише потім
розбирають старий проліт.

Приклад: перейменувати колонку `note` на `comment` у таблиці `requests`.

1. **Expand (PR 1, лише міграція).** Додати нову колонку і скопіювати дані. Старий код далі читає `note` і працює.

   ```sql
   alter table public.requests add column if not exists comment text;
   update public.requests set comment = note where comment is null;
   ```

2. **Код (PR 2, після того як `db-deploy` застосував PR 1).** Код читає й пише `comment`. Обидві колонки існують, тож
   працюють і preview, і staging, і прод, і відкат на попередній деплой.
3. **Contract (PR 3, щонайменше на один реліз пізніше).** Прибрати `note`. Це руйнівна міграція, тож **не цього
   тижня**: у WS11 міграції лише додають. Contract — після demo day, окремим PR, коли старий код давно не на проді.

Промпт для перевірки міграції — у [R01 «Аудит готовності до експлуатації»](prompts.md#r01): агент лише читає й
каже, чи сумісна міграція з кодом у `main` і чи її треба розбити.

## Спершу міграція, потім код

У шаблоні міграції застосовує лише job `db-deploy`: на `main`, після злиття, у середовищі GitHub `production`. Vercel
тим часом збирає той самий коміт `main` **паралельно**: нічого не гарантує, що схема оновиться раніше, ніж новий код
стане продом. На Hobby цей порядок не замкнений автоматично. Vercel має «Deployment Checks», що тримають прод-деплой
до зелених перевірок GitHub, але чи працюють вони на Hobby, ми не перевірили. Тому правило — у порядку PR:

- **Міграція, що додає, — окремим PR і першою.** Злили → `db-deploy` зелений (у Actions, крок «Apply migrations» —
  `success`, а не `skipped`) → лише тоді PR з кодом, який її використовує.
- **Кожна міграція сумісна з кодом, що зараз на проді.** Саме тому expand/contract.

**Чому відкат ніколи не повертає схему.** Vercel Instant Rollback перемикає домен на старий деплой і нічого не
перезбирає. До бази він не торкається: міграції, які застосував `db-deploy`, лишаються. Після відкату **старий код
працює з новою схемою**, і це безпечно, лише якщо міграції тільки додавали. Відкат схеми в Supabase — це не «undo», а
**нова міграція вперед** через звичайний PR
([Supabase: Database migrations](https://supabase.com/docs/guides/deployment/database-migrations)).

## Захист деплоїв і обхід для автоматики

**Deployment Protection** — охоронець на вході в підсобку: прод — вітрина, відкрита всім, а preview і staging Vercel
може закрити входом через акаунт Vercel (Vercel Authentication). Vercel вмикає цей захист для нових проєктів (чи
ввімкнений він саме у вас — перевірте в Settings → Deployment Protection), і режим **Standard Protection** закриває
все, **крім прод-доменів**
([Vercel: Deployment Protection](https://vercel.com/docs/deployment-protection)). У WS6 `bootstrap.md` радив
вимкнути «Require Log In», тож у багатьох із вас preview уже відкриті всім. Обидва стани робочі; головне — знати, у
якому ви, і записати це в runbook.

**Обхід для автоматики** (Protection Bypass for Automation) — перепустка для робота: секрет, з яким smoke у GitHub
Actions проходить повз охоронця. Доступний на всіх планах, зокрема Hobby
([Vercel: Protection Bypass for Automation](https://vercel.com/docs/deployment-protection/methods-to-bypass-deployment-protection/protection-bypass-automation)).

1. Vercel → Settings → Deployment Protection → Protection Bypass for Automation → створити секрет.
2. Скопіювати значення в GitHub: Settings → Secrets and variables → Actions → **New repository secret**, назва
   **`VERCEL_AUTOMATION_BYPASS_SECRET`**. Значення — ніде більше: ні в коді, ні в `docs/`, ні в чаті з агентом.
3. Smoke надсилає його **в заголовку** `x-vercel-protection-bypass`. Vercel приймає секрет і в параметрі адреси, але
   адреси потрапляють у журнали, тому лише заголовок.
4. Якщо секрет перегенерували у Vercel — оновіть і GitHub Secret, інакше smoke захищених деплоїв стане червоним.

Якщо захист у вас вимкнено, заголовок нічому не заважає, а smoke не зламається, коли захист колись увімкнуть.

## Доступ PM до staging на Hobby

PM має відкрити staging у браузері. На Hobby немає командних акаунтів, тож варіантів три — оберіть один і запишіть
його в runbook ([Vercel: usage and pricing](https://vercel.com/docs/deployment-protection/usage-and-pricing)):

| Варіант | Як | Межа на Hobby | Коли обрати |
|---|---|---|---|
| **Запит доступу** | PM відкриває staging, входить своїм акаунтом Vercel і натискає запит доступу; розробник підтверджує | **один зовнішній користувач на акаунт Hobby** ([Vercel Authentication](https://vercel.com/docs/deployment-protection/methods-to-protect-deployments/vercel-authentication)) | захист увімкнено, PM має (чи готовий створити) акаунт Vercel |
| **Спільне посилання** (Shareable Link) | розробник створює посилання на деплой і передає PM у чаті команди | **одне посилання на акаунт** | PM без акаунта Vercel |
| **Захист вимкнено** | Settings → Deployment Protection → Vercel Authentication вимкнено (як у WS6 `bootstrap.md`) | відкриті **всі** preview проєкту, не лише staging; окремого перемикача «лише для гілки» серед можливостей Hobby ми не знайшли — перевірте в інтерфейсі | PM без акаунта, а preview й так публічні з WS6 |

Дані на staging захищає не Vercel, а вхід у застосунок і RLS: без тестового акаунта PM побачить лише сторінку входу.
Пароль тестового акаунта — лише в `accounts.zip`; у коментарях, на знімках і в чаті його не пишемо.

## Smoke кожного деплою

У D0 (WS8) `smoke.yml` перевіряв лише прод. Тепер він перевіряє **кожен** успішний деплой, зокрема preview і
staging, — за адресою саме цього деплою. Готовий файл — [`materials/ops-kit/.github/workflows/smoke.yml`](../materials/ops-kit/.github/workflows/smoke.yml)
разом зі [`scripts/smoke.mjs`](../materials/ops-kit/scripts/smoke.mjs); промпт —
[R04 «Smoke кожного деплою з обходом захисту»](prompts.md#r04).

Як це працює:

1. Після кожного успішного деплою Vercel надсилає у ваше репо подію `repository_dispatch` типу
   `vercel.deployment.success`. У ній (`github.event.client_payload`) — адреса деплою `url` і середовище
   ([Vercel for GitHub](https://vercel.com/docs/git/vercel-for-github#repository-dispatch-events)).
2. **Прод-job** — як у D0, без змін у поведінці: `PROD_URL`, SHA = `main`, база відповідає.
3. **Новий job** — для кожного деплою не в production: бере `client_payload.url`, а не фіксовану адресу, і надсилає
   заголовок обходу із секрету `VERCEL_AUTOMATION_BYPASS_SECRET`. Без секрету — попередження в журналі
   («Немає секрету VERCEL_AUTOMATION_BYPASS_SECRET»), а за увімкненого захисту ще й червоний прогін (стіна входу
   Vercel), а не тиха зелена галочка.
4. **Алерт.** Крок `if: failure()` надсилає повідомлення людині, якщо прогін червоний. Налаштування — у
   [гайді сигналів](signals-guide.md); промпт — [R05](prompts.md#r05).

Дві пастки:

- **Workflow на `repository_dispatch` спрацьовує лише з файлу в гілці за замовчуванням.** Поки новий `smoke.yml` не
  злито в `main`, деплой staging його не запустить. Спершу злийте, потім робіть деплой staging.
- **Доказ для рубрики A — зелений прогін на деплой staging:** у журналі прогону видно адресу staging, а значення
  секрету — ні (GitHub маскує секрети, але не вмикайте `set -x` і не друкуйте заголовки).

## Відкат на Vercel Hobby і Promote

**Instant Rollback** — кнопка «повернути вітрину до попереднього стану»: Vercel перемикає прод-домен на один із
попередніх прод-деплоїв, без нової збірки ([Vercel: Instant Rollback](https://vercel.com/docs/instant-rollback)).

| Що | На Hobby |
|---|---|
| Куди можна відкотитися | **лише на безпосередньо попередній прод-деплой.** Далі — лише на Pro |
| Що з базою | нічого: схема й дані лишаються як є (див. [Спершу міграція, потім код](#спершу-міграція-потім-код)) |
| Що зі змінними середовища | відкат їх не змінює й нічого не перезбирає: старий деплой працює зі значеннями, з якими його колись зібрали |
| Що з наступними злиттями в `main` | **на прод не потрапляють**: після відкату Vercel вимикає автоматичне призначення прод-домену |
| Як повернути автоматичне просування | **Promote** потрібного деплою: кнопка в Vercel (Deployments → деплой → Promote, або Manage на сторінці відкату) чи `vercel promote <id або url>` ([Vercel CLI: promote](https://vercel.com/docs/cli/promote)) |
| CLI | `vercel rollback`, `vercel rollback status`, `vercel promote <id>`, `vercel promote status` ([Vercel CLI: rollback](https://vercel.com/docs/cli/rollback)) |

**Пастка для автозлиття з WS9.** Якщо у вас працює фабрика з автозлиттям, після відкату вона й далі зливає PR у
`main`. Збірки зелені, а на проді — старий деплой, і так доти, доки людина не зробить Promote. Підказка, що ви в цьому
стані: D0 smoke порівнює SHA на проді з `main`, тож після нових злиттів він червоний. Тому в runbook відкат і Promote —
**одна процедура з двох кроків**, а не два окремі пункти.

**Чим відкат не допоможе.** Якщо зламалась змінна середовища, відкат на старий деплой несе старі значення, а
виправлене значення в налаштуваннях не підхопить ніхто, доки не буде нового деплою. Виправлення: правильне значення
→ Redeploy → smoke. Саме такий сценарій — у [інцидент-дрілі](incident-guide.md).

**Як тренуватися.** Instant Rollback і Promote — функції **прод-деплоїв**. На staging відкат — це revert коміту в
`staging` (адреса гілки показує останній деплой) чи виправлення вперед. Тренуйте на staging цю процедуру, а відкат
проду й Promote опишіть у runbook покроково, з посиланнями на документацію Vercel. Промпт runbook —
[R11](prompts.md#r11).

## Реліз v1.0.0

**Тег** — незмінна закладка на конкретному коміті, як номер тиражу книжки: `v1.0.0` завжди означає саме цей код.
**Реліз GitHub** — сторінка до тегу з нотатками: що нового, що виправлено, хто робив.

Порядок (рубрика A):

1. Реліз-кандидат на staging → повідомлення в чат команди «реліз-кандидат на staging: <URL>».
2. **PM приймає** за ключовим сценарієм demo day і підписує коментарем `Приймання релізу (…)`.
3. **Між прийманням і тегом нічого не зливайте в `main`.** Інакше тег, staging і прод показуватимуть різний код.
4. Тег `v1.0.0` **на `main`** і реліз з нотатками: у вебінтерфейсі — Releases → Draft a new release → Choose a tag →
   `v1.0.0` (створити, target `main`) → **Generate release notes** → Publish. Або з терміналу:
   `gh release create v1.0.0 --target main --generate-notes`.
5. Перевірка: `/api/health` на проді показує SHA, на який вказує тег `v1.0.0`.

**Нотатки за категоріями.** GitHub збирає нотатки зі злитих PR і групує їх за мітками (labels) PR згідно з
`.github/release.yml` ([GitHub: automatically generated release notes](https://docs.github.com/en/repositories/releasing-projects-on-github/automatically-generated-release-notes)).
Готовий файл — [`materials/ops-kit/.github/release.yml`](../materials/ops-kit/.github/release.yml); можна власний.
Щоб категорії не були порожні, поставте мітки на ключові PR зрізів 1–4 до релізу. Промпт —
[R13 «Реліз v1.0.0: release.yml і нотатки»](prompts.md#r13). Нотатки для клієнта українською, мовою користувача, — бонус
PM.

## Ланцюг постачання

Ви не пишете React, Next.js чи `actions/checkout` самі, але відповідаєте за все, що вони роблять у вашому CI і на
проді. Аналогія — ресторан: овочі вирощує постачальник, а за страву відповідає кухня. Тому кухня перевіряє, хто
привіз, що привіз і коли.

### Навіщо: три історії

| Коли | Що сталося | Як спрацювало | Що б допомогло |
|---|---|---|---|
| вересень 2025 | **Shai-Hulud** — «черв'як» у npm, уражено сотні пакетів ([CISA](https://www.cisa.gov/news-events/alerts/2025/09/23/widespread-supply-chain-compromise-impacting-npm-ecosystem)) | скрипт, що запускається під час `npm install`, крав токени npm і публікував заражені версії інших пакетів жертви | lockfile, `min-release-age`, обережність зі скриптами встановлення |
| 19.03.2026 | **Trivy GitHub Action:** зловмисники з украденими обліковими даними переписали 76 із 77 тегів `trivy-action` і всі 7 тегів `setup-trivy` на шкідливі коміти ([GHSA-69fq-xp46-6x23](https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23)) | хто підключав дію **за тегом**, отримав у CI програму, що шукала ключі й секрети | закріплення дій за повним SHA; після інциденту — замінити всі секрети, до яких мав доступ CI |
| 31.03.2026 | **axios:** опубліковано заражені версії популярного пакета ([CISA](https://content.govdelivery.com/accounts/USDHSCISA/bulletins/413c7a5)) | захоплений акаунт супровідника | lockfile + `npm ci`, `min-release-age` |

Спільний шаблон: шкідливу версію публікують, і її підхоплюють ті, хто ставить «найсвіжіше» автоматично. Більшість таких
версій знаходять і знімають у перші дні. Звідси захист нижче.

### Що робимо

| Замок | Що це | Як | Рубрика |
|---|---|---|---|
| **Lockfile + `npm ci`** | `package-lock.json` фіксує точні версії всього дерева; `npm ci` ставить саме їх і падає, якщо lockfile не збігається з `package.json` | уже є в шаблоні: lockfile у git, у CI — `npm ci` | — |
| **`min-release-age`** | npm не ставить версії, молодші за N днів | рядки з [`materials/ops-kit/npmrc.add`](../materials/ops-kit/npmrc.add) у `.npmrc` репо (`min-release-age=7`); потрібен npm ≥ 11.10, одиниця — **дні** ([npm config](https://docs.npmjs.com/cli/v11/using-npm/config)) | D |
| **Dependabot з паузою** | бот відкриває PR з оновленнями щотижня, групами, і не пропонує версії, молодші за паузу (`cooldown`) | [`materials/ops-kit/.github/dependabot.yml`](../materials/ops-kit/.github/dependabot.yml): `npm` і `github-actions`, `weekly`, `cooldown`, групи | D |
| **Дії за SHA** | `uses: actions/checkout@<повний SHA> # v7.0.1` замість `@v7`: тег можна переписати, коміт — ні | промпт [R02](prompts.md#r02) | D |
| **Push protection** | GitHub відхиляє push, у якому знайшов секрет (ключ, токен) | Settings → Code security → Push protection → Enable | D |
| **CodeQL default setup** | безкоштовний статичний аналіз коду на вразливості, зокрема ваших workflow | Settings → Code security → Code scanning → CodeQL analysis → Set up → Default | D |
| Dependency review | показує в PR, які нові залежності й відомі вразливості він приносить | необов'язково: доступно на публічних репо ([GitHub](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-dependency-review)) | — |

Подробиці, які важливі на практиці:

- **`min-release-age`.** Перевірити, що працює: `npm install <пакет> --dry-run` з налаштуванням і без нього покаже
  різні версії, якщо остання вийшла нещодавно. `npm config get min-release-age` може надрукувати `null` — це відомий
  дефект відображення, а не ознака, що налаштування не діє. Для термінового винятку (свіжий безпековий реліз, який
  потрібен сьогодні) є параметр `min-release-age-exclude` — синтаксис у [документації npm](https://docs.npmjs.com/cli/v11/using-npm/config).
- **Dependabot.** Навіть без налаштування бот тримає паузу 3 дні; ми ставимо довшу. Пауза діє від 1 до 90 днів, для
  `github-actions` — лише `default-days`, і **не діє на безпекові оновлення**
  ([Dependabot options](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference)).
  PR Dependabot проходить ті самі ворота, що й ваш. Доказ для рубрики — хоча б один PR чи прогін бота: рубрика D не
  зараховує `dependabot.yml`, який жодного разу не запускався.
- **Supabase CLI 3.x — не зливайте.** `supabase` 3.0.0 змінює поведінку команд бази. `dependabot.yml` з набору
  вже ігнорує мажорне оновлення `supabase`, тож такого PR бути не має. Якщо він усе ж прийшов (ваш власний конфіг
  без `ignore` чи оновлення безпеки) — закрийте PR з коментарем «тримаємо 2.x до demo day» або напишіть у PR
  `@dependabot ignore this major version`.
- **Дії за SHA.** Повний SHA версії дає GitHub: `gh api repos/actions/checkout/commits/v7.0.1 --jq .sha`. Поточні
  версії станом на 09.11.2026: `actions/checkout` v7.0.1, `actions/setup-node` v7.1.0, `actions/upload-artifact`
  v7.0.2, `supabase/setup-cli` v3.2.0. Коментар `# v…` після SHA — для людей і для Dependabot. GitHub з 28.10.2026 має
  ще й lockfile для workflow (`actions.lock`), але чи працює він для репо особистого акаунта, ми не перевіряли, тож
  рубрика вимагає саме SHA.
- **Push protection вимкнений за замовчуванням.** Для публічних репо GitHub сам сканує секрети, і для вашого
  **акаунта** захист від push секретів у публічні репо ввімкнено. Але push protection **репозиторію** вимкнений, доки
  адміністратор репо його не ввімкне ([GitHub: push protection](https://docs.github.com/en/code-security/secret-scanning/introduction/about-push-protection)).
  Доказ — знімок налаштувань.
- **CodeQL** аналізує JavaScript/TypeScript і workflow GitHub Actions; безкоштовний на публічних репо
  ([GitHub: default setup](https://docs.github.com/en/code-security/code-scanning/enabling-code-scanning/configuring-default-setup-for-code-scanning)).
  Доказ — знімок налаштувань або перший прогін.
- **npm 12** (вийшов 08.07.2026) за замовчуванням не запускає скрипти встановлення залежностей. На Node 24 зараз
  npm 11.x — він лише попереджає. Попередження читайте, а npm 12 посеред курсу не ставте.
- **Чужі workflow не вставляйте наосліп.** Від 02.12.2026 подія `pull_request_target` у публічних репо за замовчуванням
  заблокована ([GitHub changelog](https://github.blog/changelog/2026-11-02-pull_request_target-blocked-by-default-december-2)).
  У шаблоні курсу її немає; якщо агент приніс сніпет з нею — це сигнал зупинитися.

### Next.js 16.4.5: перший PR Dependabot

03.11.2026 вийшов **безпековий реліз Next.js 16.4.5**: він закриває сім вразливостей середньої серйозності
(наприклад, [GHSA-mfr8-8qww-v76h](https://github.com/vercel/next.js/security/advisories/GHSA-mfr8-8qww-v76h)),
виправлених лише в 16.4.5. Шаблон курсу — на 16.4.3. Тому `next` ≥ 16.4.5 — нова умова D0, а оновлення — природний
перший PR Dependabot:

1. Злийте `dependabot.yml` → Dependabot відкриває PR з `next` 16.4.5 (безпекове оновлення може прийти й раніше за
   розклад, пауза на нього не діє).
2. CI зелений → злиття → `db-deploy` і smoke зелені → `/api/health` на проді показує новий SHA.
3. Якщо Dependabot ще не встиг — ваш PR з тим самим оновленням теж зараховується.

Промпт для всього розділу — [R02 «Ланцюг постачання: dependabot.yml, .npmrc, дії за SHA»](prompts.md#r02); перед
ним агент робить аудит лише на читання — [R01](prompts.md#r01).

## Коротко

- **Staging на Hobby** = гілка `staging` + адреса гілки + змінна лише для гілки (`APP_ENV=staging`), видна в
  `/api/health` чи на сторінці. Custom Environments — це Pro.
- **База спільна з продом:** лише тестові акаунти, жодних руйнівних дій, міграції лише ті, що додають, дріл — без
  міграцій. Зміна схеми — expand/contract.
- **Спершу міграція окремим PR, потім код.** Відкат коду ніколи не повертає схему.
- **Smoke кожного деплою** — за адресою деплою з `client_payload.url` і заголовком `x-vercel-protection-bypass`;
  секрет — лише в GitHub Secrets.
- **Відкат на Hobby** — лише на попередній деплой і вимикає автопросування: відкат і Promote — одна процедура.
- **`v1.0.0`** — тег на `main` після приймання PM, нотатки за категоріями з `.github/release.yml`.
- **Ланцюг постачання:** lockfile + `npm ci`, `min-release-age=7`, Dependabot з паузою, дії за SHA, push protection і
  CodeQL увімкнено, `next` 16.4.5 злито.
