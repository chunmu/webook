```javascript

import { emptyObject } from 'shared/util'
import { parseFilters } from './parser/filter-parser'

type Range = { start?: number, end?: number };

```

#### baseWarn

```javascript
export function baseWarn (msg: string, range?: Range) {
  console.error(`[Vue compiler]: ${msg}`)
}
```

#### pluckModuleFunction

```javascript
export function pluckModuleFunction<F: Function> (
  modules: ?Array<Object>,
  key: string
): Array<F> {
  return modules
    ? modules.map(m => m[key]).filter(_ => _)
    : []
    // modules.map(m => m[key]).filter(_ => _)
    // pdd: 过滤出真值属性
}
```

#### addProp

```javascript
export function addProp (el: ASTElement, name: string, value: string, range?: Range, dynamic?: boolean) {
  (el.props || (el.props = [])).push(rangeSetItem({ name, value, dynamic }, range))
  el.plain = false
  // (el.props || (el.props = []))
  // pdd: props非空判断或初始化
}
```

#### addAttr

```javascript
// pdd: 非原生attrs用的是dynamicAttrs/attrs  原生的是attrsList
export function addAttr (el: ASTElement, name: string, value: any, range?: Range, dynamic?: boolean) {
  const attrs = dynamic
    ? (el.dynamicAttrs || (el.dynamicAttrs = []))
    : (el.attrs || (el.attrs = []))
  attrs.push(rangeSetItem({ name, value, dynamic }, range))
  el.plain = false
    // (el.attrs || (el.attrs = []))
    // pdd: attrs非空判断或初始化
}
```

#### addRawAttr

```javascript
// add a raw attr (use this in preTransforms)
// pdd: 非原生attrs用的是dynamicAttrs/attrs  原生的是attrsList
export function addRawAttr (el: ASTElement, name: string, value: any, range?: Range) {
  el.attrsMap[name] = value
  el.attrsList.push(rangeSetItem({ name, value }, range))
}
```

#### addDirective

```javascript
export function addDirective (
  el: ASTElement,
  name: string,
  rawName: string,
  value: string,
  arg: ?string,
  isDynamicArg: boolean,
  modifiers: ?ASTModifiers,
  range?: Range
) {
  (el.directives || (el.directives = [])).push(rangeSetItem({
    name,
    rawName,
    value,
    arg,
    isDynamicArg,
    modifiers
  }, range))
  el.plain = false
  // pdd: 可以看出来指令是绑定在el上的
}
```

#### prependModifierMarker

```javascript
// pdd: 暂时不知道有啥用
function prependModifierMarker (symbol: string, name: string, dynamic?: boolean): string {
  return dynamic
    ? `_p(${name},"${symbol}")`
    : symbol + name // mark the event as captured
}
```

#### addHandler

