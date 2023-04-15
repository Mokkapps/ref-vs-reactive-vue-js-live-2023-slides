---
theme: mokkapps
title: "ref() vs. reactive(): What to choose using Vue 3 Composition API?"
# lineNumbers: true
exportFilename: "ref-vs-reactive-vue-js-live-2023-slides-by-michael-hoffmann"
# provide a downloadable PDF:
download: true
transition: slide-left
# globalBottomPosition: "left"
---

# ref() vs. reactive()

## What to choose using Vue 3 Composition API?

Vue.js Live on May 15th, 2023

---
layout: about-me
---

<!--
- run a weekly vue newsletter
- very active on Twitter
- follow if you are interested in Vue & Nuxt
-->

---
layout: image
image: ./ref-or-reactive.png
---



<!--
- I love Vue 3's Composition API
-  two approaches to adding a reactive state to Vue components
- cumbersome to use `.value` everywhere when using refs 
- easily lose reactivity when destructuring reactive objects
- how you can choose whether to utilize reactive, ref, or both.
-->

---
layout: section
image: ./agenda.jpg
---

# Agenda

- Reactivity in Vue 3
- reactive()
- ref()
- reactive() vs. ref()
- Conclusion

<!--
Conclusion: 

- My pinion & opinions from the Vue community
- A pattern to group ref and reactive
-->

---
layout: section
---

## Reactivity in Vue 3

---
layout: image-right
image: ./vue-reactivity-meme.jpg
---

<!-- <style>
  div {
    @apply !bg-contain;
  }
</style> -->

# What is reactivity?

And why does Vue need it?

<v-clicks>

- A reactivity system is a mechanism that automatically **keeps in sync** a data source (model) with a data representation (view) layer. 

- Every time the model changes, the view is re-rendered to reflect the changes.

- It's a crucial mechanism for any web framework.

</v-clicks>

<!--
- mechanism that automatically **keeps in sync** a data source (model) with a data representation (view) layer. 

- model changes, the view is re-rendered to reflect the changes.

- It's a crucial mechanism for any web framework.
-->

---

# JavaScript is not reactive per default

<v-clicks>

Let's take a look at a code example:

```js {1-2|1-4|1-5|1-7|1-8}
let price = 10.0
const quantity = 2

const total = price * quantity
console.log(total) // 20

price = 20.0
console.log(total) // ⚠️ total is still 20
```

<span><twemoji-face-with-monocle /> In a reactivity system, we expect that `total` is updated each time `price` or `quantity` is changed.</span>

<span><twemoji-warning /> But JavaScript usually doesn't work like this.</span>

<span><twemoji-technologist /> The Vue framework had to implement a mechanism to track the reading and writing of local variables.</span>

</v-clicks>

---

# How Vue implements reactivity

<v-clicks>

It works by intercepting the reading and writing of object properties

**Vue 3** uses [Proxies](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) for reactive objects and getters/setters for refs

```js {1|2,11|3-6|7-10|1-12} {maxHeight:'250px'}
function reactive(obj) {
  return new Proxy(obj, {
    get(target, key) {
      track(target, key)
      return target[key]
    },
    set(target, key, value) {
      target[key] = value
      trigger(target, key)
    },
  })
}
```

<span><twemoji-information /> Vue 2 used object getters/setters exclusively due to browser limitations</span>

</v-clicks>

<!--
- **Simplified code example**
- target: map storing effects (side effects), functions that modify application state
- **track()**, we check whether there is a **currently running effect**. 
- **trigger()**, lookup the subscriber effects for the property and **invoke them**
- effects are **stored in a global WeakMap**<target, Map<key, Set<effect>>> data structure
-->

---
disabled: true
---

# track & trigger

```js
// This will be set right before an effect is about
// to be run. We'll deal with this later.
let activeEffect

function track(target, key) {
  if (activeEffect) {
    const effects = getSubscribersForProperty(target, key)
    effects.add(activeEffect)
  }
}
```

```js
function trigger(target, key) {
  const effects = getSubscribersForProperty(target, key)
  effects.forEach((effect) => effect())
}
```

---
layout: section
---

# reactive()

---

# reactive()

Returns a reactive proxy of the provided object

<v-clicks>

```js {1|1-3}
import { reactive } from 'vue'

const state = reactive({ count: 0 })
```

