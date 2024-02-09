# RFC Template

- Feature Name: `limitador_multithread_inmemory`
- Start Date: 2023-11-02
- RFC PR: [Kuadrant/architecture#0000](https://github.com/Kuadrant/architecture/pull/0000)

# Summary
[summary]: #summary

Enable Limitador service to process requests in parallel when configured to use In Memory storage.

# Motivation
[motivation]: #motivation

Currently, Limitador service is single threaded, regardless of its chosen storage. This means that it can only process one request at a time. 
Alternatively, we could use a multithreading approach to process requests in parallel, which would improve the overall performance of Limitador.
However, this would introduce some particular behaviour regarding the _Accuracy_ of the defined limit counters and the _Throughput_ of the service.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In order to achieve the desired multi-threading functionality, we'd need to make sure that the user understands the
trade-offs that this approach implies and that they are aware of the possible consequences of using it, and how to
configure the service in order to obtain the desired behaviour. Regarding this last statement, we will need to
introduce an interface that allows the user to meet their requirements.

A couple of concepts we need to define in order to understand the proposal:
* **Accuracy**: The _accuracy_ is defined by the service's ability to enforce the limits in an exact factual way. This means
  that if the service is configured to allow 10 requests per second, it will only allow 10 requests per second, and
  not 11 or 9. In terms of observability, the counters, will be monotonically strictly decreasing without repeating any
  value given per service.
* **Throughput**: The _throughput_ of a service is defined by the number of requests that the service can process in a
  given time. As a consequence of having higher throughput, we need to introduce two more concepts:
  * **Overshoot**: The _overshooting_ of a limit counter is defined by the difference between the _expected_ value of the counter
    and the value that the counter has in the storage, when the stored value is **greater** than the expected one.
  * **Undershoot**: The _undershooting_ of a limit counter is defined the same way the **Overshoot** is, but the stored value
    is **lower** than the expected one.

## Behaviour

When using the multi-threading approach, we need to understand the trade-offs that we are making. The main one is that
it's not possible to have both _Accuracy_ and _Throughput_ at the same time. This means that we need to choose one of
them and configure the service accordingly, favouring accuracy comes close to the current single thread implementation,
or concede to _overshoot_ or _undershoot_ in order to have higher throughput. However, we can still have decent values for both of them, if we choose to
introduce a more complex implementation (thread pools, PID control, etc.).

[IMG Accuracy vs Throughput]

## Configuration

In order to configure Limitador, we need to introduce a clear interface that allows the user to choose the desired
behaviour. This interface should be able to seamlessly integrate with the current configuration options, and at least
at the first implementation, it should be able to be configured at initialization time. The fact that we are using
multi-threading or a single thread, it's not something that the user should be aware of, so we need to abstract that
away from them, in the end, they would only care about the "precision" of the service.

### Example

In this example, we are configuring the service to use the _InMemory_ storage and to use the _Throughput_ mode.
```bash
limitador-server --mode=throughput memory
```

In a future iteration, we might be able to provide a richer interface that allows the user to configure the service
in a balanced and/or more granular way.
```bash
limitador-server --mode=balanced --accuracy=0.1 --throughput=0.9 memory
```

or simply
```bash
limitador-server --accuracy=0.1 --throughput=0.9 memory
```

## Implications

### Accuracy

When using the _Accuracy_ mode, the service will behave in a similar way to the current implementation, most likely
in a single thread. This means that the service will be able to process one request at a time, and the _accuracy_ of
the limit counters will always be as expected. This mode is the one that one should use when it's important to enforce
the limits as accurately as possible, and the throughput is not a concern.

### Throughput

When using the _Throughput_ mode, the service will fully behave in a multi-threaded way, which means that it will be able
to process multiple requests at the same time. However, this will introduce some particular behaviour regarding the
_accuracy_ of the limit counters, which will be affected by the _overshoot_ and _undershoot_ concepts.

#### Overshoot

Given the following (simplified) limit definitions:

Limit 1:
```yaml
namespace: example.org
max_value: 4
seconds: 60
```
This could be translated to _any_ request to the namespace `example.org` can be authorized up to 4 times in a span
of 1 minute.

Limit 2:
```yaml
namespace: example.org
max_value: 2
seconds: 1
conditions:
  - "req.method == 'POST'"
```
While this one would be translated to _POST_ requests to the namespace `example.org` can be authorized up to 2 times
in a span of 1 second.

Now imagine that we have the following requests happening at the same time in parallel: `POST`, `POST`, `POST`, `GET`,
`GET`; it could happen that the service authorizes the first two `POST` requests and updates both counters to 2, then the third
`POST` request is not authorized *but* the counter of limit 1 is updated to 3 (wrongly), and finally then only one `GET` request
that comes in would be authorized, leaving one wrongly denied. In this case, we would have an _overshoot_ of 1 in the
limit 1 counter, limiting the request to the service for the next 60 seconds.

#### Undershoot

This behaviour is the opposite of the _overshoot_ one, and it happens when the service authorizes a request that should
have been denied. This scenario is less likely to happen, but it's still possible. In this case, the _undershoot_ would
be the difference between the expected value of the counter and the value that the counter has in the storage, when the
stored value is **lower** than the expected one. Usually, this would happen when trying to revert a wrongly updated counter
twice in a row, for example, when trying to revert the _overshoot_ from the previous example.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Enhancing the Limitador service to process requests in parallel and being capable of operating in the previously described
modes, entails considering various implications.

Firstly, ensuring consistency in storing counter values across multiple threads is paramount to maintinging the integrity
of rate limiting operations. This involves implementing robust concurrency control mechanisms to prevent race conditions
and data corruption when accessing and updating shared counter data.

Additionally, choosing the data structure that will store the counter values must prioritize efficiency and thread safety
to handle concurrect access effectively.

Finally, considering that initially we will implement the setup of the mode at the service initialization time, balancing
the need for strict adherence to defined limits with the desire to maximize throughput presents a trade-off that we could
give the user the ability to manage.


## Implementation guidelines

### Data structures

Limitador already possess a data structure to store the limit counters that also stores the expiration time of the counter,
the type is named `AtomicExpiringCounter`. This data structure is a thread-safe counter that can be incremented and
decremented atomically, and it also has a methods to check the value at a certain time and update the counter.

The collection of these counters should be consistent across threads and also being able to quickly sort and retrieve the
counters associated to a certain namespace and limits definitions. Currently, the data structure can't provide this
much desired functionality, so we need to introduce a new data structure that can provide this.

### Request handling and concurrency control

When a request comes in, Limitador needs to determine the appropriate Counter object(s) based on the request's namespace
and limit definitions. Taking into account the collection will be in ascending order by expiration time, iterating over the
counter objects associated with those premises and performing the following steps:

1. Retrieve the Counter object from the collection.
2. Compare the current timestamp with the expiration time of the counter to determine if the request falls within the
sliding window
3. If the request falls within the window, increment the hit count of the Counter.
4. If the hit count exceeds the limit defined for the Counter, deny the request.
5. If the request passes all limit checks, allow it to proceed.

To ensure that these operations are performed atomically and consistently across threads, we need to use fine-grained
locking or atomic operations when accessing and updating the counter data in the collection. Using synchronization
primitives such as mutexes or atomic types will help to protect the integrity of the counter data from concurrent modifications.

### Configuration and mode selection

The configuration of the service will be done at the initialization time, and it will be done through the command line as
described in the [Guide-level explanation](#guide-level-explanation) section.

#### Accuracy mode

* In Accuracy mode, we need to strictly adhere to the defined limits by denying requests that exeed the limits for each
Counter Object.
* Check and update Counter values atomically to maintin consistency and prevent race conditions.
* Enforce rate limits accurately by comparing the number of hits within the sliding window to the defined limit.

#### Throughput mode

* In Throughput mode, we need to prioritize maximizing the number of requests that can be processed concurrently over
strict adherence to defined limits.
* Allowing request to proceed even if they temproarly exceed the defined limits, favouring higher request rates.
* We should use appropriate concurrency mechanisms and CAS oeprations to handle request efficiently while maintaining
consistency in the counter data. We might not need to use locks, but we need to be careful with the CAS operations nonetheless.


# Drawbacks
[drawbacks]: #drawbacks

* The implementation of a multi-threaded Limitador service will introduce additional complexity to the codebase
* The need to manage concurrency and consistency across threads will require careful design and testing to ensure
that the service operates correctly and efficiently.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- **Single-threaded**: We could choose to keep the current single-threaded implementation of Limitador, which would
  maintain the simplicity and predictability of the service, but would limit its ability to handle concurrent requests
  and process requests at a higher throughput.

# Prior art
[prior-art]: #prior-art

- [Limitador Issue #69](https://github.com/Kuadrant/limitador/issues/69) - This issue discusses "Sharable counters
  across multiple threads" and provides some context and background on the need for multi-threading support in Limitador.
- **Redis**: Redis is a popular in-memory data store that is often used to implement rate limiting and throttling
  functionality. It provides atomic operations and data structures such as counters and sorted sets that can be used
  to implement rate limiting with high throughput and accuracy.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How to balance the need for strict adherence to defined limits with the desire to maximize throughput in the
  multi-threaded implementation of Limitador?
- What are the best concurrency control mechanisms and data structures to use for storing and updating counter data
  across multiple threads?
- What would be the algorithm to use to balance the _Accuracy_ and _Throughput_ modes?

# Future possibilities
[future-possibilities]: #future-possibilities

- **Dynamic mode selection**: We could introduce a mechanism to dynamically switch between _Accuracy_ and _Throughput_
  modes based on the current load and performance characteristics of the service.
- **Fine-grained configuration**: We could provide more granular configuration options to allow users to fine-tune the
  behaviour of the multi-threaded Limitador service based on their specific requirements.
- **Performance optimizations**: We could explore various performance optimizations such as caching, pre-fetching, and
  load balancing to further improve the throughput and efficiency of the multi-threaded Limitador service.