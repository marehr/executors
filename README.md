# Front Matter

TODO

# Minimal executor category

## Executor type traits

### Checking that a type is an `Executor`

    template<class T> struct is_executor : see-below;

    template<class T> constexpr bool is_executor_v = is_executor<T>::value;

`is_executor<T>` publicly inherits from `std::true_type` if `T` satisfies the `Executor` requirements (see Table \ref{executor_requirements}); otherwise, it publicly inherits from `std::false_type`.

XXX Is this implementable? For example, there's no way to check that `T::spawn_execute(f)` is valid for all `f` if all we have is `T`. However, we could do something like check with `function<void()>`, and
    insist that if it works for `function<void()>`, then it must work for all nullary functions.
        

### Associated future type

    template<class Executor, class T>
    struct executor_future
    {
      // XXX a future proposal can relax this to enable user-defined future types 
      using type = std::future<T>;
    };
    
    template<class Executor, class T>
    using executor_future_t = typename executor_future<Executor,T>::type;

### Classifying forward progress guarantees of executor operations

    // XXX this section could use a bikeshed

    struct possibly_blocking_execution_tag {};
    struct blocking_execution_tag {};
    struct nonblocking_execution_tag {};
    
    template<class Executor>
    struct executor_operation_forward_progress
    {
      private:
        // exposition only
        template<class T>
        using helper = typename T::operation_forward_progress;
    
      public:
        using type = std::experimental::detected_or_t<
          possibly_blocking_execution_tag, helper, Executor
        >;
    };

    template<class Executor>
    using executor_operation_forward_progress_t =
      typename executor_operation_forward_progress<Executor>::type;

## `Executor`

1. The `Executor` requirements form the basis of the executor concept taxonomy;
   every executor satisfies the `Executor` requirements. This set of
   requirements specifies operations for creating execution agents.

2. In Table \ref{executor_requirements}, `f` denotes a `MoveConstructible` function object with zero arguments whose result type is `R`,
   and `x` denotes an object of type `X`.

3. A type `X` satisfies the `Executor` requirements if:
  * `X` satisfies the `CopyConstructible` requirements (17.6.3.1).
  * For any `f` and `x`, at least one of the expressions in Table \ref{executor_requirements} are valid and have the indicated semantics.

Table: (Executor requirements) \label{executor_requirements}

| Expression                                                                         | Return Type                                                   |  Operational semantics                                       | Assertion/note/pre-/post-condition                                                                                                  |
|------------------------------------------------------------------------------------|---------------------------------------------------------------|--------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| `x.spawn_-` `execute(std::move(f))`                                                | `void`                                                        |  Creates an execution agent which invokes `f()`              | Effects: blocks the forward progress of the caller until `f` is finished as given by `executor_operation_-` `forward_progress_t<X>` |
|                                                                                    |                                                               |                                                              |                                                                                                                                     |
|                                                                                    |                                                               |                                                              |                                                                                                                                     |
|                                                                                    |                                                               |                                                              |                                                                                                                                     |
| `x.async_-` `execute(std::move(f))`                                                | `executor_-` `future_t<X,R>`                                  |  Creates an execution agent which invokes `f()`              | Effects: blocks the forward progress of the caller until `f` is finished as given by `executor_operation_-` `forward_progress_t<X>` |
|                                                                                    |                                                               |  Returns the result of `f()` via the resulting future object |                                                                                                                                     |
|                                                                                    |                                                               |                                                              |                                                                                                                                     |
|                                                                                    |                                                               |                                                              |                                                                                                                                     |

# Bulk (Parallelism TS) executor category

## Bulk executor traits

### Classifying forward progress guarantees of groups of execution agents

    struct sequenced_execution_tag {};
    struct parallel_execution_tag {};
    struct unsequenced_execution_tag {};

    // TODO a future proposal can define this category
    // struct concurrent_execution_tag {};

    template<class Executor>
    struct executor_execution_category
    {
      private:
        // exposition only
        template<class T>
        using helper = typename T::execution_category;

      public:
        using type = std::experimental::detected_or_t<
          unsequenced_execution_tag, helper, Executor
        >;
    };

    template<class Executor>
    using executor_execution_category_t = typename executor_execution_category<Executor>::type;