This state is **deeply reactive by default**:

```js {3,6|4|5|8|8-9|11,12,14|8,11-14} {maxHeight:'200px'}
import { reactive, watch } from 'vue'

const state = reactive({
  count: 0,
  nested: { count: 0 },
})

watch(state, () => console.log(state))
// "{ count: 0, nested: { count: 0 } }"

const incrementNestedCount = () => {
  state.nested.count += 1
  // Triggers watcher -> "{ count: 0, nested: { count: 1 } }"
}

```
</v-clicks>

<!-- 
- equivalent of Vue.observable() in Vue 2.6
-->

---
layout: section
---

## Limitations of reactive()

---
layout: image-right
image: ./reactive-limitations.png
---

# Limitations of reactive()

<p/>

<v-clicks>

The `reactive()` API has two limitations:

<span><emojione-monotone-digit-one /> It **only works** on object types and **doesn't work** with primitive types</span>
  
<span><emojione-monotone-digit-two /> The returned proxy object from `reactive()` **doesn't have the same identity** as the original object</span>

</v-clicks>

<v-click>

```js {1|1-2|1-5}
const plainJsObject = {}
const proxy = reactive(plainJsObject)

// proxy is NOT equal to the original plain JS object.
console.log(proxy === plainJsObject) // false
```

</v-click>

<!--
- object types like objects, arrays, and collection types such as Map and Set
- primitive values such as string, number or boolean
- strict equality (===) operator
-->

---

# Problem 1: Reactive Proxy vs. Original 

<v-clicks>

Reactivity is lost if you destructure a reactive object's property into a local variable

```js {1-3|5|5-6|8|all}
const state = reactive({
  count: 0,
})

let { count } = state
// ⚠️ count is now a local variable disconnected from state.count

count += 1 // ⚠️ Does not affect original state
```

</v-clicks>

<v-click>

`toRefs` solves that problem:

```js {3-5|1,7|1,7-8}
import { toRefs } from 'vue'

let state = reactive({
  count: 0,
})

const { count } = toRefs(state)
// count is a ref, maintaining reactivity
```

</v-click>

---

# Problem 2: Reactive Proxy vs. Original 

<v-clicks>

Reactivity is lost if you try to reassign a reactive value

```js {1-3|5|5-6|8-10|1-3,8-11|5,8-12}
let state = reactive({
  count: 0,
})

watch(state, () => console.log(state))
// "{ count: 0 }"

state = reactive({
  count: 10,
})
// ⚠️ The above reference ({ count: 0 }) is no longer being tracked (reactivity connection is lost!)
// ⚠️ The watcher doesn't fire
```

</v-clicks>

---

# Problem 3: Reactive Proxy vs. Original Problem

<v-clicks>

Reactivity connection is also lost if you pass a property into a function

```js {1-3|9|5,7|5-7}
const state = reactive({
  count: 0,
})

const useFoo = (count) => {
  // ⚠️ Here count is a plain number and not reactive
}

useFoo(state.count)
```

</v-clicks>

---

# A Good Choice for Composition API Migration

<p/>

<v-clicks>


`reactive` works very similarly to reactive properties inside of the `data` field:

```js {3-6} {'maxHeight':'150px'}
<script>
export default {
  data() {
    count: 0,
    name: 'MyCounter'
  },
  methods: {
    increment() {
      this.count += 1;
    },
  }
};
</script>
```

You can simply copy everything from `data` into `reactive` to migrate this component to Composition API:

```js {2-8} {'maxHeight':'150px'}
<script setup>
setup() {
  // Equivalent to "data" in Options API
  const state = reactive({
    count: 0,
    name: 'MyCounter'
  });
  const {count, name} = toRefs(state)

  // Equivalent to "methods" in Options API
  increment(username) {
    state.count += 1;
  }
}
</script
```

</v-clicks>
---
layout: section
---

# ref()

---
layout: image-right
image: ./ref.png
---

# ref()

ref() addresses the limitations of reactive()

<v-clicks>

`ref()` is not limited to object types but can hold any value type:

```js {1|3|4|all}
import { ref } from 'vue'

const count = ref(0)
const state = ref({ count: 0 })
```

To read & write the reactive variable created with `ref()`, you need to access it with the `.value` property:

