import os
import requests
import logging
import json
from typing import Dict, Any

logger = logging.getLogger(__name__)


class YandexGPTLiteGenerator:
    """YandexGPT Lite через REST API"""

    def __init__(self):
        self.api_key = os.getenv("YANDEX_API_KEY")
        self.folder_id = os.getenv("YANDEX_FOLDER_ID")

        if not self.api_key:
            raise ValueError("❌ Необходимо указать YANDEX_API_KEY")
        if not self.folder_id:
            raise ValueError("❌ Необходимо указать YANDEX_FOLDER_ID")

        self.url = "https://llm.api.cloud.yandex.net/foundationModels/v1/completion"
        self.headers = {
            "Content-Type": "application/json",
            "Authorization": f"Api-Key {self.api_key}"
        }

        self.model_uri = f"gpt://{self.folder_id}/yandexgpt-lite"

        self.completion_options = {
            "stream": False,
            "temperature": 0.3,
            "maxTokens": "2000"
        }

        self.system_prompt = (
            "Ты профессиональный копирайтер, специализирующийся на создании "
            "объявлений для Avito в категории 'Ремонт и строительство'. "
            "Возвращай только JSON без лишнего текста."
        )

        logger.info(f"✅ YandexGPT Lite инициализирован")

    def generate(self, description: str, should_split: bool, probability: float) -> Dict[str, Any]:
        """
        Генерация структурированного ответа под требования кейса
        """

        repair_type = "отдельные виды работ" if should_split else "комплексный ремонт под ключ"

        user_prompt = f"""
Сгенерируй черновики объявлений для Avito.

Информация от заказчика:
{description}

Тип услуги: {repair_type}
Уверенность классификации: {probability:.0%}

ВАЖНО:
- Если услуги можно разделить — создай 2-3 отдельных объявления
- Если нельзя — создай одно
- Верни ТОЛЬКО JSON

Формат ответа:

{{
  "shouldSplit": true/false,
  "drafts": [
    {{
      "mcId": число,
      "mcTitle": "название услуги",
      "text": "Заголовок: ...\\nОписание: ...\\nПреимущества: ..."
    }}
  ]
}}
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

            if response.status_code != 200:
                logger.error(f"❌ Ошибка API: {response.status_code}")
                logger.error(response.text)
                return self._fallback_structured(should_split)

            result = response.json()
            generated_text = result["result"]["alternatives"][0]["message"]["text"]

            logger.debug(f"RAW RESPONSE: {generated_text}")

            try:
                parsed = json.loads(generated_text)
                return parsed
            except json.JSONDecodeError:
                logger.error("❌ Модель вернула невалидный JSON")
                return self._fallback_structured(should_split)

        except requests.exceptions.Timeout:
            logger.error("❌ Таймаут")
            return self._fallback_structured(should_split)

        except requests.exceptions.RequestException as e:
            logger.error(f"❌ Сетевая ошибка: {str(e)}")
            return self._fallback_structured(should_split)

        except Exception as e:
            logger.error(f"❌ Неизвестная ошибка: {str(e)}")
            return self._fallback_structured(should_split)

    def _fallback_structured(self, should_split: bool) -> Dict[str, Any]:
        """
        Запасной ответ в формате кейса
        """
        if should_split:
            return {
                "shouldSplit": True,
                "drafts": [
                    {
                        "mcId": 101,
                        "mcTitle": "Сантехника",
                        "text": "Заголовок: Сантехнические работы\nОписание: Выполняю сантехнические работы отдельно.\nПреимущества: Опыт, Гарантия, Быстро"
                    },
                    {
                        "mcId": 102,
                        "mcTitle": "Электрика",
                        "text": "Заголовок: Электромонтажные работы\nОписание: Выполняю электромонтаж отдельно.\nПреимущества: Надежно, Качественно, В срок"
                    }
                ]
            }
        else:
            return {
                "shouldSplit": False,
                "drafts": [
                    {
                        "mcId": 201,
                        "mcTitle": "Ремонт под ключ",
                        "text": "Заголовок: Ремонт под ключ\nОписание: Выполняем полный комплекс ремонтных работ.\nПреимущества: Под ключ, Гарантия, Опыт"
                    }
                ]
            }

    def set_temperature(self, temperature: float):
        self.completion_options["temperature"] = max(0.0, min(1.0, temperature))
        logger.info(f"🌡 Температура: {temperature}")

    def set_max_tokens(self, max_tokens: int):
        self.completion_options["maxTokens"] = str(max_tokens)
        logger.info(f"📏 maxTokens: {max_tokens}")


# Пример использования
if __name__ == "__main__":
    from dotenv import load_dotenv

    load_dotenv()

    generator = YandexGPTLiteGenerator()

    test_description = "Нужно поклеить обои и отдельно сделать электрику"

    result = generator.generate(
        description=test_description,
        should_split=True,
        probability=0.85
    )

    print(json.dumps(result, ensure_ascii=False, indent=2))