XXX TODO the relative "strength" of these categories should be defined

### Associated shape type

    template<class Executor>
    struct executor_shape
    {
      private:
        // exposition only
        template<class T>
        using helper = typename T::shape_type;
    
      public:
        using type = std::experimental::detected_or_t<
          size_t, helper, Executor
        >;

        // exposition only
        static_assert(std::is_integral_v<type>, "shape type must be an integral type");
    };

    template<class Executor>
    using executor_shape_t = typename executor_shape<Executor>::type;

### Associated index type

    template<class Executor>
    struct executor_index
    {
      private:
        // exposition only
        template<class T>
        using helper = typename T::index_type;

      public:
        using type = std::experimental::detected_or_t<
          executor_shape_t<Executor>, helper, Executor
        >;

        // exposition only
        static_assert(std::is_integral_v<type>, "index type must be an integral type");
    };

    template<class Executor>
    using executor_index_t = typename executor_index<Executor>::type;

## `BulkExecutor`

1. The `BulkExecutor` requirements form the basis of the bulk executor concept.
   This set of requirements specifies operations for creating groups of execution agents in bulk from a single operation.

2. In Table \ref{bulk_executor_requirements},
    * `f` denotes a `CopyConstructible` function object with three arguments,
    * `n` denotes a shape object whose type is `executor_shape_t<X>`.
    * `rf` denotes a `CopyConstructible` function object with one argument whose result type is `R`,
    * `sf` denotes a `CopyConstructible` function object with one argument whose result type is `S`,
    * `i` denotes an object whose type is `executor_index_t<X>`,
    * `r` denotes an object whose type is `R`, 
    * `s` denotes an object whose type is `S`, and
    * `pred` denotes a future object whose result is `pr`.

2. A class `X` satisfies the requirements of a bulk executor if `X` satisfies
   the `Executor` requirements and the expressions of Table
   \ref{bulk_executor_requirements} are valid and have the indicated semantics.

Table: (Bulk executor requirements) \label{bulk_executor_requirements}

| Expression                                                        | Return Type                                                       |  Operational semantics                                                                                     | Assertion/note/pre-/post-condition                                                                                                                         |
|-------------------------------------------------------------------|-------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `x.bulk_-` `execute(f, n, rf, sf)`                                | `R`                                                               |  Creates a group of execution agents of shape `n` which invoke `f(i, r, s)`                                | Note: blocks the forward progress of the caller until all invocations of `f` are finished.                                                                 |
|                                                                   |                                                                   |  Returns the result of `rf(n)`                                                                             | Effects: invokes `rf(n)` on an unspecified execution agent.                                                                                                |
|                                                                   |                                                                   |                                                                                                            | Effects: invokes `sf(n)` on an unspecified execution agent.                                                                                                |
|                                                                   |                                                                   |                                                                                                            |                                                                                                                                                            |
| `x.bulk_async_-` `execute(f, n, rf, sf)`                          | `executor_-` `future_t<X,R>`                                      |  Creates a group of execution agents of shape `n` which invoke `f(i, r, s)`                                | Effects: blocks the forward progress of the caller until all invocations of `f` are finished as required by `executor_operation_-` `forward_progress_t<X>` |
|                                                                   |                                                                   |  Asynchronously returns the result of `rf(n)` via the resulting future object                              | Effects: invokes `rf(n)` on an unspecified execution agent.                                                                                                |
|                                                                   |                                                                   |                                                                                                            | Effects: invokes `sf(n)` on an unspecified execution agent.                                                                                                |
|                                                                   |                                                                   |                                                                                                            |                                                                                                                                                            |
| `x.bulk_then_-` `execute(f, n, rf, pred, sf)`                     | `executor_-` `future_t<X,R>`                                      |  Creates a group of execution agents of shape `n` which invoke `f(i, r, pr, s)` after `pred` becomes ready | Effects: blocks the forward progress of the caller until all invocations of `f` are finished as required by `executor_operation_-` `forward_progress_t<X>` |
|                                                                   |                                                                   |  Asynchronously returns the result of `rf(n)` via the resulting future.                                    | Effects: invokes `rf(n)` on an unspecified execution agent.                                                                                                |
|                                                                   |                                                                   |                                                                                                            | Effects: invokes `sf(n)` on an unspecified execution agent.                                                                                                |
|                                                                   |                                                                   |                                                                                                            | If `pred`'s result type is `void`, `pr` is ommitted from `f`'s invocation.                                                                                 |
|                                                                   |                                                                   |                                                                                                            | Post: `pred` is invalid if it is not a shared future.                                                                                                      |


