Дамп вопросов **Общереспубликанского тестирования (ОРТ)**.  
Включает **9 предметов** на **русском и кыргызском** языках в двух форматах:  
- **JSON** с локальными изображениями
- **SQLite** база данных

> **Всего вопросов**: ~12 507

---

## Структура репозитория

```
ort_dump/
├── ru/                          # Русский язык
│   ├── math_ru.json
│   ├── bio_ru.json
│   ├── mathS_ru.json
│   ├── gramm_ru.json
│   ├── chem_ru.json
│   ├── hist_ru.json
│   ├── analogy_ru.json
│   ├── phy_ru.json
│   └── Eng_ru.json
├── ky/                          # Кыргызский язык
│   ├── math_ky.json
│   ├── bio_ky.json
│   └── ...
├── images/                      # Все изображения (png, jpg)
│   ├── 056e2b36-1faf-4190-a87f-ec6f4c9c9239.png
│   ├── a1b2c3d4e5f6...jpg
│   └── ...
└── ort_bank.db                  # Единая SQLite база данных
```

---

## Быстрый старт

### Работа с JSON (Python)
```python
import json

with open("ort_dump/ru/math_ru.json", "r", encoding="utf-8") as f:
    data = json.load(f)

# Пример: первый вопрос
q = data["questions"][0]
print(q["content"])              # текст вопроса (список блоков)
print(q["correct"])              # правильный ответ ("1"-"4")
print(q["bloom_level"])          # сложность по Блуму
print(q["explanation_content"])  # структурированное объяснение
```

Изображения внутри JSON имеют относительные пути вида `images/...png` — они находятся в папке `images/`.

### Работа с SQLite

```python
import sqlite3, json

conn = sqlite3.connect("ort_dump/ort_bank.db")
conn.row_factory = sqlite3.Row

# Случайный вопрос по математике на русском
row = conn.execute("""
    SELECT q.*, s.name AS subject, l.code AS lang
    FROM questions q
    JOIN subjects s ON q.subject_id = s.id
    JOIN languages l ON q.language_id = l.id
    WHERE s.name = 'math' AND l.code = 'ru'
    ORDER BY RANDOM() LIMIT 1
""").fetchone()

question = {
    "bloom": row["bloom_level"],
    "correct": row["correct_option"],
    "content": json.loads(row["content_json"]),
    "options": json.loads(row["options_json"]),
    "explanation": json.loads(row["explanation_json"])
}
```

---

## 📋 Описание форматов

### JSON‑файлы

Каждый файл содержит объект с двумя ключами:

```json
{
  "metadata": {
    "subject": "math",
    "language_code": "ru"
  },
  "questions": [ ... ]
}
```

**Структура одного вопроса**:

| Поле                  | Тип              | Описание |
|-----------------------|------------------|----------|
| `content`             | `list[block]`    | Текст и картинки вопроса |
| `options`             | `list[option]`   | Четыре варианта ответа |
| `correct`             | `string`         | Правильный ответ (`"1"`, `"2"`, `"3"`, `"4"`) |
| `explanation_content` | `list[block]`    | Объяснение |
| `bloom_level`         | `string`         | Уровень: `"UNDERSTAND"`, `"APPLY"`, `"ANALYZE"` |

**Блок (block)**:  
```json
{ "type": "text", "content": "Текст вопроса" }
```
или  
```json
{ "type": "image", "url": "images/имя_файла.png" }
```

**Вариант ответа (option)**:
```json
{
  "id": "1",
  "content": [
    { "type": "text", "content": "Текст варианта" }
  ]
}
```

### SQLite база данных (`ort_bank.db`)

Содержит **3 таблицы**:  
- **subjects** – предметы (`math`, `bio`, …)  
- **languages** – языки (`ru`, `ky`)  
- **questions** – все вопросы  

#### Таблица `questions`

| Колонка              | Тип      | Описание |
|----------------------|----------|----------|
| `id`                 | INTEGER  | Автоинкремент |
| `subject_id`         | INTEGER  | FK → subjects.id |
| `language_id`        | INTEGER  | FK → languages.id |
| `bloom_level`        | TEXT     | Сложность |
| `correct_option`     | TEXT     | Правильный ответ (1-4) |
| `content_json`       | TEXT     | Вопрос content (JSON) |
| `options_json`       | TEXT     | Все варианты ответов (JSON) |
| `explanation_json`   | TEXT     | Объяснение с картинками (JSON) |

---

## 💻 Примеры использования

### Получить все вопросы по предмету

```python
import sqlite3, json

conn = sqlite3.connect("ort_dump/ort_bank.db")

def get_questions(subject, lang):
    cur = conn.execute("""
        SELECT q.* FROM questions q
        JOIN subjects s ON q.subject_id = s.id
        JOIN languages l ON q.language_id = l.id
        WHERE s.name = ? AND l.code = ?
    """, (subject, lang))
    
    return [{
        "bloom": row[3],
        "correct": row[4],
        "content": json.loads(row[5]),
        "options": json.loads(row[6]),
        "explanation": json.loads(row[7])
    } for row in cur.fetchall()]

math_ru = get_questions("math", "ru")
print(f"Загружено {len(math_ru)} вопросов")
```

### Прямые SQL‑запросы

```sql
-- Все вопросы по математике (русский)
SELECT * FROM questions
WHERE subject_id = (SELECT id FROM subjects WHERE name = 'math')
  AND language_id = (SELECT id FROM languages WHERE code = 'ru');

-- Количество вопросов по предметам
SELECT s.name, l.code, COUNT(*) AS cnt
FROM questions q
JOIN subjects s ON q.subject_id = s.id
JOIN languages l ON q.language_id = l.id
GROUP BY s.name, l.code
ORDER BY cnt DESC;

-- Вопросы уровня "ANALYZE"
SELECT * FROM questions WHERE bloom_level = 'ANALYZE';

-- Поиск вопросов с изображениями в content
SELECT id FROM questions 
WHERE content_json LIKE '%"type": "image"%';
```

### Фильтрация по уровню сложности

```python
def get_by_bloom(subject, lang, bloom_level):
    cur = conn.execute("""
        SELECT q.* FROM questions q
        JOIN subjects s ON q.subject_id = s.id
        JOIN languages l ON q.language_id = l.id
        WHERE s.name = ? AND l.code = ? AND q.bloom_level = ?
    """, (subject, lang, bloom_level))
    return cur.fetchall()

hard_questions = get_by_bloom("math", "ru", "ANALYZE")
```

---

## 📊 Статистика

| Предмет | Русский | Кыргызский |
|---------|---------|------------|
| math    | ~1389   | ~1356      |
| bio     | ~497   | ~500      |
| chem    | ~499   | ~500      |
| phy     | ~400   | ~400      |
| hist    | ~160   | ~160      |
| gramm   | ~1011   | ~957      |
| analogy | ~1150   | ~1198      |
| Eng     | ~166   | ~170      |
| mathS   | ~1000   | ~994      |

---

## Дисклеймер

Данный дамп предоставляется **исключительно в образовательных целях**.  
Автор не несёт ответственности за точность вопросов или возможные изменения исходных материалов.  
Используйте на свой страх и риск.

---

## 🤝 Вклад

Нашли ошибку или хотите предложить улучшения?  
Пул-реквесты приветствуются!
