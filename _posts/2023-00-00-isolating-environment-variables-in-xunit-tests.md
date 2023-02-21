---
layout: post
date:   2023-02-19 00:00:00 +1000
---

# Isolating Environment Variables in xUnit Tests

When defining automated testing suites for software systems which utilise environment variables, careful consideration needs to be taken in order to avoid unexpected collisions when manipulating and using these variables while preserving test parallelism.

In our scenario, we require multiple test suites &mdash; each defined within a separate class &mdash; which tests some code that utilises a common set of environment variables.

Sounds simple enough. So what's the problem? The goal of this article is to answer this question, dive a little deeper into the why, and equip you with the tools and knowledge necessary to tackle this issue when a faced with similar if not identical scenarios.

## The Problem

Consider the following two xUnit test suites, in which each contains a single test that sets a common environment variable and &mdash; for demonstration purposes &mdash; asserts that our [system under test](https://en.wikipedia.org/wiki/System_under_test) has visibility of that set value:

``` csharp
class TestSuiteA
{
    [Fact]
    void Should_See_Foo()
    {
        Environment.SetEnvironmentVariable("ENV_VAR", "Foo");
        var sut = new SystemUnderTest();
        Assert.Equal("Foo", sut.GetEnvironmentVariable("ENV_VAR"));
    }
}
```

``` csharp
class TestSuiteB
{
    [Fact]
    void Should_See_Bar()
    {
        Environment.SetEnvironmentVariable("ENV_VAR", "Bar");
        var sut = new SystemUnderTest();
        Assert.Equal("Bar", sut.GetEnvironmentVariable("ENV_VAR"));
    }
}
```

While at first glance these tests may seem harmless and quite functional, at least one of these tests will fail due to the way xUnit parallelises its test collections.

By default, xUnit creates a _test collection_ for each class within an assembly, and each of these test collections are run in parallel. This concurrency, however, is achieved with the usage of multiple [threads](https://en.wikipedia.org/wiki/Thread_(computing)), _not_ multiple [processes](https://en.wikipedia.org/wiki/Process_(computing)). If you are unaware, .NET's [`Environment.GetEnvironmentVariable`](https://learn.microsoft.com/en-us/dotnet/api/system.environment.getenvironmentvariable?view=net-7.0) will retrieve environment variables from the current _process_, hence our issue; each thread that the process creates will be accessing and modifying the same underlying environment in which our variables are stored.

To visualise our conundrum, see below a representation of scope between the Environment which holds our variables, the process which runs our test assembly, and the threads which run our test collections.

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

Alas we a left with a [race condition](https://en.wikipedia.org/wiki/Race_condition) when running our tests &mdash; which both attempt to read and write the same environment variable &mdash; in parallel.

## Solution A: Use Multiple Assemblies

One way we could solve our problem is by placing each of our test suites within a separate assembly.

While xUnit parallelises test collections within an assembly using threads, assemblies themselves are each executed under a separate process\*.

> \* This _can_ be dependant on the test runner. For more information please see [Running Tests in Parallel &#124; xUnit.net](https://xunit.net/docs/running-tests-in-parallel).

This, in essence, achieves our desired process&ndash;level parallelism and associated environment isolation.

However, creating an entirely new assembly for each test suite is a lot of overhead &mdash; both performance and maintenance &mdash; as well as code duplication.

I wouldn't personally recommend this solution to our problem.

## Solution B: Serial Execution

Perhaps the most obvious solution is to simply disable parallelism altogether and run all of our tests serially, that being, one after the other.

While this does cause our above tests to pass, running all of our tests sequentially is not ideal; particularly if and when there are test collections that do not require such constraints and are perfectly capable of being run in parallel without consequence.

To alleviate the impact of this serial execution, we can instead manipulate our test collections such that only a subset of tests are run sequentially &mdash; the lesser of evils &mdash; and the performance of all other tests can be maintained and appropriately run concurrently.

Utilising the `[Collection]` xUnit Attribute, we can place all test suites that modify environment variables under a single test collection such that they are run sequentially. As below:

``` csharp
[Collection("My Custom Collection")]
class TestSuiteA
{
    // ...
}
```

``` csharp
[Collection("My Custom Collection")]
class TestSuiteB
{
    // ...
}
```

> Note that the usage of hardcoded string literals are for demonstration purposes only, and I would highly recommend externalising your constants or otherwise providing a consistent static reference to be used as your collection identifiers.

<!-- > Note that I am using `nameof(Environment)` (i.e. `"Environment"`) as a convenient way to describe the test collection, elude to its purpose, and avoid hardcoding string literals.
>
> You don't need to follow suite; as long as each attribute contains the same constant string value, they will be placed within the same test collection. -->

However, there are yet additional concerns that need to be addressed with this solution. Our test suites &mdash; now running sequentially within a single test collection &mdash; still share the same process environment and, as a result, a given test's environment may be polluted by the one or more tests that run before it.

For example, if we were to add the following additional assertion to the start of each of our example tests, at least one of the tests will fail (whichever test runs second):

``` csharp
Assert.IsNull(Environment.GetEnvironmentVariable("ENV_VAR"));
```

While somewhat of a non&ndash;issue for this example given the fact that each of our tests explicitly set the associated environment variable before using it, consider the scenario in which a system under test references an environment variable which it _does not_ set &mdash; a variable that may be unknowingly set by a test that runs first &mdash; and the runtime behaviour is modified. This unexpected and variable runtime behaviour is something we certainly want to avoid, and can do so by having our tests clean up after themselves.

At a minimum, this can be achieved by simply reverting any modified environment variables back to the values they contained (or did not contain) before modification, as below:

``` csharp
[Fact]
void Test()
{
    var previousValue = Environment.GetEnvironmentVariable("ENV_VAR");
    Environment.SetEnvironmentVariable("ENV_VAR", "new value");

    // ...

    Environment.SetEnvironmentVariable("ENV_VAR", previousValue);
}
```

> This will of course need to be done for any and all environment variables that are modified during the test. I recommend developing an abstraction around this concept to reduce code duplication and room for error. <!-- For example, a disposable `TemporaryEnvironmentVariable/s` class. You can find an example of such in my GitHub repository linked at the end of this article. -->

In addition, if your system under test internally modifies one or more environment variables, you will also ideally need to restore these variables to their original values at the end of the test. If you don't know which environment variables will be modified, or the list of variables could change at runtime, you may need to perform a sort of "snapshot" of the environment before your test such that you can appropriately restore it before the next test.

## Solution C: Dependency Inversion

If do, however, govern the system under test and are able to modify its source code &mdash; which is more likely the case &mdash; implementing a layer of abstraction around its environment variable access might be the best solution.

note also however that this will require suitable support within the system under test for [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection), such that any mock implementations that are created for testing purposes can be suitably provided to and utilised by the system.

## Solution D: Implement a Shim

## Conclusion

While there are of course other solutions to this problem &mdash; including the usage of [Mutexes](https://learn.microsoft.com/en-us/dotnet/standard/threading/mutexes) or the manual creation of sub&ndash;processes to achieve isolation (consider the [Tmds.ExecFunction](https://github.com/tmds/Tmds.ExecFunction) library) &mdash; these solutions stray further from what I would consider good practice and can start to become quite convoluted.

You can find complete examples and further details of each solution above on my GitHub repository linked below.

- [jacobjmarks/xunit-environment-variable-isolation &#124; GitHub](https://github.com/jacobjmarks/xunit-environment-variable-isolation)

### References

- [Running Tests in Parallel &#124; xUnit.net](https://xunit.net/docs/running-tests-in-parallel)

## Epilogue: A Word From the Author

If you've come this far, I thank you. This is the first article I have published and while the topic of discussion is perhaps not as grandiose as I had originally imagined, I've wanted to create more analytical written content like this for a very long time, and I hope it fans the flame for more to come. Stay tuned.

These were my thoughts.

\- Jacob
