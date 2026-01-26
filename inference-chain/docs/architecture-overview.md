# Inference Chain - Обзор архитектуры

## Введение

Inference-chain - это **блокчейн на базе Cosmos SDK**, предназначенный для децентрализованного AI-инференса с механизмом консенсуса Proof of Work 2.0. Цепь построена на:

- **Cosmos SDK v0.50+** с современными паттернами dependency injection
- **CometBFT** (ранее Tendermint) для консенсуса
- **CosmWasm** для функциональности смарт-контрактов
- **7 кастомных Cosmos SDK модулей** + стандартные модули Cosmos

## Структура директорий

```
inference-chain/
├── app/                    # Конфигурация и настройка приложения
├── cmd/inferenced/         # CLI бинарник и команды
├── proto/inference/        # Protocol Buffer определения (73 .proto файла)
├── x/                      # Кастомные Cosmos SDK модули
│   ├── bls/               # BLS подписи и пороговая криптография
│   ├── bookkeeper/        # Аудит транзакций и логирование
│   ├── collateral/        # Управление залогом участников
│   ├── genesistransfer/   # Передача владения genesis-аккаунтами
│   ├── inference/         # Основная логика инференса и PoC (670 файлов)
│   ├── restrictions/      # Ограничения переводов во время bootstrap
│   └── streamvesting/     # Управление графиками vesting
├── api/                    # Сгенерированные Go-биндинги из proto
└── testutil/              # Утилиты для тестирования
```

## Кастомные модули

### 1. BLS (x/bls)

**Назначение:** Реализация схемы пороговых BLS-подписей для распределенной генерации ключей (DKG) и групповой валидации.

**Расположение:** `x/bls/keeper/keeper.go`

**Ключевые возможности:**
- Distributed Key Generation (DKG) с фазами DEALING → VERIFYING → COMPLETED
- Порог участия >50% для перехода между фазами
- Групповые публичные ключи для коллективной валидации
- Пороговые подписи для безопасности сети

**Основные типы сообщений:**
- `MsgSubmitDealerPart` - отправка полиномиальных commitment'ов и зашифрованных долей
- `MsgSubmitVerificationVector` - подтверждение завершения верификации
- `MsgSubmitGroupKeyValidationSignature` - отправка частичных подписей для валидации группового ключа
- `MsgRequestThresholdSignature` - запрос пороговой подписи от сети

### 2. Bookkeeper (x/bookkeeper)

**Назначение:** Прозрачный аудит транзакций с двойной бухгалтерией для всех переводов токенов.

**Расположение:** `x/bookkeeper/keeper/keeper.go`

**Ключевые возможности:**
- Обертывание всех банковских операций для логирования
- Двойная запись (debit/credit) для каждой транзакции
- Отслеживание под-аккаунтов модулей (например, collateral_unbonding, vesting_holding)
- Настраиваемые уровни логирования (info/debug/error/warn)

**Обернутые операции:**
- `SendCoins()` - переводы пользователь → пользователь
- `SendCoinsFromModuleToAccount()` - переводы модуль → пользователь
- `SendCoinsFromAccountToModule()` - переводы пользователь → модуль
- `SendCoinsFromModuleToModule()` - переводы модуль → модуль
- `MintCoins()` - минт токенов (логируется как "supply" → модуль)
- `BurnCoins()` - сжигание токенов (логируется как модуль → "supply")

### 3. Collateral (x/collateral)

**Назначение:** Управление депозитами залога участников, периодами unbonding, слэшингом и механизмами тюрьмы (jailing).

**Расположение:** `x/collateral/keeper/keeper.go`

**Хранилище состояния:**
- **Active Collateral:** `CollateralMap` - текущий залог в бонде для каждого участника
- **Unbonding Queue:** `UnbondingIM` - индексированная карта по (completionEpoch, participant)
- **Jailed Set:** отслеживание заблокированных участников (KeySet для O(1) поиска)
- **Slash Tracking:** `SlashedInEpoch` - предотвращение дублирования слэшинга

**Основные функции:**

```go
// Управление залогом
SetCollateral(ctx, address, amount)
GetCollateral(ctx, address) (sdk.Coin, bool)
RemoveCollateral(ctx, address)

// Управление unbonding
AddUnbondingCollateral(ctx, address, completionEpoch, amount)
GetUnbondingByEpoch(ctx, epoch) []UnbondingCollateral
ProcessUnbondingQueue(ctx, completionEpoch)

// Слэшинг
Slash(ctx, address, slashFraction, reason) (slashedAmount, error)

// Jail
SetJailed(ctx, address)
IsJailed(ctx, address) bool
RemoveJailed(ctx, address)

// Продвижение эпох
AdvanceEpoch(ctx, completedEpoch)
```

