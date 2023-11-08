# Negated `in` and `instanceof` operators

## Status

**Stage**: 1  
**Champion**: Pablo Gorostiaga Belio ([@gorosgobe](https://github.com/gorosgobe))  

## Author
Pablo Gorostiaga Belio ([@gorosgobe](https://github.com/gorosgobe))

# Proposal

Presentations

- [September 2023](https://docs.google.com/presentation/d/1vwNOjUiUvy6TzK6t0Mb8qgyKwBNszly9HibJJpUa_Eo/edit) 

## Motivation

JavaScript's `in` and `instanceof` operators have broadly the following behaviour:

```js
a in obj; // returns true if property a is in obj or in its prototype chain, false otherwise
a instanceof C; // returns true if C.prototype is in a's prototype chain, false otherwise
```

To negate the result of these expressions, we can wrap them with the logical NOT (`!`) operator:

```js
!(a in obj);
!(a instanceof C);
```

Negating an `in`/`instanceof` expression in this way suffers from a few problems:

### Error-proneness[^1]

The logical not operator, `!`, has to be applied to the whole expression to produce the intended result. Incorrect parenthesising of the sub-expression (which can be a part of an arbitrarily long expression) and/or applying the `!` operator on the wrong operand can lead to errors that are hard to debug, i.e.:

[^1]: A note about TypeScript: error-proneness is less of a concern if TypeScript is used, because TypeScript checks that the correct types are passed to the `in` and `instanceof` operators. However, incorrect or lack of types can still cause this issue. This can also happen if you don't use TypeScript, or if that particular part of your code is untyped or uses `any` explicitly.

For `in`:
```js
if (!a in obj) { 
  // will not execute, unless obj has a 'true' or 'false' key
  // `in` accepts strings or symbols as the LHS parameter, and otherwise coerces all other values to a string
  // see https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String#string_coercion
}

// correct usage
if (!(a in obj)) {
  // ...
}
```
For `instanceof`:
```js
if (!a instanceof C) { 
  // will not execute, unless C provides a @@hasInstance method that returns true for booleans
}

// correct usage
if (!(a instanceof C)) {
  // ...
}
```

This type of error is fairly common. For `in`, this [Sourcegraph query](https://sourcegraph.com/search?q=context:global+lang:javascript+/%5C%21%5B%5B:alnum:%5D%5C%27%5C%22%5D%2B+in+%5B%5B:alnum:%5D%5D%2B/+-file:%5C.min%5C.js%24+count:all&patternType=standard&sm=1&groupBy=repo) reveals that there are many instances of this issue (over ~2.1k instances when I ran it) across repos with thousands of stars on GitHub. While there are some false positives (from comments, for example), I highlight some notable examples below:
<details>
  <summary>Examples</summary>
  
|Repo                                              |Bugs                                                           |Stars |Link|Issue|
|--------------------------------------------------|---------------------------------------------------------------|------|----|-----|
|[meteor/meteor](https://github.com/meteor/meteor)                  |`!key in validDevices`         |43.6k | [Link](https://github.com/meteor/meteor/blob/57759e09746046fb75cbd1479d72cda30bba081f/tools/cordova/builder.js#L803)| [Issue](https://github.com/meteor/meteor/issues/12781) |
|[oven-sh/bun](https://github.com/oven-sh/bun)                    |`!"TZ" in process.env`                                       |42.7k |[Link](https://github.com/oven-sh/bun/blob/6bfee02301a2e2a0b79339974af0445eb5a2688f/test/js/node/process/process.test.js#L106)| [Issue](https://github.com/oven-sh/bun/issues/5800) |
|[SergioBenitez/Rocket](https://github.com/SergioBenitez/Rocket)           |`!"message" in msg \|\| !"room" in msg \|\| !"username" in MSG` |21.1k| [Link](https://github.com/SergioBenitez/Rocket/blob/c2936fcb1e4f8f4907889b54a9e4e741a565a7d7/examples/chat/static/script.js#L97)| [Issue](https://github.com/SergioBenitez/Rocket/issues/2617) |
|[jeromeetienne/AR.js](https://github.com/jeromeetienne/AR.js)            |`!'VRFrameData' in window`                                        |15.7k |[Link](https://github.com/jeromeetienne/AR.js/blob/024318c67121bd57045186b83b42f10c6560a34a/three.js/examples/vendor/webvr-polyfill.js#L6245)| [Issue](https://github.com/jeromeetienne/AR.js/issues/828) |
|[duplicati/duplicati](https://github.com/duplicati/duplicati)            |`!'IsUnencryptedOrPassphraseStored' in this.Backup`               |9.1k  |[Link](https://github.com/duplicati/duplicati/blob/d0f1498bd41b151d8512fd2acb57739f6a05587f/Duplicati/Server/webroot/ngax/scripts/controllers/RestoreController.js#L456)| [Issue](https://github.com/duplicati/duplicati/issues/5028) |
|[WebKit/WebKit](https://github.com/WebKit/WebKit)                  |`!'openDatabase' in window`                                       |6.4k  |[Link](https://github.com/WebKit/WebKit/blob/21506dd04e3ba5815f62bfd714acd73ce48ca3ef/Tools/CSSTestSuiteHarness/harness/harness.js#L1477)| [Issue](https://bugs.webkit.org/show_bug.cgi?id=261815) |
|[buildbot/buildbot](https://github.com/buildbot/buildbot)              |`!option in options`                                              |5.1k  |[Link](https://github.com/buildbot/buildbot/blob/4cf871e7378e87a5e9b811e764874645e14b57ab/www/build_common/src/webpack.js#L21)| [Issue](https://github.com/buildbot/buildbot/issues/7120) |
|[cloudflare/workerd](https://github.com/cloudflare/workerd)             |`!type in this.#recipes`                                          |4.9k  |[Link](https://github.com/cloudflare/workerd/blob/30eb1e66be0aff8ce9c6f9ec76ac2548a2bf4247/samples/extensions/burrito-shop-impl.js#L19)| [Issue](https://github.com/cloudflare/workerd/issues/1207) |
|[muicss/mui](https://github.com/muicss/mui)                     |`!'rows' in rest`                                                   |4.5k  |[Link](https://github.com/muicss/mui/blob/d1774138e025f99c870f9dbb556163028cc2d475/src/react/textarea.jsx#L21)| [Issue](https://github.com/muicss/mui/issues/336) |
|[jlord/git-it-electron](https://github.com/jlord/git-it-electron)          |`!'previous' in curCommit`                                          |4.4k  |[Link](https://github.com/jlord/git-it-electron/blob/e7551c58366787dbc62ee7ef6079fc8cc6c7acb9/assets/PortableGit/mingw32/share/gitweb/static/gitweb.js#L1374)| [Issue](https://github.com/jlord/git-it-electron/issues/393) |
|[zlt2000/microservices-platform](https://github.com/zlt2000/microservices-platform) |`!'onhashchange' in W`                                            |4.2k  |[Link](https://github.com/zlt2000/microservices-platform/blob/da821d6b598fb82d901fc67861cf902b3e60389c/zlt-web/layui-web/src/main/resources/static/assets/libs/q.js#L38)| [Issue](https://github.com/zlt2000/microservices-platform/issues/67) |
|[thechangelog/changelog.com](https://github.com/thechangelog/changelog.com)     |`!"execCommand" in document`                                       |2.6k|  [Link](https://github.com/thechangelog/changelog.com/blob/271286cc8ae68298755bf08a68d1af02dd016603/assets/app/modules/onsitePlayer.js#L449)| [Issue](https://github.com/thechangelog/changelog.com/issues/483) |
|[kiwibrowser/src](https://github.com/kiwibrowser/src)                |`!intervalName in this.intervals`                                   |2.3k|  [Link](https://github.com/kiwibrowser/src/blob/945c26be7a1e458cef098d8d782b5555611cc83b/components/chrome_apps/webstore_widget/app/main.js#L105)| [Issue](https://github.com/kiwibrowser/android/issues/285) |
|[drawcall/Proton](https://github.com/drawcall/Proton)                |`!'defineProperty' in Object`                                       |2.3k|  [Link](https://github.com/drawcall/Proton/blob/83c3caa8203c4e60c7363fb3fffbfd69c9d7ba0e/example/game/crafty/js/crafty.js#L4306)| [Issue](https://github.com/drawcall/Proton/issues/101) |
|[montagejs/collections](https://github.com/montagejs/collections)          |`!index in this`                                                    |2.1k|  [Link](https://github.com/montagejs/collections/blob/4e19cc48904dbc6313dbe9199f347969843d2308/shim-array.js#L106)| [Issue](https://github.com/montagejs/collections/issues/252) |

</details>

Similarly, for `instanceof`, this [Sourcegraph query](https://sourcegraph.com/search?q=context:global+lang:javascript+/%5C%21%5B%5B:alnum:%5D%5D%2B+instanceof+%5B%5B:alnum:%5D%5D%2B/+-file:%5C.min%5C.js%24+count:all&patternType=standard&sm=1&groupBy=repo) shows that there are also many instances of this bug (~19k occurrences when I ran it). As before, repos with thousands of stars are affected. Some examples follow below:

<details>
  <summary>Examples</summary>
  
|Repo                                              |Bugs                                                           |Stars |Link|Issue|
|--------------------------------------------------|---------------------------------------------------------------|------|----|-----|
|[odoo/odoo](https://github.com/odoo/odoo)                   |`!e instanceof o`                        |30.1k |[Link](https://github.com/odoo/odoo/blob/a05ccee899c85c95bc82fb1a8e9c0dd4b3fd5a5c/addons/web/static/lib/ace/ace.odoo-custom.js#L3393)| [Issue](https://github.com/odoo/odoo/issues/136022) |
|[facebook/flow](https://github.com/facebook/flow)               |`!flow instanceof RegExp`                   |22k   |[Link](https://github.com/facebook/flow/blob/469629e78c90ab2da056e632f02edf56a580cf86/packages/flow-parser/test/esprima_test_runner.js#L449)| [Issue](https://github.com/facebook/flow/issues/9082) |
|[v8/v8](https://github.com/v8/v8)                       |`!e instanceof RangeError`                  |21.5k |[Link](https://github.com/v8/v8/blob/29229448d9f57735d850bc49697a678c4e0a6925/test/js-perf-test/ExpressionDepth/run.js#L54) | [Issue](https://bugs.chromium.org/p/v8/issues/detail?id=14329) |
|[linlinjava/litemall](https://github.com/linlinjava/litemall)         |`!re instanceof RegExp`                     |18.2k |[Link](https://github.com/linlinjava/litemall/blob/47ea5c7420f126081e7ef17a7182890def32457d/renard-wx/lib/wxParse/showdown.js#L2238)| [Issue](https://github.com/linlinjava/litemall/issues/542) |
|[iissnan/hexo-theme-next](https://github.com/iissnan/hexo-theme-next)     |`!elem instanceof Element`                  |15.8k |[Link](https://github.com/iissnan/hexo-theme-next/blob/9c8cea69bf0d4f91c07779d71b01814b27bbb6a1/source/lib/Han/dist/han.js#L2154)| [Issue](https://github.com/iissnan/hexo-theme-next/issues/2270) |
|[chromium/chromium](https://github.com/chromium/chromium)           |`!this instanceof Test`                     |15.3k |[Link](https://github.com/chromium/chromium/blob/ec7efe1a70a533678591c239f61fba591db9bee5/third_party/qunit/src/qunit.js#L1070)| [Issue](https://bugs.chromium.org/p/chromium/issues/detail?id=1485073) |
|[arangodb/arangodb](https://github.com/arangodb/arangodb)           |`!context instanceof WebGLRenderingContext` |13.1k |[Link](https://github.com/arangodb/arangodb/blob/1e050ea77f258045aa0bb4b1c3b1ebc60709d29d/js/apps/system/_admin/aardvark/APP/frontend/js/lib/sigma.exporters.image.js#L295)| [Issue](https://github.com/arangodb/arangodb/issues/19795) |
|[ptmt/react-native-macos](https://github.com/ptmt/react-native-macos)     |`!response instanceof Map`                  |11.3k |[Link](https://github.com/ptmt/react-native-macos/blob/0f09ff48a8c1e2310ec9eef2529d64e321c0b599/local-cli/server/util/jsPackagerClient.js#L99)| N/A (deprecated) |
|[chakra-core/ChakraCore](https://github.com/chakra-core/ChakraCore)      |`!e instanceof TypeError`                   |8.9k  |[Link](https://github.com/chakra-core/ChakraCore/blob/c3ead3f8a6e0bb8e32e043adc091c68cba5935e9/test/Array/array_splice.js#L124)| [Issue](https://github.com/chakra-core/ChakraCore/issues/6950) |
|[icindy/wxParse](https://github.com/icindy/wxParse)              |`!ext.regex instanceof RegExp`              |7.7k  |[Link](https://github.com/icindy/wxParse/blob/9d5df482294b7d39f8802d413f25d28d0d6c349e/wxParse/showdown.js#L397)| [Issue](https://github.com/icindy/wxParse/issues/378) |
|[WebKit/WebKit](https://github.com/WebKit/WebKit)               |`!e instanceof Error`                       |6.4k  |[Link](https://github.com/WebKit/WebKit/blob/ea191c94955ddd2f015f7a677b138109987620b2/JSTests/stress/spread-calling.js#L78)| [Issue](https://bugs.webkit.org/show_bug.cgi?id=261815) |
|[golden-layout/golden-layout](https://github.com/golden-layout/golden-layout) |`!column instanceof lm.items.RowOrColumn`   |6k    |[Link](https://github.com/golden-layout/golden-layout/blob/95af36d1e4c596696483c5353d32ae71886b999c/website/assets/js/goldenlayout.js#L3729)| [Issue](https://github.com/golden-layout/golden-layout/issues/855) |
|[janhuenermann/neurojs](https://github.com/janhuenermann/neurojs)       |`!config instanceof network.Configuration`  |4.4k  |[Link](https://github.com/janhuenermann/neurojs/blob/9a19adc2c3d56a4276affa06fb61524dca5bbbd9/src/storage.js#L36)| [Issue](https://github.com/janhuenermann/neurojs/issues/21) |
|[gkz/LiveScript](https://github.com/gkz/LiveScript)              |`!last instanceof While`                    |2.3k  |[Link](https://github.com/gkz/LiveScript/blob/6f754f9c51d133efa8a33504157db4c059ea23c1/lib/ast.js#L3953)| [Issue](https://github.com/gkz/LiveScript/issues/1123) |
|[CloudBoost/cloudboost](https://github.com/CloudBoost/cloudboost)       |`!obj instanceof CB.CloudObject \|\| !obj instanceof CB.CloudFile \|\| !obj instanceof CB.CloudGeoPoint \|\| !obj instanceof CB.CloudTable \|\| !obj instanceof CB.Column`                                           |1.4k  |[Link](https://github.com/CloudBoost/cloudboost/blob/aa4564056047fb6ec804590f323b7f0a2a010e3b/data-service/sdk/src/PrivateMethods.js#L66)| [Issue](https://github.com/CloudBoost/cloudboost/issues/494) |


</details>  

Within Bloomberg, we encourage the use of eslint and TypeScript, each of which have an error for these cases. However, because we allow teams to make some of their own decisions about tooling, bugs creeped through: in one large set of internal projects, we found that roughly an eighth of `in`/`instanceof` usages were negated `in` and `instanceof` expressions. More than 1% of negated `in` uses had this bug. This also affected negated `instanceof`, where more than 6% of uses had the bug. Our internal results are aligned with the data from the external sourcegraph queries: there is clearly a higher incidence of the bug on negated `instanceof` expressions compared to negated `in` expressions. While we are now fixing this internally, overall these results illustrate that this is a common problem due to the lack of ergonomics around negated `in` and `instanceof` expressions.

### Generates confusion

The negation of these expressions is not aligned with operators which have a negated version, such as `===`/`!==`. This generates confusion among developers and leads to highly upvoted and viewed questions such as [Is there a “not in” operator in JavaScript for checking object properties?](https://stackoverflow.com/questions/7972446/is-there-a-not-in-operator-in-javascript-for-checking-object-properties) and [Javascript !instanceof If Statement](https://stackoverflow.com/questions/8875878/javascript-instanceof-if-statement).

### Readability

To negate the result of an `in`/`instanceof` expression, we introduce an additional grouping operator (denoted by two parentheses). In addition, the `not` is at the beginning of the expression, unlike how this would be read in natural English. Together, both of these factors result in less readable code.

### Worse developer experience

It is common to use `in`/`instanceof` as a guard in conditionals. Inverting these conditionals to reduce indentation in code, as [this is correlated with code complexity](https://www.sciencedirect.com/science/article/pii/S0167642309000379), can lead to improved code readability and quality. With the existing operators, inverting the expression in the conditional requires the expression to be both wrapped with parentheses **and** negated.

## Solution

`!in`, a negated version of `in`, where

```js
a !in obj;
```

is equivalent to
```js
!(a in obj);
```

`!instanceof`, a negated version of `instanceof`, where

```js
a !instanceof obj;
```

is equivalent to
```js
!(a instanceof obj);
```

- Safer: No longer need to introduce additional grouping, and the negation is applied directly to the operator, as opposed to applying it next to the LHS operand in the expression.
- Improved readability: No longer requires extra grouping to negate the result of the expression. This is aligned with other operators such as `!==`. Reads more naturally and is more intuitive.
- Better developer experience: Again, easier to change when refactoring code - a single `!` needs to be added to negate the expression.

## In other languages:

Python:
```python
if item not in items:
  pass

if ref1 is not ref2:
  pass
```

Kotlin:
```kotlin
if (a !in arr) {}

if (a !is SomeClass) {}
```

C#:
```csharp
if (a is not null) {}
```

Elixir:
```elixir
a not in [1, 2, 3]
```

## Related Proposals

### Pattern matching
The pattern matching proposal proposes a new relational expression like `a in b` or `a instanceof b`, using a new operator `is`: https://github.com/tc39/proposal-pattern-matching#is-expression

In the same line as `in` and `instanceof`, we could extend the proposal to include a negated `is` operator such as `!is`. 