```js {1|1,3|1,4|1,6|1,7} {'maxHeight': '130px'}
const count = ref(0)

console.log(count) // { value: 0 }
console.log(count.value) // 0

count.value = 2
console.log(count.value) // 2
```

</v-clicks>

<!--
- refs can store primitive values but also complex data structures like objects, arrays, maps, and **even DOM elements**.
- ref() takes an inner value and returns a reactive and mutable ref object. 
- The ref object has a single property .value that points to the inner value. 
- If you want to access or mutate the value you need to use.value
-->

---

# ref() internals

<v-clicks>

How can ref() hold primitive values?

```js {1,13|2,11,12|3-6|7-10|all}
function ref(value) {
  const refObject = {
    get value() {
      track(refObject, 'value')
      return value
    },
    set value(newValue) {
      value = newValue
      trigger(refObject, 'value')
    }
  }
  return refObject
}
```

For object types, `ref()` is using `reactive()` under the hood:

```js {1}
ref({}) ~= ref(reactive({}))
```

  </v-clicks>

<!--
- we learned that Vue's reactivity system works by **intercepting object properties**

- for primitive values it uses its own logic, simplified code for demonstration purpose

- when holding object types, ref automatically converts its .value with reactive().
-->

---

# Destructuring ref()

<p/>

<v-clicks>

Reactivity is lost if you destructure a reactive object created with `ref()`

```js {1,3|1-5|1-6}
import { ref } from 'vue'

const count = ref(0)

const { value: countDestructured } = count // ⚠️ disconnects reactivity, countDestructured is a plain number
const countValue = count.value // ⚠️ disconnects reactivity, countValue is a plain number
```

But reactivity is not lost if refs are grouped in a plain JavaScript object:

```js {1-4|1-6}
const state = {
  count: ref(1),
  name: ref('Michael'),
}

const { count, name } = state // count & name are still reactive
```

</v-clicks>

---

# Passing ref into functions

<p/>

<v-clicks>

Refs can be passed into functions without losing reactivity

```js {1-4|1-4,10|6,8|6-8}
const state = {
  count: ref(1),
  name: ref('Michael'),
}

const useFoo = (count) => {
  // count is a ref and fully reactive
}

useFoo(state.count)
```

