---
title: 'TypeScript + React 个人最佳实践(WIP持续补充中)'
date: 2021-07-04 00:11:47
tags: []
published: true
hideInList: false
feature:
isTop: false
---
# 前置工作
1. [阅读React Doc 中 TS 部分](https://reactjs.org/docs/static-type-checking.html#typescript)
2. [阅读 TypeScript Playground 中 React 部分](https://reactjs.org/docs/static-type-checking.html#typescript)
# 引入 React：永远使用命名空间导入（namespace import）
```typescirpt
import * as React from 'react';
import { useState } from 'react'; // 可以单独导出一遍内部项
import * as ReactDom from 'react-dom';
```
不推荐 `import React from 'react'`。[👉为什么？](https://twitter.com/dan_abramov/status/1308739731551858689)
这种引入方式被称为 `default export` 。不推荐的原因是第React 仅仅是作为一个命名空间存在的，我们并不会像使用一个值一样来使用 React。从 React 16.13.0 开始，React 的导出方式也已经更正为 `export {xxx, ...} `的形式了([commit](https://github.com/facebook/react/commit/60016c448bb7d19fc989acd05dda5aca2e124381#diff-a9c08afc9ba1304a73e4235b3016b97c))。之所以`default export`还可以使用，是因为目前 React 的构建产物是 Commonjs 规范的，webpack 等构建工具做了兼容。

# 组件开发
## 1. 尽量使用Function Component声明，即 React.FC：
```typescript
export interface Props {
  /** The user's name */
  name: string;
  /** Should the name be rendered in bold */
  priority?: boolean
}

const PrintName: React.FC<Props> = (props) => {
  return (
    <div>
      <p style={{ fontWeight: props.priority ? "bold" : "normal" }}>{props.name}</p>
    </div>
  )
}
```
我一般在开发时使用 vscode snippets 快速生成：
```json
"Functional, Folder Name": {
    "prefix": "ff",
    "body": [
      "import * as React from 'react';",
      "",
      "const { useRef, useState, useEffect, useMemo } = React;",
      "",
      "",
      "interface ${TM_DIRECTORY/.*[\\\\\\\\\\\\/](.*)$/$1/}Props {",
      "",
      "}",
      "",
			"const defaultProps: ${TM_DIRECTORY/.*[\\\\\\\\\\\\/](.*)$/$1/}Props = {};",
			"",
      "const ${TM_DIRECTORY/.*[\\\\\\\\\\\\/](.*)$/$1/}: React.FC<${TM_DIRECTORY/.*[\\\\\\\\\\\\/](.*)$/$1/}Props> = (props: React.PropsWithChildren<${TM_DIRECTORY/.*[\\\\\\\\\\\\/](.*)$/$1/}Props> = defaultProps) => {",
      "  const { } = props;",
      "",
      "  return (",
      "",
      "  );",
      "};",
      "",
      "export default ${TM_DIRECTORY/.*[\\\\\\\\\\\\/](.*)$/$1/};",
      ""
    ],
    "description": "Generate a functional component template"
  },
```
# Hook 相关
推荐安装vscode 插件：[React Hooks Snippets](https://marketplace.visualstudio.com/items?itemName=AlDuncanson.react-hooks-snippets) 快速写 hook，提高开发效率。
## 2. useState<T>： 当状态初始值为空时，推荐写出完整泛型，否则可以自动推断类型。
原因：一些状态初始值为空时，需要显示地声明类型：
```typescript
const [user, setUser] = React.useState<User | null>(null)
```
注意：如果初始值为 undefined，可不用在泛型中加上 undefined。
## 3. useMemo() 和 useCallback() 会隐式推断类型，推荐不传泛型
注：不要经常用useCallback，因为也会增加开销。仅当：
- 包装在 `React.memo()`（或 shouldComponentUpdate ）中的组件接受回调 prop;
- 当函数用作其他 hooks 的依赖项时 `useEffect(...，[callback])`。
## 4. 自定义hook如果返回为数组，需要手动添加 const 断言：
```typescript
function useLoading() {
  const [isLoading, setLoading] = React.useState(false);
  const load = (aPromise: Promise<any>) => {
    setLoading(true)
    return aPromise.then(() => setLoading(false));
  }
  // 实际需要: [boolean, typeof load] 类型
  // 而不是自动推导的：(boolean | typeof load)[]
  return [isLoading, load] as const;
}
```
或者可以直接定义返回类型：
```typescript
export function useLoading(): [
  boolean,
  (aPromise: Promise<any>) => Promise<any>
] {
  const [isLoading, setState] = React.useState(false)
  const load = (aPromise: Promise<any>) => {
    setState(true)
    return aPromise.then(() => setState(false))
  }
  return [isLoading, load]
}
```
# 其他
## 5. 使用默认参数值代替默认属性
```typescript
interface GreetProps { age?: number }
const defaultProps: GreetProps = { age: 21 };
const Greet = (props: GreetProps = defaultProps) => {
  /* ... */
}
```
原因：Function Component 的 defaultProps [最终会被废弃](https://twitter.com/hswolff/status/1133759319571345408)，不推荐使用。
如果仍要使用defaultProps，推荐如下方式：
```typescript
interface IProps {
  name: string
}
const defaultProps = {
  age: 25,
};

// 类型定义
type GreetProps = IProps & typeof defaultProps;
const Greet = (props: GreetProps) => <div></div>
Greet.defaultProps = defaultProps;

// 使用
const TestComponent = (props: React.ComponentProps<typeof Greet>) => {
  return <h1 />
}
const el = <TestComponent name="foo" />
```
## 6. 建议使用 Interface 定义组件 props（TS 官方推荐做法），使用 type 也可，不强制
type 和 interface 的区别：type 类型不能二次编辑，而 interface 可以随时扩展。

## 7. 使用 ComponentProps 获取未被导出的组件参数类型，使用 ReturnType 获取返回值类型
获取组件参数类型：
```typescript
// 获取参数类型
import { Button } from 'library' // 但是未导出props type
type ButtonProps = React.ComponentProps<typeof Button> // 获取props
type AlertButtonProps = Omit<ButtonProps, 'onClick'> // 去除onClick
const AlertButton: React.FC<AlertButtonProps> = props => (
  <Button onClick={() => alert('hello')} {...props} />
)
```
获取返回值类型：
```typescript
// 获取返回值类型
function foo() {
  return { baz: 1 }
}
type FooReturn = ReturnType<typeof foo> // { baz: number }
```
## 8. 对 type 或 interface 进行注释时尽量使用 /** */，以便获得更好的类型提示
```typescript
/* ✅ */
/**
 * @param a 注释1
 * @param b 注释2
 */
type Test = {
  a: string;
  b: number;
};
const testObj: Test = {
  a: '123',
  b: 234,
};
```
当 hover 到 `Test` 时类型提示更为友好：
![](https://Moltemort.github.io/post-images/1625414942775.png)

## 9. 组件 Props ts 类型规范
```typescript
type AppProps = {
  /** string */
  message: string;
  /** number */
  count: number;
  /** boolean */
  disabled: boolean;
  /** 基本类型数组 */
  names: string[];
  /** 字符串字面量 */
  status: 'waiting' | 'success';
  /** 对象：列出对象全部属性 */
  obj3: {
    id: string;
    title: string;
  };
  /** item 为对象的数组 */
  objArr: {
    id: string;
    title: string;
  }[];
  /** 字典*/
  dict: Record<string, MyTypeHere>;
  /** 任意完全不会调用的函数 */
  onSomething: Function;
  /** 没有参数&返回值的函数 */
  onClick: () => void;
  /** 携带参数的函数 */
  onChange: (id: number) => void;
  /** 携带点击事件的函数, 不要再用 e: any 了 */
  onClick(e: React.MouseEvent<HTMLButtonElement>): void;
  /** 可选的属性 */
  optional?: OptionalType;
  children: React.ReactNode; // 最佳，支持所有类型(JSX.Element, JSX.Element[], string)
  renderChildren: (name: string) => React.ReactNode;
  style?: React.CSSProperties;
  onChange?: React.FormEventHandler<HTMLInputElement>; // 表单事件
};
```

## 10. 组件中 event 处理
常见的Eventl类型：
```typescript
React.SyntheticEvent<T = Element>
React.ClipboardEvent<T = Element>
React.DragEvent<T = Element>
React.FocusEvent<T = Element>
React.FormEvent<T = Element>
React.ChangeEvent<T = Element>
React.KeyboardEvent<T = Element>
React.MouseEvent<T = Element>
React.TouchEvent<T = Element>
React.PointerEvent<T = Element>
React.UIEvent<T = Element>
React.WheelEvent<T = Element>
React.AnimationEvent<T = Element>
React.TransitionEvent<T = Element>
```
定义事件回调函数：
```typescript
type changeFn = (e: React.FormEvent<HTMLInputElement>) => void;
```
如果不太关心事件的类型，可以直接使用 React.SyntheticEvent，如果目标表单有想要访问的自定义命名输入，可以使用类型扩展:
```typescript
const App: React.FC = () => {
  const onSubmit = (e: React.SyntheticEvent) => {
    e.preventDefault();
    const target = e.target as typeof e.target & {
      password: { value: string; };
    }; // 类型扩展
    const password = target.password.value;
  };
  return (
    <form onSubmit={onSubmit}>
      <div>
        <label>
          Password:
          <input type="password" name="password" />
        </label>
      </div>
      <div>
        <input type="submit" value="Log in" />
      </div>
    </form>
  );
};
```

## 11. 尽量使用 optional channing

## 12. 尽量使用 React.ComponentProps<typeof Component> 减少非必要props导出

## 13. 不要在 type 或 interface 中使用函数声明
```typescript
/** ✅ */
interface ICounter {
  start: (value: number) => string
}

/** ❌ */
interface ICounter1 {
  start(value: number): string
}
```

## 14. 当局部组件结合多组件进行组件间状态通信时，如果不是特别复杂，则不需要用mobx或者 建议结合 useReducer() 和 useContext() 一起使用，频繁的组件间通信最佳实践：
`Store.ts`
```typescript
import * as React from 'react';

export interface State {
  state1: boolean;
  state2: boolean;
}

export const initState: State = {
  state1: false,
  state2: true,
};

export type Action = 'action1' | 'action2';

export const StoreContext = React.createContext<{
  state: State;
  dispatch: React.Dispatch<Action>;
}>({ state: initState, dispatch: value => { /** noop */ } });


export const reducer: React.Reducer<State, Action> = (state, action) => {
 // 用 reducer 的好处之一是可以拿到之前的 state
  switch (action) {
    case 'action1':
      return { ...state, state1: !state.state1 };
    case 'action2':
      return { ...state, state2: !state.state2 };
    default:
      return state;
  }
};
```

`WithProvider.tsx`
```typescript
import * as React from 'react';
import { StoreContext, reducer, initState } from './store';

const { useReducer } = React;

const WithProvider: React.FC<Record<string, unknown>> = (props: React.PropsWithChildren<Record<string, unknown>>) => {
  // 将 state 作为根节点的状态注入到子组件中
  const [state, dispatch] = useReducer(reducer, initState);
  const { children } = props;

  return <StoreContext.Provider value={{ state, dispatch }}>{children}</StoreContext.Provider>;
};

export default WithProvider;
```
父组件：
```typescript
import * as React from 'react';
import WithProvider from './withProvider';
import Component1 from './components/Component1';
import Component2 from './components/Component2';

const { useRef, useState, useEffect, useMemo } = React;

interface RankProps {}

const defaultProps: RankProps = {};

const Rank: React.FC<RankProps> = (props: React.PropsWithChildren<RankProps> = defaultProps) => {
  const {} = props;

  return (
    <WithProvider>
      <Component1 />
      <Component2 />
    </WithProvider>
  );
};


export default Rank;

```
子组件只需要 import `StoreContext` 之后 `useContext()` 就能得到 state 和 dispatch
```typescript
import * as React from 'react';
import { StoreContext } from '../../store';

const { useContext } = React;

interface Component1Props {}

const defaultProps: Component1Props = {};

const Component1: React.FC<Component1Props> = (props: React.PropsWithChildren<Component1Props> = defaultProps) => {
  const { state, dispatch } = useContext(StoreContext);

  const {} = props;

  return (
    <>
      state1: {state.state1 ? 'true' : 'false'}
      <button
        type="button"
        onClick={(): void => {
          dispatch('action1');
        }}
      >
        changeState1 with Action1
      </button>
    </>
  );
};

export default React.memo(Component1); // 建议有context 的地方最好 memo 一下，提高性能

```
`Store.ts` 和 `WithProvider.tsx` 可以配置成 vscode snippets，需要用到时直接使用。

# 参考
[1] [React + TypeScript 实践](https://mp.weixin.qq.com/s/mUblBpj6pmdxz9mLKEDJTw)
[2] [精读《React Hooks 最佳实践》](https://juejin.cn/post/6844903938093744135)

**欢迎在评论或 issue 中讨论，指出不合理之处，补充其他最佳实践~**


