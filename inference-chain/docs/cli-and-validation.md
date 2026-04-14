# CLI команды и механизм валидации

## Обзор CLI

Inference-chain предоставляет мощный CLI инструмент `inferenced` для взаимодействия с блокчейном.

**Директория:** `cmd/inferenced/cmd`

## Основные команды

### 1. Управление участниками

#### Регистрация участника

```bash
inferenced register-participant \
  --from <key-name> \
  --collateral 1000000ugonka \
  --models "model1,model2" \
  --node-id "mlnode-001" \
  --chain-id gonka-1 \
  --fees 5000ugonka
```

**Описание:** Регистрирует нового участника в сети с указанным залогом.

**Параметры:**
- `--from` - ключ для подписи транзакции
- `--collateral` - сумма залога (должна соответствовать минимальным требованиям)
- `--models` - список поддерживаемых моделей (через запятую)
- `--node-id` - уникальный идентификатор ML-ноды
- `--chain-id` - идентификатор цепи
- `--fees` - комиссия за транзакцию

#### Запрос информации об участнике

```bash
inferenced query inference participant <address>
```

**Пример вывода:**
```yaml
participant:
  address: gonka1abc...
  consensus_pubkey: gonkapub1...
  epochs_completed: 15
  reputation: 75
  status: ACTIVE
  collateral: 1000000ugonka
  ml_nodes:
    - node_id: mlnode-001
      models: ["llama-7b", "gpt-neo"]
      confirmation_weight: 100
```

### 2. Proof of Contribution (PoC)

#### Отправка PoC-батча

```bash
inferenced tx inference submit-poc-batch \
  --poc-stage-start-block-height 12345 \
  --nonces "nonce1,nonce2,nonce3" \
  --dist "0.1,0.2,0.3" \
  --batch-id "batch-001" \
  --node-id "mlnode-001" \
  --from <key-name> \
  --chain-id gonka-1 \
  --gas auto \
  --gas-adjustment 1.3
```

**Описание:** Отправляет батч PoC-хэшей для доказательства вычислительной работы.

**Параметры:**
- `--poc-stage-start-block-height` - высота блока начала стадии PoC
- `--nonces` - список nonce'ов для сгенерированных хэшей
- `--dist` - распределение сложности
- `--batch-id` - уникальный идентификатор батча
- `--node-id` - идентификатор ML-ноды

#### Отправка валидации PoC

```bash
inferenced tx inference submit-poc-validation \
  --poc-stage-start-block-height 12345 \
  --dealer <dealer-address> \
  --valid true \
  --from <key-name> \
  --chain-id gonka-1
```

**Описание:** Валидирует PoC-батч другого участника.

#### Запрос PoC-батчей

```bash
# Все батчи для конкретной стадии
inferenced query inference poc-batches-for-stage 12345

# Батчи конкретного участника
inferenced query inference poc-batches-for-participant <address>
```

### 3. Управление инференсом

#### Запуск инференса

```bash
inferenced tx inference start-inference \
  --inference-id "inf-001" \
  --prompt-hash "sha256:abc123..." \
  --model "llama-7b" \
  --max-tokens 1000 \
  --temperature 0.7 \
  --payment 50000ugonka \
  --from <key-name> \
  --chain-id gonka-1
```

**Описание:** Создает запрос на выполнение инференса.

**Параметры:**
- `--inference-id` - уникальный идентификатор инференса
- `--prompt-hash` - хэш промпта (для конфиденциальности)
- `--model` - название модели для использования
- `--max-tokens` - максимальное количество токенов
- `--temperature` - параметр температуры для генерации
- `--payment` - оплата за инференс

#### Завершение инференса

```bash
inferenced tx inference finish-inference \
  --inference-id "inf-001" \
  --result-hash "sha256:def456..." \
  --from <key-name> \
  --chain-id gonka-1
```

**Описание:** Отмечает инференс как завершенный с результатом.

#### Отправка валидации

```bash
inferenced tx inference validation \
  --inference-id "inf-001" \
  --is-valid true \
  --from <key-name> \
  --chain-id gonka-1
```

**Описание:** Отправляет результат валидации инференса.

#### Запрос инференса

```bash
inferenced query inference inference <inference-id>
```

**Пример вывода:**
```yaml
inference:
  id: inf-001
  requester: gonka1xyz...
  executor: gonka1abc...
  model: llama-7b
  status: COMPLETED
  prompt_hash: sha256:abc123...
  result_hash: sha256:def456...
  payment: 50000ugonka
  validations:
    - validator: gonka1val1...
      is_valid: true
    - validator: gonka1val2...
      is_valid: true
  created_height: 12345
  completed_height: 12350
```

