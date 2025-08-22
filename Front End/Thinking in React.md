原文是这篇 [React 设计哲学](https://zh-hans.react.dev/learn/thinking-in-react)
感觉这篇文章让我感觉好像……设计一个网站其实没有想象中的那么难，可能也就是现有一个设计原型，先不考虑业务逻辑，只是考虑组件如何布局，然后将 state 提取出来，再去编写任务逻辑。
感觉有点类似于数据库的样子。


不过需要考虑提取 state 后，state 的具体位置在哪里：
1. 直接放置 state 于他们共同的父组件；
2. state 放置于它们父组件上层的组件；
3. 找不到就直接新开一个组件专门管理。
![](ThinkingInReact-1.png)
上图是这个文章中，各个组件的包含关系。
不难发现 `SearchBar` 中用户的输入 `filterText` 和选择是否有库存 `inStockOnly` 这两个是 state，但是它要影响的是 `ProductTable` 的显示。
下面就是组件 `ProductTable` 的代码，这个组件的三个 props 中 `products` 是物品列表，显然这个应该是外部传递给我们的，譬如后端获取。

```jsx
function ProductTable({ products, filterText, inStockOnly }) {
  const rows = [];
  let lastCategory = null;

  products.forEach((product) => {
    if (
      product.name.toLowerCase().indexOf(
        filterText.toLowerCase()
      ) === -1
    ) {
      return;
    }
    if (inStockOnly && !product.stocked) {
      return;
    }
    if (product.category !== lastCategory) {
      rows.push(
        <ProductCategoryRow
          category={product.category}
          key={product.category} />
      );
    }
    rows.push(
      <ProductRow
        product={product}
        key={product.name} />
    );
    lastCategory = product.category;
  });

  return (
    <table>
      <thead>
        <tr>
          <th>Name</th>
          <th>Price</th>
        </tr>
      </thead>
      <tbody>{rows}</tbody>
    </table>
  );
}
```

通过 copilot 的帮助，我尝试写了一下这个从后端获取数据的组件：

```jsx
/*
apiUrl: string, 从哪个 url 请求数据
component: React component, 要封装的那个组件
dataProps: string, 封装的组件中，哪个 prop 的数据需要获取
*/
function FetchComponent({ apiUrl, component, dataProps }) {
  const [data, setData] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch(apiUrl)
      .then((response) => {
        if (!response.ok) {
          throw new Error("Network was not ok.");
        }
        return response.json();
      })
      .then((data) => {
        setData(data);
        setLoading(false);
      })
      .catch((err) => {
        setError(err);
        setLoading(false);
      });
  }, [apiUrl]);

  if (loading) {
    return <div>loading...</div>;
  }
  if (error) {
    return <div>Error: {error.message}</div>;
  }

  const clonedComponent = React.cloneElement(component, { [dataProps]: data });

  return { clonedComponent };
}
```
