Дамп вопросов **Общереспубликанского тестирования (ОРТ)**.
Включает **9 предметов** на **русском и кыргызском** языках в двух форматах:
- **JSON** с ссылками на изображения по sha256
- **SQLite** база данных

> **Всего вопросов**: ~12 500

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
│   └── subj_eng_ru.json
├── ky/                       # Кыргызский язык
│   ├── base_math_ky.json
│   ├── subj_bio_ky.json
│   └── ...
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
}
```

---

## 📋 Описание форматов

### JSON‑файлы

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
| `bloom_level`         | `string`       | `"UNDERSTAND"`, `"APPLY"`, `"ANALYZE"`   |

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

### SQLite база данных (`ort_bank.db`)

Содержит **5 таблиц**:

- **subjects** — предметы (`base_math`, `subj_bio`, …)
- **languages** — языки (`ru`, `ky`)
- **questions** — все вопросы
- **images** — все изображения (только sha256)
- **question_images** — связь вопросов с изображениями

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

#### Таблица `images`

| Колонка  | Тип     | Описание                                     |
|----------|---------|----------------------------------------------|
| `id`     | INTEGER | Автоинкремент                                |
| `sha256` | TEXT    | Хэш файла; имя файла — `images/<sha256>.webp` |

#### Таблица `question_images`

| Колонка       | Тип     | Описание                       |
|---------------|---------|--------------------------------|
| `question_id` | INTEGER | FK → questions.id              |
| `image_id`    | INTEGER | FK → images.id                 |
| `field_name`  | TEXT    | `"content"` по умолчанию       |

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
        "bloom":       row[3],
        "correct":     row[4],
        "content":     json.loads(row[5]),
        "options":     json.loads(row[6]),
        "explanation": json.loads(row[7]),
    } for row in cur.fetchall()]

math_ru = get_questions("base_math", "ru")
print(f"Загружено {len(math_ru)} вопросов")
```

### Найти файл изображения по sha256

```python
import sqlite3

conn = sqlite3.connect("ort_dump/ort_bank.db")

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

-- Количество вопросов по предметам
SELECT s.name, l.code, COUNT(*) AS cnt
FROM questions q
JOIN subjects  s ON q.subject_id  = s.id
JOIN languages l ON q.language_id = l.id
GROUP BY s.name, l.code
ORDER BY cnt DESC;

-- Вопросы уровня "ANALYZE"
SELECT * FROM questions WHERE bloom_level = 'ANALYZE';

-- Вопросы с изображениями в content
SELECT DISTINCT q.id
FROM questions q
JOIN question_images qi ON qi.question_id = q.id
WHERE qi.field_name = 'content';
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

---

## Дисклеймер

Данный дамп предоставляется **исключительно в образовательных целях**.
Автор не несёт ответственности за точность вопросов или возможные изменения исходных материалов.
Используйте на свой страх и риск.

---

## 🤝 Вклад

Нашли ошибку или хотите предложить улучшения?
Пул-реквесты приветствуются!