### 4. Вознаграждения

#### Получение вознаграждений

```bash
inferenced tx inference claim-rewards \
  --seed 1234567890 \
  --seed-signature "base64:signature..." \
  --from <key-name> \
  --chain-id gonka-1
```

**Описание:** Получает вознаграждения за эпоху, раскрывая секретный seed.

**Параметры:**
- `--seed` - секретный seed, использованный для выбора валидаций
- `--seed-signature` - криптографическая подпись seed (для доказательства заранее сгенерированного seed)

**Важно:** Seed должен быть сгенерирован и подписан до начала эпохи. Это предотвращает ретроактивный выбор валидаций.

#### Запрос суммы расчета

```bash
inferenced query inference settle-amount <address>
```

**Пример вывода:**
```yaml
settle_amount:
  participant: gonka1abc...
  epoch_index: 15
  work_coins: 100000ugonka
  reward_coins: 400000ugonka
  total: 500000ugonka
  status: PENDING_CLAIM
```

#### Запрос графика vesting

```bash
inferenced query streamvesting vesting-schedule <address>
```

**Пример вывода:**
```yaml
vesting_schedule:
  participant_address: gonka1abc...
  epoch_amounts:
    - coins: 50000ugonka
    - coins: 50000ugonka
    - coins: 50000ugonka
  remaining_epochs: 3
  total_vesting: 150000ugonka
```

### 5. Управление эпохами

#### Запрос текущей эпохи

```bash
inferenced query inference current-epoch
```

**Пример вывода:**
```yaml
epoch:
  index: 15
  poc_start_block_height: 100000
  stages:
    poc_start: 100000
    poc_generation_winddown: 100500
    poc_generation_end: 101000
    poc_validation_start: 101200
    poc_validation_winddown: 101700
    poc_validation_end: 102000
    set_new_validators: 102050
    claim_money: 102100
    next_poc_start: 110000
  status: ACTIVE
```

#### Запрос предстоящей эпохи

```bash
inferenced query inference upcoming-epoch
```

#### Запрос данных группы эпохи

```bash
inferenced query inference epoch-group-data <epoch-index>
```

### 6. Управление залогом

#### Запрос залога

```bash
inferenced query collateral balance <address>
```

**Пример вывода:**
```yaml
collateral:
  participant: gonka1abc...
  active_collateral: 1000000ugonka
  unbonding:
    - amount: 200000ugonka
      completion_epoch: 18
    - amount: 100000ugonka
      completion_epoch: 20
  is_jailed: false
```

#### Запрос статуса jail

```bash
inferenced query collateral is-jailed <address>
```

### 7. Управление моделями

#### Регистрация модели

```bash
inferenced tx inference register-model \
  --model-name "llama-7b" \
  --model-hash "sha256:model123..." \
  --unit-of-compute-price 1000 \
  --from <key-name> \
  --chain-id gonka-1
```

#### Запрос модели

```bash
inferenced query inference model <model-name>
```

**Пример вывода:**
```yaml
model:
  name: llama-7b
  hash: sha256:model123...
  unit_of_compute_price: 1000
  capacity: 500
  providers:
    - gonka1abc...
    - gonka1def...
  status: ACTIVE
```

#### Предложение цены unit-of-compute

```bash
inferenced tx inference submit-unit-of-compute-price-proposal \
  --model "llama-7b" \
  --price 950 \
  --from <key-name> \
  --chain-id gonka-1
```

### 8. Обучение (Training)

#### Создание задачи обучения

```bash
inferenced tx inference create-training-task \
  --task-id "train-001" \
  --model "llama-7b" \
  --dataset-hash "sha256:data456..." \
  --epochs 10 \
  --batch-size 32 \
  --from <key-name> \
  --chain-id gonka-1
```

#### Присоединение к обучению

```bash
inferenced tx inference join-training \
  --task-id "train-001" \
  --from <key-name> \
  --chain-id gonka-1
```

#### Отправка heartbeat обучения

```bash
inferenced tx inference training-heartbeat \
  --task-id "train-001" \
  --iteration 50 \
  --loss 0.234 \
  --from <key-name> \
  --chain-id gonka-1
```

### 9. Административные команды

#### Обновление параметров (через governance)

```bash
inferenced tx gov submit-proposal param-change proposal.json \
  --from <key-name> \
  --chain-id gonka-1
```

