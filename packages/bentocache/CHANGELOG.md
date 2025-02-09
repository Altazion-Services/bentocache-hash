# bentocache

## 1.1.0

### Minor Changes

- 07224ba: Add two new functions in the factory callback context:

  ```ts
  cache.getOrSet({
    key: 'foo',
    factory: ({ skip, fail }) => {
      const item = await getFromDb()
      if (!item) {
        return skip()
      }

      if (item.isInvalid) {
        return fail('Item is invalid')
      }

      return item
    },
  })
  ```

  ## Skip

  Returning `skip` in a factory will not cache the value, and `getOrSet` will returns `undefined` even if there is a stale item in cache.
  It will force the key to be recalculated on the next call.

  ## Fail

  Returning `fail` in a factory will not cache the value and will throw an error. If there is a stale item in cache, it will be used.

- 2578357: Added a `serialize: false` to the memory driver.

  It means that, the data stored in the memory cache will not be serialized/parsed using `JSON.stringify` and `JSON.parse`. This allows for a much faster throughput but at the expense of:

  - not being able to limit the size of the stored data, because we can't really know the size of an unserialized object
  - Having inconsistent return between the L1 and L2 cache. The data stored in the L2 Cache will always be serialized because it passes over the network. Therefore, depending on whether the data is retrieved from the L1 and L2, we can have data that does not have the same form. For example, a Date instance will become a string if retrieved from the L2, but will remain a Date instance if retrieved from the L1. So, you should put extra care when using this feature with an additional L2 cache.

### Patch Changes

- 09cd234: Refactoring of CacheEntryOptions class. We switch to a simple function that returns an object rather than a class. Given that CacheEntryOptions is heavily used : it was instantiated for every cache operation, we gain a lot in performance.
- 1939ab9: Deleted the getters usage in the CacheEntryOptions file. Looks like getters are super slow. Just removing them doubled the performance in some cases.

  Before :

  ```sh
  ┌─────────┬──────────────────────────────────┬─────────────────────┬─────────────────────┬────────────────────────┬────────────────────────┬─────────┐
  │ (index) │ Task name                        │ Latency avg (ns)    │ Latency med (ns)    │ Throughput avg (ops/s) │ Throughput med (ops/s) │ Samples │
  ├─────────┼──────────────────────────────────┼─────────────────────┼─────────────────────┼────────────────────────┼────────────────────────┼─────────┤
  │ 0       │ 'L1 GetOrSet - BentoCache'       │ '16613 ± 97.87%'    │ '1560.0 ± 45.00'    │ '613098 ± 0.10%'       │ '641026 ± 19040'       │ 83796   │
  │ 1       │ 'L1 GetOrSet - CacheManager'     │ '953451 ± 111.03%'  │ '160022 ± 3815.00'  │ '5700 ± 1.23%'         │ '6249 ± 151'           │ 1049    │
  │ 4       │ 'Tiered GetOrSet - BentoCache'   │ '16105 ± 98.11%'    │ '1515.0 ± 45.00'    │ '636621 ± 0.08%'       │ '660066 ± 20206'       │ 86675   │
  │ 5       │ 'Tiered GetOrSet - CacheManager' │ '877297 ± 111.36%'  │ '161617 ± 2876.00'  │ '5948 ± 0.67%'         │ '6187 ± 112'           │ 1140    │
  │ 6       │ 'Tiered Get - BentoCache'        │ '1542.4 ± 4.43%'    │ '992.00 ± 18.00'    │ '973931 ± 0.03%'       │ '1008065 ± 17966'      │ 648343  │
  │ 7       │ 'Tiered Get - CacheManager'      │ '1957.6 ± 0.51%'    │ '1848.0 ± 26.00'    │ '534458 ± 0.02%'       │ '541126 ± 7722'        │ 510827  │
  └─────────┴──────────────────────────────────┴─────────────────────┴─────────────────────┴────────────────────────┴────────────────────────┴─────────┘
  ```

  After:

  ```sh
  ┌─────────┬──────────────────────────────────┬─────────────────────┬─────────────────────┬────────────────────────┬────────────────────────┬─────────┐
  │ (index) │ Task name                        │ Latency avg (ns)    │ Latency med (ns)    │ Throughput avg (ops/s) │ Throughput med (ops/s) │ Samples │
  ├─────────┼──────────────────────────────────┼─────────────────────┼─────────────────────┼────────────────────────┼────────────────────────┼─────────┤
  │ 0       │ 'L1 GetOrSet - BentoCache'       │ '9610.3 ± 98.26%'   │ '1109.0 ± 29.00'    │ '879036 ± 0.05%'       │ '901713 ± 22979'       │ 143980  │
  │ 1       │ 'L1 GetOrSet - CacheManager'     │ '906687 ± 110.96%'  │ '172470 ± 1785.00'  │ '5601 ± 0.56%'         │ '5798 ± 61'            │ 1103    │
  │ 4       │ 'Tiered GetOrSet - BentoCache'   │ '8752.8 ± 98.40%'   │ '1060.0 ± 19.00'    │ '924367 ± 0.04%'       │ '943396 ± 17219'       │ 158461  │
  │ 5       │ 'Tiered GetOrSet - CacheManager' │ '925163 ± 111.45%'  │ '173578 ± 2970.00'  │ '5590 ± 0.55%'         │ '5761 ± 100'           │ 1081    │
  │ 6       │ 'Tiered Get - BentoCache'        │ '556.57 ± 0.52%'    │ '511.00 ± 10.00'    │ '1923598 ± 0.01%'      │ '1956947 ± 37561'      │ 1796720 │
  │ 7       │ 'Tiered Get - CacheManager'      │ '2060.2 ± 2.54%'    │ '1928.0 ± 20.00'    │ '513068 ± 0.02%'       │ '518672 ± 5325'        │ 485387  │
  └─────────┴──────────────────────────────────┴─────────────────────┴─────────────────────┴────────────────────────┴────────────────────────┴─────────┘
  ```

  Pretty good improvement 😁

