# Proof of Work 2.0 и механизм эпох

## Обзор

Inference-chain использует инновационный механизм консенсуса **Proof of Work 2.0**, который на самом деле представляет собой **Proof of Contribution (PoC)** - доказательство вычислительной способности через выполнение инференса AI-моделей вместо традиционного майнинга хэшей.

## Proof of Contribution (PoC)

### Ключевая документация

`mlnode/packages/pow/docs/description.md`

### Процедура PoC Race

```
1. Получение сигнала старта гонки:
   - Получить deadline и уровень сложности
   - deadline = GetDeadline()
   - difficulty = GetDifficulty()

2. Получение распределенного seed:
   - На основе блокчейна или распределенного источника
   - distributedSeed = GetDistributedSeed(chain)

3. Инициализация модели:
   - Инициализировать LLM с seed и сложностью
   - llmModel = LLM(distributedSeed, difficulty)

4. Генерация хэшей в течение X секунд:
   - Генерировать хэши до deadline
   - hashes = GenerateHashes(llmModel, X, difficulty, deadline)

5. Отправка сгенерированных хэшей в сеть:
   - Отправить батч до deadline
```

### Генерация хэшей (доказательство способности инференса)

```python
def GenerateHashes(llmModel, X, difficulty, deadline):
    hashes = []
    publicKey = GetNodePublicKey()

    while True:
        salt = GetNextSalt()
        hash = GenerateHash(publicKey, salt, llmModel)

        if CurrentTime() > deadline:
            return hashes

        if GetLeadingZerosAmount(hash) >= difficulty:
            hashes.add((hash, salt))

def GenerateHash(publicKey, salt, llmModel):
    tokens = GetModelInput(publicKey, salt)  # Token embeddings
    output = llmModel.forward(tokens)  # 1 или более forward проходов
    hash = SHA256(output)
    return hash
```

**Ключевая особенность:** Вместо бессмысленных вычислений SHA256, участники выполняют реальный AI-инференс, что обеспечивает ~100% утилизацию ресурсов для продуктивных задач.

### Валидация (может проверить любой)

```python
def ValidateHash(nodePublicKey, hash, salt, difficulty):
    llmModel = LLM(seed, difficulty)
    hash = GenerateHash(nodePublicKey, salt, llmModel)
    if GetLeadingZerosAmount(hash) >= difficulty:
        return True
    return False
```

## Отправка и валидация PoC

### Фаза 1: Генерация и обмен PoC

**Файл:** `x/inference/keeper/msg_server_submit_poc_batch.go`

```go
func (k msgServer) SubmitPocBatch(ctx, msg) (*MsgSubmitPocBatchResponse, error) {
    // 1. Контроль доступа
    if k.IsPoCParticipantBlocked(msg.Creator):
        return error("participant blocked")

    // 2. Проверка события подтверждения PoC
    activeEvent, isActive := k.GetActiveConfirmationPoCEvent()

    if isActive && activeEvent.Phase == CONFIRMATION_POC_GENERATION:
        // Обработка отправки батча подтверждения PoC
        if !activeEvent.IsInBatchSubmissionWindow(currentBlockHeight):
            return error("outside submission window")

        StorePocBatch(activeEvent.TriggerHeight, msg.Creator, msg.Nonces, msg.Dist)
        return success
    }

    // 3. Регулярная отправка PoC-батча
    upcomingEpoch := GetUpcomingEpoch()
    epochContext := NewEpochContext(upcomingEpoch)

    // Проверка времени
    if !epochContext.IsStartOfPocStage(msg.PocStageStartBlockHeight):
        return error("wrong start block height")

    if !epochContext.IsPoCExchangeWindow(currentBlockHeight):
        return error("PoC exchange window closed")

    // Сохранение батча
    StorePocBatch(msg.PocStageStartBlockHeight, msg.Creator, msg.Nonces, msg.Dist, msg.NodeId)

    return success
}
```

### Фаза 2: Обмен валидацией PoC

```go
func (k msgServer) SubmitPocValidation(ctx, msg) (*MsgSubmitPocValidationResponse, error) {
    // Аналогичная структура SubmitPocBatch, но для валидаторов
    // Валидаторы проверяют и отправляют результаты валидации батчей

    StorePocValidation(msg.PocStageStartBlockHeight, msg.Validator, msg.Dealer, msg.Valid)
}
```

## Структура эпох и фазы