**Логика слэшинга:**
1. Проверка на повторный слэшинг (по эпохе + причине)
2. Слэшинг активного залога
3. Пропорциональный слэшинг ВСЕХ unbonding записей
4. Сжигание слэшнутых токенов из модульного аккаунта
5. Маркировка как слэшнутый для текущей эпохи + причины

### 4. GenesisTransfer (x/genesistransfer)

**Назначение:** Безопасный, атомарный, одноразовый перенос владения genesis-аккаунтами, включая ликвидные балансы и графики vesting.

**Расположение:** `x/genesistransfer/keeper/keeper.go`

**Ключевые особенности:**
- **Одноразовое исполнение:** предотвращение дублирующих переносов с отслеживанием записей
- **Сохранение vesting:** сохранение временных линий vesting при переносе владения
- **Обход ограничений:** использование модульного аккаунта-посредника для обхода bootstrap-ограничений
- **Опциональный whitelist:** контролируемый governance список разрешенных аккаунтов

**Workflow переноса:**

```
1. Фаза валидации:
   - Проверка форматов адресов
   - Верификация одноразового исполнения
   - Проверка существования аккаунта с балансом
   - Проверка опционального whitelist

2. Атомарный перенос (двухшаговый для обхода ограничений):
   Шаг 1: Genesis Account → GenesisTransfer Module Account
   Шаг 2: GenesisTransfer Module Account → Recipient

3. Перенос графика vesting:
   - Для PeriodicVestingAccount: пропорциональный расчет оставшихся периодов
   - Для ContinuousVestingAccount: сохранение линейного vesting
   - Для DelayedVestingAccount: перенос cliff vesting
   - Создание нового vesting-аккаунта для получателя

4. Ведение записей:
   - Сохранение записи о переносе
   - Emit события для audit trail
```

**Поддерживаемые типы vesting-аккаунтов:**
- `PeriodicVestingAccount` - несколько периодов vesting
- `ContinuousVestingAccount` - линейный vesting во времени
- `DelayedVestingAccount` - cliff vesting с одним релизом
- `BaseVestingAccount` - базовый функционал vesting

### 5. Inference (x/inference) - ОСНОВНОЙ МОДУЛЬ

**Назначение:** Ядро бизнес-логики для управления инференсом, валидацией, PoC и вознаграждениями.

**Расположение:** `x/inference/keeper/keeper.go`

**Масштаб:** Самый большой и сложный модуль:
- **670 Go-файлов**
- **39 proto-файлов**
- **62 типа сообщений**
- **50+ collections для состояния**

**Основные категории функциональности:**

1. **Жизненный цикл инференса:**
   - `MsgStartInference` - запрос выполнения инференса
   - `MsgFinishInference` - завершение инференса и отправка результата
   - `MsgValidation` - отправка результата валидации
   - `MsgInvalidateInference` - маркировка инференса как невалидного
   - `MsgRevalidateInference` - ревалидация ранее инвалидированного инференса

2. **Управление участниками:**
   - `MsgSubmitNewParticipant` - регистрация как участник с залогом
   - `MsgClaimRewards` - получение вознаграждений эпохи с раскрытием seed