XXX TODO: need to specify how `executor_execution_category_t` describes the forward progress requirements of a group of execution agents wrt each other

# (Networking TS) executor category

XXX TODO

# Executor Customization Points

## In general

1. The functions described in this clause are *executor customization points*.
   Executor customization points provide a uniform interface to all executor types.

2. An executor customization point does not participate in overload resolution
   if its `exec` parameter is not an `Executor`. Executor customization points
   follow the design suggested by [N4381](wg21.link/N4381).

   XXX the reason p2 is included is to define some general purpose wording for executor customization points in order to avoid repetition below.
       but, there's still some repetition

## Function template `execution::spawn_execute()`

1.  ```
    template<class Executor, class Function>
    void spawn_execute(Executor& exec, Function&& f);
    ```

2. *Effects:* calls `exec.spawn_execute(std::forward<Function>(f))` if that call is well-formed; otherwise, calls
   `DECAY_COPY(std::forward<Function>(f))()` in a new execution agent with the call to `DECAY_COPY()` being evaluated
   in the thread that called `spawn_execute`. Any return value is discarded.

## Function template `execution::execute()`

1.  ```
    template<class Executor, class Function>
    result_of_t<decay_t<Function>()>
    execute(Executor& exec, Function&& f);
    ```

2. *Effects:* calls `exec.execute(std::forward<Function>(f))` if that call is well-formed; otherwise, calls
   `DECAY_COPY(std::forward<Function>(f))()` in a new execution agent with the call to `DECAY_COPY()` being
   evaluated in the thread that called `execute`.

3. *Returns:* The return value of `f`.

4. *Synchronization:* The invocation of `execute` synchronizes with (1.10) the invocation of `f`.

## Function template `execution::async_execute()`

1.  ```
    template<class Executor, class Function>
    executor_future_t<
      Executor,
      result_of_t<decay_t<Function>()>
    >
    async_execute(Executor& exec, Function&& f);
    ```

2. *Effects:* calls `exec.async_execute(std::forward<Function>(f))` if that call is well-formed;

    otherwise, calls `DECAY_COPY(std::forward<Function>(f))()` in a new execution agent with the call to `DECAY_COPY()` being
    evaluated in the thread that called `async_execute`. Any return value is stored as the result in the shared
    state. Any exception propagated from the execution of `INVOKE(DECAY_COPY(std::forward<Function>(f))` is stored as
    the exceptional result in the shared state.

3. *Returns:* An object of type

    `executor_future_t<Executor,result_of_t<decay_t<Function>()>>`
   
    that refers to the shared state created by this call to `async_execute`.

4. *Synchronization:*
    * the invocation of `async_execute` synchronizes with (1.10) the invocation of `f`.
    * the completion of the function `f` is sequenced before (1.10) the shared state is made ready.

## Function template `execution::then_execute()`

1.  ```
    template<class Executor, class Function, class Future>
    executor_future_t<Executor,see-below>
    then_execute(Executor& exec, Function&& f, Future& predecessor);
    ```