### Терминология эпох

**Документация:** `x/inference/types/epoch.md`

**Указатели эпох:**
1. **Current/Effective Epoch (Текущая эпоха):** Эпоха с валидаторами, которые являются текущими валидаторами цепи
2. **Upcoming Epoch (Предстоящая эпоха):** Эпоха, подготавливаемая для следующей стадии PoC, становится текущей после EndBlock
3. **Previous Epoch (Предыдущая эпоха):** Эпоха перед current/effective эпохой
4. **Latest Epoch (Последняя эпоха):** Последняя созданная эпоха (либо current, либо upcoming)

**Поля, связанные с эпохами:**
- `epoch_id/epoch_index`: Последовательный номер, начинающийся с 0 (увеличивается для каждого PoC)
- `epoch_group_data_id`: ID сущности Group Module, связанной с EpochGroupData
- `epoch_poc_start_block_height`: Высота блока начала PoC (используется как индекс KV)

### Стадии эпох

**Файл:** `x/inference/types/epoch_stages.go`

```go
type EpochStages struct {
    EpochIndex                uint64              // Номер текущей эпохи
    PocStart                  int64               // Начало генерации PoC
    PocGenerationWindDown     int64               // Завершение генерации
    PocGenerationEnd          int64               // Конец генерации
    PocValidationStart        int64               // Начало валидации
    PocValidationWindDown     int64               // Завершение валидации
    PocValidationEnd          int64               // Конец валидации
    SetNewValidators          int64               // Переключение набора валидаторов
    ClaimMoney                int64               // Эпоха для получения вознаграждений
    InferenceValidationCutoff int64               // Последний блок для валидаций
    NextPocStart              int64               // Начало PoC следующей эпохи
    PocExchangeWindow         EpochExchangeWindow // Окно обмена батчами
    PocValExchangeWindow      EpochExchangeWindow // Окно обмена валидациями
}

type EpochExchangeWindow struct {
    Start int64  // Inclusive начальный блок
    End   int64  // Inclusive конечный блок
}
```

### Временная линия жизненного цикла эпохи

```
Временная линия высоты блоков:
│
├── IsStartOfPocStage (например, блок 1000)
│   - Создание upcoming эпохи (индекс N+1)
│   - Создание EpochGroupData и Group
│   - Начало фазы генерации PoC
│
├── Фаза генерации PoC (блоки 1000-1100)
│   - Участники генерируют PoC-хэши
│   - Отправка PoC-батчей
│
├── Фаза обмена PoC (блоки 1100-1150)
│   - Обмен PoC-батчами
│   - Завершение генерации
│
├── Фаза валидации PoC (блоки 1150-1250)
│   - Валидаторы проверяют PoC-батчи
│   - Отправка результатов валидации
│
├── IsEndOfPoCValidationStage (например, блок 1250)
│   - Расчет аккаунтов из предыдущей эпохи
│   - Вычисление новых весов из результатов PoC
│   - Установка моделей для участников
│   - Добавление членов в группу предстоящей эпохи
│
├── IsSetNewValidatorsStage (например, блок 1260)
│   - Переключение на новый набор валидаторов
│   - Перемещение upcoming → effective эпоха
│   - EffectiveEpochIndex = N+1
│
└── Фаза инференса (блоки 1260-2000)
    - Валидаторы обслуживают инференсы
    - Валидаторы проверяют инференсы
    - Следующий цикл PoC начинается на блоке 2000
```

## Переходы стадий в EndBlock

**Файл:** `x/inference/module/module.go`

