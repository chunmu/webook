# constants

#### SSR_ATTR

```javascript
export const SSR_ATTR = 'data-server-rendered'
```

#### ASSET_TYPES

```javascript
export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]
```

#### LIFECYCLE_HOOKS 声明周期钩子

```javascript
// pdd: 生命周期钩子
export const LIFECYCLE_HOOKS = [
  'beforeCreate',
  'created',
  'beforeMount',
  'mounted',
  'beforeUpdate',
  'updated',
  'beforeDestroy',
  'destroyed',
  'activated', // keep-alive被激活时调用
  'deactivated', // keep-alive失活时调用
  'errorCaptured',
  'serverPrefetch'
]

```