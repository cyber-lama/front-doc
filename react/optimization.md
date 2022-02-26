# Мемоизация и оптимизация компонентов с помощью useMemo, React.memo

Memoization — означает, что вы создаете функцию, 
и каждый раз, когда она вызывается заново, 
она сохраняет результаты выполнения в 
кэше для предотвращения повторных вычислений.

При первом вызове этой memoized 
функции результаты рассчитываются 
(или извлекаются, что бы Вы ни делали внутри тела функции). 
Перед возвращением результатов вы 
храните их в кэше под ключом, 
который создается с входными параметрами.

## Пример приложения с проблемой

У нас есть главный компонент `App.tsx`.
Он содержит: 
- две переменных состояния `count1, count2`
- дочерний компонент `Count.tsx` 
- дочерний компонент `IsFine.tsx`

При изменении одной из переменных `count1, count2`,
с помощью соответствующего сеттера,
каждый из дочерних компонентов `Count.tsx, IsFine.tsx`
будет ререндериться.
С помощью функции `getResult()`, компонента `IsFine.tsx`,
мы можем заметить, как сильно 
это может повлиять на производительность.

```jsx
export const App = () => {
    const [count1, setCount1] = useState(0);
    const [count2, setCount2] = useState(0);
    return (
        <div className={styles.container}>
            <h5>Счетчик 1:</h5>
            <div className="counter">
                <button onClick={() => setCount1(count1 + 1)}>+</button>
                <Count id={1} value={count1}/>
            </div>

            <h5>Счетчик 2:</h5>
            <div className="counter">
                <button onClick={() => setCount2(count2 + 1)}>+</button>
                <Count id={2} value={count2}/>
                <IsFine value={count2}/>
            </div>
        </div>
    );
};
```

В компоненте `Count.tsx`,
мы кешируем и выводим в логи,
количество ререндеров данного компонента.
```tsx
const render: {[index:string]:number} = {
    count1: 0,
    count2: 0
};


export const Count = ({ id, value } :{id: number, value: number}) => {
    console.log(`🔴🌚 Count${id} render: ${++render[`count${id}`]}`);
    return (
        <div>
            <h1 className="cyan">{value}</h1>
        </div>
    );
};
```
В компоненте `IsFine.tsx` 
у нас происходит сложное 
вычисление в функции `getResult()`
```tsx
let renderCount = 0;

export const IsFine = ({value}:{value: number}) => {
    console.log(`🔴🌚 isFine render: ${++renderCount}`);
    const getResult = () => {
        let i = 0;
        while (i < 600000000) i ++;
        return value === 5 ? '💣 Это 5 :D' : '💩 Это не 5 :(';
    };
    return (
        <h3>
            {getResult()}
        </h3>
    );
};
```

## Решаем проблему сложного вычисления

В компоненте `IsFine.tsx`
у нас происходит сложное
вычисление в функции `getResult()`. 
```tsx
const getResult = () => {
        let i = 0;
        while (i < 600000000) i ++;
        return value === 5 ? '💣 Это 5 :D' : '💩 Это не 5 :(';
    };
```
Нам нужно чтобы вычисление производилось только в случае 
когда value будет меняться.

Делать мы это будем с помощью хука `useMemo`

```tsx
const getResult = useMemo(() => {
    let i = 0;
    while (i < 600000000) i ++;
    return value === 5 ? '💣 Это 5 :D' : '💩 Это не 5 :(';
}, [value]);
```
useMemo хранит результат выполнение анонимной функции,
поэтому вызывать `getResult` в компоненте
`IsFine.tsx`
будем так:

```tsx
export const IsFine = ({value}:{value: number}) => {
    console.log(`🔴🌚 isFine render: ${++renderCount}`);
    const getResult = useMemo(() => {
        let i = 0;
        while (i < 600000000) i ++;
        return value === 5 ? '💣 Это 5 :D' : '💩 Это не 5 :(';
    }, [value]);
    return (
        <h3>
            {getResult} // вместо getResult()
        </h3>
    );
};
```

## Избавляемся от лишнего ререндера компонентов. React.memo

Если ваш компонент всегда рендерит одно и то
же при неменяющихся пропсах,
вы можете обернуть его в вызов
React.memo для повышения производительности в некоторых
случаях, мемоизируя тем самым результат.

`React.memo !== React.useMemo`
React.memo - компонент высшего порядка, используется 
для мемоизaции компонентов.

```tsx
const render: {[index: string]: number} = {
    count1: 0,
    count2: 0
};

export const Count = memo(({ id, value } : {id:number, value:number}) => {
    console.log(`🔴🌚 Count${id} render: ${++render[`count${id}`]}`);
    return (
        <div>
            <h1 className="cyan">{value}</h1>
        </div>
    );
});
```

```tsx
let renderCount = 0;

export const IsFine = memo(({value}:{value: number}) => {
    console.log(`🔴🌚 isFine render: ${++renderCount}`);
    const getResult = useMemo(() => {
        let i = 0;
        while (i < 600000000) i ++;
        return value === 5 ? '💣 Это 5 :D' : '💩 Это не 5 :(';
    }, [value]);
    return (
        <h3>
            {getResult}
        </h3>
    );
});
```

## Контролируем ререндер компонента с помощью своей функции

После мемоизaции IsFine, на каждый рендер компонента,
запускается `getResult()`.

Но мы видим, что это вычисление нужно делать,
когда props, равен 5. Иначе, это лишнее расходы ресурсов.

Во втором колбэке можем описать условие отрисовки компонента
как на примере ниже.
```tsx
export const IsFine = memo(({value}:{value: number}) => {
    console.log(`🔴🌚 isFine render: ${++renderCount}`);
    const getResult = useMemo(() => {
        let i = 0;
        while (i < 600000000) i ++;
        return value === 5 ? '💣 Это 5 :D' : '💩 Это не 5 :(';
    }, [value]);
    return (
        <h3>
            {getResult}
        </h3>
    );
}, (prevProps, nextProps) => {
    return !(nextProps.value === 5 || nextProps.value === 6);
});
```






