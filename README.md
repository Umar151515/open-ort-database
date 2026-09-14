# Дамп вопросов Общереспубликанского тестирования (ОРТ)

Дамп вопросов **Общереспубликанского тестирования (ОРТ)**.
Включает **9 предметов** на **русском и кыргызском** языках, а также задания на **чтение** (`base_read`), в двух форматах:

- **JSON** с ссылками на изображения по sha256
- **SQLite** база данных

> **Всего вопросов**: ~13 520
> **Заданий на чтение** (`base_read`): отдельная структура, см. ниже

> **Примечание.** Часть заданий могла быть сгенерирована с помощью LLM. В частности, **все задания на чтение на кыргызском языке** (`base_read_ky`) созданы LLM.

---

## Структура репозитория

```
ort_dump/
├── ru/                       # Русский язык
│   ├── base_math_ru.json
│   ├── subj_bio_ru.json
│   ├── subj_math_ru.json
│   ├── base_gramm_ru.json
│   ├── subj_chem_ru.json
│   ├── subj_hist_ru.json
│   ├── base_analogy_ru.json
│   ├── subj_phy_ru.json
│   ├── subj_eng_ru.json
│   └── base_read_ru.json     # Задания на чтение (особый формат)
├── ky/                       # Кыргызский язык
│   ├── base_math_ky.json
│   ├── subj_bio_ky.json
│   ├── ...
│   └── base_read_ky.json     # Полностью сгенерировано LLM
├── images/                   # Все изображения (WebP)
│   ├── 056e2b36f1af4190a87fec6f4c9c9239....webp
│   ├── a1b2c3d4e5f6....webp
│   └── ...
└── ort_bank.db               # Единая SQLite база данных
```

---

## Быстрый старт

### Работа с JSON (Python)

```python
import json

with open("ort_dump/ru/base_math_ru.json", "r", encoding="utf-8") as f:
    data = json.load(f)

q = data["questions"][0]
print(q["content"])               # текст вопроса (список блоков)
print(q["correct"])               # правильный ответ (int, 1..N)
print(q["bloom_level"])           # сложность по Блуму
print(q["explanation_content"])   # структурированное объяснение
print(q["type"])                  # тип задания (см. раздел "Типы заданий")
```

Изображения внутри JSON ссылаются на файл по его **sha256** — сам файл лежит в `images/<sha256>.webp`.

### Работа с SQLite

```python
import sqlite3, json

conn = sqlite3.connect("ort_dump/ort_bank.db")
conn.row_factory = sqlite3.Row

row = conn.execute("""
    SELECT q.*, s.name AS subject, l.code AS lang
    FROM questions q
    JOIN subjects  s ON q.subject_id  = s.id
    JOIN languages l ON q.language_id = l.id
    WHERE s.name = 'base_math' AND l.code = 'ru'
    ORDER BY RANDOM() LIMIT 1
""").fetchone()

question = {
    "bloom":       row["bloom_level"],
    "correct":     row["correct_option"],
    "content":     json.loads(row["content_json"]),
    "options":     json.loads(row["options_json"]),
    "explanation": json.loads(row["explanation_json"]),
    "type":        row["type"],
}
```

---

## 📋 Описание форматов

### JSON‑файлы обычных заданий

Каждый файл содержит объект с двумя ключами:

```json
{
  "metadata": {
    "subject": "base_math",
    "language_code": "ru"
  },
  "questions": [ ... ]
}
```

**Структура одного вопроса**:

| Поле                  | Тип            | Описание                                 |
|-----------------------|----------------|------------------------------------------|
| `content`             | `list[block]`  | Текст и картинки вопроса                 |
| `options`             | `list[option]` | Варианты ответа                          |
| `correct`             | `int`          | Номер правильного варианта (1..N)        |
| `explanation_content` | `list[block]`  | Объяснение                               |
| `bloom_level`         | `string`       | `"REMEMBER"`, `"UNDERSTAND"`, `"APPLY"`, `"ANALYZE"`, `"EVALUATE"` |
| `type`                | `int`          | Тип задания (1 или 2, см. ниже)          |

**Блок (block)** — либо текст, либо изображение:

```json
{ "type": "text",  "content": "Текст вопроса" }
```

```json
{ "type": "image", "url": "056e2b36f1af4190a87fec6f4c9c9239...." }
```

В поле `url` у изображения лежит **sha256** файла без расширения. Сам файл — `images/<sha256>.webp`.

**Вариант ответа (option)**:

```json
{
  "id": 1,
  "content": [
    { "type": "text", "content": "Текст варианта" }
  ]
}
```

`id` — целое число.

### JSON‑файлы заданий на чтение (`base_read_*.json`)

