```python
import logging
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List

from services.yandex_generator import YandexGPTLiteGenerator

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(
    title="Avito Draft Generator API",
    description="Генерация черновиков объявлений с разбиением по микрокатегориям",
    version="1.0"
)


class DraftRequest(BaseModel):
    description: str
    should_split: bool
    probability: float


class Draft(BaseModel):
    mcId: int
    mcTitle: str
    text: str


class DraftResponse(BaseModel):
    shouldSplit: bool
    drafts: List[Draft]


try:
    generator = YandexGPTLiteGenerator()
except Exception as e:
    logger.error(f"Ошибка инициализации генератора: {e}")
    generator = None


@app.post("/generate-draft", response_model=DraftResponse)
def generate_draft(request: DraftRequest):
    if generator is None:
        raise HTTPException(status_code=500, detail="Генератор не инициализирован")

    try:
        result = generator.generate(
            description=request.description,
            should_split=request.should_split,
            probability=request.probability
        )

        if not isinstance(result, dict):
            raise ValueError("Generator returned non-dict")

        if "shouldSplit" not in result or "drafts" not in result:
            raise ValueError("Invalid response structure")

        if not isinstance(result["drafts"], list):
            raise ValueError("drafts must be a list")

        return result

    except Exception as e:
        logger.error(f"Ошибка генерации: {str(e)}")
        raise HTTPException(status_code=500, detail="Ошибка генерации черновика")


@app.get("/health")
def health():
    return {"status": "ok"}
```
