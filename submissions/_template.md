# Здача WS11

<!--
Скопіюйте цей файл у submissions/<team-slug>/<github-login>.md і заповніть УСІ поля.
Ім'я та прізвище сюди НЕ пишіть — лише в опис PR, pull request — запиту на злиття змін (репозиторій публічний).
Поле або розділ, що не стосується вашої ролі, — рівно «—». Коментарі-підказки після заповнення можна видалити.
Посилання — повні: https://github.com/… або https://….vercel.app. Файли — на SHA (Secure Hash Algorithm — тут: 40-символьний ідентифікатор коміту): …/blob/<SHA>/….
PM (project manager — менеджер проєкту) здає репо вимог <pm>/<team-slug>-spec, розробник — своє репо <dev>/<team-slug>.
Жодних паролів: акаунти — лише адреси *@example.test. Пароль від accounts.zip теж не пишемо.
Жодних значень токенів, ключів і chat id (TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID, VERCEL_AUTOMATION_BYPASS_SECRET,
SENTRY_AUTH_TOKEN, BACKUP_PASSPHRASE, SUPABASE_DB_URL) — лише назви змінних.
Адреси prod і staging — без параметрів після «?». Секрет обходу захисту Vercel (x-vercel-protection-bypass) в адресі — ніколи.
-->

## Дані

- Роль: <!-- pm або dev -->
- Команда: <!-- property-workflows | therapy-scheduling | school-purchasing | leave-portal -->
- GitHub-логін: <!-- як у назві файлу -->
- Ваше репо: <!-- PM: https://github.com/<ваш-логін>/<team-slug>-spec · розробник: https://github.com/<ваш-логін>/<team-slug> -->
- SHA: <!-- SHA (Secure Hash Algorithm — тут: 40-символьний ідентифікатор коміту): 40 hex-символів (hexadecimal — шістнадцяткові: 0–9, a–f), останній коміт main ВАШОГО репо на момент здачі. Розробник: git fetch origin → git rev-parse origin/main. PM: сторінка комітів main → кнопка копіювання повного SHA -->
- Репо вимог PM: <!-- https://github.com/<pm>/<team-slug>-spec · PM (project manager — менеджер проєкту): те саме, що «Ваше репо» -->
- Прод-URL: <!-- URL (Uniform Resource Locator) — адреса. Розробник: https://<проєкт>.vercel.app (той самий, що першим рядком README) · PM: «—» -->
- Staging-URL: <!-- розробник: домен гілки staging у Vercel, https://….vercel.app — без параметрів після «?» · PM: «—» -->

## PM

<!-- Розробник: заголовок «## PM» залиште, а все під ним (і підрозділи) замініть одним рядком «—».
     Посилання A і C — на ваше репо вимог; посилання B — здебільшого на репо розробників. -->

### Операційні вимоги (трек A)

- Сховище: <!-- docs/operations.md або issue «Операційні вимоги» -->
- Операційні вимоги: <!-- …/blob/<SHA>/docs/operations.md або …/issues/N -->
- SLO: <!-- SLO — Service Level Objective, ціль рівня сервісу. 2–3 рядки «SLO-1: що отримує користувач — ціль за 28 днів — де міряємо — бюджет помилок» -->
- Матриця серйозності: <!-- SEV — severity, рівень серйозності. «SEV1–3, по 2 приклади, хто реагує і як швидко» + якщо треба, посилання на розділ …#… -->
- RPO і RTO: <!-- RPO — Recovery Point Objective (скільки даних можна втратити), RTO — Recovery Time Objective
  (скільки продукт може не працювати). Значення й одним реченням, що це означає на безкоштовному тарифі -->
- Хто отримує алерт: <!-- канал і ролі, напр. «Telegram-група команди: розробник, PM». Без chat id, імен і телефонів -->
- Після курсу: <!-- Pro, пауза чи архів — одним реченням -->

### Приймання релізу (трек B)

