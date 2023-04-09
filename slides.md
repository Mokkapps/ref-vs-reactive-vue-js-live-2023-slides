---
theme: mokkapps
title: "ref() vs. reactive(): What to choose using Vue 3 Composition API?"
lineNumbers: true
colorSchema: "light"
exportFilename: "ref-vs-reactive-vue-js-live-2023"
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
layout: image-right
image: ./agenda.jpg
---

# Agenda

- Reactivity in Vue 3
- reactive()
  - Limitations of reactive()
  - My Opinion about reactive()
- ref()
  - Unwrapping refs
  - My Opinion about ref()
- Summary
- Composing ref() and reactive()
- Vue Community Opinions

---
layout: section
---

## Reactivity in Vue 3

---
layout: image-right
image: ./vue-reactivity-meme.jpg
---

<style>
  div.w-full.w-full {
    background-size: contain !important;
  }
</style>

# Why does Vue need a reactivity system?

<p></p>

<v-clicks>

The state of a Vue component consists of **reactive JavaScript objects**. 

When you modify them, the view or dependent reactive objects are updated.

</v-clicks>

---

# JavaScript is not reactive per default

Let's take a look at a code example:

```js {1-2|4|4,5|7|7,8}
let price = 10.0
const quantity = 2

const total = price * quantity
console.log(total) // 20

price = 20.0
console.log(total) // ⚠️ total is still 20
```

<v-clicks>


<span><twemoji-face-with-monocle /> In a reactivity system, we expect that `total` is updated each time `price` or `quantity` is changed.</span>

<span><twemoji-warning /> But JavaScript usually doesn't work like this.</span>

<span><twemoji-technologist /> The Vue framework had to implement another mechanism to track the reading and writing of local variables.</span>

</v-clicks>

---

# How Vue implements reactivity

It works by intercepting the reading and writing of object properties

<v-clicks>

**Vue 2** used getters/setters exclusively due to browser limitations

**Vue 3** uses [Proxies](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) for reactive objects and getters/setters for refs

</v-clicks>

<v-click>

```js {1|2|3-6|7-10|1-12|14-26} {maxHeight:'250px'}
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

function ref(value) {
  const refObject = {
    get value() {
      track(refObject, 'value')
      return value
    },
    set value(newValue) {
      value = newValue
      trigger(refObject, 'value')
    },
  }
  return refObject
}
```
</v-click>

---
layout: section
---

# reactive()

---

# reactive()

Returns a reactive proxy of the provided object.

<v-clicks>

```js {1|3|all}
import { reactive } from 'vue'

const state = reactive({ count: 0 })
```

This state is **deeply reactive by default**:

```js {3-6|8|9|11,12,14|8,11-14} {maxHeight:'200px'}
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

```js {1|2|4-5|all}
const plainJsObject = {}
const proxy = reactive(plainJsObject)

// proxy is NOT equal to the original plain JS object.
console.log(proxy === plainJsObject) // false
```

</v-click>

---

# Reactive Proxy vs. Original Problem #1

Reactivity is lost if you destructure a reactive object's property into a local variable:

<v-click>

```js {1-3|5-6|8|all}
const state = reactive({
    count: 0,
})

// ⚠️ count is now a local variable disconnected from state.count
let { count } = state

count += 1 // ⚠️ Does not affect original state
```

</v-click>

<v-click>

`toRefs` solves that problem:

```js {3-5|1,7-8}
import { toRefs } from 'vue'

let state = reactive({
  count: 0,
})

// count is a ref, maintaining reactivity
const { count } = toRefs(state)
```

</v-click>

---

# Reactive Proxy vs. Original Problem #2

Reactivity is lost if you try to reassign a reactive value:

```js {1-3|5|5-6|8-10|11|12}
const state = reactive({
  count: 0,
})

watch(state, () => console.log(state), { deep: true })
// "{ count: 0 }"

state = reactive({
  count: 10,
})
// ⚠️ The above reference ({ count: 0 }) is no longer being tracked (reactivity connection is lost!)
// ⚠️ The watcher doesn't fire
```

---

# Reactive Proxy vs. Original Problem #3

Reactivity connection is also lost if you pass a property into a function:

```js {1-3|10|5,8|5-7}
const state = reactive({
  count: 0,
})

const useFoo = (count) => {
  // ⚠️ Here count is a plain number and not reactive
}

