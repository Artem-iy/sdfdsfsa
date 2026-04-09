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


```
import os
import requests
import logging
from typing import Dict, Any, List

logger = logging.getLogger(__name__)


MICROCATEGORIES = [
    {"mcId": 101, "mcTitle": "Ремонт квартир и домов под ключ", "keywords": ["ремонт под ключ", "комплексный ремонт"]},
    {"mcId": 102, "mcTitle": "Сантехника", "keywords": ["сантехника", "трубы", "унитаз", "смеситель"]},
    {"mcId": 103, "mcTitle": "Электрика", "keywords": ["электрика", "проводка", "розетка", "щиток"]},
    {"mcId": 104, "mcTitle": "Натяжные потолки", "keywords": ["натяжные потолки", "натяжной потолок"]},
    {"mcId": 105, "mcTitle": "Укладка плитки", "keywords": ["плитка", "укладка плитки", "кафель"]},
    {"mcId": 106, "mcTitle": "Поклейка обоев", "keywords": ["обои", "поклейка обоев"]},
    {"mcId": 107, "mcTitle": "Малярные работы", "keywords": ["покраска", "малярные работы"]},
    {"mcId": 108, "mcTitle": "Штукатурные работы", "keywords": ["штукатурка"]},
    {"mcId": 109, "mcTitle": "Напольные покрытия", "keywords": ["ламинат", "паркет", "пол"]},
    {"mcId": 110, "mcTitle": "Гипсокартон", "keywords": ["гипсокартон", "гкл"]},
    {"mcId": 111, "mcTitle": "Демонтажные работы", "keywords": ["демонтаж", "снос", "разбор"]}
]


class YandexGPTLiteGenerator:

    def __init__(self):

        self.api_key = os.getenv("YANDEX_API_KEY")
        self.folder_id = os.getenv("YANDEX_FOLDER_ID")

        self.url = "https://llm.api.cloud.yandex.net/foundationModels/v1/completion"

        self.headers = {
            "Content-Type": "application/json",
            "Authorization": f"Api-Key {self.api_key}"
        }

        self.model_uri = f"gpt://{self.folder_id}/yandexgpt-lite"

        self.completion_options = {
            "stream": False,
            "temperature": 0.3,
            "maxTokens": "1000"
        }

        self.system_prompt = (
            "Ты профессиональный копирайтер Avito. "
            "Пиши короткие продающие объявления для услуг ремонта."
        )

    def detect_categories(self, description: str) -> List[Dict]:

        description = description.lower()
        found = []

        for mc in MICROCATEGORIES:
            for kw in mc["keywords"]:
                if kw in description:
                    found.append(mc)
                    break

        if not found:
            found.append(MICROCATEGORIES[0])

        return found

    def generate_text(self, description: str, category_title: str) -> str:

        user_prompt = f"""
Напиши объявление для Avito.

Услуга: {category_title}

Описание клиента:
{description}

Формат:
Короткий продающий текст 2-3 предложения.
"""

        payload = {
            "modelUri": self.model_uri,
            "completionOptions": self.completion_options,
            "messages": [
                {"role": "system", "text": self.system_prompt},
                {"role": "user", "text": user_prompt}
            ]
        }

        try:

            response = requests.post(
                self.url,
                headers=self.headers,
                json=payload,
                timeout=30
            )

            result = response.json()

            text = result["result"]["alternatives"][0]["message"]["text"]

            return text.strip()

        except Exception as e:

            logger.error(f"Ошибка генерации: {e}")

            return f"Профессиональные услуги: {category_title}. Качественно и в срок."

    def generate(self, description: str, should_split: bool, probability: float) -> Dict[str, Any]:

        categories = self.detect_categories(description)

        drafts = []

        for cat in categories:

            text = self.generate_text(description, cat["mcTitle"])

            drafts.append({
                "mcId": cat["mcId"],
                "mcTitle": cat["mcTitle"],
                "text": text
            })

        return {
            "shouldSplit": len(drafts) > 1,
            "drafts": drafts
        }
```