2. *Effects:* calls

    `exec.then_execute(std::forward<Function>(f), std::forward<Future>(predecessor))`
    
    if that call is well-formed; otherwise:

    * Creates a shared state that is associated with the returned future object.

    * Creates a new execution agent `predecessor` becomes ready. The execution agent calls `f(pred)`, where `pred` is a reference to
      the `predecessor` state if it is not `void`. Otherwise, the execution agent calls `f()`.

    * Any return value of `f` is stored as the result in the shared state of the resulting future.

3. *Returns:* `executor_future_t<Executor,result_of_t<decay_t<Function>()>` when `predecessor` is a `void` future. Otherwise,
   `executor_future_t<Executor,result_of_t<decay_t<Function>(T&)>>` where `T` is the result type of the `predecessor` future.

4. *Synchronization:*
    * the invocation of `then_execute` synchronizes with (1.10) the invocation of `f`.
    * the completion of the invocation of `f` is sequenced before (1.10) the shared state is made ready.

5. *Postconditions:* If the `predecessor` future is not a shared future, then `predecessor.valid() == false`.


## Function template `execution::bulk_execute()`

1.  ```
    template<class Executor, class Function1, class Function2, class Function3>
    result_of_t<Function2(executor_shape_t<Executor>)>
    bulk_execute(Executor& exec, Function1 f, executor_shape_t<Executor> shape,
                 Function2 result_factory, Function3 shared_factory);
    ```

2. *Effects:* calls `exec.bulk_execute(f, shape, result_factory, shared_factory)` if that call is well-formed;
   otherwise:
   
   * Calls `result_factory(shape)` and `shared_factory(shape)` in an unspecified execution agent. The results
     of these invocations are stored to shared state.

   * Creates a new group of execution agents of shape `shape`. Each execution agent calls `f(idx, result, shared)`, where
     `idx` is the index of the execution agent, and `result` and `shared` are references to the respective shared state.
     Any return value of `f` is discarded.

3. *Returns:* An object of type `result_of_t<Function2(executor_shape_t<Executor>)>` that refers to the result shared state created by
   this call to `bulk_execute`.

4. *Synchronization:* The completion of the functions `result_factory` and
   `shared_factory` happen before the creation of the group of execution
   agents.

## Function template `execution::bulk_async_execute()`

1.  ```
    template<class Executor, class Function1, class Function2, class Function3>
    executor_future_t<
      Executor,
      result_of_t<Function2(executor_shape_t<Executor>)>
    >
    bulk_async_execute(Executor& exec, Function1 f, executor_shape_t<Executor> shape,
                       Function2 result_factory, Function3 shared_factory);
    ```

2. *Effects:* calls `exec.bulk_async_execute(f, shape, result_factory, shared_factory)` if that call is well-formed;
   otherwise:

    * Calls `result_factory(shape)` and `shared_factory(shape)` in an unspecified execution agent. The results of these
      invocations are stored to shared state.

    * Creates a new group of execution agents of shape `shape`. Each execution agent calls `f(idx, result, shared)`, where
      `idx` is the index of the execution agent, and `result` and `shared` are references to the respective shared state.
      Any return value of `f` is discarded.

3. *Returns:* An object of type

    `executor_future_t<Executor,result_of_t<Function2(executor_shape_t<Executor>)>`
    
    that refers to the shared result state created by this call to `bulk_async_execute`.

4. *Synchronization:*
    * the invocation of `bulk_async_execute` synchronizes with (1.10) the invocations of `f`.
    * the completion of the functions `result_factory` and `shared_factory` happen before the creation of the group of execution agents.
    * the completion of the invocations of `f` are sequenced before (1.10) the result shared state is made ready.

## `execution::bulk_then_execute()`

