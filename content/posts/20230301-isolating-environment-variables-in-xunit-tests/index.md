---
layout: post
title: Isolating Environment Variables in xUnit Tests
date: 2023-03-01T00:00:00.000+1000
tags: [.NET, Testing]
cover:
  image: "cover.png"
  relative: true
  hidden: true
---

When testing systems that utilise environment variables at runtime, careful consideration needs to be given to the design of both the system, if governed, and the test suite to avoid unexpected and seemingly irreproducible runtime and assertion failures when these variables are used in parallel.

<!--more-->

In our scenario, we require multiple test suites &mdash; each defined within a separate class &mdash; which tests some code that utilises a common set of environment variables.

Sounds simple enough. So what's the problem? The goal of this article is to answer this question, dive a little deeper into the _why_, and equip you with the tools and knowledge necessary to tackle this issue when faced with similar if not identical situations.

## The Problem

Consider the following two xUnit test suites. Each contains a single test that sets a common environment variable and &mdash; for demonstration purposes &mdash; asserts that our [system under test](https://en.wikipedia.org/wiki/System_under_test) has visibility of that set value:

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

By default, xUnit creates a _test collection_ for each class within an assembly, and each of these test collections are run in parallel. However, this concurrency is achieved with the usage of multiple [threads](https://en.wikipedia.org/wiki/Thread_(computing)), _not_ multiple [processes](https://en.wikipedia.org/wiki/Process_(computing)). If you are unaware, .NET's [`Environment.GetEnvironmentVariable`](https://learn.microsoft.com/en-us/dotnet/api/system.environment.getenvironmentvariable?view=net-6.0) will retrieve environment variables from the current _process_, hence our issue; each thread that the process creates will be accessing and modifying the same underlying environment in which our variables are stored.

To visualise our conundrum, see below a representation of scope between the Environment which holds our variables, the process which runs our test assembly, and the threads which run our test collections.

{{< figure src="env-scope.drawio.svg" alt="Environment scope" align="center" width="435em" >}}

As can be seen, irrespective of the thread used, the same Environment will be utilised. Alas, we are left with a [race condition](https://en.wikipedia.org/wiki/Race_condition) when running our tests, which both attempt to read and write the same environment variable in parallel.

So how should we go about resolving this issue?

## Solution A: Use Multiple Assemblies

One way we could solve our problem is by placing each of our test suites within a separate assembly.

While xUnit parallelises test collections within an assembly using threads, assemblies themselves are each executed under a separate process\*.

> \* This _can_ be dependent on the test runner. For more information see [Running Tests in Parallel &#124; xUnit.net](https://xunit.net/docs/running-tests-in-parallel).

This, in essence, achieves our desired process&ndash;level parallelism and associated environment isolation.

However, creating an entirely new assembly for each test suite is a lot of overhead &mdash; both performance and maintenance &mdash; as well as code duplication.

This is a band-aid fix. I wouldn't personally recommend this solution.

## Solution B: Serial Execution

Perhaps the most obvious solution is to simply disable parallelism altogether and run all of our tests serially, that is, one after the other.

While this does cause our above tests to pass, running all of our tests sequentially is not ideal; particularly if and when there are test collections that do not require such constraints and are perfectly capable of being run in parallel without consequence.

To alleviate the impact of serial execution, we can instead manipulate our test collections such that only a subset of tests are run sequentially &mdash; the lesser of evils &mdash; and the performance of all other tests can be maintained and appropriately run concurrently.

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

> Note that the use of hardcoded string literals is for demonstration purposes only, and I would highly recommend externalising your constants or otherwise providing a consistent static reference to be used as your collection identifiers.

However, there are yet additional concerns that need to be addressed. Our test suites &mdash; now running sequentially within a single test collection &mdash; still share the same process environment and, as a result, a given test's environment may be "polluted" by the one or more tests that run before it.

For example, if we were to add the following additional assertion to the start of each of our example tests, at least one of the tests will fail (whichever test runs second):

``` csharp
Assert.IsNull(Environment.GetEnvironmentVariable("ENV_VAR"));
```

While somewhat of a non&ndash;issue for this example, given the fact that each of our tests explicitly set the associated environment variable before using it, consider the scenario in which a system under test references an environment variable which it _does not_ set &mdash; a variable that may be unknowingly set by a test that runs first &mdash; and the runtime behaviour is modified. This unexpected and variable runtime behaviour is something we certainly want to avoid and can do so by having our tests clean up after themselves.

At a minimum, this can be achieved by simply reverting any modified environment variables to the values they contained (or did not contain) at the start of the test, as below:

``` csharp
[Fact]
void Test()
{
    var previousValue = Environment.GetEnvironmentVariable("ENV_VAR");
    Environment.SetEnvironmentVariable("ENV_VAR", "Baz");

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

> This will of course need to be done for _all_ environment variables that are modified during the test. I recommend developing an [`IDisposable`](https://learn.microsoft.com/en-us/dotnet/api/system.idisposable?view=net-7.0) abstraction around this concept to reduce code duplication and room for error. You can find an example of one [here](https://github.com/jacobjmarks/xunit-environment-variable-isolation/blob/main/Examples.SerialExecution/Support/TemporaryEnvironmentVariable.cs).

<!-- In addition, if your system under test internally modifies one or more environment variables, you will also ideally need to restore these variables to their original values at the end of the test. If you don't know which environment variables will be modified, or the list of variables could change at runtime, you may need to perform a sort of "snapshot" of the environment at the start of your test such that you can appropriately restore it before the next test runs. -->

With this additional cleanup in place, we have now not only resolved the issues in running our tests but have also achieved a level of environment "isolation" between them. However, we're still losing a lot of performance due to limiting &mdash; even partially &mdash; the ability for our tests to run concurrently.

<!-- If you are not able to modify the internals of the system under test, this may just be the best you'll get. -->

## Solution C: Dependency Inversion

If you govern the system under test and can modify its source code &mdash; which is more than likely the case &mdash; implementing a layer of abstraction around its environment variable access may just be the best solution overall.

The concept of decoupling high-level components from low-level implementations is known as [Dependency Inversion](https://en.wikipedia.org/wiki/Dependency_inversion_principle) (see [Microsoft's documentation](https://learn.microsoft.com/en-us/dotnet/architecture/modern-web-apps-azure/architectural-principles#dependency-inversion)), and there's a good reason it's one of the five pillars of the broadly-known [SOLID](https://en.wikipedia.org/wiki/SOLID) design principles; when adhering to this pattern, software systems become more modular, maintainable, extensible and _testable_.

Currently, our system under test is depending directly on the concrete methods provided via the static [`System.Environment`](https://learn.microsoft.com/en-us/dotnet/api/system.environment?view=net-6.0) class. We can represent this dependency with the following simple diagram:

{{< figure src="uml-a.drawio.svg" alt="UML Diagram A" align="center" width="165em" >}}

Making use of the dependency inversion principle, we can implement a layer of abstraction and refactor our design as follows:

{{< figure src="uml-b.drawio.svg" alt="UML Diagram B" align="center" width="465em" >}}

If the system under test is refactored to depend only on the _interface_, the _implementation_ can be substituted as we see fit. A default implementation can be provided to preserve the existing runtime requirements, and a stub or mock implementation can be created and used during our tests.

> Note that this will require a suitable level of support within the system under test for [dependency injection](https://en.wikipedia.org/wiki/Dependency_injection) (see [Microsoft's documentation](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection)). In our case, this could be as simple as passing the implementation via the system under test's class constructor.

Consider the following simple interface which we could utilise:

``` csharp
interface IEnvironmentVariableProvider
{
    string? Get(string variable);
}
```

While some additional code excerpts have been omitted here for brevity (I'm going to assume you have a basic understanding of interfaces, implementations, and dependency injection), after refactoring our system under test, we can update our tests to make use of a stub in-memory environment variable provider &mdash; which implements our new interface &mdash; as below:

> Full implementation details can be found within this article's associated GitHub repository [here](https://github.com/jacobjmarks/xunit-environment-variable-isolation).

``` csharp
class TestSuiteA
{
    [Fact]
    void Should_See_Foo()
    {
        IEnvironmentVariableProvider environmentVariables = new InMemoryEnvironmentVariableProvider(new()
        {
            new("ENV_VAR", "Foo"),
        });

        // provide the stub via constructor-level dependency injection
        var sut = new SystemUnderTest(environmentVariables);
        Assert.Equal("Foo", sut.GetEnvironmentVariable("ENV_VAR"));
    }
}
```

And likewise for our `TestSuiteB`, replacing `"Foo"` with `"Bar"`.

With this stub in place, our tests are now _fully_ isolated. There are no dependencies on concrete external components and our tests are capable of being run with full parallelism without any risk of environment variable collision or pollution.

> Note however that we are now utilising the system under test in a way that is discrepant from how it would be used during normal operation. As such, we need to ensure we create suitable tests for the default implementation of our new interface, and could even go so far as to assert that this implementation is utilised when initialising the system for regular use.

I believe dependency inversion to be the best solution to our problem. Implementing an additional layer of abstraction provides us with immense flexibility, not only when it comes to writing automated tests.

## Honorable Mention: Shims

There exists one more solution approach worth mentioning, that being the utilisation of [shims](https://en.wikipedia.org/wiki/Shim_(computing)).

A shim is essentially a transparent middleware that can be implicitly injected into the context of a running application, such that it can be utilised in place of another dependency.

Consider the layer of abstraction that we introduced in the previous solution, a shim could be used instead of these additional components and no modifications would need to be introduced to the system under test; we can directly override the static `System.Environment` methods.

Sounds great, right?

Unfortunately, standard testing frameworks (including xUnit) do not support shims and you may find it difficult to find libraries that do; [Pose](https://github.com/tonerdo/pose) was once promising however it has not been updated in some time and does not support the latest versions of .NET.

Currently, the most prominent way to use shims in .NET is via the [Microsoft Fakes Framework](https://learn.microsoft.com/en-us/visualstudio/test/code-generation-compilation-and-naming-conventions-in-microsoft-fakes?view=vs-2022) &mdash; however, it requires Visual Studio Enterprise and only supports MSTest testing projects.

While shims can be a powerful method of test isolation &mdash; especially if you cannot modify the system under test &mdash; I could not find a viable method of utilising them for our scenario. Additionally, their necessity can generally be avoided through the use of dependency inversion.

## Conclusion

While there are of course other solutions to this problem &mdash; including the usage of [Mutexes](https://learn.microsoft.com/en-us/dotnet/standard/threading/mutexes) or the manual creation of sub&ndash;processes to achieve isolation (consider the [Tmds.ExecFunction](https://github.com/tmds/Tmds.ExecFunction) library) &mdash; these solutions stray further from what I would consider good practice and can start to become quite convoluted.

I would also strongly suggest considering for your use case whether it would make sense to move away from the direct consumption of environment variables throughout your system under test, instead opting for a more traditional [.NET Configuration](https://learn.microsoft.com/en-us/dotnet/core/extensions/configuration) and [Options pattern](https://learn.microsoft.com/en-us/dotnet/core/extensions/options) approach; a robust, well-documented and common solution which lends itself to the previously discussed dependency inversion principle, among others.

You can find complete examples and implementation details of solutions B and C within the GitHub repository linked below:

- [jacobjmarks/xunit-environment-variable-isolation &#124; GitHub](https://github.com/jacobjmarks/xunit-environment-variable-isolation)

## Epilogue: A Word From the Author

If you've made it this far, I thank you. This is the first blog post/article I have published (since my [university days](https://github.com/jacobjmarks/cvtree-parallel/blob/master/assets/9188100%20Report.pdf)) and while the topic of discussion is perhaps not as grandiose as I had originally imagined, I've wanted to create more analytical written content like this for a very long time, and I hope it fans the flame for more to come. Stay tuned.

\- Jacob

These were my thoughts.
