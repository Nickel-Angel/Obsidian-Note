# 项目结构
* `/app`：路由、组件和主要业务逻辑所在的地方；
* `/app/lib`：一些通用的接口函数；
* `/app/ui`：一些 ui 组件；
* `/public`：静态资源。
# 样式
在 `app/ui` 中会找到 `global.css` 这个文件，这是全局的样式。
可以像这样引入模块式的 CSS。
```typescript
import styles from '@/app/ui/home.module.css';
```
可以使用 clsx 来切换组件不同状态的样式：
```typescript
import clsx from 'clsx';
 
export default function InvoiceStatus({ status }: { status: string }) {
  return (
    <span
      className={clsx(
        'inline-flex items-center rounded-full px-2 py-1 text-sm',
        {
          'bg-gray-100 text-gray-500': status === 'pending',
          'bg-green-500 text-white': status === 'paid',
        },
      )}
    >
    // ...
)}
```