Формат отличается: в корне лежит массив `tasks`, каждый из которых содержит общий текст и пачку вопросов к нему.

```json
{
  "metadata": {
    "subject": "base_read",
    "language_code": "ru"
  },
  "tasks": [
    {
      "task_type": 1,
      "text_content": [ ... ],
      "questions": [
        {
          "content": [ ... ],
          "options": [ ... ],
          "correct": 3,
          "explanation_content": [ ... ],
          "bloom_level": "UNDERSTAND"
        }
      ]
    }
  ]
}
```

| Поле            | Тип            | Описание                                  |
|-----------------|----------------|-------------------------------------------|
| `task_type`     | `int`          | Тип задания на чтение (1 или 2, см. ниже) |
| `text_content`  | `list[block]`  | Общий текст/пассаж                        |
| `questions`     | `list[question]` | Вопросы к этому тексту                  |

---

## 🏷 Типы заданий

Колонка `type` (в JSON — ключ `"type"`) отличает обычные задания от специальных. Значения зависят от предмета:

| Предмет         | `type = 1`                              | `type = 2`                          |
|-----------------|-----------------------------------------|-------------------------------------|
| Все остальные   | обычное задание                         | —                                   |
| `base_math`     | обычное задание                         | задание на сравнение                |
| `base_analogy`  | аналогии слов                           | дополнение                          |
| `base_read`     | цельный текст                           | текст, разбитый на 2 части          |

У всех предметов, кроме перечисленных, `type` всегда равен `1`.

---

### SQLite база данных (`ort_bank.db`)

Содержит **5 таблиц**:

- **subjects** — предметы (`base_math`, `subj_bio`, …)
- **languages** — языки (`ru`, `ky`)
- **questions** — обычные вопросы (все предметы, кроме `base_read`)
- **reading_tasks** — задания на чтение (пассажи)
- **reading_questions** — вопросы к пассажам

Изображения в базе отдельно **не хранятся**: их sha256 лежит прямо внутри `content_json` / `options_json` / `explanation_json` в поле `url`, а сам файл — `images/<sha256>.webp`.

#### Таблица `questions`

| Колонка            | Тип     | Описание                                  |
|--------------------|---------|-------------------------------------------|
| `id`               | INTEGER | Автоинкремент                             |
| `subject_id`       | INTEGER | FK → subjects.id                          |
| `language_id`      | INTEGER | FK → languages.id                         |
| `bloom_level`      | TEXT    | Сложность                                 |
| `correct_option`   | INTEGER | Номер правильного варианта (1..N)         |
| `content_json`     | TEXT    | Вопрос content (JSON)                     |
| `options_json`     | TEXT    | Все варианты ответов (JSON)               |
| `explanation_json` | TEXT    | Объяснение с картинками (JSON)            |
| `type`             | INTEGER | Тип задания (по умолчанию 1)              |

Индексы: `ix_questions_subject_lang`, `ix_questions_type`.

#### Таблица `reading_tasks`

| Колонка             | Тип     | Описание                                     |
|---------------------|---------|----------------------------------------------|
| `id`                | INTEGER | Автоинкремент                                |
| `subject_id`        | INTEGER | FK → subjects.id                             |
| `language_id`       | INTEGER | FK → languages.id                            |
| `task_type`         | INTEGER | Тип задания на чтение (1 или 2)              |
| `text_content_json` | TEXT    | Общий текст/пассаж (JSON)                    |

Индекс: `ix_reading_tasks_subject_lang`.

#### Таблица `reading_questions`

| Колонка            | Тип     | Описание                                  |
|--------------------|---------|-------------------------------------------|
| `id`               | INTEGER | Автоинкремент                             |
| `reading_task_id`  | INTEGER | FK → reading_tasks.id                     |
| `bloom_level`      | TEXT    | Сложность                                 |
| `correct_option`   | INTEGER | Номер правильного варианта (1..N)         |
| `content_json`     | TEXT    | Вопрос content (JSON)                     |
| `options_json`     | TEXT    | Все варианты ответов (JSON)               |
| `explanation_json` | TEXT    | Объяснение (JSON)                         |
| `type`             | INTEGER | Тип вопроса (по умолчанию 1)              |

Индекс: `ix_reading_questions_task`.

---

## 💻 Примеры использования

### Получить все вопросы по предмету

```python
import sqlite3, json

conn = sqlite3.connect("ort_dump/ort_bank.db")

def get_questions(subject, lang):
    cur = conn.execute("""
        SELECT q.* FROM questions q
        JOIN subjects  s ON q.subject_id  = s.id
        JOIN languages l ON q.language_id = l.id
        WHERE s.name = ? AND l.code = ?
    """, (subject, lang))

    return [{
        "bloom":       row["bloom_level"],
        "correct":     row["correct_option"],
        "content":     json.loads(row["content_json"]),
        "options":     json.loads(row["options_json"]),
        "explanation": json.loads(row["explanation_json"]),
        "type":        row["type"],
    } for row in cur.fetchall()]

math_ru = get_questions("base_math", "ru")
print(f"Загружено {len(math_ru)} вопросов")
```

