# Алгоритмы и вычисления

## Обзор

Inference-chain использует сложные статистические и математические алгоритмы для управления репутацией участников, валидацией, распределением вознаграждений и принятием решений о консенсусе.

## 1. Расчет репутации

### Документация

**Файл:** `x/inference/calculations/reputation.md`

### Входные данные

```go
type ReputationInputs struct {
    EpochCount           uint64    // Количество эпох, в которых участвовал
    EpochMissPercentages []float64 // Список процентов пропущенных запросов по эпохам
}

type ValidationParams struct {
    EpochsToMax           int64   // Эпохи для достижения максимальной репутации
    MissPercentageCutoff  float64 // Порог процента пропусков перед штрафом
    MissRequestsPenalty   float64 // Множитель штрафа за пропуски
}
```

### Алгоритм

```go
func Reputation(epochCount uint64, epochMissPercentages []float64, params ValidationParams) uint64 {
    // 1. Расчет AddMissCost (штраф за пропуски)
    penalty := 0.0
    for _, missPercentage := range epochMissPercentages {
        if missPercentage > params.MissPercentageCutoff {
            penalty = (penalty + missPercentage/float64(params.EpochsToMax)) * params.MissRequestsPenalty
        }
    }
    addMissCost := penalty * float64(params.EpochsToMax)

    // 2. Расчет ActualEpochCount
    actualEpochCount := float64(epochCount) - addMissCost

    // 3. Репутация (ограничена 0-100)
    if actualEpochCount > float64(params.EpochsToMax) {
        return 100
    }
    if actualEpochCount <= 0 {
        return 0
    }

    return uint64(math.Floor(actualEpochCount / float64(params.EpochsToMax) * 100))
}
```

### Пример

```
Входные данные:
- EpochCount = 50
- EpochMissPercentages = [0.05, 0.02, 0.15, 0.01]
- EpochsToMax = 100
- MissPercentageCutoff = 0.10
- MissRequestsPenalty = 2.0

Расчет:
1. Эпоха 0: 0.05 < 0.10 → без штрафа
2. Эпоха 1: 0.02 < 0.10 → без штрафа
3. Эпоха 2: 0.15 > 0.10 → penalty = (0 + 0.15/100) * 2.0 = 0.003
4. Эпоха 3: 0.01 < 0.10 → без штрафа

AddMissCost = 0.003 * 100 = 0.3
ActualEpochCount = 50 - 0.3 = 49.7
Reputation = floor(49.7 / 100 * 100) = 49
```

### Интерпретация

- **Репутация 0:** Новый участник или сильно штрафован
- **Репутация 1-99:** Прогрессирующий участник (частично надежен)
- **Репутация 100:** Полностью надежный участник (100+ эпох без проблем)

## 2. Расчет статуса участника

### Документация

**Файл:** `x/inference/calculations/status.md`

### Входные данные

```go
type StatusInputs struct {
    ConsecutiveInvalidInferences uint64  // Последовательные невалидные инференсы
    ValidatedInferences          uint64  // Валидированные инференсы
    InvalidatedInferences        uint64  // Инвалидированные инференсы
    EpochsCompleted              uint64  // Завершенные эпохи
}

type ValidationParams struct {
    FalsePositiveRate       float64  // Ожидаемый FPR для хороших участников (например, 0.01)
    MinRampUpMeasurements   int64    // Мин. измерений перед проверками статуса
}
```

### Статусы

- **ACTIVE:** Участник работает нормально
- **RAMPING:** Новый участник в процессе набора репутации
- **INVALID:** Участник демонстрирует статистически значимую нечестность

### Алгоритм

