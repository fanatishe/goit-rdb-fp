# goit-rdb-fp — фінальний проєкт з PostgreSQL

Фінальний проєкт курсу GoIT RDB на датасеті `infectious_cases.csv`. Реалізація розрахована на **Google Colab + PostgreSQL 16 + Psycopg 3** та повторює інфраструктуру попередніх домашніх завдань.

## Структура репозиторію

```text
goit-rdb-fp/
├── README.md
├── goit-rdb-fp.ipynb
├── sql/
│   └── final_project.sql
├── requirements.txt
├── data/
│   └── infectious_cases.csv
└── screenshots/
    └── README.md
```

## Дані

`data/infectious_cases.csv` — CSV, наданий у завданні. У файлі 10 521 рядок і 12 колонок: `Entity`, `Code`, `Year` та 9 числових показників інфекційних захворювань. Діапазон років у джерелі — 1920–2023.

Під час імпорту порожні CSV-поля завантажуються як `NULL`. Числові показники мають тип `NUMERIC`, щоб не втрачати десяткову точність.

## Встановлення PostgreSQL 16

Notebook встановлює PostgreSQL 16 з офіційного PGDG APT-репозиторію. `pgserver` не використовується: у попередніх роботах його замінили прямим встановленням сервера через APT через проблеми сумісності `pgserver==0.1.4` з новими версіями Python у Colab.

Інсталяційна комірка виконує такі кроки:

1. Встановлює `ca-certificates` та `curl`.
2. Додає `https://apt.postgresql.org/pub/repos/apt` і офіційний ключ PGDG.
3. Встановлює `postgresql-16` та `postgresql-client-16`.
4. Запускає кластер `pg_ctlcluster 16 main start`.
5. Створює PostgreSQL superuser `root`, що відповідає OS-користувачу Colab для локальної peer-аутентифікації.
6. Встановлює `psycopg[binary]` і `pandas`, перевіряє `server_version`.

Підключення виконується через Unix socket `/var/run/postgresql`:

```python
conn = psycopg.connect(
    dbname="postgres",
    user="root",
    host="/var/run/postgresql",
    connect_timeout=10,
)
```

## Запуск у Google Colab

1. Відкрийте `goit-rdb-fp.ipynb` у Google Colab.
2. Виконайте **Runtime → Restart session and run all**.
3. Коли notebook попросить, завантажте `infectious_cases.csv` з папки `data/` (якщо файл ще не доступний у середовищі Colab).
4. Дочекайтеся завершення всіх комірок. Notebook перевіряє версію PostgreSQL, кількість імпортованих рядків, кількість унікальних пар `Entity` + `Code` та відповідність кількості рядків після нормалізації.
5. Перегляньте таблиці результатів. Notebook зберігає CSV-результати у `results/`, а зображення запитів і результатів — у `screenshots/` поточної файлової системи Colab.
6. Додайте створені PNG зі `screenshots/` у цей репозиторій, щоб виконати критерій здачі щодо скриншотів.

Notebook містить усі SQL-запити та є основним відтворюваним способом виконання проєкту.

## Запуск SQL-файлу через psql

SQL-файл `sql/final_project.sql` також містить повну реалізацію проєкту та команду `\copy`. Запускати її потрібно з кореня репозиторію, оскільки шлях до CSV відносний:

```bash
psql -d postgres -U root -f sql/final_project.sql
```

Це **psql-скрипт**, а не SQL, призначений для виконання напряму через Psycopg: команда `\copy` є метакомандою клієнта `psql`. У notebook імпорт виконується еквівалентним `COPY FROM STDIN` через Psycopg 3.

## Нормалізація до 3НФ

Після імпорту створюються дві таблиці:

- `pandemic.entities` — довідник унікальних комбінацій `entity` + `code`; первинний ключ `entity_id`.
- `pandemic.infectious_indicators` — числові показники та `year`; первинний ключ `(entity_id, year)`, `entity_id` — зовнішній ключ на `entities`.

Унікальність комбінації забезпечує `UNIQUE NULLS NOT DISTINCT (entity, code)`. Це важливо, оскільки у джерелі є рядки без `Code`: різні роки однієї такої сутності мають посилатися на один і той самий `entity_id`, а не створювати окремий запис довідника для кожного року.

Контрольні значення для наданого CSV:

| Перевірка | Очікуване значення |
|---|---:|
| Рядки у вихідному CSV / `infectious_cases` | 10 521 |
| Унікальні пари `Entity` + `Code` / `entities` | 245 |
| Рядки `infectious_indicators` | 10 521 |
| Непорожні `Code` у staging | 9 393 |
| `NULL` у `Code` | 1 128 |
| Непорожні числові значення `Number_rabies` | 6 780 |
| `NULL` у `Number_rabies` | 3 741 |

## Що реалізовано за завданнями

1. Створено схему `pandemic`, імпортовано оригінальний CSV у `pandemic.infectious_cases`.
2. Створено `pandemic.entities` і `pandemic.infectious_indicators`; збережено усі 10 521 спостереження.
3. Для кожної пари `Entity` + `Code` обчислюються `AVG`, `MIN`, `MAX`, `SUM` для `number_rabies`; результат сортується за середнім у спадному порядку та обмежується 10 рядками. `AVG` та інші агрегати ігнорують SQL `NULL`.
4. Створено view `pandemic.v_infectious_cases_year_difference` з `MAKE_DATE`, `CURRENT_DATE` та `AGE`/`EXTRACT`.
5. Створено функції:
   - `pandemic.year_difference(integer)` — кількість повних років між 1 січня заданого року та поточною датою;
   - `pandemic.average_cases_per_period(numeric, numeric)` — ділить річне значення на заданий додатний дільник; `NULL` повертає `NULL`, нульовий або від’ємний дільник відхиляється.

Для функції середньої кількості захворювань notebook показує окремі розрахунки з дільниками `12`, `4` та `2`. Застосування до `number_rabies` обмежене рядками, де `number_rabies IS NOT NULL`; тип `NUMERIC` гарантує, що всі збережені значення є числовими.

## Нюанс інтерпретації

Розрахунки `annual cases / 12`, `/ 4` та `/ 2` — це математичний розподіл річного значення на однакові періоди, як вимагає умова. Це не фактичні помісячні або поквартальні вимірювання з датасету.

## Публікація в GitHub

Створіть на GitHub **публічний** репозиторій з назвою `goit-rdb-fp` (без автоматичного додавання README, щоб уникнути конфлікту), потім із кореня розпакованої папки виконайте:

```bash
git init
git add .
git commit -m "Add RDB final project"
git branch -M main
git remote add origin https://github.com/<YOUR_USERNAME>/goit-rdb-fp.git
git push -u origin main
```

Після завантаження notebook і скриншотів перевірте, що репозиторій відкривається без авторизації, та додайте його публічне посилання в LMS. Архів `ФП_ПІБ.zip` перед завантаженням до LMS перейменуйте, підставивши замість `ПІБ` власні ініціали та прізвище.
