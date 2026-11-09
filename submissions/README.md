# Здача WS11

Здача — **один PR у цей репозиторій з одним новим файлом**:

```text
submissions/<team-slug>/<github-login>.md
```

- `<team-slug>` — `property-workflows`, `therapy-scheduling`, `school-purchasing` або `leave-portal`;
- `<github-login>` — ваш логін GitHub, точно як у профілі (той самий, з якого відкрито PR);
- вміст — за [шаблоном](_template.md), усі поля заповнені, зайва для вашої ролі частина — «—».

Здають окремо: PM і розробники, у кожного — свій файл. Роль у файлі — `pm` або `dev`. Інших файлів PR не змінює: ні
README, ні `docs/`, ні `materials/`, ні чужих файлів у `submissions/`. **Строк — [[DATE:WS11-DEADLINE]]** (до 23:59 за
Києвом).

## Яке репо й SHA вказати

Кожен здає **своє** репо — те саме, що у WS6–WS10:

| Роль | «Ваше репо» | «Репо вимог PM» |
|---|---|---|
| `pm` | репо вимог `https://github.com/<pm>/<team-slug>-spec` | те саме |
| `dev` | своє репо `https://github.com/<dev>/<team-slug>` | репо вимог PM вашої команди |

Конвеєр перевірки читає «Ваше репо» **на цьому коміті**, тому коміт має бути запушений, а SHA — повний (40 символів):
коміт `main` вашого репо на момент здачі. Реліз `v1.0.0`, прогони CI й теги конвеєр теж читає з вашого репо.

- **Розробник** (PowerShell чи Git Bash):

  ```bash
  git fetch origin
  git rev-parse origin/main
  ```

- **PM (без git):** на сторінці репо вимог відкрийте історію комітів `main` і скопіюйте повний SHA останнього коміту
  кнопкою копіювання біля нього (точну назву перевірте в інтерфейсі). Короткий SHA з 7 символів не підходить.

**Staging-URL (розробник)** — домен гілки `staging` у Vercel, `https://….vercel.app`, **без параметрів після «?»**. Як
PM відкриває staging (запит доступу, спільне посилання чи вимкнений захист — він відкриває всі preview), записано у вашому runbook.
Секрет обходу захисту (`x-vercel-protection-bypass`) у файл не потрапляє ні окремо, ні в адресі: він відкриває
staging будь-кому і живе лише в GitHub Secrets.

## Посилання — у якому вигляді

| Що | Де | Вигляд |
|---|---|---|
| PR | ваше репо | `https://github.com/<ваш-логін>/<repo>/pull/N` |
| Файл на SHA (`smoke.yml`, `dependabot.yml`, `.npmrc`, `runbook.md`, postmortem, стенограма, `accounts.zip`) | ваше репо | `https://github.com/<ваш-логін>/<repo>/blob/<SHA>/docs/incidents/<YYYY-MM-DD>-<slug>.md` |
| Рядок файлу (крок алерту) | ваше репо | `…/blob/<SHA>/.github/workflows/smoke.yml#L<N>` |
| Знімок (алерт, подія трекера, налаштування безпеки) | ваше репо | `…/blob/<SHA>/docs/ops/<що-на-знімку>.png` — токен, chat id і секрети замазано |
| Прогін CI (`smoke`, `CI`, CodeQL, Dependabot, бонусний workflow) | ваше репо | `https://github.com/<ваш-логін>/<repo>/actions/runs/N` |
| Гілка `staging` | ваше репо | `https://github.com/<ваш-логін>/<repo>/tree/staging` |
| Реліз `v1.0.0` | ваше репо | `https://github.com/<ваш-логін>/<repo>/releases/tag/v1.0.0` |
| Коментар у PR чи issue | ваше репо | `…/pull/N#issuecomment-M`, `…/issues/N#issuecomment-M` |
| `/api/health` | ваш прод | `https://<проєкт>.vercel.app/api/health` |
| Staging | ваш Vercel | `https://<домен гілки staging>.vercel.app` — без «?» і параметрів |
| Операційні вимоги, повідомлення клієнту, чекліст demo day, записка про вартість | репо вимог PM | `https://github.com/<pm>/<team-slug>-spec/blob/<SHA>/docs/operations.md` (або `…/issues/N`) |
| Підпис PM «Приймання релізу», розбір postmortem і runbook, issue «Відхилення від вимог» | репо розробника | `https://github.com/<dev>/<team-slug>/pull/N#issuecomment-M`, `…/issues/N` |
| Бонус: workflow, прогін, issue з чернеткою агента | ваше репо | `…/blob/<SHA>/.github/workflows/backup.yml`, `…/actions/runs/N`, `…/issues/N` |