```go
func ComputeStatus(validationParams ValidationParams, participant Participant) (status Status, reason string) {
    p := validationParams.FalsePositiveRate
    N := participant.ConsecutiveInvalidInferences
    V := participant.CurrentEpochStats.ValidatedInferences
    I := participant.CurrentEpochStats.InvalidatedInferences
    E := participant.EpochsCompleted

    // 1. Проверка последовательных неудач: p^N < 10^-6
    if math.Pow(p, float64(N)) < 0.000001 {
        return INVALID, "consecutive_failures"
    }

    // 2. Расчет Z-оценки
    n := V + I
    var z float64
    if n > 0 && p*(1-p)/float64(n) > 0 {
        observedRate := float64(I) / float64(n)
        standardError := math.Sqrt(p * (1 - p) / float64(n))
        z = (observedRate - p) / standardError
    } else {
        z = 0
    }

    // 3. Порог ramp-up
    needed := int64(math.Min(
        math.Ceil((3+math.Sqrt(5))/(2*p)),
        float64(validationParams.MinRampUpMeasurements),
    ))

    // 4. Определение статуса
    if n < uint64(needed) && E < 1 {
        return RAMPING, "ramping"
    }
    if z > 1 {
        return INVALID, "statistical_invalidations"
    }
    return ACTIVE, ""
}
```

### Примеры

**Пример 1: Последовательные неудачи**
```
p = 0.01, N = 4
p^4 = 0.01^4 = 10^-8 < 10^-6
→ INVALID (consecutive_failures)
```

**Пример 2: Статистические инвалидации**
```
p = 0.02, n = 500, I = 20
Ожидаемая скорость инвалидации: 0.02
Наблюдаемая скорость: 20/500 = 0.04
z = (0.04 - 0.02) / sqrt(0.02 * 0.98 / 500) ≈ 3.19 > 1
→ INVALID (statistical_invalidations)
```

**Пример 3: Ramp-up**
```
p = 0.02, n = 100, E = 0
needed = ceil((3+sqrt(5))/(2*0.02)) = 131
100 < 131 и E < 1
→ RAMPING (ramping)
```

### Обоснование

Алгоритм использует статистическую проверку гипотез:

- **Null Hypothesis (H0):** Участник честен (скорость инвалидации = p)
- **Alternative Hypothesis (H1):** Участник нечестен (скорость инвалидации > p)
- **Test Statistic:** Z-оценка (стандартизированное отклонение от ожидаемой скорости)
- **Rejection Region:** z > 1 (односторонний тест)

## 3. Алгоритм ShouldValidate

### Документация

**Файл:** `x/inference/calculations/should_validate.go`

### Назначение

Определяет, должен ли валидатор проверять конкретный инференс на основе:
- Секретного seed валидатора
- Репутации исполнителя
- Относительной вычислительной мощности

### Входные данные

```go
type ShouldValidateInputs struct {
    Seed                 int64     // Секретный seed валидатора
    InferenceId          string    // ID инференса
    ExecutorReputation   uint64    // Репутация исполнителя (0-100)
    TrafficBasis         string    // Базис трафика для расчетов
    TotalPower           uint32    // Общая мощность сети
    ValidatorPower       uint32    // Мощность валидатора
    ExecutorPower        uint32    // Мощность исполнителя
}

type ValidationParams struct {
    MinValidationAverage float64  // Мин. средние валидации на инференс
    MaxValidationAverage float64  // Макс. средние валидации на инференс
}
```

### Алгоритм

```go
func ShouldValidate(
    seed int64,
    inferenceDetails *InferenceValidationDetails,
    totalPower uint32,
    validatorPower uint32,
    executorPower uint32,
    validationParams *ValidationParams,
) (bool, string) {
    // 1. Расчет целевых валидаций на основе репутации исполнителя
    executorReputation := decimal.NewFromInt(int64(inferenceDetails.ExecutorReputation)).Div(decimal.NewFromInt(100))

    maxValidationAverage := validationParams.MaxValidationAverage
    minValidationAverage := CalculateMinimumValidationAverage(inferenceDetails.TrafficBasis, validationParams)
    rangeSize := maxValidationAverage - minValidationAverage

    // Высокая репутация (100%) → minValidationAverage валидаций
    // Низкая репутация (0%) → maxValidationAverage валидаций
    targetValidations := maxValidationAverage - (rangeSize * executorReputation)

    // 2. Расчет нашей вероятности быть выбранным
    ourProbability := targetValidations * decimal.NewFromInt(int64(validatorPower)).Div(
        decimal.NewFromInt(int64(totalPower - executorPower)),
    )

    if ourProbability.GreaterThan(decimal.NewFromInt(1)) {
        ourProbability = decimal.NewFromInt(1)
    }

    // 3. Детерминированный случайный выбор используя seed и inferenceId
    randFloat := DeterministicFloat(seed, inferenceDetails.InferenceId)

    // 4. Выбор
    shouldValidate := randFloat.LessThan(ourProbability)

    return shouldValidate, ""
}

func DeterministicFloat(seed int64, identifier string) decimal.Decimal {
    // Объединение seed и идентификатора в детерминированные байты
    b := []byte(strconv.FormatInt(seed, 10) + ":" + identifier)

    // Хэш для получения детерминированного "случайного" числа
    hash := sha256.Sum256(b)
    hashInt := binary.BigEndian.Uint64(hash[:8])

    // Конвертация в decimal [0, 1)
    return decimal.NewFromUint64(hashInt).Div(decimal.NewFromUint64(math.MaxUint64))
}
```