## 1.0.0

- eeb3c8c: BREAKING CHANGES:
  This commit changes the API of the `gracePeriod` option.
  - `gracePeriod` is now `grace` and should be either `false` or a `Duration`.
  - If you were using the `fallbackDuration` option, you should now use the `graceBackoff` option at the root level.
- 82e9d6c: Previously, `suppressL2Errors` was automatically enabled even when we had just a L2 layer. Which can be confusing, because errors were filtered out.

  Now `suppressL2Errors` is a bit more intelligent and will only be enabled if you have a L1 layer. Unless you explicitly set it to `true`.

- 716a423: BREAKING CHANGES :

  `undefined` values are forbidden in the cache. If you are trying to cache `undefined`, you will now get an error. This is a breaking change because it was previously allowed.

  If you want to cache something to represent the absence of a value, you can use `null` instead of `undefined`.

- 4478db6: BREAKING CHANGES

  ## API Changes for timeouts

  The timeout options have changed APIs:
  `{ soft: '200ms', hard: '2s' }`

  Becomes:

  ```ts
  getOrSet({ timeout: '200ms', hardTimeout: '2s' })
  ```

  You can now also use `0` for `timeout` which means that, if a stale value is available, then it will be returned immediately, and the factory will run in the background. SWR-like, in short.

  ## Default timeout

  Now, the default timeout is `0`. As explained above, this enables the SWR-like behavior by default, which is a good default for most cases and what most people expect.

- 7ae55e2: Added an `onFactoryError` option that allows to catch errors that happen in factories, whether they are executed in background or not.

  ```ts
  const result = await cache.getOrSet({
    key: 'foo',
    grace: '5s',
    factory: () => {
      throw new MyError()
    },
    onFactoryError: (error) => {
      // error is an instance of errors.E_FACTORY_ERROR
      // error.cause is the original error thrown by the factory
      // you can also check if the factory was executed in background with error.isBackgroundFactory
      // and also get the key with error.key. Will be `foo` in this case
    },
  })
  ```

- 27a295d: Keep only POJO syntax

  This commit remove the "legacy" syntax and only keep the POJO syntax.
  For each method, the method signature is a full object, for example :

  ```ts
  bento.get({ key: 'foo ' })
  bento.getOrSet({
    key: 'foo',
    factory: () => getFromDb(),
  })
  ```

- f0b1008: The memory driver can now accept `maxSize` and `maxEntrySize` in human format. For example, `maxSize: '1GB'` or `maxEntrySize: '1MB'`.

  We use https://www.npmjs.com/package/bytes for parsing so make sure to respect the format accepted by this module.

- a8ac574: This commit adds a new custom behavior for handling GetSet operation when end-user is using a single L2 storage without L1 cache.
