在 React 中，我们使用 Hook 机制来维护状态，我们如果使用 js 变量记录数据的话，React 会在 re-render 后重新加载变量内容。
有 React hooks 有三个原则：
 * Hooks 只能在 React 函数组件中声明
 * Hooks 只能在顶层容器中声明
 * Hooks 不能作为条件语句中使用
需要声明的一点是 Hooks 不能在条件语句中使用的意思是，不能出现如下写法：
```jsx
function myComponent() {
  if (Condition) {
    const [state, setState] = useState(0);
  }
}
```
这是因为 React 需要依赖 hooks 调用的顺序来正确地管理组件的状态和副作用。如果 hooks 调用是条件化的，React 将无法保证每次渲染时 hooks 调用的顺序一致，从而导致错误。
