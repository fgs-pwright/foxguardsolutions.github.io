---
layout: post
title: Google Test and Google Mock
author: Phil Wright
excerpt: Google's Test and Mock frameworks for C++ are powerful tools to support clean, object-oriented C++ applications

---

[Google Test](https://github.com/google/googletest#google-test) is a popular C++ [unit testing](https://en.wikipedia.org/wiki/Unit_testing) framework developed by Google that can be used together with the closely related [mocking](https://martinfowler.com/articles/mocksArentStubs.html) extension framework, [Google Mock](https://github.com/google/googletest/blob/master/googlemock/README.md#google-mock), to test code that conforms to the [SOLID principles](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design)) for object-oriented software design. In order to follow best practices for testing while supporting a legacy application written in C++ and a new project in its early stages, our team recently invested some time into incorporating these frameworks into our test pipeline alongside similar tools already used for .NET development. This post documents lessons learned during the setup of Google Test and Google Mock for the legacy application. Work done on the legacy application was in a Windows environment using Visual Studio for development and MSVC to compile the test and mock libraries and the application.

## Setting Up the Libraries ##

The recommended way to incorporate Google Test and Mock into a project is to compile the source code from [the `googletest` repository](https://github.com/google/googletest) and to link it into the project under test. The source files needed to build these two projects are in `googletest/src` and `googlemock/src`.

Google provided Visual C++ project and solution files for building in `googletest/msvc` and `googlemock/msvc`, but the Google Test files were targeted at older versions of Visual C++ and Visual Studio. When loading these projects into the solution for our legacy application, Visual Studio performed a one-way upgrade of the project files to match the version we were using. We moved these upgraded project files into a `GoogleTest` folder in the repo housing our application and included `googletest` as a submodule within this folder, so that the only Google Test code we would need to maintain in our own source control would be these upgraded project files.

![The 'googletest' submodule and upgraded project files within our solution]({{ site.baseurl }}/images/2017-10-10-Google-Test-And-Google-Mock/Google_Test_Within_Our_Solution.PNG)

## Test Project Configuration ##

A test project should be an executable project, and it will need additional `include` directories:

```
$(SolutionDir)GoogleTest\googletest\googletest\include;$(SolutionDir)GoogleTest\googletest\googlemock\include;
```

The runtime library that is specified for the test project, the one specified for Google Test, and the one for Google Mock all need to match the runtime library specified for the system under test, or SUT.

![Configure the runtime library to match across projects]({{ site.baseurl }}/images/2017-10-10-Google-Test-And-Google-Mock/Configure_Runtime_Library.gif)

It is possible to create a test project whose SUT is an executable project, but this requires additional configuration as well as some additional attributes in SUT class declarations. Even with this additional configuration, we encountered bugs that were difficult to explain or address when running test code that wrote to managed memory, so we advise making the SUT a static library and referencing it from a separate executable project with minimal functionality, as needed.

## Google Test ##

Basic testing structure includes _tests_ and _assertions_. Tests do not have to be written as part of a test fixture, but it is good OO practice to do so.
Google Test uses macros for tests and assertions. A test in simple, declarative code uses the `TEST()` macro, and a test as part of a fixture uses the `TEST_F()` macro ([see example from the Google Test primer](https://github.com/google/googletest/blob/master/googletest/docs/Primer.md#test-fixtures-using-the-same-data-configuration-for-multiple-tests)).
A test fixture is written as a class that inherits `testing::Test`, and then each test that belongs to that fixture is expanded by the `TEST_F()` macro into a class that inherits the fixture class.
```cpp
namespace CarWash
{
    class WaxApplicator : public IWaxApplicator
    {
    public:
        explicit WaxApplicator();
        void ApplyWax() const override;
        bool WaxIsRemaining() const override;
        void BuffSurface() const override;
    }
}
```

```cpp
#include "gtest/gtest.h"

#include "WaxApplicator.h"

namespace CarWashTests
{
    class WaxApplicatorTests : public testing::Test
    {
    protected:
        WaxApplicator _waxApplicator;

        WaxApplicatorTests()
            : _waxApplicator() {}
    };

    TEST_F(WaxApplicatorTests,
        WaxIsRemaining_GivenNoWaxApplied_ReturnsFalse)
    {
        ASSERT_FALSE(_waxApplicator.WaxIsRemaining());
    }

    TEST_F(WaxApplicatorTests,
        WaxIsRemaining_GivenWaxApplied_ReturnsTrue)
    {
        _waxApplicator.ApplyWax();

        ASSERT_TRUE(_waxApplicator.WaxIsRemaining());
    }
}
```

Internally, through macro expansion, Google Test will mangle test names using underscores (eg., `WaxApplicatorTests_WaxIsRemaining_Given...`). This could create name uniqueness problems, if both test fixtures and test names included underscores. Though the documentation is not quite clear on the matter, [Google Test maintainers have stated](https://groups.google.com/forum/#!topic/googletestframework/N5A07bgEvp4) that it is acceptable to use underscores in test names, but that underscores should be avoided in test fixture names.

## Google Mock ##

A mock should inherit from an abstract class, generally, that defines a virtual destructor explicitly and defines a pure virtual function for any method that will be needed by the mock objects. This is not a strict requirement, but it avoids some complications. We followed the [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html) specifications for creating _interfaces_ (a construct, not a language feature) to define these base classes.

```cpp
namespace CarWash
{
    class ICarFacade
    {
    public:
        virtual ~ICarFacade() {}
        virtual WaxType GetPreferredWax(int) const = 0;
    };
}
```

The reason the destructor is defined explicitly is that if a mock were to be derived from a class without a virtual destructor, then the derived destructor would not be called when a mock object is deleted through a pointer to the base class.

![Memory leaks are bad]({{ site.baseurl }}/images/2017-10-10-Google-Test-And-Google-Mock/Memory_Leaks.gif)

Mock class declaration uses some strange syntax, but [the documentation gives clear instructions for how to write mock method declarations](https://github.com/google/googletest/blob/master/googlemock/docs/ForDummies.md#how-to-define-it).

```cpp
#include "gtest/gtest.h"
#include "gmock/gmock.h"

#include "ICarFacade.h"

namespace CarWashTests
{
    class MockCarFacade : public CarWash::ICarFacade
    {
    public:
        MOCK_CONST_METHOD1(GetPreferredWax, WaxType(int));
    };
}
```

A virtual class cannot be instantiated in C++, so for any dependencies on interfaces a pointer to the interface type should be passed into the constructor. It is best practice to use a smart pointer in these cases, to avoid memory management issues. This could be a `shared_ptr`, a `unique_ptr`, or some other lifetime-aware pointer type. In the production code, the constructor will be called with a pointer to an instance of the implementation class, and in the tests, the constructor will be called with a pointer to an instance of the mock class. Note that this mock instance should not be instantiated in the test code; rather, the mock object is declared as a field of the test class, and Google Mock handles creating and destroying the mock.

Call verification is done by setting an expectation, which is then verified when the mock object goes into garbage collection (recall that Google Mock handles destroying the mock—this is part of it). Note that an expectation will only be satisfied if the method call occurs *after* the expectation is set. For example, if you instantiate a class under test with a mock object dependency in the test fixture's constructor, and that class's constructor makes a call to the mock object, an `EXPECT_CALL()` for that method call will go unsatisfied. You can do setup of a mock using the `ON_CALL()` macro. Different matchers can be used in `EXPECT_CALL()`, eg., `testing::_`, `testing::Pointee()`, `testing::StrEq()`.

Working with a `shared_ptr` to a mock object can cause some complications. When a test class with a mock field is instantiated, memory for that mock is allocated on the stack. As a consequence, when the test class is destructed, the deleter of the `shared_ptr` to the mock will by default call `delete` on a stack-allocated object. This results in undefined behavior. These complications can be avoided by passing a trivial deleter to the constructor of the `shared_ptr`, as shown below.

```cpp
#include "gtest/gtest.h"

#include "WaxApplicator.h"
#include "MockCarFacade.h"

namespace CarWashTests
{
    class WaxApplicatorTests : public testing::Test
    {
    protected:
        WaxApplicator _waxApplicator;
        MockCarFacade _mockCarFacade;

        WaxApplicatorTests()
            : _waxApplicator(
                shared_ptr<ICarFacade>(&_mockCarFacade, [](auto x) {})) {}

        void Given_1990sCar()
        {
            ON_CALL(_mockCarFacade, GetPreferredWax(_))
                .WillByDefault(Return(ClearcoatWax())));
        }
    };

    TEST_F(WaxApplicatorTests,
        ApplyWax_GetsPreferredWaxFromCarFacade)
    {
        EXPECT_CALL(_mockCarFacade, GetPreferredWax(_))
            .Times(Exactly(1));

        Given_1990sCar();
        _waxApplicator.ApplyWax();
    }
}
```

## Automating Test Run for Pull Requests Using Jenkins ##

Incorporating Google Test into the continuous integration pipeline the team already had in place was fairly straightforward, since the framework is based on xUnit. The [Jenkins xUnit plugin](https://plugins.jenkins.io/xunit)'s `archiveXUnit` tool includes a utility for reading Google Test reports. Building our test project produced an executable, and producing the necessary test report required running the executable and passing it the argument `--gtest_output="xml:{outputFileName}"`.

Building a solution that includes Google Test on our Jenkins server required us to configure our source code management with the "Recursively update submodules" option, so that Google Test would be cloned recursively inside the solution that required it for reference when building the test project. Outside this change, no additional modifications were needed to our Jenkins configuration to incorporate Google Test.

![Jenkins SCM configuration]({{ site.baseurl }}/images/2017-10-10-Google-Test-And-Google-Mock/Jenkins_Job_Configuration_Recursive_Submodules.PNG)

## Conclusion ##

We found Google Test and Google Mock to be a fully-featured pair of frameworks that we could integrate into our development process and continuous integration pipeline without a great deal of effort. The frameworks integrate well with many of the tools we use on a regular basis, including Visual Studio, ReSharper's unit test runner, and Jenkins. The frameworks are well-documented with many [examples](https://github.com/google/googletest/blob/master/googletest/docs/Samples.md) and guides targeted at [beginners](https://github.com/google/googletest/blob/master/googletest/docs/Primer.md) and [advanced users](https://github.com/google/googletest/blob/master/googletest/docs/AdvancedGuide.md), and they seem to have a fairly substantial community of users. We are looking forward to using Google Test and Google Mock on future projects!