- Сценарій demo day для UAT: <!-- UAT — User Acceptance Testing, приймальне тестування. 5–8 кроків одним рядком або посилання на розділ …/blob/<SHA>/docs/demo-day.md -->
- UAT на staging: <!-- на кожного розробника команди з «реліз-кандидат на staging» один рядок:
     «@<dev>: staging https://….vercel.app, сценарій ✅, підпис https://github.com/<dev>/<team-slug>/pull/N#issuecomment-M»
     або «@<dev>: ❌ крок N → https://github.com/<dev>/<team-slug>/issues/M, підпис …#issuecomment-K»
     або «@<dev>: не прийнято: <причина, дата>». Адреси staging — без параметрів після «?» -->
- Відхилення від вимог: <!-- issues у репо розробників https://github.com/<dev>/<team-slug>/issues/N, … — або «відхилень немає» -->
- Розбір postmortem і runbook: <!-- на кожного розробника — коментар у його PR: https://github.com/<dev>/<team-slug>/pull/N#issuecomment-M, … — або «не прийнято: postmortem не надходив, <дата>» -->
- Повідомлення клієнту про інцидент: <!-- …/blob/<SHA>/docs/incident-message.md — за чиїм postmortem: @<dev> -->

### Готовність до demo day і вартість (трек C)

- Чекліст demo day: <!-- …/blob/<SHA>/docs/demo-day.md -->
- Точки відмови з планом Б: <!-- перелік одним рядком, напр. «прод не відкривається, база на паузі, ліміт моделей, вхід не проходить» -->
- Записка про вартість: <!-- …/blob/<SHA>/docs/cost-note.md -->
- Цифри станом на: <!-- дата, на яку перевірено ціни й ліміти -->

## Розробник

<!-- PM: заголовок «## Розробник» залиште, а все під ним (і підрозділи) замініть одним рядком «—».
     PR, файли й прогони — у вашому репо. Файли — посиланнями на SHA (…/blob/<SHA>/…).
     Знімки — файлами у вашому репо, напр. docs/ops/<що-на-знімку>.png, з замазаними токеном, chat id і секретами. -->

### D0 — умова прийняття

- `/api/health`: <!-- https://<проєкт>.vercel.app/api/health — відповідь має містити "db":"ok" і SHA вашого main -->
- Зелений прогін `smoke` на `main`: <!-- https://github.com/<я>/<team-slug>/actions/runs/N -->
- `accounts.zip` на SHA: <!-- https://github.com/<я>/<team-slug>/blob/<SHA>/accounts.zip -->
- Ролі тестових акаунтів: <!-- лише ролі й адреси: «працівник — pm-employee@example.test; …». Паролі — лише в accounts.zip -->
- Версія `next`: <!-- з npm ls next на main, 16.4.5 чи новіша, напр. next@16.4.5 -->

### Середовища й реліз (трек A)

