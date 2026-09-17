# AI Engineer — требования к команде и мои обязательства

Документ фиксирует, какие данные, интерфейсы и продуктовые определения нужны AI Engineer от команды, и что AI Engineer должен предоставить остальным участникам.

## Быстрая навигация

**Кратко:** [что мне нужно от команды](#short-team) · [что я должен команде](#short-ai)

**По ролям:** [Backend](#backend) · [Model / Infra](#infra) · [Product / Scenario](#product) · [Frontend](#frontend)

---

<a id="backend"></a>
# 1. Что мне нужно от Backend

Backend должен предоставить стабильные контракты для работы с переговорной сессией.

Необходимые сущности и данные:

- `session_id`
- `turn_id`
- `ScenarioConfig`
- текущий `SessionState`
- Transcript / conversation history
- результат STT текущей реплики либо способ его получить
- механизм получения актуального состояния сессии
- механизм сохранения результатов AI
- механизм завершения сессии

Backend также должен определить:

- кто создаёт и завершает session
- кто является `source of truth` для `SessionState`
- кто применяет `StateUpdate`
- кто хранит Transcript
- порядок сохранения user/opponent turns
- idempotency по `turn_id`
- timeout policy
- retry policy
- формат ошибок
- поведение при недоступности модели
- поведение при invalid structured output

Backend является `source of truth` для состояния сессии и её persistence.

---

<a id="infra"></a>
# 2. Что мне нужно от Model / Infra

Нужны стабильные interfaces / endpoints для:

- Whisper STT
- Qwen Opponent
- Qwen Judge
- Qwen TTS
- Embedding Model

По каждому endpoint должны быть известны:

- request format
- response format
- context window
- max output tokens
- timeout
- error format
- streaming support
- structured output / JSON mode
- возможность задавать system prompt
- возможность задавать temperature / top_p
- возможность включать / выключать reasoning на уровне отдельного запроса

Нужны следующие режимы работы:

### Guard

- Qwen
- reasoning OFF
- low temperature
- structured output

### Opponent

- Qwen
- reasoning OFF
- structured output

### Turn Analyzer

- Qwen
- reasoning OFF
- structured output

### Preparation Coach

- Qwen
- reasoning ON

### Final Judge

- Qwen
- reasoning ON
- structured output

Переключение reasoning не должно требовать перезапуска модели.

---

<a id="product"></a>
# 3. Что мне нужно от Product / Scenario Team

Для каждого сценария нужны структурированные данные.

## Shared Context

Информация, известная обеим сторонам:

- тема переговоров
- предмет переговоров
- роли сторон
- общая ситуация
- известные обеим сторонам условия
- tone
- difficulty

## Player Private Context

Информация, известная только пользователю:

- цель пользователя
- приоритеты
- внутренние ограничения
- внутренняя информация пользователя

При наличии в сценарии:

- BATNA
- reservation point
- допустимые уступки
- quantitative constraints

## Opponent Private Context

Информация, известная только AI-оппоненту:

- цель оппонента
- приоритеты
- hidden interests
- внутренние ограничения
- информация, которую нельзя раскрывать пользователю

При наличии в сценарии:

- BATNA
- reservation point
- допустимые уступки
- quantitative constraints

Также для сценария должны быть определены:

- условия успеха
- условия провала
- возможные исходы
- ограничения переговоров

---

# 4. Judge Rubric

Product должен определить систему оценки переговоров.

Минимальные категории:

- goal achievement
- discovery
- questions
- argumentation
- concessions
- relationship
- closing

Для каждой категории должны быть определены:

- диапазон score
- признаки сильного поведения
- признаки среднего поведения
- признаки слабого поведения
- типичные ошибки

Final Judge не должен самостоятельно придумывать критерии оценки.

---

<a id="frontend"></a>
# 5. Что мне нужно от Frontend

Мне нужен список AI-данных, которые реально будут использоваться интерфейсом.

Во время переговоров потенциально:

- opponent text
- opponent audio
- emotion
- negotiation stage
- reaction indicators

После переговоров:

- overall score
- skill scores
- strengths
- mistakes
- evidence
- better responses
- recommendations
- plan vs reality

Frontend не должен парсить произвольный текст LLM для получения структуры.

AI должен возвращать structured data.

---

# 6. Что я должен предоставить команде

Я определяю структуру и семантику основных AI-объектов:

- `GuardResult`
- `OpponentResult`
- `StateUpdate`
- `StrategyPlan`
- `TurnAnalysis`
- `JudgeResult`

Для каждого объекта должны быть описаны:

- required fields
- optional fields
- types
- enum values
- семантика каждого поля

---

# 7. GuardResult

Минимально:

- `allowed`
- `category`
- `reason_code`

Минимальные категории:

- `normal`
- `prompt_injection`
- `jailbreak`
- `private_information_request`
- `unsafe`
- `abusive`
- `strong_offtopic`

Chain-of-thought модели наружу не передаётся.

---

# 8. OpponentResult

Минимально:

- `response_text`
- `intent`
- `emotion`
- `proposed_state_update`

При необходимости:

- negotiation action
- offer / counteroffer
- requested stage transition
- metadata

Opponent должен возвращать structured output, а не только текст.

`OpponentResult.proposed_state_update` — это только предложение изменения состояния.

Оно ещё не считается принятым `StateUpdate`.

---

# 9. StateUpdate

`StateUpdate` появляется только после прохождения Validator.

Минимально:

- accepted / rejected
- state patch либо новое состояние
- reason code при reject
- corrected values при необходимости

Логика:

```text
OpponentResult
    ↓
proposed_state_update
    ↓
Validator
    ↓
StateUpdate
    ↓
SessionState
```

Backend остаётся `source of truth` для SessionState.

---

# 10. StrategyPlan

`StrategyPlan` создаётся Preparation Coach до начала переговоров.

Минимально:

- primary goal
- priorities
- referenced BATNA / constraints
- unknowns to discover
- planned questions
- concession policy
- if-then plan
- desired outcome

Если BATNA, limits или reservation point уже заданы в Player Private Context, Preparation Coach не должен их менять или придумывать заново.

Он строит стратегию вокруг заданных ограничений.

`StrategyPlan` НЕ передаётся Opponent.

Он сохраняется для Final Judge.

---

# 11. TurnAnalysis

Асинхронный анализ пользовательского хода.

Минимально:

- `turn_id`
- detected user actions
- negotiation tactics
- demonstrated skills
- mistakes
- strong actions
- optional retrieval query

TurnAnalysis не должен блокировать real-time ответ Opponent.

---

# 12. JudgeResult

Минимально:

- overall score
- skill scores
- strengths
- mistakes
- evidence
- better responses
- recommendations
- plan vs reality
- goal achievement summary

Для конкретной ошибки желательно возвращать:

- `turn_id`
- user quote / evidence
- skill
- explanation
- better response

---

# 13. Prompts и Model Profiles

Я должен предоставить system prompts минимум для:

- Guard
- Opponent
- Preparation Coach
- Turn Analyzer
- Final Judge

Для каждой роли также определяются:

- reasoning ON / OFF
- generation parameters
- max output tokens
- expected structured output
- temperature requirements
- latency requirements, если критично

---

# 14. Guardrails

Я определяю правила Guard и защиты AI-контура.

Минимально:

- prompt injection
- jailbreak
- попытка получить system prompt
- попытка получить private context
- попытка узнать hidden BATNA
- попытка узнать reservation point
- unsafe input
- abusive input
- strong off-topic

Также должны соблюдаться архитектурные правила доступа к данным.

### Preparation Coach получает:

- Shared Context
- Player Private Context
- Coach Retrieval

### Opponent получает:

- Shared Context
- Opponent Private Context
- OpponentStrategy
- SessionState
- Transcript

### Opponent НЕ получает:

- Player Private Context
- StrategyPlan

### Preparation Coach НЕ получает:

- Opponent Private Context

### Final Judge может получать:

- полный ScenarioConfig
- StrategyPlan
- Transcript
- Final SessionState
- TurnAnalysis
- Judge Rubric
- Judge Retrieval

Данные, которые модель не должна видеть, не должны передаваться ей физически.

---

# 15. State Validator

Я определяю требования и правила Validator.

Validator должен проверять:

- reservation limits, если они есть
- допустимость уступок
- допустимость сделки
- stage transitions
- trust / pressure bounds
- запрещённые state changes
- private constraints

Opponent формирует:

```text
response_text
+
proposed_state_update
```

После этого Validator проверяет результат.

Только валидированный ответ может попасть в TTS и Transcript как ответ оппонента.

Если validation failed:

- выполняется retry Opponent
- либо используется safe fallback response

Конкретная retry policy определяется совместно AI + Backend.

Физическое место реализации Validator — backend или AI service — определяется командой.

---

# 16. RAG

Я отвечаю за требования к RAG:

- preprocessing
- chunking
- embedding model
- metadata schema
- indexing
- retrieval logic
- filters
- top-k
- reranking при необходимости

Используется одна Negotiation Knowledge Base и одна RAG-инфраструктура:

- Embedding Model
- Qdrant

При этом используются разные retrieval-профили:

- Coach Retrieval
- Opponent Retrieval
- Judge Retrieval

---

# 17. Coach Retrieval

Используется Preparation Coach.

Ищет знания про:

- подготовку к переговорам
- BATNA
- постановку целей
- стратегию
- вопросы
- concessions
- planning techniques

---

# 18. Opponent Retrieval

Используется для формирования поведения / стратегии Opponent.

Ищет:

- tactics
- reaction patterns
- concession strategies
- pressure techniques
- behavior policies

Opponent желательно получать operational / behavioral knowledge, а не преподавательские объяснения.

`OpponentStrategy` и внутренние retrieval results являются внутренними сущностями AI-контура и не обязаны быть внешними API-контрактами.

---

# 19. Judge Retrieval

Используется Final Judge.

Ищет:

- negotiation theory
- evaluation principles
- типичные ошибки
- рекомендации
- alternative responses

---

# 20. Metadata Knowledge Base

Chunks должны иметь metadata минимум:

- `topic`
- `source`
- `section`
- `type`
- `usable_for`

`usable_for` может содержать:

- `coach`
- `opponent`
- `judge`

Документы индексируются заранее.

Embedding Model также должна быть доступна в runtime для embedding retrieval-запросов.

---

# 21. AI Evals

Я должен подготовить набор тестовых кейсов для автоматической и ручной проверки AI-компонентов.

Это не training dataset, а regression / evaluation suite.

## Guard

Проверяем:

- normal request
- prompt injection
- jailbreak
- private information extraction
- off-topic
- unsafe request

## Opponent

Проверяем:

- остаётся в роли
- не раскрывает private context
- не знает StrategyPlan пользователя
- соблюдает ограничения сценария
- не соглашается слишком легко
- корректно реагирует на сильные и слабые действия пользователя
- использует тактики соответствующей сложности

## RAG

Проверяем retrieval по темам:

- BATNA
- concessions
- discovery
- anchoring
- questions
- closing

## Judge

Проверяем оценку:

- хорошей уступки
- плохой уступки
- открытого вопроса
- слабого discovery
- хорошего discovery
- преждевременной скидки
- хорошего closing
- нарушения StrategyPlan
- достижения / недостижения цели

После существенных изменений model / prompt / RAG evals должны запускаться повторно.

---

# 22. Что я должен передать Frontend

Я предоставляю:

- структуру AI outputs
- описание и семантику полей
- enum values
- required / optional поля
- диапазоны и смысл scores
- semantics emotion
- semantics stages
- semantics strengths / mistakes / recommendations

Определения:

`mistake`  
Конкретная ошибка пользователя в конкретном ходе.

`strength`  
Конкретное сильное действие пользователя.

`recommendation`  
Общий совет на будущее.

`better_response`  
Пример более сильной формулировки для конкретного момента.

`evidence`  
Ссылка на `turn_id` и/или цитата из Transcript.

Frontend / Product определяют способ визуального отображения этих данных.

---

# 23. Что я должен передать Product

Я определяю технические требования к сценариям:

- обязательные поля
- типы данных
- public / private разделение
- поддерживаемые ограничения
- какие параметры реально влияют на Opponent
- какие параметры проверяются Validator
- какие параметры используются Final Judge

Product определяет:

- бизнес-смысл параметров
- содержание сценариев
- цели сторон
- ограничения
- критерии успеха
- Judge Rubric

---

# 24. Общие требования к контрактам

Для каждого объекта или API команда должна явно определить:

1. Input
2. Output
3. Required fields
4. Optional fields
5. Types
6. Enum values
7. Owner
8. Source of truth
9. Error behaviour
10. Timeout behaviour
11. Retry behaviour
12. Privacy / access rules
13. Version

Shared contracts нельзя менять без согласования сторон.

---

<a id="short-team"></a>
# 25. Кратко: что мне нужно от команды

## Backend

- ScenarioConfig
- SessionState contract
- Transcript contract
- session / turn lifecycle
- persistence rules
- retry / timeout rules
- error handling

## Model / Infra

- model endpoints
- inference capabilities
- reasoning switching
- structured output
- streaming
- model limits
- timeout / error format

## Product

- Shared Context
- Player Private Context
- Opponent Private Context
- цели сторон
- ограничения
- BATNA / reservation points при наличии
- Judge Rubric
- success / failure criteria

## Frontend

- список реально используемых AI-полей
- требования к данным во время переговоров
- требования к итоговому feedback

---

<a id="short-ai"></a>
# 26. Кратко: что я должен команде

- GuardResult
- OpponentResult
- StateUpdate
- StrategyPlan
- TurnAnalysis
- JudgeResult
- описание AI schemas
- system prompts
- model profiles
- guardrails
- State Validator requirements
- RAG specification
- chunking / metadata / retrieval rules
- AI evals
- семантика AI-полей
- технические требования к ScenarioConfig