### Получить задание на чтение с вопросами

```python
def get_reading_tasks(subject, lang):
    cur = conn.execute("""
        SELECT t.id AS task_id, t.task_type, t.text_content_json
        FROM reading_tasks t
        JOIN subjects  s ON t.subject_id  = s.id
        JOIN languages l ON t.language_id = l.id
        WHERE s.name = ? AND l.code = ?
    """, (subject, lang))

    result = []
    for row in cur.fetchall():
        qs = conn.execute("""
            SELECT content_json, options_json, explanation_json,
                   correct_option, bloom_level, type
            FROM reading_questions
            WHERE reading_task_id = ?
        """, (row["task_id"],)).fetchall()
        result.append({
            "task_type": row["task_type"],
            "text":      json.loads(row["text_content_json"]),
            "questions": [{
                "content":     json.loads(q["content_json"]),
                "options":     json.loads(q["options_json"]),
                "explanation": json.loads(q["explanation_json"]),
                "correct":     q["correct_option"],
                "bloom":       q["bloom_level"],
                "type":        q["type"],
            } for q in qs],
        })
    return result

reading_ru = get_reading_tasks("base_read", "ru")
print(f"Загружено {len(reading_ru)} заданий на чтение")
```

### Найти файл изображения по sha256

```python
sha = "056e2b36f1af4190a87fec6f4c9c9239...."
path = f"ort_dump/images/{sha}.webp"
print(path)
```

### Прямые SQL‑запросы

```sql
-- Все вопросы по математике (русский)
SELECT * FROM questions
WHERE subject_id  = (SELECT id FROM subjects  WHERE name = 'base_math')
  AND language_id = (SELECT id FROM languages WHERE code = 'ru');

-- Количество обычных вопросов по предметам
SELECT s.name, l.code, COUNT(*) AS cnt
FROM questions q
JOIN subjects  s ON q.subject_id  = s.id
JOIN languages l ON q.language_id = l.id
GROUP BY s.name, l.code
ORDER BY cnt DESC;

-- Вопросы уровня "ANALYZE"
SELECT * FROM questions WHERE bloom_level = 'ANALYZE';

-- Задания на сравнение в математике
SELECT * FROM questions
WHERE type = 2
  AND subject_id = (SELECT id FROM subjects WHERE name = 'base_math');

-- Аналогии-дополнения
SELECT * FROM questions
WHERE type = 2
  AND subject_id = (SELECT id FROM subjects WHERE name = 'base_analogy');

-- Чтение: цельные тексты (type = 1)
SELECT * FROM reading_tasks WHERE task_type = 1;

-- Чтение: тексты, разбитые на 2 части (type = 2)
SELECT * FROM reading_tasks WHERE task_type = 2;
```

### Фильтрация по уровню сложности

```python
def get_by_bloom(subject, lang, bloom_level):
    cur = conn.execute("""
        SELECT q.* FROM questions q
        JOIN subjects  s ON q.subject_id  = s.id
        JOIN languages l ON q.language_id = l.id
        WHERE s.name = ? AND l.code = ? AND q.bloom_level = ?
    """, (subject, lang, bloom_level))
    return cur.fetchall()

hard_questions = get_by_bloom("base_math", "ru", "ANALYZE")
```

---

## 📊 Статистика

| Предмет        | Русский | Кыргызский |
|----------------|---------|------------|
| base_math      | ~1389   | ~1356      |
| subj_bio       | ~497    | ~500       |
| subj_chem      | ~499    | ~500       |
| subj_phy       | ~400    | ~400       |
| subj_hist      | ~160    | ~160       |
| base_gramm     | ~1011   | ~957       |
| base_analogy   | ~1150   | ~1198      |
| subj_eng       | ~166    | ~170       |
| subj_math      | ~1000   | ~994       |
| base_read      | ~510    | ~510       |

---

## Дисклеймер

Данный дамп предоставляется **исключительно в образовательных целях**.
Часть заданий могла быть сгенерирована с помощью LLM; в особенности это касается **всех заданий на чтение на кыргызском языке** (`base_read_ky`).
Автор не несёт ответственности за точность вопросов или возможные изменения исходных материалов.
Используйте на свой страх и риск.

---

## 🤝 Вклад

Нашли ошибку или хотите предложить улучшения? Пул-реквесты приветствуются!