- Гілка `staging`: <!-- https://github.com/<я>/<team-slug>/tree/staging -->
- Зміни з `main` у `staging`: <!-- PR чи merge: …/pull/N або …/commit/<sha> -->
- Змінна лише для гілки `staging`: <!-- назва і де її видно, напр. «APP_ENV — поле appEnv у /api/health на staging». Значень секретів не пишемо -->
- Проєкт Supabase для staging: <!-- назва проєкту, напр. «leave-portal-staging»; supabaseRef staging: <ref>; supabaseRef проду: <ref> — мають бути різні. Ref (reference — короткий ідентифікатор проєкту Supabase з його адреси) не секрет; ключів і рядків підключення не пишемо -->
- Міграції в базу staging через CI: <!-- CI — continuous integration, автоматичні перевірки й кроки в GitHub Actions. Прогін CI на main …/actions/runs/N: job міграцій staging друкує «Ціль міграцій: проєкт <ref> (staging)» (той самий ref, що вище), крок «Apply migrations» — success; файл …/blob/<SHA>/.github/workflows/ci.yml#L<N> -->
- Workflow smoke деплою: <!-- …/blob/<SHA>/.github/workflows/smoke.yml — адреса з client_payload.url, заголовок x-vercel-protection-bypass із секрету -->
- Зелений smoke на деплой staging: <!-- …/actions/runs/N — у журналі JSON (JavaScript Object Notation — текстовий формат даних) health з "appEnv":"staging" і supabaseRef staging -->
- Smoke на деплой одного PR-preview: <!-- PR — pull request, запит на злиття змін; …/actions/runs/N — у журналі supabaseRef staging (preview теж ходить у базу staging, а не в прод) -->
- Доступ PM до staging: <!-- спосіб (запит доступу / спільне посилання / захист вимкнено — відкриває всі preview) і де це в runbook: …/blob/<SHA>/docs/runbook.md#… -->
- Реліз `v1.0.0`: <!-- https://github.com/<я>/<team-slug>/releases/tag/v1.0.0 -->
- Нотатки за категоріями: <!-- …/blob/<SHA>/.github/release.yml або «власні категорії: …» -->
- Прод на тегу: <!-- SHA тегу v1.0.0 і SHA у /api/health — мають збігатися -->
- Приймання PM перед тегом: <!-- коментар PM «Приймання релізу (@<pm>, <дата>): …» …/pull/N#issuecomment-M або …/issues/N#issuecomment-M · або «приймання PM не отримано: запит <дата>, <посилання>» -->

### Сигнали (трек B)

- Крок алерту: <!-- …/blob/<SHA>/.github/workflows/smoke.yml#L<N> — if: failure(), токен із Secrets, без токена — попередження, а не падіння -->
- Навмисно червоний прогін: <!-- …/actions/runs/N -->
- Знімок алерту: <!-- …/blob/<SHA>/docs/ops/<…>.png — видно, що впало, середовище й посилання на прогін; токен і chat id замазано -->
- Хто отримує алерт: <!-- канал і ролі, напр. «Telegram-група команди: розробник, PM». Без chat id -->
- Трекер помилок: <!-- «Sentry» або «власний onRequestError» і файл: …/blob/<SHA>/src/instrumentation.ts (шлях може бути інший) -->
- Дані моделі в трекері: <!-- Sentry: де вимкнено збір вхідних і вихідних даних моделі …/blob/<SHA>/…#L<N> · власний onRequestError: «без заголовків, cookie і тексту запиту» + рядок коду -->
- Подія навмисної помилки: <!-- …/blob/<SHA>/docs/ops/<…>.png — маршрут і стек; preview чи staging -->
- Тестовий маршрут прибрано чи закрито: <!-- …/pull/N — прибрано або лише для ролі адміністратора -->
- Журнал `llm.call`: <!-- …/blob/<SHA>/src/lib/ai/llm-log.ts (шлях може бути інший) — поля: модель, затримка, токени, стан, fallback, версія промпту, id запиту -->
- Тест «без тексту користувача»: <!-- …/blob/<SHA>/tests/llm-log.test.ts і зелений прогін CI …/actions/runs/N -->

### Інцидент-дріл (трек C)

