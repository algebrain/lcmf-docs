# Документация: App-level слой frontend

Этот документ описывает app-level слой в `LCMF`.

Речь здесь идет о той части frontend-приложения, которая собирает модули
состояния в одну систему, создает общую шину, создает runtime состояния и
передает модулям общие зависимости.

Если вам нужна общая архитектурная рамка, сначала прочитайте
[`ARCH.md`](./ARCH.md).
Если вам нужна форма отдельного модуля, сначала прочитайте
[`MODULE.md`](./MODULE.md).
Если вам нужен механизм публичных синхронных чтений, переходите в
[`REGISTRY.md`](./REGISTRY.md).

## 1. Что такое app-level слой в `LCMF`

App-level слой в `LCMF` — это слой сборки приложения.

Он не владеет бизнес-логикой модулей. Его задача:

- создать общую шину;
- собрать runtime состояния;
- создать `registry` публичных чтений;
- создать HTTP и realtime runtime;
- вызвать `init!` модулей;
- выполнить стартовые проверки;
- собрать целое приложение из независимых модулей.

Коротко:

- app owns infrastructure
- modules own state and behavior

## 2. Что именно собирает приложение

Обычная картина app-level слоя такая:

- `bus`
- `registry`
- runtime состояния
- HTTP runtime
- realtime runtime, если он нужен
- `logger`
- конфиг
- app-level wiring между этими частями

При этом приложение не должно забирать ownership состояния у модулей. Оно
создает среду, в которой модули подключаются к системе и дальше владеют
своими переходами состояния, сообщениями и сетевыми операциями.

## 3. App-level слой и состояние

App-level слой собирает модульные участки состояния в runtime приложения.

Это не означает, что `LCMF`-модули получают право читать внутреннее состояние
друг друга напрямую. Это означает только то, что приложение собирает
инфраструктуру, внутри которой каждый модуль владеет своим участком.

На этом этапе `LCMF` не обязан фиксировать одну физическую форму runtime
состояния. Возможны разные реализации, пока сохраняются базовые инварианты:

- ownership состояния остается у модуля;
- межмодульная координация идет через `bus`;
- публичные синхронные чтения идут через `registry`;
- внутреннее состояние другого модуля не читается напрямую.

### Пример простой app-level формы состояния

Это app-level пример. Он показывает форму состояния, которое собирает
приложение, а не внутренний код отдельного `LCMF`-модуля.

```clojure
(defn make-app-state []
  {:booking (atom booking.state/initial-state)
   :accounts (atom accounts.state/initial-state)
   :catalog (atom catalog.state/initial-state)})
```

Это только пример формы. Важен не `atom` как таковой, а то, что приложение
собирает модульные участки, а не превращает state runtime в новый канал
межмодульной координации.

## 4. Что `LCMF`-модуль видит из app-level слоя

`LCMF`-модуль получает от приложения общие зависимости, но не должен получать
право свободно обходить архитектурные границы.

Практически это означает:

- модуль получает `bus`;
- модуль получает `registry`;
- модуль получает HTTP runtime;
- модуль может получать доступ к своему состоянию или к выделенному runtime
  своего состояния;
- модуль не должен использовать app-level wiring как способ читать чужое
  внутреннее состояние.

### Пример допустимого shape зависимостей

Это app-level пример. Он показывает, как приложение передает зависимости в
`LCMF`-модули.

```clojure
(defn init-app! []
  (let [bus (bus/make-bus)
        registry (registry/make-registry)
        app-state (make-app-state)
        http-runtime (http/make-runtime)]
    (booking/init! {:bus bus
                    :registry registry
                    :http http-runtime
                    :state (:booking app-state)})
    (accounts/init! {:bus bus
                     :registry registry
                     :http http-runtime
                     :state (:accounts app-state)})))
```

### Пример недопустимого shape зависимостей

Это тоже app-level пример, но уже с неправильным wiring.

```clojure
;; Плохо: модуль получает весь app-state и может читать чужие внутренности.
(booking/init! {:bus bus
                :registry registry
                :http http-runtime
                :app-state app-state})
```

Во втором варианте app-level слой сам размывает архитектурную границу.

## 5. App-level слой и шина

Шина создается на уровне приложения и передается в `LCMF`-модули как общая
инфраструктурная зависимость.

Это важно по двум причинам:

- все модули координируются через один и тот же `bus`;
- шина остается app-level механизмом, а не внутренней деталью отдельного
  модуля.

### Пример app-level создания шины

Это app-level пример создания общей шины.

```clojure
(defn make-system []
  (let [app-bus (bus/make-bus)]
    {:bus app-bus}))
```

## 6. App-level слой и HTTP runtime

HTTP runtime тоже создается на уровне приложения.

Модули используют его для своих операций, но не должны каждый по-своему
создавать отдельный HTTP-клиент как способ обойти общую дисциплину ошибок,
trace context и безопасного поведения.

### Пример app-level HTTP runtime

Это app-level пример создания общего HTTP runtime.

```clojure
(defn make-system []
  (let [http-runtime (http/make-runtime {:base-url "/api"})]
    {:http http-runtime}))
```

## 7. App-level слой и стартовая последовательность

Обычный порядок app-level сборки такой:

- создать `bus`;
- создать `registry`;
- создать runtime состояния;
- создать HTTP runtime;
- создать realtime runtime, если он нужен;
- вызвать `init!` модулей;
- выполнить стартовые проверки;
- собрать финальное приложение.

### Пример app-level сборки

Это app-level пример полной стартовой последовательности.

```clojure
(defn init-app! []
  (let [bus (bus/make-bus)
        registry (registry/make-registry)
        app-state (make-app-state)
        http-runtime (http/make-runtime {:base-url "/api"})]
    (accounts/init! {:bus bus
                     :registry registry
                     :http http-runtime
                     :state (:accounts app-state)})
    (catalog/init! {:bus bus
                    :registry registry
                    :http http-runtime
                    :state (:catalog app-state)})
    (booking/init! {:bus bus
                    :registry registry
                    :http http-runtime
                    :state (:booking app-state)})
    (registry/assert-requirements! registry)
    {:bus bus
     :registry registry
     :state app-state
     :http http-runtime}))
```

## 8. Что app-level слой не должен делать

App-level слой не должен:

- забирать ownership состояния у модулей;
- превращать app-state в общий обход границ модулей;
- подменять `bus` прямыми вызовами между модулями;
- подменять `registry` прямым чтением чужих внутренних данных;
- втягивать в `LCMF` устройство компонентов и view-layer.

## 9. Минимальный итог

App-level слой в `LCMF` — это слой сборки, а не слой бизнес-логики.

Приложение создает шину, runtime состояния и сетевые runtime, а `LCMF`-модули
через эти зависимости подключаются к системе и дальше координируются прежде
всего через `bus`.