### Расчет минимальной средней валидации

```go
func CalculateMinimumValidationAverage(trafficBasis string, params *ValidationParams) decimal.Decimal {
    minValidationAverage := params.MinValidationAverage
    minValidationHalfway := params.MinValidationHalfway
    fullValidationTrafficCutoff := params.FullValidationTrafficCutoff
    minValidationTrafficCutoff := params.MinValidationTrafficCutoff

    if trafficBasis == "" {
        return decimal.NewFromFloat(minValidationAverage)
    }

    trafficBasisInt, _ := strconv.ParseInt(trafficBasis, 10, 64)

    if trafficBasisInt >= fullValidationTrafficCutoff {
        // Высокий трафик → полная валидация
        return decimal.NewFromFloat(minValidationAverage)
    }

    if trafficBasisInt <= minValidationTrafficCutoff {
        // Низкий трафик → валидация на полпути
        return decimal.NewFromFloat(minValidationHalfway)
    }

    // Линейная интерполяция между cutoff точками
    trafficRange := decimal.NewFromInt(fullValidationTrafficCutoff - minValidationTrafficCutoff)
    trafficDiff := decimal.NewFromInt(trafficBasisInt - minValidationTrafficCutoff)
    trafficFraction := trafficDiff.Div(trafficRange)

    validationRange := decimal.NewFromFloat(minValidationAverage - minValidationHalfway)
    interpolated := decimal.NewFromFloat(minValidationHalfway).Add(validationRange.Mul(trafficFraction))

    return interpolated
}
```

### Пример

```
Входные данные:
- Seed = 1234567890
- InferenceId = "inf-001"
- ExecutorReputation = 80 (высокая репутация)
- TotalPower = 10000
- ValidatorPower = 100
- ExecutorPower = 500
- MinValidationAverage = 1.0
- MaxValidationAverage = 5.0

Расчет:
1. executorReputation = 80/100 = 0.80
2. rangeSize = 5.0 - 1.0 = 4.0
3. targetValidations = 5.0 - (4.0 * 0.80) = 5.0 - 3.2 = 1.8
4. ourProbability = 1.8 * (100 / (10000 - 500)) = 1.8 * 0.0105 = 0.0189
5. randFloat = DeterministicFloat(1234567890, "inf-001") = 0.742... (пример)
6. shouldValidate = (0.742 < 0.0189) = false

Вывод: Валидатор НЕ должен проверять этот инференс.
```

### Интерпретация

- **Высокая репутация исполнителя:** Меньше валидаций требуется (доверие сети)
- **Низкая репутация исполнителя:** Больше валидаций требуется (более тщательная проверка)
- **Высокая мощность валидатора:** Больше вероятность быть выбранным
- **Детерминированность:** Один и тот же seed всегда дает тот же результат для инференса

## 4. Расчет вознаграждений и субсидий

### Документация

**Файл:** `x/inference/keeper/accountsettle.go`

### Параметры субсидий