1.  ```
    template<class Executor, class Function1, class Future, class Function2, class Function3>
    executor_future_t<
      Executor,
      result_of_t<Function2(executor_shape_t<Executor>)>
    >
    bulk_then_execute(Executor& exec, Function1 f, executor_shape_t<Executor> shape,
                      Future& predecessor,
                      Function2 result_factory, Function3 shared_factory);
    ```

2. *Effects:* calls `exec.bulk_then_execute(f, shape, predecessor, result_factory, shared_factory)` if that call is well-formed;
   otherwise:

   * Calls `result_factory(shape)` and `shared_factory(shape)` in an unspecified execution agent. The results of these
     invocations are stored to shared state.

   * Creates a new group of execution agents of shape `shape` after `predecessor` becomes ready. Each execution agent calls `f(idx, result, pred, shared)`, where
     `idx` is the index of the execution agent, `result` is a reference to the result shared state, `pred` is a reference to
     the `predecessor` state if it is not `void`. Otherwise, each execution agent calls `f(idx, result, shared)`.
     Any return value of `f` is discarded.

3. *Returns:* An object of type

    `executor_future_t<Executor,result_of_t<Function2(executor_shape_t<Executor>)>`
    
    that refers to the shared result state created by this call to `bulk_then_execute`.

4. *Synchronization:*
    * the invocation of `bulk_then_execute` synchronizes with (1.10) the invocations of `f`.
    * the completion of the functions `result_factory` and `shared_factory` happen before the creation of the group of execution agents.
    * the completion of the invocations of `f` are sequenced before (1.10) the result shared state is made ready.

5. *Postconditions:* If the `predecessor` future is not a shared future, then `predecessor.valid() == false`.

## Networking TS-specific customization points

XXX TODO

# Execution policy interoperation

```
class parallel_execution_policy
{
  public:
    // types:
    using execution_category = parallel_execution_tag;
    using executor_type = implementation-defined;

    // executor access
    const executor_type& executor() const noexcept;

    // execution policy factory
    template<class Executor>
    see-below on(Executor&& exec) const;
};

class sequenced_execution_tag { by-analogy-to-parallel_execution_policy };
class parallel_unsequenced_execution_tag { by-analogy-to-parallel_execution_policy };
```

## Associated executor

1. Each execution policy is associated with an executor, and this executor is called its *associated executor*.

2. The type of an execution policy's associated executor shall satisfy the requirements of `BulkExecutor`.

3. When an execution policy is used as a parameter to a parallel algorithm, the
   execution agents that invoke element access functions are created by the
   execution policy's associated executor.

4. The type of an execution policy's associated executor is the same as the member type `executor_type`.

## Execution category

1. Each execution policy is categorized by an *execution category*.

2. When an execution policy is used as a parameter to a parallel algorithm, the
   execution agents it creates are guaranteed to make forward progress and
   execute invocations of element access functions as ordered by its execution
   category.

3. An execution policy's execution category is given by the member type `execution_category`.

4. The execution category of an execution policy's associated executor shall not be weaker than the execution policy's execution category.

## Associated executor access

1.  ```
    const executor_type& executor() const noexcept;
    ```

2. *Returns:* The execution policy's associated executor.

## Execution policy factory

1.  ```
    template<class Executor>
    see-below on(Executor&& exec) const;
    ```

2. Let `T` be `decay_t<Executor>`.

3. *Returns:* An execution policy whose execution category is `execution_category`. If `T` satisfies the requirements of
   `BulkExecutor`, the returned execution policy's associated executor is equivalent to `exec`. Otherwise,
   the returned execution policy's associated executor is an adaptation of `exec`.

   XXX TODO: need to define what adaptation means

3. *Remarks:* This member function shall not participate in overload resolution unless `is_executor_v<T>` is `true` and
   `executor_execution_category_t<T>` is as strong as `execution_category`.

# Control structure interoperation

## Function template `async`

1. The function template `async` provides a mechanism to invoke a function in a new
   execution agent created by an executor and provides the result of the function in the
   future object with which it shares a state.

    ```
    template<class Executor, class Function, class... Args>
    executor_future_t<Executor, result_of_t<decay_t<Function>(decay_t<Args>...)>>
    async(Executor& exec, Function&& f, Args&&... args);
    ```

