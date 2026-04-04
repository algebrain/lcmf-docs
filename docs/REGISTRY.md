# Документация: Реестр публичных чтений

Этот документ описывает реестр публичных чтений в `LCMF`.

Речь идет о механизме, который позволяет `LCMF`-модулю получить чужие
публичные данные без прямой связанности с внутренней формой состояния другого
модуля.

Для общей архитектурной рамки см. [`ARCH.md`](./ARCH.md).
Для формы отдельного модуля см. [`MODULE.md`](./MODULE.md).
Для app-level слоя, который создает реестр и выполняет стартовую проверку, см.
[`APP.md`](./APP.md).

## 1. Назначение

`registry` нужен для контролируемого синхронного чтения между `LCMF`-модулями.

Он решает такие задачи:

- явно фиксирует, какие публичные чтения модуль предоставляет;
- явно фиксирует, какие внешние чтения модулю обязательны;
- позволяет делать fail-fast стартовую проверку до запуска приложения.

## 2. Когда использовать

Используйте `registry`, когда:

- нужен sync-read данных другого `LCMF`-модуля;
- зависимость должна быть явной и проверяемой;
- читать чужие публичные данные удобнее, чем городить для этого отдельную
  цепочку сообщений.

Не используйте `registry` для:

- основной межмодульной координации;
- side effects и реактивных процессов;
- обхода внутренних границ другого модуля;
- каскадных вызовов provider -> provider.

В `LCMF` важно держать в голове общую рамку:

- `registry` — узкий контрактный канал sync-read;
- `registry` не должен занимать место `bus`;
- если через `registry` начинает проходить все больше координации, это
  архитектурный сигнал пересмотреть границы модулей или сценарий
  взаимодействия.

## 3. Уровни применения

Реестр применяется на трех уровнях:

- app-level слой создает `registry` и выполняет стартовую проверку;
- `LCMF`-модуль-владелец регистрирует свои публичные чтения;
- `LCMF`-модуль-потребитель объявляет обязательные зависимости и получает
  provider для вызова.

Обычная картина использования такая:

- приложение создает один общий `registry`;
- модули-владельцы регистрируют свои provider-функции;
- модули-потребители объявляют обязательные чтения;
- приложение вызывает `assert-requirements!`;
- после этого provider используется в нужном модуле.

## 4. Основные функции

### `make-registry`

Создает общий in-memory реестр.

```clojure
(def registry
  (registry/make-registry))
```

Это пример app-level слоя: приложение создает один общий `registry`.

### `register-provider!`

Регистрирует публичное чтение модуля с уникальным `provider-id`.

```clojure
(registry/register-provider!
 registry
 {:provider-id :accounts/get-by-id
  :module :accounts
  :provider-fn (fn [{:keys [user-id]}]
                 (when-let [user (get-in @accounts-state [:users user-id])]
                   {:id (:id user)
                    :login (:login user)
                    :role (:role user)}))})
```

`provider-fn` должен возвращать минимально достаточные данные для внешнего
read-case.
Это пример кода `LCMF`-модуля-владельца.

### `resolve-provider`

Возвращает provider-функцию или `nil`, если она отсутствует.

```clojure
(if-let [get-user (registry/resolve-provider registry :accounts/get-by-id)]
  (get-user {:user-id "u-alice"})
  nil)
```

Это пример кода `LCMF`-модуля или app-level слоя, который мягко обрабатывает
отсутствие provider.

### `require-provider`

Возвращает provider-функцию, а при отсутствии сразу сигнализирует об ошибке.

```clojure
(let [get-user (registry/require-provider registry :accounts/get-by-id)]
  (get-user {:user-id "u-alice"}))
```

Это основной выбор для обязательных зависимостей.
Это пример кода `LCMF`-модуля-потребителя.

### `declare-requirements!`

Объявляет обязательные внешние read-зависимости `LCMF`-модуля.

```clojure
(registry/declare-requirements! registry
                                :booking
                                #{:accounts/get-by-id
                                  :catalog/get-slot-by-id})
```

Это пример `init!` уровня `LCMF`-модуля-потребителя.

### `assert-requirements!`

Выполняет fail-fast проверку готовности перед запуском приложения.

```clojure
(registry/assert-requirements! registry)
```

Эта проверка должна жить в app-level слое после `init!` модулей, но до старта
приложения.
Это app-level пример.

## 5. Что должен возвращать provider

Provider проектируется вокруг внешнего read-case, а не вокруг удобства
внутренней модели владельца.

Практические правила:

- provider возвращает минимально достаточные данные;
- provider не раскрывает весь внутренний участок состояния;
- provider не выполняет side effects;
- provider не превращается в сырой дамп внутреннего store.

### Пример хорошего provider

Это пример кода `LCMF`-модуля-владельца.

```clojure
(registry/register-provider!
 registry
 {:provider-id :accounts/get-by-id
  :module :accounts
  :provider-fn (fn [{:keys [user-id]}]
                 (when-let [user (get-in @accounts-state [:users user-id])]
                   {:id (:id user)
                    :login (:login user)
                    :role (:role user)}))})
```

### Пример плохого provider

Это тоже пример кода `LCMF`-модуля-владельца, но с неправильной формой
публичного чтения.

```clojure
(registry/register-provider!
 registry
 {:provider-id :accounts/raw-state
  :module :accounts
  :provider-fn (fn [_]
                 @accounts-state)})
```

Во втором варианте модуль раскрывает весь свой внутренний участок состояния и
ломает границу владения.

## 6. Связь с `bus`

`registry` не заменяет `bus`.

В `LCMF` роли разделены так:

- `bus` координирует модули;
- `registry` закрывает отдельные sync-read кейсы;
- если `registry` начинает выглядеть как главный механизм системы, это ошибка
  архитектурного акцента.

### Пример хорошего разделения ролей

Это пример кода `LCMF`-модуля-потребителя и `LCMF`-модуля-владельца, показанных
в одном фрагменте.

```clojure
;; booking читает публичные данные catalog через registry
(let [get-slot (registry/require-provider registry :catalog/get-slot-by-id)
      slot (get-slot {:slot-id "slot-09-00"})]
  ...)

;; а о результате своей работы сообщает через bus
(bus/publish! bus
              :booking/created
              {:id "b-1" :slot-id "slot-09-00" :user-id "u-alice"}
              {:module :booking})
```

## 7. Пример обычной картины использования

Это app-level пример.

```clojure
(defn init-app! []
  (let [registry (registry/make-registry)]
    (accounts/init! {:registry registry ...})
    (catalog/init! {:registry registry ...})
    (booking/init! {:registry registry ...})
    (registry/assert-requirements! registry)
    {:registry registry}))
```

Что показывает этот пример:

- приложение создает один общий `registry`;
- `LCMF`-модули-владельцы регистрируют свои публичные чтения;
- `LCMF`-модули-потребители объявляют зависимости;
- стартовая проверка живет на app-level.

## 8. Минимальный итог

`registry` в `LCMF` — это узкий контрактный механизм публичных синхронных
чтений.

Он полезен, когда модулю нужно получить чужие публичные данные, но он не
должен занимать место `bus` как главного механизма межмодульной координации.
