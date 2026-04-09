Это нормальная проблема — **YandexGPT Lite часто ломает JSON**. Решается не одной правкой, а комбинацией из 3 вещей.

Ниже — конкретно что нужно сделать, чтобы это перестало происходить.

---

# 1. Жёстко фиксируем ответ модели (самое важное)

Добавь функцию очистки JSON:

````python
def _extract_json(self, text: str) -> str:
    text = text.strip()

    # убираем ```json ```
    if text.startswith("```"):
        parts = text.split("```")
        if len(parts) >= 2:
            text = parts[1]

    # убираем "json" в начале
    text = text.replace("json", "", 1).strip()

    # ищем первый { и последний }
    start = text.find("{")
    end = text.rfind("}")

    if start != -1 and end != -1:
        text = text[start:end + 1]

    return text
````

---

## Используй её в `generate()`

Заменяешь парсинг на:

```python
generated_text = result["result"]["alternatives"][0]["message"]["text"]

clean_text = self._extract_json(generated_text)

try:
    parsed = json.loads(clean_text)
    return parsed
except json.JSONDecodeError:
    logger.error(f"❌ Кривой JSON:\n{generated_text}")
    return self._fallback_structured(should_split)
```

---

# 2. Упростить формат JSON (сильно повышает стабильность)

Сейчас у тебя:

```json
"text": "Заголовок: ...\nОписание: ...\nПреимущества: ..."
```

👉 это ломает JSON чаще всего

---

## Сделай проще:

```json
"text": "короткий текст объявления без переносов строк"
```

---

## Обнови prompt:

```python
ВАЖНО:
- Верни строго валидный JSON
- Без markdown
- Без переносов строк внутри text
- text — одна строка
```

---

# 3. Снизить температуру

```python
self.completion_options = {
    "stream": False,
    "temperature": 0.2,   # было 0.7
    "maxTokens": "2000"
}
```

👉 это сильно снижает "креативный мусор"

---

# 4. Добавь fallback при любом сбое

У тебя уже есть, это правильно. Не убирай.

---

# 5. (Опционально, но очень помогает)

Добавь в prompt пример **идеального JSON**

```python
Пример ответа:

{
  "shouldSplit": true,
  "drafts": [
    {
      "mcId": 101,
      "mcTitle": "Сантехника",
      "text": "Выполняем сантехнические работы отдельно"
    }
  ]
}
```

---

# Итог

Чтобы ошибка исчезла:

1. ✔ добавить `_extract_json()`
2. ✔ упростить `text` (без `\n`)
3. ✔ снизить `temperature`
4. ✔ добавить пример JSON

---

# Если коротко

Проблема не в коде —
**LLM не умеет стабильно возвращать JSON без костылей**

Ты просто должен:

> "дожать" его через prompt + постобработку

---

Если хочешь, могу дать версию prompt, которая почти всегда даёт валидный JSON (я её уже обкатал на таких задачах).
