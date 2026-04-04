# Документация: Frontend-модули

Этот документ описывает стандартный подход к созданию модулей в `LCMF`.
Frontend-модуль представляет собой самодостаточную единицу глобального
состояния со своей сетевой частью, которая взаимодействует с другими модулями
прежде всего через шину сообщений. Публичные синхронные чтения используются как
дополнительный механизм для отдельных read-case.

Если вам нужна не module-level форма, а целостная картина app-level сборки
вокруг `init!`, сначала прочитайте [`APP.md`](./APP.md).

## 1. Анатомия модуля

Каждый модуль должен предоставлять функцию инициализации `init!`, которая
служит его точкой входа. Эта функция отвечает за подключение модуля к шине,
подготовку сетевых операций и регистрацию тех публичных чтений, которые модуль
действительно экспортирует наружу.

### Функция `init!`

Сигнатура:

```clojure
(defn init! [deps])
```

Параметр `deps` — карта зависимостей.

Обязательные ключи:

- `:bus` — экземпляр frontend message bus
- `:registry` — реестр публичных чтений
- `:http` — app-level HTTP runtime
- `:logger` — функция для технических сообщений

Возможные дополнительные ключи:

- `:ws` — app-level realtime runtime
- `:config` — уже загруженный конфиг приложения или модуля

Внутри `init!` модуль обычно делает следующее:

- подписывается на нужные сообщения шины;
- регистрирует свои публичные чтения;
- объявляет обязательные чтения, которые ему нужны от других модулей;
- подготавливает свои HTTP-операции;
- возвращает карту с модульными дескрипторами, если это предусмотрено
  приложением.

### Пример общей формы модуля

Это пример целого `LCMF`-модуля.

```clojure
(ns booking.module
  (:require [lcmf.bus :as bus]
            [lcmf.registry :as registry]
            [lcmf.http :as http]))

(def module-id :booking)

(def initial-module-state
  {:items {}
   :create-request {:status :idle
                    :error nil}})

(defn make-module-state []
  (atom initial-module-state))

(defn- public-state [state]
  {:bookings (vals (:items @state))})

(defn- register-public-reads! [registry state]
  (registry/register-provider!
   registry
   {:provider-id :booking/list-bookings
    :module module-id
    :provider-fn (fn [_]
                   (public-state state))}))

(defn init!
  [{:keys [bus registry http logger]}]
  (logger :info {:component ::booking
                 :event :module-initializing})

  (let [state (make-module-state)]
    (register-public-reads! registry state)

    (logger :info {:component ::booking
                   :event :module-initialized})

    {:module module-id
     :state state}))
```

## 2. Работа с шиной сообщений

Шина является главным способом межмодульной координации в `LCMF`.

Модуль может публиковать сообщения о произошедших фактах и подписываться на
сообщения других модулей. При этом модуль не должен знать, кто именно
подписан на его сообщения.

### Пример публикации сообщения

Это пример внутренней функции `LCMF`-модуля.

```clojure
(defn- publish-booking-created! [bus-instance booking]
  (bus/publish! bus-instance
                :booking/created
                {:id (:id booking)
                 :slot-id (:slot-id booking)
                 :user-id (:user-id booking)}
                {:module module-id}))
```

### Пример реакции на сообщение другого модуля

Это пример внутреннего обработчика `LCMF`-модуля.

```clojure
(defn- handle-account-signed-out [state _bus envelope]
  (let [user-id (get-in envelope [:payload :user-id])]
    (swap! state update :items
           (fn [items]
             (into {}
                   (remove (fn [[_ booking]]
                             (= user-id (:user-id booking))))
                   items)))))

(defn- subscribe! [bus-instance state]
  (bus/subscribe! bus-instance
                  :accounts/signed-out
                  (fn [bus envelope]
                    (handle-account-signed-out state bus envelope))
                  {:module module-id}))
```

Если обработчик сообщения публикует новое сообщение, он должен сохранять
причинно-следственную связность через `:parent-envelope`, если это предусмотрено
API конкретной библиотеки шины.

## 3. Чем модуль владеет

Модуль в `LCMF` владеет своим участком глобального состояния и своей сетевой
частью.

Это означает, что модуль сам определяет:

- начальное состояние своего участка;
- допустимые переходы состояния;
- форму своих внутренних данных;
- свои сообщения;
- свои HTTP-операции;
- свои публичные чтения.

Другой модуль должен зависеть не от внутренней формы этого участка состояния, а
от публичных контрактов модуля.

### Пример контролируемого обновления состояния модуля

Это пример внутренних функций `LCMF`-модуля, а не app-level кода.

```clojure
(defn- mark-create-pending! [state]
  (swap! state assoc
         :create-request {:status :pending
                          :error nil}))

(defn- store-created-booking! [state booking]
  (swap! state
         (fn [slice]
           (-> slice
               (assoc-in [:items (:id booking)] booking)
               (assoc :create-request {:status :ok
                                       :error nil})))))
```

