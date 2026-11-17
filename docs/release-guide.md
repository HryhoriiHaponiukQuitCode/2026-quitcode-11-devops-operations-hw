# Середовища, реліз і ланцюг постачання: як зміна безпечно доходить до проду

**Середовище** (environment) — окрема копія застосунку зі своєю адресою і своїми налаштуваннями, де зміну можна
подивитися до того, як її побачать користувачі. **Реліз** (release) — домовленість «ось ця версія готова», з номером і
списком змін. **Ланцюг постачання** (supply chain) — усе чуже, з чого складається ваш продукт: пакети npm, дії GitHub
Actions, інструменти в CI. Цей гайд — про те, як на безкоштовних Vercel Hobby, Supabase Free і GitHub Free провести
зміну через перевірене середовище, поставити тег `v1.0.0` і не впустити в репо шкідливий пакет.

Корисні посилання: [README, трек розробника](../README.md#трек-розробника--100--10-балів) ·
[покроково](walkthrough-dev.md) · [промпти R01–R04, R13](prompts.md#r01) ·
[сигнали й алерт](signals-guide.md) · [інцидент і runbook](incident-guide.md) ·
[ліміти й вартість](cost-guide.md) · [набір `ops-kit`](../materials/ops-kit/README.md) ·
необов'язково — [прапорці, канарка, blue-green, аварійний вимикач](release-strategies.md).

Факти про тарифи й інтерфейси — **станом на 09.11.2026**, про окрему базу staging — **станом на 17.11.2026**, з
посиланням на джерело. Назви кнопок у Vercel і GitHub
змінюються частіше, ніж можливості: якщо пункту меню не знайшли — шукайте за змістом і перевірте в інтерфейсі.

## Навіщо в роботі

Зміна, яка йде на прод без перевіреного середовища, означає, що PM приймає не те, що побачить клієнт: «у мене на
preview працювало» — не доказ для релізу. Staging і тег `v1.0.0` дають одну відповідь на питання «що саме зараз у
клієнта». А кожен чужий пакет у `package.json` — ваша відповідальність: шкідлива версія з npm потрапляє на прод
так само, як ваш власний код, тому оновлення мають бути контрольованими, а не «коли згадали».

## Чотири середовища

Аналогія — театр. **Local** — репетиція вдома перед дзеркалом. **Preview** — прогін однієї сцени: кожен PR отримує
власну адресу. **Staging** — генеральна репетиція в костюмах на справжній сцені: увесь реліз-кандидат цілком, і його
приймає PM. **Production** — прем'єра з глядачами.

| Середовище | Звідки береться | Адреса | Хто дивиться | База Supabase |
|---|---|---|---|---|
| **Local** | `npm run dev` на вашому комп'ютері | `http://localhost:3000` | ви | **проєкт staging** (`.env.local` = ключі staging) |
| **Preview** | кожен push у гілку чи PR | своя на кожен деплой | ви, CodeRabbit, PM за посиланням | **проєкт staging** |
| **Staging** | гілка `staging` | стабільна адреса гілки | PM приймає реліз-кандидат | **проєкт staging** |
| **Production** | гілка `main` | прод-URL з першого рядка README | користувачі, клієнт, demo day | **прод-проєкт** |

Чим staging відрізняється від preview: адреса **не змінюється** від деплою до деплою (PM зберігає її в закладках),
у staging є **власні змінні середовища**, і туди потрапляє лише те, що вже злито в `main`. Preview — чернетка одного
PR; staging — уся наступна версія продукту перед виходом. І preview, і staging працюють **на базі staging, ніколи на
проді**: як це налаштувати — у розділі [«Staging з окремою базою»](#staging-з-окремою-базою).

## Staging на Vercel Hobby

Vercel має окрему можливість «Custom Environments» (іменоване середовище `staging` поруч із Preview і Production), але
вона **лише на Pro та Enterprise**. На Hobby Vercel офіційно описує інший шлях — «Preview branch for staging»,
доступний на всіх планах ([Vercel: Environments](https://vercel.com/docs/deployments/environments)):

**staging на Hobby = гілка `staging` + адреса цієї гілки + змінні середовища лише для цієї гілки + власний проєкт
Supabase.** Проєкт Supabase, міграції й змінні бази — у розділі [«Staging з окремою базою»](#staging-з-окремою-базою);
його кроки 1–7 робіть **до** кроку 1 нижче, щоб перший деплой гілки `staging` одразу пішов у базу staging.

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
5. **Покажіть, що це staging і яка база.** На staging системна змінна `VERCEL_ENV` все одно дорівнює `preview`, тому
   D0 `/api/health` сам staging не відрізнить. Додайте у відповідь `/api/health` назву середовища з `APP_ENV` і поле
   `supabaseRef` — ref проєкту Supabase з `NEXT_PUBLIC_SUPABASE_URL` (обидва значення не секрети). Назву середовища
   рубрика A приймає й у підвалі сторінки, а `supabaseRef` — лише в health: саме за ним видно, що база окрема. Промпт —
   [R03 «Staging: гілка, власна база, змінна для гілки, доступ PM»](prompts.md#r03).

Що **не** зараховується як staging: просто preview з PR; змінна, однакова для всіх середовищ; окрема розробка в
`staging` повз `main`; staging чи preview, що ходить у прод-базу.

**Ключ OpenRouter на staging.** Якщо у WS10 ви додали `OPENROUTER_API_KEY` у Preview, staging його успадкує. Тоді
staging витрачає ті самі 50 безкоштовних запитів на добу, що й прод і ваш eval: додаткові ключі ліміт не збільшують.
Плануйте UAT PM на staging і живий eval на різні UTC-доби. Детальніше — у [гайді вартості](cost-guide.md).

## Staging з окремою базою

**Навіщо в роботі.** Staging — генеральна репетиція: той самий код, що завтра піде на прод, але **свої дані**. Якби
staging писав у прод-базу, кожен клік PM на UAT, кожен preview з PR і кожен дріл змінював би те, що бачать
користувачі. А на безкоштовному Supabase автоматичних резервних копій немає («Not included»,
[Supabase Pricing](https://supabase.com/pricing)), тож відновити зіпсоване нічим. Тому сам Supabase радить для staging
окремий проєкт: «You need a *new* project for staging»
([Supabase: Managing environments](https://supabase.com/docs/guides/deployment/managing-environments)), а на сторінці
цін пише, що Free дає два проєкти саме для цього: «one for development and one for production»
([Supabase Pricing](https://supabase.com/pricing)). Усі факти цього розділу — станом на 17.11.2026.

Раніше (WS6) staging ділив базу з продом — тепер так не робимо, бо тоді staging тренує не реліз, а обережність з
бойовими даними, і будь-яка помилка на репетиції стає інцидентом на проді.

**Терміни.**

- **Проєкт Supabase** — окрема база Postgres зі своїм Auth, своїми ключами й своєю адресою `https://<ref>.supabase.co`.
- **ref** — короткий ідентифікатор проєкту з цієї адреси. Він не секрет: браузер і так отримує його з
  `NEXT_PUBLIC_SUPABASE_URL`.
- **Слот** — одне з двох місць під активний безкоштовний проєкт. Призупинений проєкт слота не займає.
- **Environment у GitHub** — «сейф» для секретів: відкрити його може лише job, що на нього посилається, і лише з
  дозволених гілок.

### Схема: хто в яку базу ходить

| Середовище | База Supabase | Змінні бази у Vercel | Адреси входу (Auth URL) — у якому проєкті |
|---|---|---|---|
| Local | **staging** | `.env.local` = ключі staging (не в git) | `http://localhost:3000/**` — у staging |
| Preview (PR) | **staging** | Preview, усі гілки = значення staging; секретного ключа немає | `https://*-<scope>.vercel.app/**` — у staging |
| Staging | **staging** | ті самі Preview + лише для гілки `staging`: `APP_ENV=staging` і, якщо застосунку він потрібен, секретний ключ **staging** | Site URL і `https://<адреса-staging>/**` — у staging |
| Production | **прод** | Production = значення проду | лише прод-адреси — у проді |

Міграції: PR → CI (`db-check` на одноразовій базі) → злиття в `main` → job **`db-deploy-staging`** (база staging) →
лише якщо зелено, job **`db-deploy`** (прод). Дані: у проді — те, що там уже є; у staging — міграції, seed, тестові
акаунти й тестові записи. **Прод-дані в staging не копіюємо ніколи**: ні дампом, ні експортом CSV, ні «щоб було схоже».

Порядок кроків нижче продиктований залежностями: база staging має бути готова (схема, акаунти, дані) **до** того, як
на неї перемкнуться Preview; секрет і змінні в GitHub — **до** злиття job міграцій, інакше CI на `main` почервоніє;
Preview перемикаємо **до** першого деплою гілки `staging`. У
[walkthrough-dev, крок 2](walkthrough-dev.md#2-staging-з-окремою-базою-і-smoke-кожного-деплою-25-год--трек-a) ці кроки
розкладено разом із гілкою й smoke.

### 1. Звільнити слот

1. Supabase Dashboard → список проєктів у **всіх** ваших організаціях: «The project limit applies across all
   organizations where you are an Owner or Administrator», а «Paused projects do not count towards your free project
   limit» ([Supabase: Billing](https://supabase.com/docs/guides/platform/billing-on-supabase)).
2. Якщо активних уже два (прод і, наприклад, бонусний двійник WS9 `<team-slug>-factory`), у зайвому проєкті:
   **Settings → General → Project availability → Pause Project**
   ([Supabase: Pause or delete a project](https://supabase.com/docs/guides/platform/delete-project)). Призупинити можна
   лише Free-проєкти.
3. **Пауза, а не видалення:** призупинений проєкт можна відновити (Resume project) «up to 1 year after it was paused»
   ([Supabase: Project pausing](https://supabase.com/docs/guides/platform/free-project-pausing)), а видалений — ні: «We
   cannot recover deleted projects».
4. Якщо це двійник WS9: його прод перестане відповідати, а розклади в його репо (smoke, фабрика) почнуть падати. У репо
   двійника вимкніть ці workflow (Actions → workflow → Disable workflow; назву пункту перевірте в інтерфейсі). Оцінки
   WS9 фінальні, докази лежать у git і Actions.
5. **Прод не призупиняйте й не видаляйте.** Якщо зайвого проєкту немає, а обидва слоти потрібні, напишіть у чат курсу:
   варіанту «staging на прод-базі» немає.

### 2. Створити проєкт staging

- Назва `<team-slug>-staging`, **регіон той самий, що в проду**: однакові затримки, і одна сторінка статусу стосується
  обох проєктів.
- Пароль бази — одразу в менеджер паролів: він знадобиться для рядка підключення.
- **Налаштування Data API — як у проді** (точні назви перемикачів перевірте в інтерфейсі). Supabase попереджає:
  «Default privileges differ between projects, depending on when the project was created and its Data API settings»,
  і «Supabase is changing the platform default to revoke these automatic grants so that exposure becomes opt-in»
  ([Supabase: Securing your API](https://supabase.com/docs/guides/api/securing-your-api)). Тому новий проєкт може
  видати новим таблицям інші права, ніж ваш прод. Захист — у таблиці [«Що не переїде само»](#що-не-переїде-само).
- Що забрати з Dashboard (лише в менеджер паролів, у Vercel чи в GitHub — не в git і не в чат): адресу проєкту,
  publishable key, secret key і рядок **Session pooler** (порт 5432) з кнопки **Connect** проєкту **staging**. Формат
  рядка: `postgresql://postgres.[PROJECT-REF]:[YOUR-DATABASE-PASSWORD]@[POOLER-HOST]:5432/postgres`
  ([Supabase: Connect to your database](https://supabase.com/docs/guides/database/connecting-to-postgres)). **Копіюйте
  рядок цілком з проєкту staging; не правте прод-рядок, замінюючи ref і пароль** — так найлегше помилитися базою.
- Ref проєкту — частина адреси `https://<ref>.supabase.co` (і `postgres.<ref>` у рядку pooler).

### 3. GitHub: environment `staging`, секрет і змінні з ref

Робите **до** злиття PR з job міграцій staging (крок 4), у вебінтерфейсі GitHub:

1. Settings → Environments → **New environment** `staging` → Deployment branches: **Selected branches** → `main`. Так,
   саме `main`: міграції в обидві бази запускає лише конвеєр на `main`; PR-гілки й гілка `staging` секрету бази staging
   не отримають.
2. У environment `staging` — секрет **`SUPABASE_DB_URL`** = рядок Session pooler проєкту staging (крок 2). Секрети
   environment бачать лише jobs, що на нього посилаються: «These secrets are only available to workflow jobs that use
   the environment» ([GitHub: Managing environments](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments)).
   На GitHub Free environments доступні лише в публічних репо; ваші репо публічні, а environment `production` у вас уже
   є. Якщо репо приватне — напишіть у чат курсу.
3. Settings → Secrets and variables → Actions → **Variables** → Repository variables: **`SUPABASE_STAGING_REF`** = ref
   staging, **`SUPABASE_PROD_REF`** = ref проду. Це не секрети, а запобіжник: job звіряє з ними, куди веде рядок
   підключення.
4. Гігієна: `SUPABASE_DB_URL` має бути **лише** секретом середовищ (`staging` і `production`), а не секретом
   репозиторію (Repository secrets).

### 4. Job міграцій staging: спершу staging, потім прод

Зразок — [`materials/ops-kit/ci-examples/db-deploy-staging.yml`](../materials/ops-kit/ci-examples/db-deploy-staging.yml)
(**приклад, не запускали**); агенту ту саму зміну доручає промпт [R03](prompts.md#r03). Що в ньому:

- новий job `db-deploy-staging`: лише на `main` (push і ручний запуск), `environment: staging`, ті самі кроки dry run →
  `supabase db push --db-url` закріпленою CLI 2.x, що й у `db-deploy`;
- **запобіжник** першим кроком: бере ref з імені користувача `postgres.<ref>` у рядку підключення і зупиняє job
  (`exit 1`), якщо секрету немає, рядок не схожий на Session pooler або ref ≠ `SUPABASE_STAGING_REF`. У журнал іде лише
  ref (він публічний) — рядок «Ціль міграцій: проєкт <ref> (staging)»;
- **без пропуску й без запасного шляху через `link`**: зник секрет чи веде не туди — job червоний, і прод не мігрує;
- `db-deploy` (прод) тепер `needs: … db-deploy-staging` і має дзеркальний запобіжник: ref = `SUPABASE_PROD_REF`;
- новий вхід ручного запуску `include_seed_staging`: seed у staging, а наявний `include_seed` і далі стосується лише
  проду.

**Перший прогін.** Злиття PR уже запускає `db-deploy-staging`: у порожню базу staging накотяться **всі** міграції з
нуля. Це заодно доводить, що історію міграцій можна відтворити. Прод-job після нього нічого нового не застосує. Далі —
Actions → CI → Run workflow (гілка `main`, `include_seed_staging` = true) для seed. Чи застосує CLI seed, коли всі
міграції вже накочено, ми не перевіряли: дивіться журнал кроку dry run; якщо seed там немає — дані для UAT введіть
через застосунок (крок 5).

**Якщо staging впав або спить.** Міграція впала на staging → `db-deploy` (прод) **не запускається**: це і є ворота,
зламана міграція не доходить до проду. Виправлення — нова міграція вперед, окремим PR. Staging на паузі → `db push` не
під'єднається → прод теж не мігрує: Resume project у staging і перезапуск впалих jobs цього прогону в Actions (назву
кнопки перевірте в інтерфейсі).

**Еквіваленти в оцінюванні.** Зараховуємо будь-який шлях, де міграції в базу staging застосовує **CI з секрету
environment** із запобіжником ref, а не ноутбук, — зокрема тригер на push у гілку `staging`, як у прикладі Supabase.
Тоді в runbook запишіть пастку: PR-preview між злиттям і оновленням `staging` працюватимуть на старій схемі.

### 5. Тестові акаунти, локальна розробка й дані для UAT

1. Поточний `.env.local` (там прод) за бажання збережіть як `.env.prod.local`: правило `.env*` у `.gitignore` шаблону
   його вже покриває. Агент ці файли не читає.
2. У `.env.local` — значення **staging**: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`,
   `SUPABASE_SECRET_KEY` і **той самий** `DEMO_USER_PASSWORD`, що й для проду.
3. `npm run users:demo` створить ті самі акаунти з `supabase/demo-users.json` у staging (скрипт ідемпотентний, код не
   змінюємо). PM користується тим самим `accounts.zip`, нічого нового не надсилаємо. У справжньому проєкті паролі
   тестових акаунтів staging і проду варто розводити; тут їх зводимо, бо дані синтетичні.
4. **Дані для UAT на staging.** Свіжа база staging має лише міграції, seed і акаунти. Довідники й записи, які ви
   колись створили на проді через інтерфейс (у демо — підрозділи й посади, без яких не подати заявку), на staging
   **відсутні**, і сценарій PM упаде. Два правильні шляхи: синтетичні довідники — у `supabase/seed.sql` звичайним PR
   (ідемпотентно, `on conflict …`, лише `*@example.test`) і Run workflow з `include_seed_staging`; або введіть їх на
   staging через застосунок як тестовий адміністратор. **Прод-дані не копіюємо.**
5. **Хто ще читає `.env.local`.** З WS8 його читають і `npm run uat` / `npm run test:e2e` (Playwright завантажує
   `.env.local`, створює й видаляє синтетичних `uat-*` користувачів секретним ключем). Тепер ці записи йдуть у
   staging, тож UAT запускайте **проти staging**: `npm run uat -- --url <staging>`, а не проти проду — інакше браузер
   увійде в прод, а синтетичні користувачі з'являться в staging. Якщо захист Vercel на staging увімкнено, Playwright
   потрібен заголовок обходу: приклад `extraHTTPHeaders` з `x-vercel-protection-bypass` і
   `x-vercel-set-bypass-cookie` — у [документації Vercel](https://vercel.com/docs/deployment-protection/methods-to-bypass-deployment-protection/protection-bypass-automation)
   (станом на 17.11.2026; у шаблоні курсу цього заголовка немає — доручіть агенту через [R03](prompts.md#r03)).
   Те, що справді має йти в прод (наприклад, `users:demo` для проду), запускайте явно з окремим файлом:
   `node --env-file=.env.prod.local scripts/create-demo-users.mjs`. `.env.local` туди-сюди не перемикайте.
6. **`vercel env pull` перепише `.env.local`.** Ця команда «will export development environment variables to a local
   `.env.local` file» ([Vercel CLI: env](https://vercel.com/docs/cli/env), станом на 17.11.2026). Тож у Vercel у
   середовищі **Development** мають бути значення staging або нічого (крок 7), інакше одна команда тихо поверне
   локальну розробку на прод.

### 6. Auth у проєкті staging

- Authentication → налаштування **як у проді**: реєстрацію вимкнено, «Confirm email» — як у вашому прод-проєкті (точні
  назви перевірте в інтерфейсі). Якщо в проді ви щось змінювали, повторіть це в staging.
- URL Configuration у **staging**: Redirect URLs — `https://*-<scope>.vercel.app/**` (preview) і
  `http://localhost:3000/**`; коли з'явиться адреса гілки `staging` — Site URL = ця адреса і ще
  `https://<адреса-staging>/**`. Wildcard для Vercel preview і точні адреси для проду радить
  [Supabase: Redirect URLs](https://supabase.com/docs/guides/auth/redirect-urls).
- URL Configuration у **проді**: після перемикання змінних Vercel (крок 7) лишіть **лише прод-адреси**: приберіть
  `localhost`, preview-wildcard і адресу staging, якщо додавали.
- **Пастка:** `supabase config push` з прод-значеннями `site_url` у `supabase/config.toml` переписав би Auth staging.
  URL staging задаємо в Dashboard, а `config push` у staging не робимо.

### 7. Vercel: змінні бази для кожного середовища

Спершу інвентаризація: які змінні Supabase читає **ваш** застосунок —
`git grep -n -o -E "process\.env\.[A-Z_]+" -- src` (лише назви). Зазвичай це три:

| Змінна | Production | Preview, усі гілки | Preview, лише гілка `staging` | Development |
|---|---|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | прод | **staging** | — (бере загальне Preview) | staging або нічого |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | прод | **staging** | — | staging або нічого |
| `SUPABASE_SECRET_KEY` (якщо є з WS7: запрошення через Admin API) | прод | **нічого** | **ключ staging** | нічого |

**Правило секретного ключа:** прод-ключ — лише Production; ключ staging — лише гілка `staging`; у PR-preview —
жодного `sb_secret_…`. Preview збирається з коду будь-якої гілки, а секретний ключ обходить RLS: «A secret key bypasses
every Row Level Security policy you have»
([Supabase: API keys](https://supabase.com/docs/guides/api/api-keys)). Це правило **замінює** вказівку WS7 «лише
Production»: без ключа staging адмінські функції на staging впадуть з «SUPABASE_SECRET_KEY не задано», а «виправлення»
галочкою Preview на прод-ключі віддало б прод-ключ кожному PR-preview.

**Як розділити змінну, не зламавши прод.** У WS6 кожна змінна — один запис з кількома галочками (Production, Preview,
можливо Development). Якщо просто змінити значення, воно зміниться **в усіх** позначених середовищах: «The new value
applies to every Environment that the variable covers» ([Vercel CLI: env](https://vercel.com/docs/cli/env)). Тому для
кожної змінної з таблиці:

1. Відкрийте наявний запис → лишіть галочку **лише Production** → Save. Значення — прод, без змін.
2. **Add new** → та сама назва → лише **Preview**, усі гілки (без вибору конкретної) → значення staging. Для
   `SUPABASE_SECRET_KEY` замість цього: Preview → **гілка `staging`** → ключ staging.
3. Development: значення staging або без галочки.
4. Перевірка: жодне значення **не-Production** (Preview і Development) не вказує на прод-проєкт. За бажанням —
   `vercel env ls` (лише назви й середовища; значення не друкуйте).

Перевизначення для гілки має пріоритет над загальним Preview: «Any branch-specific variables will override other
preview environment variables with the same name» ([Vercel: Environment variables](https://vercel.com/docs/environment-variables)).
Тому `NEXT_PUBLIC_SUPABASE_*` задаємо на рівні **всього** Preview, а не лише для гілки `staging`: тоді PR-preview теж
ходять у staging, а крок дрілу «видалити хибне перевизначення» повертає staging до бази staging. Якщо Supabase
підключено через Vercel Marketplace, перевірте, у які середовища інтеграція пише змінні.

**Redeploy.** Зміни змінних діють лише на нові деплої, а `NEXT_PUBLIC_*` вбудовуються під час збірки. Після
перемикання: Redeploy останнього деплою `staging` (якщо гілка вже є) і новий push у кожен відкритий PR. **Старі
preview-адреси**, зібрані раніше, далі ходять у прод: не відкривайте їх і не надсилайте PM.

**Чому саме так на Hobby.** «Preview branch for staging» доступний на «All plans, including Hobby», а Custom
Environments — лише Pro і Enterprise ([Vercel: Environments](https://vercel.com/docs/deployments/environments)). Там
само Vercel радить «a custom environment or preview branch with staging credentials when you need separate resources».

### 8. Перевірка: база справді окрема

1. Прод `/api/health` (GET без входу) → `supabaseRef` = ref проду.
2. Прогін smoke на деплой `staging`: у журналі рядок JSON health з `"appEnv":"staging"` і `supabaseRef` = ref staging
   (≠ прод). `scripts/smoke.mjs` з набору вже друкує всю відповідь health.
3. Прогін smoke на деплой **будь-якого PR-preview** після перемикання змінних: `supabaseRef` = ref staging. Без цього
   пункту перевизначення лише для гілки `staging` пройшло б перевірку, хоча PR-preview далі писали б у прод.
4. Прогін CI на `main`: у `db-deploy-staging` рядок «Ціль міграцій: проєкт <ref> (staging)» з тим самим ref, що й у
   п. 2, і крок «Apply migrations» — `success`.
5. Вхід на staging акаунтом з `accounts.zip` працює; прод `/api/health` не змінився.

### Правила staging

| Правило | Чому |
|---|---|
| **Лише синтетичні дані**, акаунти `*@example.test`; прод-даних у staging немає | staging бачать PM, preview і агент на дрілі; копія проду — це витік, а не «реалістичність» |
| Міграції — лише через `main` і CI (`db-deploy-staging`, потім `db-deploy`), ніколи вручну з ноутбука | лише так схема спершу проходить справжню базу Supabase, а прод не мігрує після помилки |
| Ніякого `supabase migration down` і `db reset --linked` проти **жодного** з ваших проєктів | обидві команди знищують дані: `migration down` не «відкочує крок», а скидає схему й накочує міграції заново; резервних копій на Free немає |
| Supabase CLI — закріплена в шаблоні версія (`npx supabase` з проєкту), не `npx supabase@latest` | CLI 3.0.0 (02.11.2026) за замовчуванням спрямовує команди бази в **локальну** базу ([Supabase CLI v3](https://supabase.com/changelog/supabase-cli-v3)) |
| Прод-ключ (будь-який) — лише Production; секретний ключ staging — лише гілка `staging`; у PR-preview — жодного `sb_secret_…` | preview збирається з коду будь-якої гілки, а секретний ключ обходить RLS |
| **Інцидент-дріл — лише staging, без міграцій і без видалення даних**, без ротації ключів у Supabase | схему не відкотити, лише нова міграція вперед; резервних копій немає; базою staging користуються PM на UAT і всі preview, тож зламана база staging зупиняє всю команду |
| Перед UAT, демо чи дрілом — Dashboard → проєкт staging активний | staging на Free засинає так само, як прод ([нижче](#пауза-staging)) |

### Що не переїде само

Новий проєкт отримує лише міграції, seed і акаунти з `users:demo`. Симптом: на staging щось не працює, а на проді
працює. Ліки щоразу ті самі — **перенести в міграцію через звичайний PR** (тоді воно з'явиться в обох базах), а не
клацати вдруге руками лише в staging.

| Що | Як виправити |
|---|---|
| Те, що на проді колись зробили руками в Dashboard: бакети Storage, розширення, політики поза міграціями | нова міграція |
| **Права (GRANT) на таблиці.** Проєкти, створені в різний час, можуть мати різні права за замовчуванням (цитата в кроці 2) | кожна міграція містить явні `grant` для `anon` / `authenticated` / `service_role` поруч з RLS, як міграція `profiles` у шаблоні. Бачите на staging «permission denied» — міграція вперед з потрібним `grant`, а не перемикач лише в staging |
| Довідники й записи, створені на проді через інтерфейс | синтетичні — у `seed.sql` або через застосунок на staging (крок 5); прод-даних не копіюємо |

### Пауза staging

Free-проєкт засинає «after 1 week of inactivity» ([Supabase Pricing](https://supabase.com/pricing)). «A few requests a
day is usually enough to keep a project running, though that isn't a guarantee», а запити застосунку й відкриття
проєкту в Dashboard рахуються як активність ([Supabase KB](https://supabase.com/kb/faqs/platform/free-project-pausing)).
Staging будять локальна розробка, PR-preview і smoke кожного деплою — усі вони тепер ходять у staging. Але щоденний
розклад smoke будить лише прод. Тож правило просте: **перед UAT, демо чи дрілом відкрийте Dashboard → проєкт staging і,
якщо він на паузі, натисніть Resume project.** Дані й налаштування повернуться
([Supabase: Project pausing](https://supabase.com/docs/guides/platform/free-project-pausing)). Скільки триває
відновлення, документація не каже.

### Якщо staging уже ходив у прод-базу

1. Звільнити слот і створити `<team-slug>-staging` (кроки 1–2).
2. GitHub: environment `staging`, секрет, змінні з ref (крок 3).
3. PR за оновленим [R03](prompts.md#r03): `supabaseRef` у health, job `db-deploy-staging` із запобіжником,
   `include_seed_staging` → злиття → прогін зелений → Run workflow з `include_seed_staging` (крок 4).
4. `.env.local` → staging, `npm run users:demo`, дані для UAT (крок 5); Auth staging (крок 6).
5. Vercel: розділити змінні за таблицею кроку 7, видалити старе перевизначення гілки `staging` з прод-значеннями, якщо
   воно є (`APP_ENV=staging` лишається); Redeploy `staging` і новий push у відкриті PR.
6. Перевірка (крок 8); прод-Auth — лише прод-адреси.
7. Runbook: таблиця середовищ з двома проєктами (назва і ref, без ключів), «база на паузі» для кожного, ротація ключа
   окремо для Production і для гілки `staging`.
8. Тестові записи, які раніше потрапили з staging у прод, прибирайте лише свої, через інтерфейс застосунку й по
   одному — без масових запитів до бази.

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

2. **Код (PR 2, після того як `db-deploy-staging` і `db-deploy` застосували PR 1).** Код читає й пише `comment`.
   Обидві колонки існують в обох базах, тож працюють і preview, і staging, і прод, і відкат на попередній деплой.
3. **Contract (PR 3, щонайменше на один реліз пізніше).** Прибрати `note`. Це руйнівна міграція, тож **не цього
   тижня**. Contract — після demo day, окремим PR, коли старого коду вже немає ні на проді, ні на staging (спершу
   оновіть `staging` з `main`).

**Чому це потрібно і з окремою базою.** У кожен момент існує код, старіший за схему: прод під час збірки Vercel, прод
після Instant Rollback і staging між злиттям у `main` і оновленням гілки `staging`. Тому кожна міграція має бути
сумісна з кодом, що зараз на проді і на staging. PR-preview працюють на схемі бази staging (= схема `main`): якщо PR
містить і міграцію, і код, що її використовує, preview буде червоним до злиття міграції — живий приклад, навіщо
міграція йде окремим PR першою.

Промпт для перевірки міграції — у [R01 «Аудит готовності до експлуатації»](prompts.md#r01): агент лише читає й
каже, чи сумісна міграція з кодом у `main` і чи її треба розбити.

## Спершу міграція, потім код

Міграції застосовує лише CI на `main`, після злиття: спершу job `db-deploy-staging` у базу staging (середовище GitHub
`staging`), і лише якщо там зелено — job `db-deploy` у прод (середовище `production`); див.
[крок 4](#4-job-міграцій-staging-спершу-staging-потім-прод). Vercel тим часом збирає той самий коміт `main`
**паралельно**: нічого не гарантує, що схема оновиться раніше, ніж новий код стане продом. На Hobby цей порядок не замкнений автоматично. Vercel має «Deployment Checks», що тримають прод-деплой
до зелених перевірок GitHub, але чи працюють вони на Hobby, ми не перевірили. Тому правило — у порядку PR:

- **Міграція — окремим PR і першою.** Злили → **обидва** jobs, `db-deploy-staging` і `db-deploy`, зелені (у Actions
  крок «Apply migrations» — `success`, а не `skipped`) → лише тоді PR з кодом, який її використовує.
- **Кожна міграція сумісна з кодом, що зараз на проді і на staging.** Саме тому expand/contract.

**Чому відкат ніколи не повертає схему.** Vercel Instant Rollback перемикає домен на старий деплой і нічого не
перезбирає. До бази він не торкається: міграції, які застосував `db-deploy`, лишаються — і в проді, і тим паче в
staging. Після відкату **старий код працює з новою схемою**, і це безпечно, лише якщо міграція сумісна зі старим кодом
(expand/contract). Відкат схеми в Supabase — це не «undo», а
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

Дані бази staging захищає не Vercel, а вхід у застосунок і RLS: без тестового акаунта PM побачить лише сторінку входу.
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

**Відкат чи аварійний вимикач.** Відкат повертає всю попередню версію, разом з усіма іншими змінами. Вимкнути лише
одну функцію без нового деплою — це аварійний вимикач (kill switch), і змінна середовища для нього не годиться: вона
діє лише після нового деплою. Що це, як зробити в нашому стеку і що з канарки й blue-green доступно на Hobby — у
[стратегіях викочування](release-strategies.md) (необов'язково, без балів; станом на 17.11.2026).

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
у гілці 16.x виправлених у 16.4.5. Шаблон курсу — на 16.4.3. Тому `next` ≥ 16.4.5 — нова умова D0, а оновлення — природний
перший PR Dependabot:

1. Злийте `dependabot.yml` → Dependabot відкриває PR з `next` 16.4.5 чи новішим (безпекове оновлення може прийти й
   раніше за розклад, пауза на нього не діє; версійне — у груповому PR `npm-minor-and-patch`, станом на 17.11 уже з
   16.4.6 — теж зараховується).
2. CI зелений **і деплой preview Vercel у PR теж зелений** (чому це окрема перевірка — розділ нижче) → злиття →
   `db-deploy` і smoke зелені → `/api/health` на проді показує новий SHA.
3. Якщо Dependabot ще не встиг — ваш PR з тим самим оновленням теж зараховується.

### «CI зелений, деплой червоний»: PR #42 → #46 у демо (17.11.2026)

Необов'язкове читання, рубрику не змінює. Справжній випадок з демо-репо курсу.

**Навіщо в роботі.** Зелені галочки CI здаються дозволом зливати. Але CI і хостинг можуть ставити залежності
**по-різному** — і тоді CI перевіряє не те, що потрапить на прод.

**Що сталося.** Груповий PR Dependabot [#42](https://github.com/koldovsky/2026-quitcode-06-leave-portal-demo/pull/42)
(21 оновлення, `npm-minor-and-patch`): усі перевірки CI зелені або пропущені, а деплой Vercel — `Error` на кроці встановлення
залежностей: `npm error code ERESOLVE … Conflicting peer dependency`.

- **Причина.** `@vitest/eslint-plugin` **1.6.28** — patch-реліз, тобто третя цифра версії — підняв свою
  **peer-залежність** (пакет, який він очікує поруч у проєкті) з `typescript >=5.0.0` до `>=6.0.0`. У демо TypeScript
  5.9.3, а `typescript-eslint` 8 допускає лише `<6.1.0`.
- **Чому CI зелений.** CI ставить залежності командою `npm ci` — строго за `package-lock.json`, нічого не перераховуючи.
  Vercel сам визначає команду встановлення за lock-файлом: для npm з `package-lock.json` це `npm install`
  ([документація Vercel «Package Managers»](https://vercel.com/docs/package-managers), станом на 17.11.2026), а вона
  заново перевіряє peer-залежності. Відтворено локально на npm 11.16.0: `npm ci` → успіх, `npm install` →
  `ERESOLVE` (опис [#46](https://github.com/koldovsky/2026-quitcode-06-leave-portal-demo/pull/46)).

**Як виправили** — [#46](https://github.com/koldovsky/2026-quitcode-06-leave-portal-demo/pull/46), злито 17.11.2026
(коміт злиття `084bdec`); GitHub позначив і #42 злитим, бо його коміт увійшов у #46:

1. Решта 20 оновлень з #42 — без змін, зокрема `next` 16.4.3 → **16.4.6** (≥ 16.4.5, безпековий реліз).
2. `@vitest/eslint-plugin` лишився на **1.6.27**.
3. У `.github/dependabot.yml` — `ignore` для `@vitest/eslint-plugin` версій `>=1.6.28` **з коментарем**: чому, з якого
   PR і коли рядок прибрати (під час переходу на TypeScript 6 окремим PR). Без коментаря такий `ignore` через місяць
   стає загадкою.
4. `vercel.json`: `"installCommand": "npm ci"` — деплой ставить залежності так само, як CI, і лише за lockfile.

**Уроки:**

- **Однакова збірка всюди.** CI і хостинг ставлять залежності однією командою (`npm ci`), інакше CI перевіряє інший
  проєкт, ніж той, що їде на прод. Перевірка перед злиттям — **усі** статуси PR, включно з деплоєм preview.
- **Patch ≠ безпечно.** Номер версії — обіцянка автора пакета, а не гарантія: patch-реліз може змінити вимоги до
  сусідніх пакетів.
- **Lockfile — це контракт.** `package-lock.json` фіксує дерево, яке ви перевірили; `npm ci` його виконує, а
  `npm install` може його переписати.
- **Пауза (`cooldown`, `min-release-age=7`) тут не допомогла б:** версію 1.6.28 опубліковано 22.10.2026, за кілька
  тижнів до PR. Пауза захищає від **свіжих шкідливих** релізів, а від несумісних — лише перевірка збірки так, як на
  проді. Потрібні обидва замки.

Промпт для всього розділу — [R02 «Ланцюг постачання: dependabot.yml, .npmrc, дії за SHA»](prompts.md#r02); перед
ним агент робить аудит лише на читання — [R01](prompts.md#r01).

## Коротко

- **Staging на Hobby** = гілка `staging` + адреса гілки + змінна лише для гілки (`APP_ENV=staging`), видна в
  `/api/health` чи на сторінці. Custom Environments — це Pro.
- **Staging має власну базу Supabase:** preview, staging і локальна розробка — на ній, прод — лише на прод-базі;
  міграції CI застосовує спершу в staging, потім у прод (із запобіжником ref); прод-даних у staging немає; дріл — без
  міграцій. Зміна схеми — expand/contract.
- **Спершу міграція окремим PR, потім код.** Відкат коду ніколи не повертає схему.
- **Smoke кожного деплою** — за адресою деплою з `client_payload.url` і заголовком `x-vercel-protection-bypass`;
  секрет — лише в GitHub Secrets.
- **Відкат на Hobby** — лише на попередній деплой і вимикає автопросування: відкат і Promote — одна процедура.
- **`v1.0.0`** — тег на `main` після приймання PM, нотатки за категоріями з `.github/release.yml`.
- **Ланцюг постачання:** lockfile + `npm ci`, `min-release-age=7`, Dependabot з паузою, дії за SHA, push protection і
  CodeQL увімкнено, `next` ≥ 16.4.5 злито.