- Сценарій: <!-- одним рядком: що зламали на staging і як, напр. «хибне значення змінної лише для гілки staging». Без міграцій і без видалення даних, прод і база проду не зачеплені -->
- Сигнал і час: <!-- червоний smoke …/actions/runs/N чи подія трекера; поломка о HH:MM, алерт о HH:MM — від поломки до алерту N хв -->
- Профіль «лише пропонує»: <!-- …/blob/<SHA>/.claude/settings.json (правила зі зразка materials/ops-kit/.claude/settings.incident.json; окремий файл профілю в .claude/ — теж можна) — інструменти запису MCP (Model Context Protocol — спосіб дати агенту інструменти зовнішнього сервісу) Vercel і Supabase заборонені поіменно -->
- MCP-конфіг: <!-- MCP (Model Context Protocol — спосіб дати агенту інструменти зовнішнього сервісу): …/blob/<SHA>/.mcp.json — Vercel і, за потреби, Supabase з read_only=true і project_ref проєкту staging, без токенів; або «Supabase MCP не підключали» -->
- Стенограма розбору: <!-- …/blob/<SHA>/docs/incidents/<YYYY-MM-DD>-<slug>-rca.md (назва може бути інша) — дослівно, із заголовком: дата, модель, профіль, що агент міг і чого не міг -->
- Виправлення людиною, не агентом: <!-- …/pull/N або …/commit/<sha> -->
- Postmortem: <!-- …/blob/<SHA>/docs/incidents/<YYYY-MM-DD>-<slug>.md — хронологія з часом, вплив, причина з доказами, «що НЕ було причиною», дії з власником -->
- Runbook: <!-- …/blob/<SHA>/docs/runbook.md -->
- Дія з postmortem → перевірка: <!-- PR …/pull/N з посиланням на postmortem і прогін CI …/actions/runs/N, де нова перевірка працює -->
- Розбір PM: <!-- коментар PM у вашому PR …/pull/N#issuecomment-M, ваша відповідь чи зміни — або «не надходило» -->

### Ланцюг постачання (трек D)

- `dependabot.yml`: <!-- …/blob/<SHA>/.github/dependabot.yml — npm і github-actions, щотижня, cooldown, групи -->
- Dependabot запускався: <!-- PR Dependabot …/pull/N або прогін …/actions/runs/N -->
- `.npmrc`: <!-- …/blob/<SHA>/.npmrc — з min-release-age -->
- Дії за SHA: <!-- …/blob/<SHA>/.github/workflows/<…>.yml — uses: …@<повний sha> # v… -->
- Push protection і CodeQL: <!-- знімок налаштувань …/blob/<SHA>/docs/ops/<…>.png і/або перший прогін CodeQL …/actions/runs/N -->
- `next` ≥ 16.4.5: <!-- PR Dependabot (безпековий чи груповий) або ваш …/pull/N — злитий, CI зелений; версія 16.4.5 чи новіша -->

## Бонус

<!-- Необов'язково. Якщо бонусу немає — рівно «—» (і видаліть підрозділи нижче).
     Розробник: заповніть «Розробник», а «PM» — «—». PM: навпаки.
     Розробник обирає ОДИН варіант; поля іншого варіанта — «—».
     Частковий результат теж заповнюйте: що зроблено, де зупинилось і чому. -->

### Розробник

- Варіант: <!-- «резервна копія і відновлення» або «асистент чергового» -->
- Workflow: <!-- …/blob/<SHA>/.github/workflows/backup.yml або …/oncall-assistant.yml — дії за SHA -->
- Прогін: <!-- …/actions/runs/N -->
- Резервна копія: шифрування й строк зберігання: <!-- як шифрується, строк зберігання артефакту; «дамп у відкритому вигляді — ніде» · або «—» -->
- Резервна копія: тренування відновлення: <!-- прогін …/actions/runs/N з порівнянням кількості рядків і розділ «Відновлення» …/blob/<SHA>/docs/runbook.md#… · або «—» -->
- Асистент чергового: чернетка розбору: <!-- issue …/issues/N з позначкою «чернетка агента, не перевірено» · або «—» -->
- Асистент чергового: межі, ліміти й вартість: <!-- без інструментів запису і без секретів проду; модель, ліміти, оцінка вартості запуску · або «—» -->

### PM

- Що зроблено: <!-- «нотатки релізу v1.0.0 для клієнта» або «настільне тренування інциденту» -->
- Посилання: <!-- …/blob/<SHA>/docs/release-notes-v1.0.0.md або …/blob/<SHA>/docs/tabletop.md чи …/issues/N -->
- Висновок: <!-- до пів сторінки: що з'ясували і що змінили -->