```go
func (am AppModule) EndBlock(ctx context.Context) error {
    blockHeight := ctx.BlockHeight()

    // Обработка подтверждения PoC trigger и переходов
    am.handleConfirmationPoC(ctx, blockHeight)

    // Получение текущей эпохи и параметров
    params := am.keeper.GetParams(ctx)
    currentEpoch := am.keeper.GetEffectiveEpoch(ctx)
    epochContext := types.NewEpochContext(*currentEpoch, *params.EpochParams, blockHeight)

    // Обработка таймаутов инференса
    timeouts := am.keeper.GetAllInferenceTimeoutForHeight(ctx, blockHeight)
    am.expireInferences(ctx, timeouts)

    // Очистка старых данных
    am.keeper.Prune(ctx, currentEpoch.Index)

    // Отслеживание обновлений
    if upgradePlan.Height == blockHeight:
        am.keeper.SetLastUpgradeHeight(ctx, blockHeight)

    // ===== ПЕРЕХОДЫ СТАДИЙ ЭПОХ =====

    // Стадия 1: Завершение формирования эпохи
    if epochContext.IsEndOfPoCValidationStage(blockHeight):
        am.onEndOfPoCValidationStage(ctx, blockHeight, blockTime)
        // - Расчет аккаунтов из предыдущей эпохи
        // - Продвижение эпохи collateral (обработка unbonding)
        // - Продвижение эпохи streamvesting (разблокировка токенов с vesting)
        // - Вычисление новых весов PoC
        // - Установка моделей участников
        // - Регистрация топ-майнеров
        // - Добавление членов в группу предстоящей эпохи

    // Стадия 2: Переключение валидаторов
    if epochContext.IsSetNewValidatorsStage(blockHeight):
        am.onSetNewValidatorsStage(ctx, blockHeight, blockTime)
        // - Перемещение upcoming → effective эпоха
        // - Обновление индекса effective эпохи

    // Стадия 3: Начало нового PoC
    if epochContext.IsStartOfPocStage(blockHeight):
        // - Создание новой upcoming эпохи
        // - Создание группы эпохи для участников PoC
        // - Начало фазы генерации PoC
        upcomingEpoch := createNewEpoch(currentEpoch, blockHeight)
        am.keeper.SetEpoch(ctx, upcomingEpoch)

        newGroup := am.keeper.CreateEpochGroup(ctx, blockHeight, upcomingEpoch.Index)
        newGroup.CreateGroup(ctx)

    // Обновление набора валидаторов, если группа эпохи изменилась
    if currentEpochGroup.IsChanged(ctx):
        computeResult := currentEpochGroup.GetComputeResults(ctx)
        finalResult := am.applyEarlyNetworkProtection(ctx, computeResult)
        am.keeper.Staking.SetComputeValidators(ctx, finalResult)

    return nil
}
```

## Выбор валидаторов на основе результатов PoC

**Файл:** `x/inference/epochgroup/epoch_group.go`

```go
func (eg *EpochGroup) GetComputeResults(ctx) ([]ComputeResult, error) {
    participants := eg.GetParticipants(ctx)
    members := []EpochMember{}

    for _, participant := range participants:
        // Расчет репутации
        reputation := calculations.Reputation(
            participant.EpochsCompleted,
            participant.MissPercentages,
            validationParams
        )

        // Расчет веса подтверждения из PoC
        confirmationWeight := calculateInferenceServingWeight(participant.MlNodes)

        member := NewEpochMemberFromActiveParticipant(
            participant,
            reputation,
            confirmationWeight
        )
        members = append(members, member)

    // Создание обновлений валидаторов для CometBFT
    computeResults := []ComputeResult{}
    for _, member := range members:
        computeResults = append(computeResults, ComputeResult{
            Address:         member.Address,
            Power:           member.Weight,           // Вес голоса
            ConsensusPubkey: member.Pubkey,           // Ed25519 ключ
        })

    return computeResults
}
```

### Интеграция с CometBFT

```go
// Вызывается модулем Staking для обновления CometBFT
func (k Keeper) SetComputeValidators(ctx, computeResults, isTestNet) ([]abci.ValidatorUpdate, error) {
    validatorUpdates := []abci.ValidatorUpdate{}

    for _, result := range computeResults:
        // Конвертация в формат обновления валидатора CometBFT
        pubkey := ed25519.PubKey{...}

        validatorUpdates = append(validatorUpdates, abci.ValidatorUpdate{
            PubKey: pubkey,
            Power:  result.Power,  // Вес голоса в консенсусе
        })

    // Применение обновлений к CometBFT
    return validatorUpdates, nil
}
```

## Поток создания эпох

### 1. Точки создания

- **EndBlock** когда `IsStartOfPocStage`: Создание новой upcoming эпохи и EpochGroupData
- **InitGenesis**: Установка группы эпохи 0

### 2. Жизненный цикл