```go
type SettleParameters struct {
    CurrentSubsidyPercentage float32  // Текущая ставка субсидии (например, 0.80 = 80%)
    TotalSubsidyPaid         int64    // Всего субсидий выплачено
    StageCutoff              float64  // Доля поставки на стадию снижения (например, 0.10 = 10%)
    StageDecrease            float32  // Снижение на стадию (например, 0.05 = 5%)
    TotalSubsidySupply       int64    // Всего доступно для субсидий
}
```

### Формула субсидии

```
Базовая формула:
subsidy = work / (1 - subsidyRate)

Пример:
work = 100, subsidyRate = 0.80
subsidy = 100 / (1 - 0.80) = 100 / 0.20 = 500

Общая выплата:
total = work + subsidy = 100 + 500 = 600

Разбивка:
- 100 (work coins) от escrow (оплата клиента)
- 500 (reward coins) от модуля (субсидия сети)
```

### Алгоритм расчета субсидии

```go
func (sp *SettleParameters) GetTotalSubsidy(workCoins int64) SubsidyResult {
    // 1. Проверка, остались ли субсидии
    if sp.TotalSubsidyPaid >= sp.TotalSubsidySupply {
        return SubsidyResult{Amount: 0, CrossedCutoff: false}
    }

    // 2. Расчет следующего cutoff
    nextCutoff := sp.getNextCutoff()

    // 3. Расчет субсидии при текущей ставке
    subsidyAtCurrentRate := getSubsidy(workCoins, sp.CurrentSubsidyPercentage)

    // 4. Проверка пересечения границы стадии
    if sp.TotalSubsidyPaid + subsidyAtCurrentRate > nextCutoff {
        // Пересечение границы - разделение расчета

        // 4a. Субсидия до cutoff
        subsidyUntilCutoff := nextCutoff - sp.TotalSubsidyPaid
        workUntilNextCutoff := getWork(subsidyUntilCutoff, sp.CurrentSubsidyPercentage)

        // 4b. Снижение ставки субсидии для следующей стадии
        nextRate := sp.CurrentSubsidyPercentage * (1.0 - sp.StageDecrease)

        // 4c. Субсидия при следующей ставке
        remainingWork := workCoins - workUntilNextCutoff
        subsidyAtNextRate := getSubsidy(remainingWork, nextRate)

        return SubsidyResult{
            Amount: subsidyUntilCutoff + subsidyAtNextRate,
            CrossedCutoff: true,
        }
    }

    // Не пересекли границу - простой расчет
    return SubsidyResult{Amount: subsidyAtCurrentRate, CrossedCutoff: false}
}

// Формула: subsidy = work / (1 - subsidyRate)
func getSubsidy(work int64, rate float32) int64 {
    return decimal.NewFromInt(work).Div(
        decimal.NewFromInt(1).Sub(decimal.NewFromFloat32(rate))
    ).IntPart()
}

// Обратная формула: work = subsidy * (1 - subsidyRate)
func getWork(subsidy int64, rate float32) int64 {
    return decimal.NewFromInt(subsidy).Mul(
        decimal.NewFromInt(1).Sub(decimal.NewFromFloat32(rate))
    ).IntPart()
}
```

### Пример снижения по стадиям

```
Параметры:
- TotalSubsidySupply = 10,000,000
- StageCutoff = 0.10 (10% на стадию)
- StageDecrease = 0.05 (5% снижения)
- CurrentSubsidyPercentage = 0.80 (80%)

Стадия 0 (0 - 1,000,000):
- Ставка: 80%
- work=100 → subsidy=400 → total=500

Стадия 1 (1,000,000 - 2,000,000):
- Ставка: 80% * 0.95 = 76%
- work=100 → subsidy=316.67 → total=416.67

Стадия 2 (2,000,000 - 3,000,000):
- Ставка: 76% * 0.95 = 72.2%
- work=100 → subsidy=259.35 → total=359.35

... и так далее до исчерпания субсидий
```

### Пересечение границы

