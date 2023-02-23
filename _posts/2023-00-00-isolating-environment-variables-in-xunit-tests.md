---
layout: post
date:   2023-00-00 00:00:00 +1000
---

# Isolating Environment Variables in xUnit Tests

When testing systems that utilise environment variables at runtime, careful consideration needs to be given to the design of both the system, if governed, and the test suite in order to avoid unexpected and seemingly irreproducible runtime and assertion failures.

When defining automated and testing suites for software systems which utilise environment variables at runtime, careful consideration is needed in order to avoid unexpected and non-reputable failures.

In our scenario, we require multiple test suites &mdash; each defined within a separate class &mdash; which tests some code that utilises a common set of environment variables.

Sounds simple enough. So what's the problem? The goal of this article is to answer this question, dive a little deeper into the _why_, and equip you with the tools and knowledge necessary to tackle this issue when a faced with similar if not identical scenarios.

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

By default, xUnit creates a _test collection_ for each class within an assembly, and each of these test collections are run in parallel. This concurrency, however, is achieved with the usage of multiple [threads](https://en.wikipedia.org/wiki/Thread_(computing)), _not_ multiple [processes](https://en.wikipedia.org/wiki/Process_(computing)). If you are unaware, .NET's [`Environment.GetEnvironmentVariable`](https://learn.microsoft.com/en-us/dotnet/api/system.environment.getenvironmentvariable?view=net-6.0) will retrieve environment variables from the current _process_, hence our issue; each thread that the process creates will be accessing and modifying the same underlying environment in which our variables are stored.

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

Alas we a left with a [race condition](https://en.wikipedia.org/wiki/Race_condition) when running our tests, which both attempt to read and write the same environment variable in parallel.

So how should we go about resolving this issue?

## Solution A: Use Multiple Assemblies

One way we could solve our problem is by placing each of our test suites within a separate assembly.

While xUnit parallelises test collections within an assembly using threads, assemblies themselves are each executed under a separate process\*.

> \* This _can_ be dependant on the test runner. For more information see [Running Tests in Parallel &#124; xUnit.net](https://xunit.net/docs/running-tests-in-parallel).

This, in essence, achieves our desired process&ndash;level parallelism and associated environment isolation.

However, creating an entirely new assembly for each test suite is a lot of overhead &mdash; both performance and maintenance &mdash; as well as code duplication.

This is a band-aid fix. I wouldn't personally recommend this solution for the reasons mentioned above.

## Solution B: Serial Execution

Perhaps the most obvious solution is to simply disable parallelism altogether and run all of our tests serially, that being, one after the other.

While this does cause our above tests to pass, running all of our tests sequentially is not ideal; particularly if and when there are test collections that do not require such constraints and are perfectly capable of being run in parallel without consequence.

To alleviate the impact of this serial execution, we can instead manipulate our test collections such that only a subset of tests are run sequentially &mdash; the lesser of evils &mdash; and the performance of all other tests can be maintained and appropriately run concurrently.

Utilising the `[Collection]` xUnit Attribute, we can place all test suites that modify environment variables under a single custom test collection such that they are run sequentially. As below:

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

However, there are yet additional concerns that need to be addressed. Our test suites &mdash; now running sequentially within a single test collection &mdash; still share the same process environment and, as a result, a given test's environment may be "polluted" by the one or more tests that run before it.

For example, if we were to add the following additional assertion to the start of each of our example tests, at least one of the tests will fail (whichever test runs second):

``` csharp
Assert.IsNull(Environment.GetEnvironmentVariable("ENV_VAR"));
```

While somewhat of a non&ndash;issue for this example given the fact that each of our tests explicitly set the associated environment variable before using it, consider the scenario in which a system under test references an environment variable which it _does not_ set &mdash; a variable that may be unknowingly set by a test that runs first &mdash; and the runtime behaviour is modified. This unexpected and variable runtime behaviour is something we certainly want to avoid, and can do so by having our tests clean up after themselves.

At a minimum, this can be achieved by simply reverting any modified environment variables back to the values they contained (or did not contain) at the start of the test, as below:

``` csharp
[Fact]
void Test()
{
    var previousValue = Environment.GetEnvironmentVariable("ENV_VAR");
    Environment.SetEnvironmentVariable("ENV_VAR", "new value");

    try
    {
        // perform the test using the environment variable ...
    }
    finally
    {
        Environment.SetEnvironmentVariable("ENV_VAR", previousValue);
    }
}
```

> This will of course need to be done for any and all environment variables that are modified during the test. I recommend developing an [`IDisposable`](https://learn.microsoft.com/en-us/dotnet/api/system.idisposable?view=net-7.0) abstraction around this concept to reduce code duplication and room for error. You can find an example [here]().

In addition, if your system under test internally modifies one or more environment variables, you will also ideally need to restore these variables to their original values at the end of the test. If you don't know which environment variables will be modified, or the list of variables could change at runtime, you may need to perform a sort of "snapshot" of the environment at the start of your test such that you can appropriately restore it before the next test runs.

At this point, we have resolved the issues in running our tests and have now achieved a level of environment "isolation" between tests. However, we're still losing a lot of performance due to limiting &mdash; even partially &mdash; the ability for our tests to run concurrently.

If you are not able to modify the internals of the system under test, this may just be the best you'll get. But what if you can?

## Solution C: Dependency Inversion

If you govern the system under test and are able to modify its source code &mdash; which is more than likely the case &mdash; implementing a layer of abstraction around its environment variable access may just be the best solution overall.

The concept of decoupling high-level components from low-level implementations is known as [Dependency Inversion](https://en.wikipedia.org/wiki/Dependency_inversion_principle) (see [Microsoft's documentation](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/architectural-principles#dependency-inversion)), and there's a good reason its one of the five pillars of the broadly-known [SOLID](https://en.wikipedia.org/wiki/SOLID) design principles; When adhering to this pattern, software systems become more modular, maintainable, extensible and _testable_.

<!-- The dependency inversion principle says that software components should "Depend upon abstractions, [not] concretions". -->

Currently, our system under test is depending directly on the concrete methods provided via the static [`System.Environment`](https://learn.microsoft.com/en-us/dotnet/api/system.environment?view=net-6.0) class. We can represent this dependency with the following simple diagram:

``` mermaid
classDiagram
    direction LR
    System Under Test ..> Environment
```

<!-- A design with such little abstraction &mdash; while it can be quick to put together &mdash; lacks a level of flexibility that would be greatly benefit us in our scenario. -->

Making use of the dependency inversion principle, we can implement a layer of abstraction and refactor our design as follows:

``` mermaid
classDiagram
    direction LR

    class System Under Test
    class interface { <<interface>> }
    class implementation
    class Environment

    System Under Test ..> interface
    interface <|-- implementation
    implementation ..> Environment
```

<!-- If the system under test is refactored as to not directly depend on the static `System.Environment` methods, instead depending only on a defined interface, the implementation can be [stubbed or mocked](https://martinfowler.com/articles/mocksArentStubs.html) as we see fit during our tests. -->

If the system under test is refactored to depend only on the _interface_, the _implementation_ can be substituted as we see fit; a default implementation can be provided to preserve the existing runtime requirements, and a [stubbed or mocked](https://martinfowler.com/articles/mocksArentStubs.html) implementation can be created and used during our tests.

> Note that this will require a suitable level of support within the system under test for [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) (see [Microsoft's documentation](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection)).
>
> In our case, this could be as simple as passing the implementation via the system under test's class constructor.

Consider the following simple interface and associated default implementation which we could utilise:

``` csharp
interface IEnvironmentVariableProvider
{
    string? Get(string variable);
}
```

``` csharp
class EnvironmentVariableProvider : IEnvironmentVariableProvider
{
    string? Get(string variable)
    {
        return Environment.GetEnvironmentVariable(variable);
    }
}
```

After revising the system under test to depend on this interface, and receive an implementation via the class constructor, we can suitably provide an alternative mock implementation during our tests.

<!-- ``` csharp
class SystemUnderTest
{
    string? GetEnvironmentVariable(string variable)
    {
        return Environment.GetEnvironmentVariable(variable);
    }
}
```

``` csharp
class SystemUnderTest
{
    private readonly IEnvironmentVariableProvider _environmentVariables;

    SystemUnderTest(IEnvironmentVariableProvider environmentVariables)
    {
        _environmentVariables = environmentVariables;
    }

    string? GetEnvironmentVariable(string variable)
    {
        return _environmentVariables.Get(variable);
    }
}
``` -->

``` csharp
class TestSuiteA
{
    [Fact]
    void Should_See_Foo()
    {
        var environmentVariables = new MockEnvironmentVariableProvider(new()
        {
            new("ENV_VAR", "Foo"),
        });

        var sut = new SystemUnderTest(environmentVariables);
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
        var environmentVariables = new MockEnvironmentVariableProvider(new()
        {
            new("ENV_VAR", "Bar"),
        });

        var sut = new SystemUnderTest(environmentVariables);
        Assert.Equal("Bar", sut.GetEnvironmentVariable("ENV_VAR"));
    }
}
```

## Solution D: Implement a Shim

``` csharp
using (ShimsContext.Create())
{
    ShimEnvironment.GetEnvironmentVariableGet = (string variable) => "Foo";
}
```

## Conclusion

and a lot of these concepts are not specific to xUnit, or testing altogether, and could be used in a variety of contexts.

consider whether the utilisation of environment variables

[options pattern](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/options?view=aspnetcore-6.0)

While there are of course other solutions to this problem &mdash; including the usage of [Mutexes](https://learn.microsoft.com/en-us/dotnet/standard/threading/mutexes) or the manual creation of sub&ndash;processes to achieve isolation (consider the [Tmds.ExecFunction](https://github.com/tmds/Tmds.ExecFunction) library) &mdash; these solutions stray further from what I would consider good practice and can start to become quite convoluted.

You can find complete examples and further details of each solution above on my GitHub repository linked below.

- [jacobjmarks/xunit-environment-variable-isolation &#124; GitHub](https://github.com/jacobjmarks/xunit-environment-variable-isolation)

### References

- [Running Tests in Parallel &#124; xUnit.net](https://xunit.net/docs/running-tests-in-parallel)

## Epilogue: A Word From the Author

If you've come this far, I thank you. This is the first article I have published and while the topic of discussion is perhaps not as grandiose as I had originally imagined, I've wanted to create more analytical written content like this for a very long time, and I hope it fans the flame for more to come. Stay tuned.

These were my thoughts.

\- Jacob