**Паролів, ключів, токенів і chat id у файлі немає ніде.** Акаунти — лише адреси `*@example.test`. Пароль від
`accounts.zip` теж ніде не пишемо. Значення `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`, `VERCEL_AUTOMATION_BYPASS_SECRET`,
`SENTRY_AUTH_TOKEN`, `BACKUP_PASSPHRASE`, `SUPABASE_DB_URL` — тим паче: лише назву змінної. Адрес Google Таблиць у файл
не копіюємо.

## Покроково: розробник

PowerShell (Git Bash — так само, лише `cp` замість `Copy-Item`):

```powershell
gh repo fork koldovsky/2026-quitcode-11-devops-operations-hw --clone
cd 2026-quitcode-11-devops-operations-hw
git switch -c ws11/<github-login>
Copy-Item submissions/_template.md submissions/<team-slug>/<github-login>.md
# заповніть файл
git add submissions/<team-slug>/<github-login>.md
git commit -m "WS11 submission: <team-slug>/<github-login>"
git push -u origin ws11/<github-login>
gh pr create --repo koldovsky/2026-quitcode-11-devops-operations-hw --web
```

Fork з WS10 тут не підходить: це інший репозиторій.

## Покроково: PM (у браузері)

1. Відкрийте [шаблон здачі](_template.md) в цьому репозиторії, перегляньте сирий текст (кнопка Raw; точну назву
   перевірте в інтерфейсі) і скопіюйте його повністю.
2. Перейдіть у теку `submissions/<team-slug>/` цього репозиторію → Add file → Create new file (точні назви перевірте
   в інтерфейсі). GitHub попередить, що права запису у вас немає, і запропонує працювати у вашій копії (fork) — це
   правильно.
3. Ім'я файлу — `<github-login>.md`, вміст — скопійований шаблон. Заповніть поля просто в редакторі. Приймання
   релізу — по рядку на кожного розробника команди: staging, результат сценарію і посилання на ваш коментар-підпис у
   його репо. Розробник без «реліз-кандидат на staging» — рядок «не прийнято: <причина, дата>».
4. Внизу оберіть створення нової гілки й запропонуйте зміну (Propose new file → Create pull request; точні назви
   перевірте в інтерфейсі).

## Далі — для всіх

Fork цього репозиторію потрібен **лише для здачі**. Ваша робота живе у вашому репо вимог чи репо розробника.

1. Відкриється сторінка нового PR з шаблоном опису. **Заголовок — `WS11: Ім'я Прізвище`.** У шаблоні заповніть
   **«Ім'я та Прізвище»** (обов'язково: сертифікат виписується на реальне ім'я), роль і Definition of Done своєї
   ролі. Розробник відкриває PR лише з `--web` і не додає `--fill`: з ним опис береться з комітів, і шаблон не
   підставляється.
2. Цей репозиторій має менше 10 зірок, тому CodeRabbit не почне сам. Напишіть у PR **окремим коментарем**
   `@coderabbitai review`.
3. CodeRabbit перевіряє лише файл здачі: шлях, заповнені поля, SHA, вигляд посилань, відсутність паролів, токенів і
   chat id і що нічого іншого не змінено. Виправте зауваження в тій самій гілці: розробник — новим комітом і push, PM
   — правкою файлу у своїй копії (іконка олівця на сторінці файлу; точну назву перевірте в інтерфейсі).

Ім'я та прізвище — лише в PR (заголовок і опис), не у файлі: файли цього репозиторію публічні.