```javascript

// pdd: 添加handler
export function addHandler (
  el: ASTElement,
  name: string,
  value: string,
  modifiers: ?ASTModifiers,
  important?: boolean,
  warn?: ?Function,
  range?: Range,
  dynamic?: boolean
) {
  modifiers = modifiers || emptyObject
  // warn prevent and passive modifier
  /* istanbul ignore if */
  // pdd: passive 曾经有看到一篇文章提及这个的作用 就是告诉浏览器我不会阻止事件冒泡
  // pdd: 在性能上有比较明显的提升
  // pdd: 有些事件需要及时冒泡 但是如果用户定义了自定义处理事件 浏览器就不能确定代码块中是否有prevent阻止冒泡的代码
  // pdd: 所以passive通知浏览器明确不会有阻止的逻辑 则可以及时冒泡 自定义处理可以延后
  if (
    process.env.NODE_ENV !== 'production' && warn &&
    modifiers.prevent && modifiers.passive
  ) {
    warn(
      'passive and prevent can\'t be used together. ' +
      'Passive handler can\'t prevent default event.',
      range
    )
  }

  // normalize click.right and click.middle since they don't actually fire
  // this is technically browser-specific, but at least for now browsers are
  // the only target envs that have right/middle clicks.
  if (modifiers.right) {
    if (dynamic) {
      name = `(${name})==='click'?'contextmenu':(${name})`
    } else if (name === 'click') {
      name = 'contextmenu'
      delete modifiers.right
    }
  } else if (modifiers.middle) {
    if (dynamic) {
      name = `(${name})==='click'?'mouseup':(${name})`
    } else if (name === 'click') {
      name = 'mouseup'
    }
  }

  // check capture modifier
  if (modifiers.capture) {
    delete modifiers.capture
    name = prependModifierMarker('!', name, dynamic)
  }
  if (modifiers.once) {
    delete modifiers.once
    name = prependModifierMarker('~', name, dynamic)
  }
  /* istanbul ignore if */
  if (modifiers.passive) {
    delete modifiers.passive
    name = prependModifierMarker('&', name, dynamic)
  }

  let events
  if (modifiers.native) {
    delete modifiers.native
    events = el.nativeEvents || (el.nativeEvents = {})
  } else {
    events = el.events || (el.events = {})
  }

  const newHandler: any = rangeSetItem({ value: value.trim(), dynamic }, range)
  if (modifiers !== emptyObject) {
    newHandler.modifiers = modifiers
  }

  const handlers = events[name]
  /* istanbul ignore if */
  if (Array.isArray(handlers)) {
    important ? handlers.unshift(newHandler) : handlers.push(newHandler)
  } else if (handlers) {
    events[name] = important ? [newHandler, handlers] : [handlers, newHandler]
  } else {
    events[name] = newHandler
  }

  el.plain = false
}

```

#### getRawBindingAttr

```javascript
export function getRawBindingAttr (
  el: ASTElement,
  name: string
) {
  return el.rawAttrsMap[':' + name] ||
    el.rawAttrsMap['v-bind:' + name] ||
    el.rawAttrsMap[name]
}
```

#### getBindingAttr

```javascript
export function getBindingAttr (
  el: ASTElement,
  name: string,
  getStatic?: boolean
): ?string {
  // pdd: 通过 :key="xxx" v-bind:key="xx" 这种形式获取
  const dynamicValue =
    getAndRemoveAttr(el, ':' + name) ||
    getAndRemoveAttr(el, 'v-bind:' + name)
  if (dynamicValue != null) {
    return parseFilters(dynamicValue)
  } else if (getStatic !== false) {
    // pdd: 获取原生属性  key="xxx"
    // pdd: 把静态值json stringify处理一下
    const staticValue = getAndRemoveAttr(el, name)
    if (staticValue != null) {
      return JSON.stringify(staticValue)
    }
  }
}
```

#### getAndRemoveAttr

```javascript
// note: this only removes the attr from the Array (attrsList) so that it
// doesn't get processed by processAttrs.
// By default it does NOT remove it from the map (attrsMap) because the map is
// needed during codegen.
export function getAndRemoveAttr (
  el: ASTElement,
  name: string,
  removeFromMap?: boolean
): ?string {
  let val
  if ((val = el.attrsMap[name]) != null) {
    const list = el.attrsList
    for (let i = 0, l = list.length; i < l; i++) {
      if (list[i].name === name) {
        list.splice(i, 1)
        break
      }
    }
  }
  if (removeFromMap) {
    delete el.attrsMap[name]
  }
  return val
}
```

#### getAndRemoveAttrByRegex

```javascript
export function getAndRemoveAttrByRegex (
  el: ASTElement,
  name: RegExp
) {
  const list = el.attrsList
  for (let i = 0, l = list.length; i < l; i++) {
    const attr = list[i]
    if (name.test(attr.name)) {
      list.splice(i, 1)
      return attr
    }
  }
}
```

#### rangeSetItem

```javascript
// pdd: 设置起止索引
function rangeSetItem (
  item: any,
  range?: { start?: number, end?: number }
) {
  if (range) {
    if (range.start != null) {
      item.start = range.start
    }
    if (range.end != null) {
      item.end = range.end
    }
  }
  return item
}
```