This capability is quite important as it is frequently used when extracting logic into [Composable Functions](https://vuejs.org/guide/reusability/composables.html)

</v-clicks>
---

# Replacing ref object

<p/>

<v-clicks>

A `ref` containing an object value can reactively replace the entire object

```js {1-4|6-9|10}
const state = ref({
  count: 1,
  name: 'Michael',
})

state.value = {
  count: 2,
  name: 'Chris',
}
// state is still reactive
```

</v-clicks>

---
layout: section
---

# Unwrapping refs

Vue helps us unwrapping refs without calling `.value` everywhere

---

# Template Unwrapping

<p/>

<v-clicks>


Vue automatically "unwraps" a `ref` when you call it in a template:

```vue {1-5|4,7-12}
<script setup>
import { ref } from 'vue'

const count = ref(0)
</script>

<template>
  <span>
    <!-- no .value needed -->
    {{ count }}
  </span>
</template>
```

</v-clicks>

---

# Watcher Unwrapping

<p/>

<v-clicks>

We can directly pass a `ref` as a watcher dependency:

```js {3|5-6}
import { watch, ref } from 'vue'

const count = ref(0)

watch(count, (newCount) => console.log(newCount)) // no .value needed
```

</v-clicks>

---

# Unwrapping refs with unref()

<p/>

<v-clicks>

[unref()](https://vuejs.org/api/reactivity-utilities.html#unref) is a handy utility function that is especially useful if your value **could be** a `ref`:

```js {3-4|1,3,6|1,3,6,7|1,4,9|1,4,9,10}
import { ref, unref } from 'vue'

const count = ref(0)
const name = 'Michael'

const unwrappedCount = unref(count)
console.log(unwrappedCount) // 0

const unwrappedName = unref(name)
console.log(name) // 'Michael'
```

`unref()` is a sugar function for `isRef(count) ? count.value : count`

</v-clicks>

---

# Volar VS Code Extension

<p/>

<v-clicks>

[Volar VS Code extension](https://marketplace.visualstudio.com/items?itemName=Vue.volar) can automatically add `.value` to `refs`:

<div class="grid grid-cols-2 gap-12">
  <img src="/volar-auto-complete-ref.gif"/>
  <img src="/volar-setting.png"/>
</div>


</v-clicks>

---
layout: section
---

# reactive() vs. ref()

---

# reactive() vs. ref()

| reactive                                                          | ref                                                                  |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| 👎 **only** works on object types                                   | 👍 works with **any** value                                            |
| 👍 no difference in accessing values in `<script>` and `<template>` | 👎 accessing values in `<script>` and `<template>` behaves differently |
| 👎 re-assigning a new object "disconnects" reactivity               | 👍 object references can be reassigned                                 |
| 🫱 properties can be accessed without `.value`                       | 🫱 need to use `.value` to access properties                            |
|                                                                     | 👍 references can be passed across functions                           |
| 👎 destructured values are not reactive                             |                                                                        |
| 👍 Similar to Vue 2’s data object                                   |                                                                        |

<!--
- ref() scores as it can be used with any value
- ref() values are accessed differently in script and template, plus point for reactive()
- ref() object references can be re-assigned
- ref() references can be passed across functions
- ref() can be destructured if grouped in a plain JS object
- reactive() is better for Composition API migration
-->

---
layout: section
---

# Conclusion

---

# My Opinion

<p/>

<v-clicks>

What I like most about `ref` is that you know that it's a reactive value if you see that its property is accessed via `.value`. 

It's not that clear if you use an object that is created with `reactive`:

```js {1|3}
anyObject.property = 'new' // anyObject could be a plain JS object or a reactive object

anyRef.value = 'new' // likely a ref
```

</v-clicks>

---

# Composing ref() and reactive()

<p/>

<v-clicks>


A recommended pattern is to group refs inside a `reactive` object:

```js {1-2|1-7|9-10|12-13|15-18}
const loading = ref(true)
const error = ref(null)

const state = reactive({
  loading,
  error,
})

// You can watch the reactive object...
watchEffect(() => console.log(state.loading))

// ...and the ref directly
watch(loading, () => console.log('loading has changed'))

setTimeout(() => {
  loading.value = false
  // Triggers both watchers
}, 500)
```

</v-clicks>

<!--
If you don't need the reactivity of the "state" object itself group the refs in a plain JS object

**Grouping refs results in a single object**
- easier to handle
- keeps your code organized
- At a glance, you can see that the grouped refs belong together and are related.

This pattern is also used in libraries like Vuelidate where they use reactive() for setting up state for validations
-->

---
layout: section
---

# Opinions from Vue Community

---
layout: image-right
image: ./ref.jpeg
---

# Twitter Poll

<Tweet id="1645744629193617416" />

---
layout: image-right
image: ./michael-thiessen.jpg
---

# Opinions from Vue Community

<p/>

<v-click>

The amazing [Michael Thiessen](https://twitter.com/MichaelThiessen) wrote a [brilliant in-depth article](https://michaelnthiessen.com/ref-vs-reactive/#act-3-why-i-prefer-ref) about this topic and collected the opinions of famous people in the Vue community:

</v-click>

<v-clicks>

- Eduardo, creator of Pinia
- Daniel Roe, leader of the Nuxt team
- Matt from LearnVue
- and more...

</v-clicks>

<v-click>

<span><twemoji-backhand-index-pointing-right />
Summarized, they all use `ref` by default and `reactive` when they need to group things.</span>

</v-click>

---
layout: image-right
image: ./ref-or-reactive.png
---

# Final Words

<p/>

So, should you use `ref` or `reactive`?

<v-clicks>

My recommendation is to use `ref` by default and `reactive` when you need to group things. 

The Vue community has the same opinion but it's totally fine if you decide to use `reactive` by default.

Both `ref` and `reactive` are powerful tools to create reactive variables in Vue 3. 

You can even use both of them without any technical drawbacks. 

Just pick the one you like and try to stay **consistent** in how you write your code!

</v-clicks>

---
layout: outro
---


# Thank you for listening!

Questions?

[Repository](https://github.com/Mokkapps/ref-vs-reactive-vue-js-live-2023-slides) / [Slides](https://ref-vs-reactive-vuejs-live-2023.netlify.app/)
