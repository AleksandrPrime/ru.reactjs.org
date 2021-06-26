---
id: concurrent-mode-reference
title: Справочник API конкурентного режима (экспериментально)
permalink: docs/concurrent-mode-reference.html
prev: concurrent-mode-adoption.html
---

<style>
.scary > blockquote {
  background-color: rgba(237, 51, 21, 0.2);
  border-left-color: #ed3315;
}
</style>

<div class="scary">

>Внимание:
>
>Эта страница посвящена **экспериментальным возможностям, которых еще нет в стабильной версии**. Информация предназначена для ранних пользователей и просто интересующихся.
>
>Большая часть информации на данной странице уже не актуальна и оставлена для истории. **Актуальная информация приведена в посте блога [React 18 Alpha announcement](/blog/2021/06/08/the-plan-for-react-18.html).**
>
>Перед выходом React 18 информация на этой странице будет обновлена.

</div>

Страница содержит справочник API [конкурентного режима](/docs/concurrent-mode-intro.html) React. Если вам нужно вводное руководство по этому режиму, прочитайте о [паттернах конкурентного UI](/docs/concurrent-mode-patterns.html).

**Примечание: это не стабильный релиз, а предварительная версия для сообщества. Скорее всего, в будущем API изменится. Используйте его на свой страх и риск!**