```
Сценарий:
- TotalSubsidyPaid = 990,000
- NextCutoff = 1,000,000
- WorkCoins = 100
- CurrentRate = 0.80

Расчет:
1. subsidyAtCurrentRate = 100 / (1 - 0.80) = 500
2. 990,000 + 500 = 990,500 < 1,000,000 → НЕТ пересечения
   → Результат: 500

Сценарий с пересечением:
- TotalSubsidyPaid = 999,800
- NextCutoff = 1,000,000
- WorkCoins = 100
- CurrentRate = 0.80

Расчет:
1. subsidyAtCurrentRate = 500
2. 999,800 + 500 = 1,000,300 > 1,000,000 → ПЕРЕСЕЧЕНИЕ

3. subsidyUntilCutoff = 1,000,000 - 999,800 = 200
4. workUntilNextCutoff = 200 * (1 - 0.80) = 40
5. nextRate = 0.80 * 0.95 = 0.76
6. remainingWork = 100 - 40 = 60
7. subsidyAtNextRate = 60 / (1 - 0.76) = 250

Результат: 200 + 250 = 450
```

## 5. Штрафы за простои

### Алгоритм

```go
func CheckAndPunishForDowntimeForParticipant(
    participant Participant,
    rewards uint64,
    p0 float64,
    logger log.Logger,
) uint64 {
    stats := participant.CurrentEpochStats
    if stats == nil {
        return rewards
    }

    // 1. Расчет процента простоя
    totalRequests := stats.MissedRequests + stats.CompletedRequests
    if totalRequests == 0 {
        return rewards
    }

    downtimePercentage := float64(stats.MissedRequests) / float64(totalRequests)

    // 2. Применение SPRT (Sequential Probability Ratio Test) для простоя
    if downtimePercentage > params.DowntimeBadPercentage {
        // Снижение вознаграждений на основе серьезности простоя
        punishment := CalculateDowntimePunishment(downtimePercentage, params)
        reducedRewards := float64(rewards) * (1 - punishment)

        logger.Info("Downtime punishment applied",
            "participant", participant.Address,
            "downtime_percentage", downtimePercentage,
            "punishment", punishment,
            "original_rewards", rewards,
            "reduced_rewards", reducedRewards,
        )

        return uint64(reducedRewards)
    }

    return rewards
}

func CalculateDowntimePunishment(downtimePercentage float64, params ValidationParams) float64 {
    // Линейная интерполяция штрафа
    goodPercentage := params.DowntimeGoodPercentage
    badPercentage := params.DowntimeBadPercentage

    if downtimePercentage <= goodPercentage {
        return 0.0  // Без штрафа
    }

    if downtimePercentage >= badPercentage {
        return 1.0  // Максимальный штраф (100%)
    }

    // Интерполяция между good и bad
    rangeSize := badPercentage - goodPercentage
    excess := downtimePercentage - goodPercentage
    punishmentFraction := excess / rangeSize

    return punishmentFraction
}
```

### Пример

```
Параметры:
- DowntimeGoodPercentage = 0.05 (5%)
- DowntimeBadPercentage = 0.20 (20%)

Сценарии:
1. downtimePercentage = 0.03 (3%)
   → punishment = 0.0 (без штрафа)
   → rewards = 1000 * (1 - 0.0) = 1000

2. downtimePercentage = 0.10 (10%)
   → excess = 0.10 - 0.05 = 0.05
   → rangeSize = 0.20 - 0.05 = 0.15
   → punishment = 0.05 / 0.15 = 0.333
   → rewards = 1000 * (1 - 0.333) = 667

3. downtimePercentage = 0.25 (25%)
   → punishment = 1.0 (максимальный штраф)
   → rewards = 1000 * (1 - 1.0) = 0
```

## Заключение

Алгоритмы inference-chain обеспечивают:

1. **Справедливую репутацию:** Прогрессивное увеличение доверия с накоплением честного поведения
2. **Статистическую строгость:** Использование Z-оценок и проверки гипотез для обнаружения нечестности
3. **Эффективную валидацию:** Адаптивное количество валидаций на основе репутации
4. **Экономическое выравнивание:** Снижающиеся субсидии стимулируют раннее участие
5. **Штрафы за производительность:** Простои наказываются пропорционально серьезности

Все алгоритмы детерминированы, верифицируемы и устойчивы к манипуляциям, обеспечивая безопасность и честность сети.