3. **Proof of Contribution (PoC):**
   - `MsgSubmitPocBatch` - отправка PoC-батча (nonce'ы доказывающие вычисления)
   - `MsgSubmitPocValidation` - валидация PoC-батча другого участника
   - `MsgSubmitSeed` - отправка случайного seed для выбора валидации

4. **Модели и ценообразование:**
   - `MsgRegisterModel` - регистрация AI-модели для инференса
   - `MsgSubmitUnitOfComputePriceProposal` - предложение цены для модели

5. **Обучение (децентрализованное обучение):**
   - `MsgCreateTrainingTask` - создание новой задачи обучения
   - `MsgJoinTraining` - присоединение к задаче обучения
   - `MsgAssignTrainingTask` - назначение обучения участнику
   - `MsgSubmitTrainingKvRecord` - отправка записи ключ-значение обучения
   - `MsgTrainingHeartbeat` - сигнал прогресса обучения
   - `MsgSetBarrier` - установка барьера синхронизации для обучения

6. **Мост (кросс-чейн):**
   - `MsgBridgeExchange` - обмен токенами через мост
   - `MsgRequestBridgeWithdrawal` - запрос вывода на внешнюю цепь
   - `MsgRequestBridgeMint` - запрос минта с моста

**Ключевые collections:**
```go
type Keeper struct {
    Participants                  collections.Map[sdk.AccAddress, types.Participant]
    RandomSeeds                   collections.Map[Pair[uint64, sdk.AccAddress], types.RandomSeed]
    PoCBatches                    collections.Map[Triple[int64, sdk.AccAddress, string], types.PoCBatch]
    PoCValidations                collections.Map[Triple[int64, sdk.AccAddress, sdk.AccAddress], types.PoCValidation]
    Models                        collections.Map[string, types.Model]
    Inferences                    collections.Map[string, types.Inference]
    InferenceTimeouts             collections.Map[Pair[uint64, string], types.InferenceTimeout]
    InferenceValidationDetailsMap collections.Map[Pair[uint64, string], types.InferenceValidationDetails]
    EpochGroupDataMap             collections.Map[Pair[uint64, string], types.EpochGroupData]
    Epochs                        collections.Map[uint64, types.Epoch]
    EffectiveEpochIndex           collections.Item[uint64]
    SettleAmounts                 collections.Map[sdk.AccAddress, types.SettleAmount]
    TopMiners                     collections.Map[sdk.AccAddress, types.TopMiner]
    // ... и множество других
}
```

### 6. Restrictions (x/restrictions)

**Назначение:** Временные ограничения на переводы токенов пользователь→пользователь во время bootstrap блокчейна с сохранением необходимых сетевых операций.

**Расположение:** `x/restrictions/keeper/keeper.go`

**Ключевые особенности:**
- **SendRestriction Function:** перехватывает все банковские переводы
- **Автоматическая дерегистрация:** удаление функции ограничения по истечении периода
- **Аварийные исключения:** утвержденные governance шаблоны аварийных переводов
- **Нулевые накладные расходы после истечения:** автоматически дерегистрируется для производительности

**Логика переводов:**

```
Разрешено (всегда проходит):
1. Оплата комиссий за газ → модуль fee collector
2. Переводы Пользователь → Модуль (стейкинг, governance и т.д.)
3. Переводы Модуль → Пользователь (вознаграждения, возвраты)
4. Совпадения с аварийными исключениями

Ограничено (заблокировано во время bootstrap):
1. Прямые переводы Пользователь → Пользователь

Жизненный цикл:
- Ограничения активны до RestrictionEndBlock
- SendRestriction авто-дерегистрируется по истечении
- Emit события: "restriction_lifted"
```

### 7. StreamVesting (x/streamvesting)

**Назначение:** Управление графиками vesting с разблокировкой на основе эпох для вознаграждений участников.

**Расположение:** `x/streamvesting/keeper/keeper.go`

**Ключевые типы:**
```go
type VestingSchedule struct {
    ParticipantAddress string
    EpochAmounts       []EpochCoins  // Монеты для разблокировки по эпохам
}

type EpochCoins struct {
    Coins sdk.Coins
}
```

**Основные функции:**

1. **Добавление вознаграждений с vesting (с агрегацией):**
```go
func AddVestedRewards(ctx, participantAddress, fundingModule, amount, vestingEpochs, memo) {
    // 1. Перевод из модуля финансирования в модуль streamvesting
    SendCoinsFromModuleToModule(fundingModule, streamvesting, amount)

    // 2. Определение эпох vesting
    epochs := vestingEpochs ?? params.RewardVestingPeriod

    // 3. Получение или создание графика vesting
    schedule := GetVestingSchedule(participantAddress) ?? CreateNew()

    // 4. Расширение графика при необходимости
    while len(schedule.EpochAmounts) < epochs:
        schedule.EpochAmounts.append(EpochCoins{Coins: NewCoins()})

    // 5. Разделение и агрегация вознаграждений по эпохам
    for each coin in amount:
        amountPerEpoch := coin.Amount / epochs
        remainder := coin.Amount % epochs

        for i in 0..epochs:
            epochCoin := NewCoin(coin.Denom, amountPerEpoch)
            if i == 0 && remainder > 0:
                epochCoin += remainder

            // Агрегация с существующими суммами
            schedule.EpochAmounts[i].Coins += epochCoin

    // 6. Сохранение обновленного графика
    SetVestingSchedule(schedule)
}
```

2. **Обработка разблокировок по эпохам:**
```go
func ProcessEpochUnlocks(ctx) {
    schedules := GetAllVestingSchedules()

    for each schedule in schedules:
        if len(schedule.EpochAmounts) == 0:
            continue

        // Получение монет первой эпохи
        coinsToUnlock := schedule.EpochAmounts[0].Coins

        if !coinsToUnlock.IsZero():
            // Перевод от модуля к участнику
            SendCoinsFromModuleToAccount(streamvesting, participant, coinsToUnlock)

        // Удаление первой эпохи
        schedule.EpochAmounts = schedule.EpochAmounts[1:]

        // Обновление или удаление графика
        if len(schedule.EpochAmounts) == 0:
            RemoveVestingSchedule(participant)
        else:
            SetVestingSchedule(schedule)
}
```

**Интеграция с модулем Inference:**
- Вызывается функцией `AdvanceEpoch()` модуля inference во время переходов эпох
- Разблокирует одну эпоху токенов с vesting для всех участников
- Поддерживает несколько одновременных графиков vesting для каждого участника (агрегировано)

## Архитектурные паттерны

### Dependency Injection

Приложение использует современный `depinject` для связывания модулей:

```go
func AppConfig() depinject.Config {
    return depinject.Configs(
        appConfig,
        depinject.Supply(
            map[string]module.AppModuleBasic{
                genutiltypes.ModuleName: genutil.NewAppModuleBasic(...),
                govtypes.ModuleName:     gov.NewAppModuleBasic(...),
            },
        ),
    )
}
```

### Collections Framework

Современное управление состоянием с типобезопасностью:

```go
// Простые карты ключ-значение
Participants   collections.Map[sdk.AccAddress, types.Participant]

// Композитные ключи
RandomSeeds    collections.Map[collections.Pair[uint64, sdk.AccAddress], types.RandomSeed]

// Индексированные карты для эффективных запросов
UnbondingIM    collections.IndexedMap[
    collections.Pair[uint64, sdk.AccAddress],
    types.UnbondingCollateral,
    UnbondingIndexes  // Вторичный индекс: ByParticipant
]

// KeySets для отслеживания членства (O(1) поиск)
Jailed         collections.KeySet[sdk.AccAddress]

// Singleton элементы
EffectiveEpochIndex collections.Item[uint64]
```

### Двойная бухгалтерия

Модуль Bookkeeper оборачивает все банковские операции:

```go
func (k Keeper) logTransaction(ctx, to, from, coin, memo, subAccount) {
    if k.logConfig.DoubleEntry {
        // Дебетовая запись: to аккаунт получает
        log("type", "debit", "account", to, "amount", amount, "memo", memo)
        // Кредитовая запись: from аккаунт отправляет
        log("type", "credit", "account", from, "amount", -amount, "memo", memo)
    }
}
```

## Интеграция с Cosmos SDK

### Порядок инициализации модулей

```go
var genesisModuleOrder = []string{
    // Модули Cosmos SDK / IBC
    capabilitytypes.ModuleName,
    authtypes.ModuleName,
    banktypes.ModuleName,
    // ...

    // Кастомные модули цепи
    bookkeepermoduletypes.ModuleName,    // Сначала аудит транзакций
    blstypes.ModuleName,                  // BLS криптография
    inferencemoduletypes.ModuleName,      // Основная логика инференса
    collateralmoduletypes.ModuleName,     // Управление залогом
    wasmtypes.ModuleName,                 // Смарт-контракты
    streamvestingmoduletypes.ModuleName,  // Графики vesting
    restrictionsmoduletypes.ModuleName,   // Ограничения переводов
    genesistransfermoduletypes.ModuleName, // Перенос genesis-аккаунтов
}
```

### Порядок выполнения EndBlock

```go
var endBlockers = []string{
    crisistypes.ModuleName,
    govtypes.ModuleName,
    stakingtypes.ModuleName,

    // Кастомные модули
    blstypes.ModuleName,
    inferencemoduletypes.ModuleName,  // Критично: обрабатывает переходы эпох
    feegrant.ModuleName,
    group.ModuleName,
    wasmtypes.ModuleName,
}
```

## Protocol Buffers структура

**Всего Proto-файлов:** 73 .proto файла во всех модулях

```
proto/inference/
├── bls/                    # 9 proto файлов
├── bookkeeper/            # 5 proto файлов
├── collateral/            # 8 proto файлов
├── genesistransfer/       # 5 proto файлов
├── inference/             # 39 proto файлов (самый большой)
├── restrictions/          # 5 proto файлов
└── streamvesting/         # 6 proto файлов
```

## Заключение

Inference-chain представляет собой сложный, production-ready блокчейн, объединяющий:

- **Инновационный консенсус:** Proof of Work 2.0 через AI-вычисления
- **Передовую криптографию:** BLS пороговые подписи
- **Продвинутую экономику:** Подсистемы vesting, collateral и rewards
- **Прозрачность:** Полный аудит через двойную бухгалтерию
- **Защиту bootstrap:** Временные ограничения с аварийными исключениями
- **Модульность:** 7 специализированных модулей на современном Cosmos SDK

Архитектура спроектирована для масштабируемости, безопасности и децентрализации AI-инфраструктуры.
