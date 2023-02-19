---
layout: post
date:   2023-02-19 00:00:00 +1000
---

# Isolating Environment Variables in xUnit Tests

When defining automated testing suites for software systems which utilise environment variables, careful consideration needs to be taken in order to avoid unexpected collisions when manipulating and using these variables while preserving test parallelism.

In our scenario, we require multiple test suites &mdash; each defined within a separate class &mdash; which tests some code that utilises a common set of environment variables.

Sounds simple enough. So what's the problem?

## The Problematic Tests

Consider the following two xUnit test suites, in which each contains a single test that sets a common environment variable and &mdash; for demonstration purposes &mdash; asserts that our [system under test](https://en.wikipedia.org/wiki/System_under_test) has visibility of that set value:

<!-- ``` csharp
class SystemUnderTest
{
    string? GetEnvironmentVariable(string variable)
    {
        return Environment.GetEnvironmentVariable(variable);
    }
}
``` -->

``` csharp
class TestSuiteA
{
    [Fact]
    void Should_See_Foo()
    {
        Environment.SetEnvironmentVariable("ENVIRONMENT_VARIABLE", "Foo");
        var sut = new SystemUnderTest();
        Assert.Equal("Foo", sut.GetEnvironmentVariable("ENVIRONMENT_VARIABLE"));
    }
}
```

``` csharp
class TestSuiteB
{
    [Fact]
    void Should_See_Bar()
    {
        Environment.SetEnvironmentVariable("ENVIRONMENT_VARIABLE", "Bar");
        var sut = new SystemUnderTest();
        Assert.Equal("Bar", sut.GetEnvironmentVariable("ENVIRONMENT_VARIABLE"));
    }
}
```

While at first glance these tests may seem harmless and quite functional, at least one of these tests will fail due to the way xUnit parallelises its test collections.

By default, xUnit creates a _test collection_ for each class within an assembly, and each of these test collections are run in parallel. This concurrency, however, is achieved with the usage of multiple [threads](https://en.wikipedia.org/wiki/Thread_(computing)), _not_ multiple [processes](https://en.wikipedia.org/wiki/Process_(computing)). If you are unaware, [`Environment.GetEnvironmentVariable`](https://learn.microsoft.com/en-us/dotnet/api/system.environment.getenvironmentvariable?view=net-7.0) will retrieve environment variables from the current _process_, hence our issue; each thread that the process creates will be accessing and modifying the same underlying environment in which our variables are stored.

Alas a [race condition](https://en.wikipedia.org/wiki/Race_condition) is created when running our tests &mdash; which both attempt to modify and read the same environment variable &mdash; in parallel.

<!-- > \* When running multiple test assemblies &mdash; regardless of parallelism configuration &mdash; each assembly runs on a separate Process. As such, environment collisions between assemblies should be a non&ndash;issue. -->

To visualise our conundrum, see below a representation of scope between the Environment which holds our variables, the process which runs our test assembly, and the Threads which run our test collections.

``` mermaid
flowchart TD
    subgraph "Environment<sup>n</sup>"
    subgraph "Process<sup>n</sup>"
    E2P3T1("Thread<sup>1</sup>")
    E2P3T2("Thread<sup>2</sup>")
    E2P3TN("Thread<sup>n</sup>")
    end
    end
    subgraph "Environment<sup>2</sup>"
    subgraph "Process<sup>2</sup>"
    E2P2T1("Thread<sup>1</sup>")
    E2P2T2("Thread<sup>2</sup>")
    E2P2TN("Thread<sup>n</sup>")
    end
    end
    subgraph "Environment<sup>1</sup>"
    subgraph "Process<sup>1</sup>"
    E1P1T1("Thread<sup>1</sup>")
    E1P1T2("Thread<sup>2</sup>")
    E1P1TN("Thread<sup>n</sup>")
    end
    end
```

As can be seen, unless our test collections are run using separate processes,

<!-- ``` mermaid
erDiagram
    Environment ||--|| Process : "attached to"
    Process ||--|{ Thread : "creates"
``` -->

## Solution A: Assembly per Test Suite

One way we could solve our problem is by placing each of our test suites within a separate assembly.

While xUnit's _Test Framework_ parallelises test collections with threads, an xUnit test _Runner_ will run each assembly under a separate process ().

This, in essence, achieves our desired process&ndash;level parallelism and associated environment isolation.

## Solution A: Serial Execution (or, Avoiding the Collision)

Perhaps the most obvious solution is to simply disable parallelism and run all of our tests serially, that being, one after the other.

While this does cause our above tests to pass, running all of our tests sequentially is not ideal; particularly when there are test collections that do not require such constraints and are perfectly capable of being run in parallel without consequence.

To alleviate the impact of this serial execution, we can instead manipulate our test collections such that only a subset of tests are run sequentially, and the performance of all other tests can be maintained and appropriately run in parallel.

Utilising the `[Collection]` xUnit Attribute, we can place all test suites that modify environment variables under a single test collection such that they are run sequentially; as below:

``` csharp
[Collection(nameof(Environment))]
class TestSuiteA
{
    // ...
}
```

``` csharp
[Collection(nameof(Environment))]
class TestSuiteB
{
    // ...
}
```

However, there are yet additional concerns that need to be addressed with this solution. Our test suites &mdash; now running sequentially within a single test collection &mdash; still share the same process environment and, as a result, a given test's environment may be polluted by the one or more tests that run before it.

For example, if we were to add the following additional assertion to the start of each of our example tests, at least one of the tests will fail (whichever test runs second):

``` csharp
Assert.IsNull(Environment.GetEnvironmentVariable("ENVIRONMENT_VARIABLE"));
```

While somewhat of a non&ndash;issue for this example given the fact that each of our tests explicitly set the associated environment variable before using it, consider the scenario in which a system under test references an environment variable which it _does not_ set &mdash; a variable that may be unknowingly set by a test that runs first &mdash; and the runtime behaviour is modified. This unexpected and variable runtime behaviour is something we certainly want to avoid, and can do so by having our tests clean up after themselves.

At a minimum, this can be achieved by simply reverting any modified environment variables back to the values they contained (or did not contain) before modification, as below:

``` csharp
[Fact]
void Test()
{
    var previousValue = Environment.GetEnvironmentVariable("ENVIRONMENT_VARIABLE");
    Environment.SetEnvironmentVariable("ENVIRONMENT_VARIABLE", "new value");

    // perform the test using our environment variable ...

    Environment.SetEnvironmentVariable("ENVIRONMENT_VARIABLE", previousValue);
}
```

> This will of course need to be done for any and all environment variables that are modified during the test. I recommend developing an abstraction around this concept to reduce code duplication and room for error. For example, a disposable `TemporaryEnvironmentVariable/s` class. You can find an example of such in my GitHub repository linked at the end of this article.

In addition, if your system under test internally modifies one or more environment variables, you will also ideally need to restore these variables to their original values at the end of the test. If you don't know which environment variables will be modified, or the list of variables could change at runtime, you may need to perform a sort of "snapshot" of the environment before your test such that you can appropriately restore it before the next test.

## Solution B: Inversion of Control

## Solution C: Implement a Shim

## Conclusion

While there are of course other solutions to this problem &mdash; including the usage of [Mutexes](https://learn.microsoft.com/en-us/dotnet/standard/threading/mutexes) or the manual creation of sub&ndash;processes to achieve isolation (consider the [Tmds.ExecFunction](https://github.com/tmds/Tmds.ExecFunction) library) &mdash; these solutions stray further from what I would consider good practice and can start to become quite convoluted.

You can find complete examples and further details of each solution above on my GitHub repository linked below.

- [jacobjmarks/xunit-environment-variable-isolation \| GitHub](https://github.com/jacobjmarks/xunit-environment-variable-isolation)

### Further Reading

- [Running Tests in Parallel \| xUnit.net](https://xunit.net/docs/running-tests-in-parallel)
