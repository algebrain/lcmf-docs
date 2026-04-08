# Документация: Типовое состояние запроса во frontend

Этот документ описывает роль типового request-state слоя в `LCMF`.

Речь идет о небольшом прикладном слое поверх `lcmf-http`, который помогает
модулю однообразно вести состояние выполнения запроса, не забирая у него
владение предметной логикой.

Для общей архитектурной рамки см. [`ARCH.md`](./ARCH.md).
Для формы frontend-модуля см. [`MODULE.md`](./MODULE.md).
Для HTTP-модели клиента см. [`HTTP.md`](./HTTP.md).
Для app-level слоя, который создает общий HTTP runtime, см. [`APP.md`](./APP.md).

Реализация библиотеки:

- [`lcmf-request-state`](https://github.com/algebrain/lcmf-request-state)

Важно понимать этот документ правильно:

- он не делает request-state новым центром архитектуры;
- он не подменяет `HTTP.md`;
- он не описывает кэш, общий query-layer или межмодульную координацию;
- он описывает узкий и practically useful слой типового состояния одного
  request-flow внутри модуля.

## Быстрый старт

Если нужен короткий рабочий ориентир, достаточно держать в голове такую
модель:

- один `atom` обслуживает один request-flow;
- модуль использует общий `lcmf-http` runtime;
- request-state helper ведет типовые переходы состояния;
- предметный смысл операции остается у модуля.

Минимальный пример:

```clojure
(def catalog-load-state
  (atom (request-state/init-state)))

(defn load-catalog! [http-runtime]
  (request-state/start! catalog-load-state)
  (-> (http/request! http-runtime {:method :get
                                   :path "/catalog/slots"})
      (.then (fn [response]
               (request-state/succeed! catalog-load-state
                                       (:body response)
                                       response)))
      (.catch (fn [error]
                (let [data (ex-data error)]
                  (case (:kind data)
                    :cancelled (request-state/cancel! catalog-load-state)
                    (request-state/fail! catalog-load-state data)))))))
```

Что здесь важно:

- transport остается в `lcmf-http`;
- request-state helper не решает за модуль, когда загружать данные;
- отмена не смешивается с обычной ошибкой;
- если у модуля есть еще и создание записи, для него обычно нужен другой
  `atom`.

Если нужен refresh без потери уже показанных данных:

```clojure
(request-state/start-refresh! catalog-load-state)
```

Если нужен latest-only сценарий:

```clojure
(let [run-id (request-state/begin-run! catalog-load-state)]
  (request-state/complete-run! catalog-load-state
                               run-id
                               {:status :success
                                :data {:items [...]}
                                :response response}))
```

Если после этого нужна полная картина и границы ответственности, читайте
документ дальше по порядку.

## 1. Зачем нужен отдельный request-state слой

В реальном модульном frontend-коде снова и снова повторяются одни и те же
вопросы:

- как показать первую загрузку;
- как сохранить успешные данные;
- как показать ошибку;
- как не потерять прошлые данные при refresh;
- как отличить отмену от обычной ошибки;
- как не дать более старому ответу перетереть более новый.

Если каждый модуль решает это заново, код начинает расползаться на почти
одинаковые, но несовпадающие куски логики.

Поэтому в `LCMF` допустим и полезен отдельный request-state слой, если он
сохраняет правильные границы:

- transport остается в `lcmf-http`;
- ownership состояния остается у модуля;
- предметные решения остаются у модуля;
- request-state слой помогает только с типовой механикой одного запроса.

## 2. Где request-state живет в архитектуре

Request-state слой живет между `lcmf-http` и модульной предметной логикой.

Короткая картина такая:

- app-level слой создает общий HTTP runtime;
- модуль использует этот runtime для своих операций;
- модуль хранит у себя локальный request-state;
- request-state helper помогает аккуратно проводить типовые переходы;
- модуль сам решает, как эти переходы встроены в его state и UI.

Практически это означает:

- `lcmf-http` отвечает за запрос, ответ, ошибку и trace-метаданные;
- request-state слой отвечает за `:idle`, `:pending`, `:success`, `:error`,
  `:cancelled` и related transitions;
- модуль отвечает за смысл операции, refresh policy, публикацию событий и
  связь с экраном.

## 3. Главная рекомендуемая модель

Для малого и среднего проекта полезнее всего такая модель:

- один `atom` обслуживает один request-flow.

Это означает:

- загрузка списка — это один request-flow;
- отправка формы — другой request-flow;
- удаление записи — третий request-flow;
- latest-only сценарий живет внутри этого же одного flow.

Практический пример:

```clojure
(def catalog-load-state
  (atom (request-state/init-state)))

(def booking-create-state
  (atom (request-state/init-state)))
```

Почему это хороший базовый режим:

- он проще в голове;
- он проще в UI;
- он лучше совпадает с реальными пользовательскими сценариями;
- он не создает ложной многозадачности в одном общем state.

Для `LCMF` это особенно важно, потому что модуль должен оставаться маленьким и
ясным по ownership.

## 4. Что request-state слой должен уметь

Минимально practically useful request-state слой должен уметь:

- создать начальное состояние;
- перевести его в первую загрузку;
- сохранить успех;
- сохранить ошибку без автоматического стирания прошлых данных;
- отдельно выразить отмену;
- отдельно выразить фоновое обновление;
- помочь latest-only сценарию не пропускать устаревший результат.

Типовая форма состояния может выглядеть так:

```clojure
{:status :idle
 :data nil
 :error nil
 :revalidating? false
 :started-at nil
 :finished-at nil
 :last-success-at nil
 :request-id nil
 :correlation-id nil
 :retry-after-sec nil
 :current-run-id nil
 :last-completed-run-id nil}
```

Практически здесь важны две вещи:

- набор основных статусов;
- отдельный флаг `:revalidating?` вместо попытки сделать "обновление" еще
  одним самостоятельным статусом.

## 5. Что request-state слой не должен делать

Request-state слой в `LCMF` не должен:

- становиться общим кэшем приложения;
- становиться универсальным query-framework;
- координировать модули между собой;
- сам решать, когда делать retry;
- сам решать, когда делать resync;
- сам решать, какие сообщения публиковать в `bus`;
- строить общую модель stale/fresh для всей системы;
- становиться скрытым app-level orchestration-слоем.

Это важная граница.

Если request-state слой начинает брать на себя эти обязанности, он перестает
быть supporting library и начинает размывать ownership модуля.

## 6. Отношение к `lcmf-http`

Request-state слой должен быть поверх `lcmf-http`, а не рядом с ним как
конкурирующая модель.

Причина простая:

- `lcmf-http` уже различает HTTP error, transport failure, timeout, cancel и
  invalid response;
- request-state слой должен использовать уже приведенный результат;
- request-state слой не должен заново придумывать разбор транспортного слоя.

Практическая картина такая:

```clojure
(-> (http/request! http-runtime {:method :get :path "/bookings"})
    (.then (fn [response]
             (request-state/succeed! bookings-state
                                     (:body response)
                                     response)))
    (.catch (fn [error]
              (let [data (ex-data error)]
                (case (:kind data)
                  :cancelled (request-state/cancel! bookings-state)
                  (request-state/fail! bookings-state data))))))
```

Здесь transport-ответственность и request-state ответственность остаются
разделенными.

## 7. Request-state и модульный ownership

Даже при наличии request-state helper модуль остается владельцем:

- своего state;
- своих операций;
- связи между запросом и UI;
- решения, нужен ли refresh;
- решения, нужен ли latest-only;
- решения, какие сообщения публиковать в `bus`.

Это означает:

- helper не владеет модулем;
- helper не знает предметный смысл данных;
- helper не должен сам встраивать себя в глобальный app-state.

Именно поэтому request-state слой хорошо сочетается с `LCMF`: он снимает
boilerplate, но не переносит ownership от модуля к библиотеке.

## 8. Latest-only как локальная дисциплина

Во frontend-коде обычна ситуация, когда старый запрос завершается позже нового.

Для одного request-flow полезна локальная discipline:

- новый запуск получает `run-id`;
- завершение применимо только к текущему активному `run-id`;
- если flow уже ушел в новый жизненный цикл, старый запуск больше не имеет
  права менять state.

Важно:

- это локальная дисциплина одного request-flow;
- это не общий менеджер гонок для всего модуля;
- это не межмодульный coordination-layer.

Практический пример:

```clojure
(let [run-id (request-state/begin-run! catalog-load-state)]
  (request-state/complete-run! catalog-load-state
                               run-id
                               {:status :success
                                :data {:items [{:id "slot-10-00"}]}
                                :response {:request-id "rid-refresh"}}))
```

Если позже завершится более старый запуск, он должен быть проигнорирован.

## 9. Обычный и фоновой режим

Для малого и среднего проекта полезно различать:

- первую или явную загрузку;
- фоновое обновление.

Это различие важно не как абстракция ради абстракции, а как обычная
пользовательская практика:

- при первой загрузке экран может быть пустым и ждать данных;
- при refresh часто лучше сохранить старые данные и показать мягкое обновление;
- ошибка refresh не обязана стирать уже показанную картину.

Именно поэтому practically useful request-state модель обычно содержит:

- основные terminal status;
- отдельный флаг `:revalidating?`.

## 10. Когда request-state библиотека особенно уместна

Отдельная request-state библиотека особенно уместна, если:

- в двух-трех модулях повторился почти одинаковый boilerplate загрузки;
- проект уже использует общий `lcmf-http` runtime;
- модулям нужен единый подход к pending/success/error/cancelled;
- проект хочет latest-only поведение без ad hoc решений в каждом модуле.

Если же request-state паттерны в проекте сильно различаются и не складываются в
одну устойчивую форму, лучше не раздувать helper раньше времени.

## 11. Что важно не делать

Вокруг request-state слоя в `LCMF` особенно важно не делать следующее:

- не держать много разных операций в одном общем request-state;
- не превращать helper в новый центр архитектуры;
- не смешивать transport logic и request-state logic;
- не переносить сюда bus orchestration;
- не строить тут общий cache framework;
- не пытаться через request-state читать и координировать соседние модули.

Коротко:

- один request-flow;
- один локальный state;
- один маленький helper-слой;
- никакой подмены архитектурных границ.

## 12. Минимальный итог

Request-state слой в `LCMF` — это полезная, но вторичная supporting library.

Его роль:

- убрать типовой boilerplate одного request-flow;
- сохранить ясные переходы состояния;
- помочь latest-only сценарию;
- не нарушить ownership модуля и границы между `HTTP`, `bus`, `registry` и
  app-level слоем.

Для малого и среднего проекта хороший базовый режим такой:

- app-level слой дает общий `lcmf-http` runtime;
- модуль держит отдельный `atom` на отдельный request-flow;
- request-state helper ведет типовое состояние этого flow;
- предметные решения и координация остаются у модуля.