**Пример proposal.json:**
```json
{
  "title": "Update Validation Parameters",
  "description": "Adjust false positive rate and validation averages",
  "changes": [
    {
      "subspace": "inference",
      "key": "ValidationParams",
      "value": {
        "false_positive_rate": "0.02",
        "min_validation_average": "1.5",
        "max_validation_average": "6.0"
      }
    }
  ],
  "deposit": "10000000ugonka"
}
```

### 10. Запросы статистики

#### Производительность эпохи

```bash
inferenced query inference epoch-performance-summary <address> <epoch-index>
```

**Пример вывода:**
```yaml
performance_summary:
  participant: gonka1abc...
  epoch_index: 15
  completed_inferences: 1234
  validated_inferences: 567
  invalidated_inferences: 12
  missed_requests: 45
  poc_batches_submitted: 1
  poc_validations_submitted: 89
  uptime_percentage: 96.5
  reputation_change: +5
```

#### Топ-майнеры

```bash
inferenced query inference top-miners
```

**Пример вывода:**
```yaml
top_miners:
  - rank: 1
    address: gonka1abc...
    poc_qualification: 150
    work_coins: 500000ugonka
    bonus_reward: 50000ugonka
  - rank: 2
    address: gonka1def...
    poc_qualification: 145
    work_coins: 480000ugonka
    bonus_reward: 40000ugonka
```

## Механизм валидации инференса

### Обзор

Inference-chain использует **приватную рандомизированную валидацию** для эффективной проверки результатов инференса.

**Документация:** `docs/specs/inference-validation-flow.md`

### Фаза 1: Генерация секретного seed

**На каждой API-ноде:**
```
1. Генерация приватного случайного seed для каждой эпохи
2. Сохранение seed в секрете в локальной конфигурации
3. Использование seed для приватного выбора валидаций
```

**Пример генерации seed:**
```go
func GenerateSeed() (int64, []byte, error) {
    // Генерация криптографически безопасного случайного числа
    seed := crypto.RandomInt64()

    // Подпись seed валидаторским ключом
    signature := SignWithValidatorKey(seed)

    // Сохранение в конфигурацию
    SaveSeedToConfig(seed, signature)

    return seed, signature, nil
}
```

### Фаза 2: Приватный выбор валидации

**Когда инференс завершается:**

```go
func (iv *InferenceValidator) SampleInferenceToValidate(finishedInferenceIds []string) {
    // Получение текущего секретного seed
    currentSeed := configManager.GetCurrentSeed().Seed

    for each inferenceId in finishedInferenceIds:
        // Получение деталей инференса из блокчейна
        inferenceDetails := GetInferenceValidationDetails(inferenceId)

        // Приватный расчет используя секретный seed
        shouldValidate, _ := calculations.ShouldValidate(
            currentSeed,                    // Секретный seed
            inferenceDetails,               // Детали инференса
            totalPower,                     // Общая мощность сети
            validatorPower,                 // Наша мощность
            executorPower,                  // Мощность исполнителя
            validationParams,               // Параметры валидации
        )

        if shouldValidate:
            // Немедленное выполнение верификации
            result := PerformValidation(inferenceId)

            // Публикация результата валидации в блокчейн
            SubmitValidation(inferenceId, result)
}
```

### Фаза 3: Раскрытие seed и ретроактивная верификация

**При получении вознаграждений:**

```go
func (k msgServer) ClaimRewards(ctx, msg) (*MsgClaimRewardsResponse, error) {
    // 1. Участник раскрывает свой секретный seed
    revealedSeed := msg.Seed

    // 2. Верификация подписи seed (предотвращает ретроактивное изменение)
    ValidateSeedSignature(revealedSeed, msg.SeedSignature, participant.ValidatorKey)

    // 3. Реконструкция списка инференсов, которые ДОЛЖНЫ были быть проверены
    requiredValidations := getMustBeValidatedInferences(
        ctx,
        participant,
        revealedSeed,
        epochInferences,
    )

    // 4. Сравнение фактических и требуемых валидаций
    actualValidations := GetValidationsSubmitted(participant, currentEpoch)

    for each requiredInference in requiredValidations:
        if requiredInference not in actualValidations:
            return error(ErrValidationsMissed)

    // 5. Если все требуемые валидации присутствуют, разрешить вознаграждения
    PayoutRewards(participant, settleAmount)

    return &MsgClaimRewardsResponse{Amount: totalCoins}
}

func getMustBeValidatedInferences(ctx, participant, seed, inferences) []string {
    mustValidate := []string{}

    for each inference in inferences:
        inferenceDetails := GetInferenceValidationDetails(inference.Id)

        // Повторный запуск точно такого же расчета, который валидатор использовал приватно
        shouldValidate, _ := calculations.ShouldValidate(
            seed,  // Теперь используя раскрытый seed
            inferenceDetails,
            totalPower,
            participant.Power,
            executorPower,
            validationParams,
        )

        if shouldValidate:
            mustValidate.append(inference.Id)

    return mustValidate
}
```