```
Создание эпохи (блок X):
├── Создать Epoch структуру
│   - Index: N+1
│   - PocStartBlockHeight: X
│   - GroupId: <генерируется>
│
├── Создать EpochGroupData
│   - EpochIndex: N+1
│   - GroupDataId: <генерируется>
│   - Phase: FORMATION
│
├── Создать Group (через Group модуль)
│   - Members: пусто (добавляются в конце валидации PoC)
│   - Policy: консенсусная политика
│
└── Сохранить все в state

Формирование эпохи (блоки X до Y):
├── Фаза генерации PoC
│   - Участники генерируют и отправляют батчи
│
├── Фаза валидации PoC
│   - Валидаторы проверяют батчи
│
└── Завершение валидации (блок Y)
    ├── Расчет весов PoC
    ├── Добавление участников в группу
    ├── Phase: ACTIVE
    └── Подготовка к переключению валидаторов

Активация эпохи (блок Y+offset):
├── SetNewValidatorsStage
│   - Upcoming → Effective
│   - Обновить EffectiveEpochIndex
│   - Переключить CometBFT валидаторов
│
└── Эпоха теперь активна для обслуживания инференсов
```

## Параметры эпох

**Из:** `proto/inference/inference/params.proto`

```protobuf
message EpochParams {
    int64 epoch_length = 1;                      // Базовая длительность эпохи в блоках
    int64 epoch_multiplier = 2;                  // Множитель для расчетов эпохи
    int64 epoch_shift = 3;                       // Смещение для выравнивания эпохи
    int64 default_unit_of_compute_price = 4;     // Цена инференса по умолчанию
    int64 poc_stage_duration = 5;                // Длительность генерации PoC
    int64 poc_exchange_duration = 6;             // Окно обмена PoC-батчами
    int64 poc_validation_delay = 7;              // Задержка перед началом валидации
    int64 poc_validation_duration = 8;           // Длительность валидации PoC
    int64 set_new_validators_delay = 9;          // Задержка перед переключением валидаторов
    int64 inference_validation_cutoff = 10;      // Cutoff для валидаций инференса
    uint64 inference_pruning_epoch_threshold = 11; // Эпохи перед очисткой инференсов
    int64 inference_pruning_max = 12;            // Макс. инференсов для очистки за блок
    int64 poc_pruning_max = 13;                  // Макс. данных PoC для очистки за блок
    Decimal poc_slot_allocation = 14;            // Доля слотов для PoC (0.0-1.0)
}
```

## Преимущества Proof of Work 2.0

### 1. Продуктивное использование ресурсов

В отличие от традиционного PoW, который тратит электроэнергию на бессмысленные вычисления SHA256, PoW 2.0 использует вычислительную мощность для реального AI-инференса.

**Утилизация ресурсов:** ~100% вычислительной мощности идет на продуктивные AI-задачи.

### 2. Прямая корреляция вычислений и влияния

```
Больше вычислительной мощности
    → Больше успешных PoC-хэшей
    → Выше вес голоса
    → Больше влияния в управлении
    → Больше вознаграждений
```

### 3. Защита от Sybil-атак

Создание множества фейковых нод не дает преимущества - требуется реальная вычислительная мощность для генерации валидных PoC-батчей.

### 4. Детерминированная верификация

Любой может верифицировать PoC-батчи, запустив те же вычисления модели с теми же параметрами (seed, salt, difficulty).

### 5. Адаптивная сложность

Сложность может быть настроена через governance для балансировки требований к вычислительной мощности и доступности участия.

## Безопасность

### 1. Временные ограничения

PoC-батчи должны быть отправлены в определенные окна времени, предотвращая pre-mining или отложенные отправки.

### 2. Распределенные seeds

Использование блокчейн-генерируемых seeds гарантирует, что никто не может предварительно вычислить хэши.

### 3. Валидация консенсусом

Несколько валидаторов проверяют каждый PoC-батч, предотвращая обман.

### 4. Слэшинг за мошенничество

Отправка невалидных PoC-батчей приводит к слэшингу залога и потенциальному jailing.

## Заключение

Proof of Work 2.0 представляет собой фундаментальное переосмысление консенсуса для AI-эпохи:

- **Вычисления служат двойной цели:** Консенсус + полезная работа
- **Децентрализация без расточительства:** Зеленый консенсус через продуктивные вычисления
- **Экономическое выравнивание:** Провайдеры вычислительной мощности напрямую получают вознаграждения
- **Проверяемость:** Детерминированная валидация обеспечивает прозрачность
- **Масштабируемость:** Epoch-based ротация позволяет динамический набор валидаторов

Это делает inference-chain уникальным блокчейном, где консенсус и полезная работа объединены в единый механизм.