useFoo(state.count)
```

---
layout: section
---

# My Opinion about reactive()

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
  const {count, name} = toRefs(statee)

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

`ref()` is not limited to object types but can hold any value type:

```js {1|3|4}
import { ref } from 'vue'

const count = ref(0)
const state = ref({ count: 0 })
```

To read & write the reactive variable created with `ref()`, you need to access it with the `.value` property:

```js {0|1-2|4|4-5|7|7-8|10|10-11} {'maxHeight': '130px'}
const count = ref(0)
const state = ref({ count: 0 })

console.log(count) // { value: 0 }
console.log(count.value) // 0

count.value++
console.log(count.value) // 1

state.value.count = 1
console.log(state.value) // { count: 1 }
```

---

# ref() internals

How can ref() hold primitive values?

<v-clicks>

`ref()` is using `reactive()` under the hood:

```js {1-3|1-5|1-6|1-7}
const ref = reactive({
  value: 0,
})

ref.value // 0
ref.value += 1
ref.value // 1
```

  </v-clicks>

---

# Destructuring ref()

<p/>

<v-clicks>

Reactivity is lost if you destructure a reactive object created with `ref()`:

```js {1,3|1-5|1-6}
import { ref } from 'vue'

const count = ref(0)

const countValue = count.value // ⚠️ disconnects reactivity
const { value: countDestructured } = count // ⚠️ disconnects reactivity
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

Refs can be passed into functions without losing reactivity:

```js {1-4|1-4,14|6-12}
const state = {
  count: ref(1),
  name: ref('Michael'),
}

const useFoo = (count) => {
  /**
   * The function receives a ref
   * It needs to access the value via .value but it
   * will retain the reactivity connection
   */
  }

useFoo(state.count)
```

This capability is quite important as it is frequently used when extracting logic into [Composable Functions](https://vuejs.org/guide/reusability/composables.html)

</v-clicks>
---

# Replacing ref object

<p/>

<v-clicks>

A `ref` containing an object value can reactively replace the entire object:

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

## My Opinion about ref()

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
layout: section
---

# Unwrapping refs

Vue helps us unwrapping refs without calling `.value` everywhere

---

# Unwrapping refs with unref()

<p/>

<v-clicks>

[unref()](https://vuejs.org/api/reactivity-utilities.html#unref) is a handy utility function that is especially useful if your value **could be** a `ref`:

```js {3|1,5}
import { ref, unref } from 'vue'

const count = ref(0)

const unwrappedCount = unref(count)
```

`unref()` is a sugar function for `isRef(count) ? count.value : count`

</v-clicks>

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

// Vue automatically unwraps this ref for us
watch(count, (newCount) => console.log(newCount))
```

</v-clicks>

---

# Volar VS Code Extension

<p/>

<v-clicks>


[Volar VS Code extension](https://marketplace.visualstudio.com/items?itemName=Vue.volar) can automatically add `.value` to `refs`:

<img src="/volar-setting.png" style="height: 200px"/>

The corresponding JSON setting:

```json
"volar.autoCompleteRefs": true
```

</v-clicks>

---
layout: section
---

# Summary

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

---
layout: section
---

# Composing ref() and reactive()

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

<!-- If you don't need the reactivity of the `state` object itself you could instead group the refs in a plain JavaScript object.

Grouping refs results in a single object that is easier to handle and keeps your code organized. At a glance, you can see that the grouped refs belong together and are related.

This pattern is also used in libraries like [Vuelidate](https://vuelidate.js.org/) where they [use reactive() for setting up state for validations](https://blog.logrocket.com/form-validation-in-vue-with-vuelidate/). -->

---
layout: section
---

# Opinions from Vue Community

---
layout: image-right
image: ./michael-thiessen.jpg
---

# Opinions from Vue Community

<p/>

<v-click>

The amazing [Michael Thiessen](https://twitter.com/MichaelThiessen) wrote a [brilliant in-depth article](https://michaelnthiessen.com/ref-vs-reactive/#act-3-why-i-prefer-ref) about this topic and collected the opinions of famous people in the Vue community:

</v-click>

<v-click>

Some names:

</v-click>

<v-click>

- Eduardo, creator of Pinia
- Daniel Roe, leader of the Nuxt team
- Matt from LearnVue
- and more...

</v-click>

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

[Repository](https://github.com/Mokkapps/vuejs-athen-meetup-2023-lightning-talk-polite-popup-nuxt-3-slides) / [Slides](https://vuejs-athen-meetup-2023-popup-talk.netlify.app/)