### Механизмы безопасности

#### 1. Предотвращение манипуляций

**Проблема:** Валидаторы могут захотеть пропустить сложные валидации.

**Решение:**
- Seed генерируется и подписывается **до** начала эпохи
- Валидаторы не могут предсказать, какие инференсы им нужно будет проверять
- Ретроактивная верификация при получении вознаграждений гарантирует соблюдение

**Пример атаки (не удается):**
```
Злоумышленник видит:
- inference-001: простая валидация
- inference-002: сложная валидация

Злоумышленник хочет:
- Проверить только inference-001, пропустить inference-002

Почему не получится:
1. Seed уже сгенерирован и подписан до начала эпохи
2. ShouldValidate() детерминированно выбирает на основе seed
3. При claim rewards, сеть реконструирует список требуемых валидаций
4. Если пропущена inference-002, rewards отклоняются
```

#### 2. Криптографическая подотчетность

**Механизм:**
- Seeds криптографически подписываются при генерации
- Подпись использует валидаторский ключ (Ed25519)
- Невозможно изменить seed после того, как он подписан
- Публичная верификация во время claims обеспечивает прозрачность

**Код проверки подписи:**
```go
func ValidateSeedSignature(seed int64, signature []byte, validatorPubkey crypto.PubKey) error {
    // Сериализация seed
    seedBytes := make([]byte, 8)
    binary.BigEndian.PutUint64(seedBytes, uint64(seed))

    // Верификация подписи
    if !validatorPubkey.VerifySignature(seedBytes, signature) {
        return errors.New("invalid seed signature")
    }

    return nil
}
```

#### 3. Применение для конкретной модели

**Механизм:**
- Требования валидации используют мощность для конкретной модели
- Только валидаторы, поддерживающие модель, имеют право проверять
- Предотвращаются назначения между моделями

**Проверка:**
```go
func canValidateModel(validator Participant, modelName string) bool {
    for _, mlNode := range validator.MlNodes {
        for _, supportedModel := range mlNode.Models {
            if supportedModel == modelName {
                return true
            }
        }
    }
    return false
}
```

### Преимущества рандомизированной валидации

#### 1. Эффективность

**Традиционная валидация:**
- Каждый инференс проверяется N валидаторами
- Избыточность 100% × N

**Рандомизированная валидация:**
- Каждый инференс проверяется ~1-10% валидаторов (в зависимости от репутации)
- Избыточность 10-100% (адаптивная)

**Экономия:** 90-99% вычислительных ресурсов при сохранении безопасности.

#### 2. Адаптивная безопасность

**Высокая репутация исполнителя:**
```
Репутация = 95/100
→ MinValidationAverage = 1.0
→ TargetValidations ≈ 1.25
→ Меньше валидаторов проверяют (доверие)
```

**Низкая репутация исполнителя:**
```
Репутация = 10/100
→ MaxValidationAverage = 5.0
→ TargetValidations ≈ 4.55
→ Больше валидаторов проверяют (тщательная проверка)
```

#### 3. Защита от сговора

**Проблема:** Исполнитель и валидаторы могут сговориться.

**Решение:**
- Случайный выбор валидаторов (неизвестен заранее)
- Детерминированный после раскрытия seed (не может быть изменен)
- Множественные валидаторы снижают риск полного сговора

## Заключение

CLI inference-chain предоставляет:

1. **Полный контроль:** Управление участниками, инференсом, обучением, вознаграждениями
2. **Мониторинг:** Запросы статуса, производительности, эпох
3. **Безопасность:** Криптографические подписи, проверка seed
4. **Гибкость:** Поддержка множества моделей, параметров, конфигураций

Механизм валидации обеспечивает:

1. **Эффективность:** 90-99% экономия ресурсов
2. **Безопасность:** Криптографическая подотчетность и ретроактивная верификация
3. **Справедливость:** Адаптивная валидация на основе репутации
4. **Защиту от манипуляций:** Детерминированный выбор с секретными seeds

Все вместе создает надежную, масштабируемую и экономически эффективную систему для децентрализованного AI-инференса.