Такой подход нужен, чтобы ownership участка состояния оставался у модуля, а
изменения не превращались в произвольную запись во внутреннее состояние из
внешнего кода.

## 4. Что модуль экспортирует наружу

Наружу модуль экспортирует не весь свой участок состояния, а только явно
объявленные публичные поверхности.

В `LCMF` такими поверхностями являются:

- сообщения, которые модуль публикует в шину;
- публичные чтения через `registry`;
- модульные функции верхнего уровня, если они явно входят в его контракт.

Публичное чтение должно давать достаточно данных для межмодульной координации,
но не превращаться в главный способ архитектурной координации. В `LCMF` эту
роль выполняет шина.

### Пример публичного чтения

Это пример внутренней регистрации публичных чтений внутри `LCMF`-модуля.

```clojure
(defn- get-booking-by-id [state booking-id]
  (get-in @state [:items booking-id]))

(defn- register-public-reads! [registry state]
  (registry/register-provider!
   registry
   {:provider-id :booking/get-by-id
    :module module-id
    :provider-fn (fn [{:keys [booking-id]}]
                   (when booking-id
                     (get-booking-by-id state booking-id)))})
  (registry/register-provider!
   registry
   {:provider-id :booking/list-bookings
    :module module-id
    :provider-fn (fn [_]
                   (vals (:items @state)))}))
```

## 5. Что остается внутренним

Внутренними для модуля остаются:

- точная форма его участка состояния;
- внутренние функции переходов;
- служебные поля сетевых операций;
- промежуточные вычисления;
- внутренняя организация файла или файлов модуля.

Другой модуль не должен читать эти данные напрямую и не должен строить свою
логику на знании внутренней структуры чужого состояния.

## 6. Работа с HTTP

HTTP-операции являются частью модуля, а не внешним случайным слоем вокруг
него.

Модуль сам определяет:

- какие запросы ему нужны;
- как нормализуется их результат;
- как успех и ошибка отражаются в его участке состояния;
- какие сообщения публикуются по итогам операции.

### Пример HTTP-операции модуля

Это пример публичной операции `LCMF`-модуля.

```clojure
(defn create-booking!
  [{:keys [state http bus]} payload]
  (mark-create-pending! state)
  (-> (http/request! http
                     {:method :post
                      :path "/bookings"
                      :json payload})
      (.then (fn [booking]
               (store-created-booking! state booking)
               (publish-booking-created! bus booking)
               booking))
      (.catch (fn [error]
                (swap! state assoc
                       :create-request {:status :error
                                        :error error})
                (throw error)))))
```

Внешний код может вызывать такую операцию как публичную функцию модуля, но сам
не должен напрямую решать, как именно обновляется состояние модуля `booking`.

## 7. Обязательные межмодульные чтения

Если модулю нужны публичные данные другого модуля, он должен явно объявить эту
зависимость через `registry`.

Это нужно для того, чтобы приложение могло проверить все обязательные
межмодульные чтения на старте.

Важно: `registry` не заменяет `bus`. Если через публичные чтения начинает
проходить основная координация между модулями, это сигнал пересмотреть
границы модулей или сценарий взаимодействия.

### Пример объявления зависимостей

Это пример кода `LCMF`-модуля на этапе `init!`.

```clojure
(defn- declare-requirements! [registry-instance]
  (registry/declare-requirements! registry-instance
                                  module-id
                                  #{:accounts/get-by-id
                                    :catalog/get-slot-by-id}))

(defn init!
  [{:keys [bus registry http logger]}]
  (logger :info {:component ::booking
                 :event :module-initializing})

  (let [state (make-module-state)]
    (declare-requirements! registry)
    (register-public-reads! registry state)
    (subscribe! bus state)

    (logger :info {:component ::booking
                   :event :module-initialized})
    {:module module-id
     :state state
     :ops {:create-booking! (fn [payload]
                              (create-booking! {:state state
                                                :http http
                                                :bus bus}
                                               payload))}}))
```

Само приложение после инициализации модулей должно выполнить общую проверку
обязательных зависимостей.

## 8. Наблюдаемость и технические сообщения

Модуль должен использовать переданный `:logger` для технических сообщений и не
создавать свою отдельную систему журналирования.

Практическое правило такое:

- модуль пишет структурированные технические сообщения;
- модуль использует единые `correlation-id` и `request-id`, если они доступны;
- модуль не задает глобальную policy наблюдаемости на уровне приложения.

### Пример технического сообщения

Это пример внутреннего технического сообщения `LCMF`-модуля.

```clojure
(logger :info {:component ::booking
               :event :booking/create-started
               :slot-id (:slot-id payload)
               :user-id (:user-id payload)})
```

## 9. Минимальный итог

Хороший frontend-модуль в `LCMF` — это модуль глобального store со своей
сетевой частью, который:

- владеет своим участком состояния;
- экспортирует только публичные чтения и осознанные публичные операции;
- координируется с другими модулями через шину;
- читает чужие данные только через публичный контракт;
- получает app-level зависимости через `init!`.
