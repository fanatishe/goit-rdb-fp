# Запуск проєкту

## Google Colab

1. Завантажте `goit-rdb-fp.ipynb` у [Google Colab](https://colab.research.google.com/) через **File → Upload notebook**.
2. У меню виберіть **Runtime → Restart session and run all**.
3. Коли з’явиться діалог завантаження даних, виберіть файл `data/infectious_cases.csv`.
4. Дочекайтеся завершення всіх комірок. Для запуску потрібне підключення до інтернету: notebook автоматично встановлює PostgreSQL 16 та необхідні Python-бібліотеки.

Після завершення роботи файли результатів і зображення будуть збережені в каталогах `results/` та `screenshots/` середовища Colab. Архів із цими файлами створюється в кінці notebook.

## Альтернативний запуск через `psql`

Для цього способу потрібні встановлений і запущений PostgreSQL 16 та клієнт `psql`.

1. Відкрийте термінал у кореневому каталозі репозиторію `goit-rdb-fp/`.
2. Переконайтеся, що доступні база даних `postgres` і роль PostgreSQL `root`.
3. Виконайте команду:

```bash
psql -d postgres -U root -f sql/final_project.sql
```

Запускайте команду саме з кореневого каталогу репозиторію: SQL-файл завантажує `data/infectious_cases.csv` за відносним шляхом.