2. *Returns:* Equivalent to:

    `return execution::async_execute(exec, [=]{ return INVOKE(f, args...); });`

    XXX This forwarding doesn't look correct to me

## `std::future::then()`

1. The member function template `then` provides a mechanism for attaching a *continuation* to a `std::future` object,
   which will be executed on a new execution agent created by an executor.

    ```
    template<class T>
    template<class Executor, class Function>
    executor_future_t<Executor, see-below>
    future<T>::then(Executor& exec, Function&& f);
    ```

2. *Returns:* Equivalent to:

    `return execution::then_execute(exec, std::forward<Function>(f), *this);`

    XXX This forwarding doesn't look correct to me

## `std::shared_future::then()`

1. The member function template `then` provides a mechanism for attaching a *continuation* to a `std::shared_future` object,
   which will be executed on a new execution agent created by an executor.

    ```
    template<class T>
    template<class Executor, class Function>
    executor_future_t<Executor, see-below>
    shared_future<T>::then(Executor& exec, Function&& f);
    ```

2. *Returns:* Equivalent to:

    `return execution::then_execute(exec, std::forward<Function>(f), *this);`

## Function template `invoke`

1. The function template `invoke` provides a mechanism to invoke a function in a new
   execution agent created by an executor and return result of the function.

    ```
    template<class Executor, class Function, class... Args>
    result_of_t<F&&(Args&&...)>
    invoke(Executor& exec, Function&& f, Args&&... args);
    ```

2. *Returns:* Equivalent to:

    `return execution::execute(exec, [&]{ return INVOKE(f, args...); });`

## Task block

### Function template `define_task_block_restore_thread()`

1.  ```
    template<class Executor, class F>
    void define_task_block_restore_thread(Executor& exec, F&& f);
    ```

2. *Requires:* Given an lvalue `tb` of type `task_block`, the expression `f(tb)` shall be well-formed.

3. *Effects:* Constructs a `task_block tb`, creates a new execution agent, and calls `f(tb)` on that execution agent.

4. *Throws:* `exception_list`, as specified in version two of the Paralellism TS.

5. *Postconditions:* All tasks spawned from `f` have finished execution.

6. *Remarks:* Unlike `define_task_block`, `define_task_block_restore_thread` always returns on the same thread as the one on which it was called.

### `task_block` member function template `run`

1.  ```
    template<class Executor, class F>
    void run(Executor& exec, F&& f);
    ```

2. *Requires:* `F` shall be `MoveConstructible`. `DECAY_COPY(std::forward<F>(f))()` shall be a valid expression.

3. *Preconditions:* `*this` shall be an active `task_block`.