- [Включение конкурентного режима](#concurrent-mode)
    - [`createRoot`](#createroot)
- [API задержек](#suspense)
    - [`Suspense`](#suspensecomponent)
    - [`SuspenseList`](#suspenselist)
    - [`useTransition`](#usetransition)
    - [`useDeferredValue`](#usedeferredvalue)

## Включение конкурентного режима {#concurrent-mode}

### `createRoot` {#createroot}

```js
ReactDOM.createRoot(rootNode).render(<App />);
```

Заменяет `ReactDOM.render(<App />, rootNode)` и включает конкурентный режим.

Дополнительную информацию можно узнать в [документации конкурентного режима](/docs/concurrent-mode-intro.html).

## API задержек {#suspense}

### `Suspense` {#suspensecomponent}

```js
<Suspense fallback={<h1>Загрузка...</h1>}>
  <ProfilePhoto />
  <ProfileDetails />
</Suspense>
```

`Suspense` позволяет вашим компонентам "ждать", отображая запасной интерфейс, прежде чем они будут отрендерены.

В примере компонент `ProfileDetails` получает данные с помощью асинхронного вызова. Во время ожидания загрузки `ProfileDetails` и `ProfilePhoto`, отображается запасной заголовок `Загрузка...`. Следует отметить, что до тех пор, пока все дочерние компоненты `<Suspense>` не отрендерены, мы увидим запасной интерфейс.

`Suspense` принимает два пропса:
* **fallback** принимает индикатор загрузки. Запасной интерфейс отображается до тех пор, пока не завершится рендер всех дочерних компонентов `Suspense`.
* **unstable_avoidThisFallback** принимает логическое значение. Оно сообщает React, что не нужно отображать запасной интерфейс при первоначальной загрузке. Скорее всего, этот API будет исключён в будущих релизах.

### `<SuspenseList>` {#suspenselist}

```js
<SuspenseList revealOrder="forwards">
  <Suspense fallback={'Загрузка...'}>
    <ProfilePicture id={1} />
  </Suspense>
  <Suspense fallback={'Загрузка...'}>
    <ProfilePicture id={2} />
  </Suspense>
  <Suspense fallback={'Загрузка...'}>
    <ProfilePicture id={3} />
  </Suspense>
  ...
</SuspenseList>
```

`SuspenseList` помогает скоординировать загрузку множества компонентов с помощью определения порядка, в котором они будут показываться пользователю.

Бывают ситуации, когда нескольким компонентам требуется получить данные, последовательность загрузки которых нельзя предсказать заранее. Если обернуть такие компоненты в `SuspenseList`, то React не покажет компонент из списка до тех пор, пока не будет отображён предыдущий (этим поведением можно управлять).

`SuspenseList` принимает два пропса:
* **revealOrder (forwards, backwards, together)** определяет последовательность, в которой отображаются дочерние компоненты `SuspenseList`.
  * `together` позволяет отобразить *все* компоненты одновременно вместо того, чтобы показывать их последовательно.
* **tail (collapsed, hidden)** определяет, как отображаются компоненты из `SuspenseList`, которые ещё не загружены.
    * По умолчанию `SuspenseList` отображает запасной интерфейс для всех компонентов в списке.
    * `collapsed` показывает запасной интерфейс только для компонента, который будет загружен следующим.
    * `hidden` не показывает компоненты, которые ещё не загружены.

Учтите, что `SuspenseList` управляет только ближайшими вложенными компонентами `Suspense` и `SuspenseList`. Поиск осуществляется не более, чем на один уровень вниз, но при этом можно делать многоуровневые вложения `SuspenseList` для построения более сложной разметки.

### `useTransition` {#usetransition}

```js
const SUSPENSE_CONFIG = { timeoutMs: 2000 };

const [startTransition, isPending] = useTransition(SUSPENSE_CONFIG);
```

`useTransition` позволяет не отображать состояние загрузки, когда контент обновляется перед **переходом к следующему экрану**. Это также позволяет компонентам так откладывать более медленные обновления выборки данных до последующих рендеров, что более важные обновления могут быть выполнены немедленно.

Хук `useTransition` возвращает массив из двух элементов.
* `startTransition` -- это функция, которая получает колбэк. Её используют, чтобы сообщить React, какое состояние нам нужно отложить.
* `isPending` -- это логическое значение, которое позволяет React проинформировать нас, ожидаем ли мы в данный момент окончания перехода.

**Если обновление состояния компонента является причиной задержки, то его нужно обернуть по паттерну перехода.**

```js
const SUSPENSE_CONFIG = { timeoutMs: 2000 };

function App() {
  const [resource, setResource] = useState(initialResource);
  const [startTransition, isPending] = useTransition(SUSPENSE_CONFIG);
  return (
    <>
      <button
        disabled={isPending}
        onClick={() => {
          startTransition(() => {
            const nextUserId = getNextId(resource.userId);
            setResource(fetchProfileData(nextUserId));
          });
        }}
      >
        Next
      </button>
      {isPending ? " Загрузка..." : null}
      <Suspense fallback={<Spinner />}>
        <ProfilePage resource={resource} />
      </Suspense>
    </>
  );
}
```

В этом примере мы обернули получение данных в функцию `startTransition`. Это позволяет сразу загружать данные профиля пользователя, откладывая рендер страницы и компонента `Spinner` на 2 секунды (время указано в `timeoutMs`).

Логическое значение `isPending` показывает, что наш компонент в состоянии перехода. Это позволяет, находясь на предыдущем экране, показать пользователю сообщение о загрузке.

**Паттерн перехода более подробно рассматривается в разделе [Паттерны конкурентного UI](/docs/concurrent-mode-patterns.html#transitions).**

#### Конфигурирование useTransition {#usetransition-config}

```js
const SUSPENSE_CONFIG = { timeoutMs: 2000 };
```

Хук `useTransition` принимает **объект настройки задержек** со свойством `timeoutMs`. Этот тайм-аут, заданный в миллисекундах, указывает React, сколько времени нужно ждать появления следующего состояния, например, как это было сделано в программе выше.

**Примечание: мы рекомендуем делать объект настройки задержек глобальным для всех модулей проекта.**


### `useDeferredValue` {#usedeferredvalue}

```js
const deferredValue = useDeferredValue(value, { timeoutMs: 2000 });
```

Хук возвращает отложенную версию для значения, получение которого может «отставать» от рендера всего компонента более чем на `timeoutMs`.

Обычно этот хук используется для создания отзывчивого интерфейса, когда часть компонента нужно немедленно отрендерить на основе пользовательского ввода и при этом другая часть ожидает загрузки данных.

Хорошим примером является ввод текста.

```js
function App() {
  const [text, setText] = useState("hello");
  const deferredText = useDeferredValue(text, { timeoutMs: 2000 });

  return (
    <div className="App">
      {/* Вводим текст в поле ... */}
      <input value={text} onChange={handleChange} />
      ...
      {/* ... при этом список может отображать отложенное значение */}
      <MySlowList text={deferredText} />
    </div>
  );
 }
```

Такой подход позволяет немедленно показывать новый текст для `input`, чтобы улучшить отзывчивость UI. В тоже время, если рендер `MySlowList` "отстает" на 2 секунды, установленные в `timeoutMs`, то в фоновом режиме отображается отложенное значение.

**Паттерн ожидания значений более подробно рассматривается в разделе [Паттерны конкурентного UI](/docs/concurrent-mode-patterns.html#deferring-a-value).**

#### Конфигурирование useDeferredValue {#usedeferredvalue-config}

```js
const SUSPENSE_CONFIG = { timeoutMs: 2000 };
```

`useDeferredValue` принимает **объект настройки задержек** со свойством `timeoutMs`. Этот тайм-аут (в миллисекундах) сообщает React, на сколько долго отложенное значение может отставать.

React будет пытаться использовать более короткую задержку, когда сеть и устройство это позволяют.