4. *Effects:* Evaluates `DECAY_COPY(std::forward<F>(f))()`, where `DECAY_COPY(std::forward<F>(f))` is evaluated synchronously within the current thread.
   The call to the resulting copy of the function object is permitted to run on an execution agent created by `exec` in an unordered fashion relative to
   the sequence of operations following the call to `run(exec, f)` (the continuation), or indeterminately-sequenced within the same thread as the continuation.
   The call to `run` synchronizes with the next invocation of `wait` on the same `task_block` or completion of the nearest enclosing `task_block` (i.e., the `define_task_block` or
   `define_task_block_restore_thread` that created this `task_block`.

5. *Throws:* `task_cancelled_exception`, as described in version 2 of the Parallelism TS.

# Polymorphic executor wrapper

## Class `executor`

XXX this section has a lot of wording redundant with `any`. it would be nice if the constructors & 
    modifiers effects/return clauses could say something like "as if by corresponding-expression-involving-some-expository-`std::any`-object"


```
class executor
{
  public:
    // XXX could contemplate introducing a type indicating that the
    //     forward progress type has been erased
    using operation_forward_progress = possibly_blocking_execution_tag;

    // construction and destruction
  
    // XXX a future proposal can introduce the default constructor
    //     and account for empty states
    // executor() noexcept;

    executor(const executor& other);

    // XXX this proposal omits move construction because
    //     it seems like it would leave the other executor
    //     in an empty state
    // executor(executor&& other) noexcept;

    template<class Executor>
    executor(Executor&& exec);

    // assignments
    executor& operator=(const executor& rhs);

    template<class Executor>
    executor& operator=(exec& rhs);

    // modifiers
    void swap(executor& rhs) noexcept;

    // execution agent creation
    template<class Function>
    future<result_of_t<decay_t<Function>()>>
    async_execute(Function&& f);

    template<class Function>
    void spawn_execute(Function&& f);

    // XXX future proposals can introduce member functions for each
    //     Executor customization point

  private:
    std::any contained_executor_; // exposition only
};
```

1. An object of class `executor` stores an instance of any `Executor` type. The stored instance is called the *contained executor*.

### Construction and destruction

1.  ```
    executor(const executor& other);
    ```

2. *Effects:* Constructs an object of type `executor` with a copy of `other`'s contained executor.

3. *Throws:* Any exceptions arising from calling the selected constructor of the contained executor.


4.  ```
    executor(executor&& other);
    ```

5. *Effects:* Constructs an object of type `executor` with a contained executor move constructed from `other`'s contained executor.

6.  ```
    template<class Executor>
    executor(Executor&& exec);
    ```

7. Let `T` be `decay_t<Executor>`.

8. *Effects:* Constructs an object of type `executor` that contains an executor of type `T` direct-initialized with
   `std::forward<Executor>(exec)`.

9. *Remarks:* This constructor shall not participate in overload resolution unless `is_executor_v<T>` is `true` or if `T` is the same type as `executor`.

10. *Throws:* Any exception thrown by the selected constructor of `T`.

### Assignment

1.  ```
    executor& operator=(const executor& rhs);
    ```

2. *Effects:* As if by `executor(rhs).swap(*this)`. No effects if an exception is thrown.

3. *Returns:* `*this`.

4. *Throws:* Any exceptions arising from the copy constructor of the contained executor.

5.  ```
    template<class Executor>
    executor& operator=(Executor&& rhs);
    ```

6. Let `T` be `decay_t<Executor>`.

7. *Effects:* Constructs an object `tmp` of type `executor` that contains an executor of type `T` direct-initialized with
   `std::forward<Executor>(rhs)`, and `tmp.swap(*this)`. No effects if an exception is thrown.

7. *Remarks*: This operator does not participate in overload resolution unless `is_executor_v<T>` is `true` or if `T` is the same type as `executor`.

8. *Throws:* Any exception thrown by the selected constructor of `T`.

### Modifiers

1.  ```
    void swap(executor& rhs) noexcept;
    ```

2. *Effects:* Exchanges the contained executors of `*this` and `rhs`.

### Execution agent creation

#### Member function `async_execute`

1.  ```
    template<class Function>
    future<result_of_t<decay_t<Function>()>>
    async_execute(Function&& f);
    ```

2. Let `exec` be this `executor`'s contained executor.

3. *Returns:* Equivalent to `execution::async_execute(exec, std::forward<Function>(f))`.

XXX This equivalent expression requires giving `std::future<T>` a constructor which would move convert from `executor_future_t<X,T>`, where `X` is the type of `exec`.

#### Member function `spawn_execute`

1.  ```
    template<class Function>
    void spawn_execute(Function&& f);
    ```

2. Let `exec` be this `executor`'s contained executor.

3. *Effects:* As if by `execution::spawn_execute(exec, std::forward<Function>(f))`.

### Non-member functions

1.  ```
    void swap(executor& x, executor& y) noexcept;
    ```

2. *Effects:* As if by `x.swap(y)`.


# Thread pool type

XXX TODO

