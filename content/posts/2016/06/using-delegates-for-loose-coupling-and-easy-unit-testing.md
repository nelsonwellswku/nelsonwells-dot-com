+++
title = 'Using delegates for loose coupling and easy unit testing'
date = 2016-06-14T15:00:00-05:00
tags = ['programming', 'csharp', 'automapper', 'unit testing']
draft = false
+++
In a typical ASP.NET MVC or WebApi project, controller actions often do nothing more than make a service call, map the returned domain object to a view model, and either render a view with the model or simply return it for the framework to serialize as JSON. Using [AutoMapper](http://automapper.org/), a simplified WebApi controller action may look something like

    public EmployeeViewModel GetEmployee(string employeeName)
    {
        var employee = _employeeService.GetByName(employeeName);
        var viewModel = Mapper.Map<EmployeeViewModel>(employee);
        return viewModel;
    }

This is a pretty common but difficult to test pattern because of the static Mapper dependency. You can't actually unit test the controller action without having a properly (or at least minimally) configured static `Mapper`. New versions of AutoMapper provide different ways to solve this problem, like the injectable `IMapper` interface, but I want to demonstrate another method to reduce coupling and ease unit testing.

In this entry, I'll demonstrate how to get rid of the static `Mapper` dependency without using the IMapper interface, which exposes several overloads to map one data type to another, or creating your own interface that serves a similar purpose. Instead, we'll inject a `delegate` whose signature is very expressive.

    public delegate TDest MapperFunc<in TSrc, out TDest>(TSrc source);

<!--more-->
## Writing the controller class

Now that we've defined our mapper delegate, we can inject it into a controller class as a constructor parameter and tweak our `GetEmployee` method. Here's a snippet that demonstrates the change.

    public EmployeeController(
        EmployeeService employeeService, 
        MapperFunc<Employee, EmployeeViewModel> mapEmployee)
    {
        // initialize private fields
        _employeeService = employeeService;
        _mapEmployee = mapEmployee;
    }

    public EmployeeViewModel GetEmployee(string employeeName)
    {
        var employee = _employeeService.GetByName(employeeName);
        var viewModel = _mapEmployee(employee);
        return viewModel;
    }

Now, how does this change our unit test? Previously, we'd have to assert on the values of the returned view model, but that's because we were mapping from the entity type to the view model with a static Mapper class. That's not ideal because that means we're not just testing the controller method logic, but we're also testing if the static Mapper is properly defined. Instead, we should be testing the controller method independently from our entity-to-view-model mapping code.

Now that we've removed the dependency on the static Mapper by injecting and utilizing the `MapperFunc<,>`, our unit test logic changes slightly. What we _really_ want to test is that the controller method

1. calls the employee service to retrieve an entity
2. calls the mapper func on that particular entity instance and
3. returns the particular view model that itself was created by the mapper func

For the controller method unit test, we don't really care about what the mapper func does. The mapper func that we use in production can (and should!) be unit tested separately now.

## Writing the unit test

All we need to do is set up a mock for the employee service to return an instance of an `Employee`. Then, the mapper func we inject just needs to check if the instance passed in is the same instance that the service returned. If it is, then we can return a view model instance or null and assert on the return value. Here's an example

    class EmployeeControllerTests
    {
        private Mock<IEmployeeService> _employeeServiceMock;
        private Employee _returnedEmployee;
        private EmployeeViewModel _returnedEmployeeViewModel;
        private MapperFunc<Employee, EmployeeViewModel> _mapEmployee;

        [SetUp]
        public void SetUp()
        {
            _returnedEmployee = new Employee();
            _returnedEmployeeViewModel = new EmployeeViewModel();
            _mapEmployee = emp => emp == _returnedEmployee 
                ? _returnedEmployeeViewModel 
                : null;

            _employeeServiceMock = new Mock<IEmployeeService>();
            _employeeServiceMock
                .Setup(x => x.GetEmployee(It.Is<int>(i => i == 1)))
                .Returns(_returnedEmployee);
         }

         [Test]
         public void GetEmployee_HappyPath()
         {
            // Arrange
            var controller = new EmployeeController(
                    _employeeServiceMock.Object, 
                    _mapEmployee);

            // Act
            var viewModel = controller.GetEmployee(1);

            // Assert
            Assert.NotNull(viewModel);
         }

        // more tests
    }

## When should this technique be used?

A colleague introduced me to this technique, and it worked very well for the product we worked on together for this exact use case, mapping from an entity to a view model and in some other places in the application as well. _Almost_ the same result could have achieved by using an IMapper interface that exposes the one method that matches the signature of the MapperFunc delegate. However, the code to set up the IMapper mock is longer and, once you've gotten your head around this technique, is much more difficult to understand at a glance. And, after all, what _is_ an interface that exposes only one method with no side effects if _not_ a delegate?

Any time you have an interface that exposes one method and only has one concrete class, you have an opportunity to use this technique. If the concrete implementation takes no dependencies itself, it behooves you to scrap the class completely and replace it with a delegate.

## Final Thoughts

You may recognize the MapperFunc delegate from another BCL type. Specifically, this one

    public delegate TResult Func<in T, out TResult>(T arg);

and you're probably wondering why not inject `Func<Employee, EmployeeViewModel>` into the `EmployeeController` instead of creating a new delegate type. Well, if you want, you can use `Func<,>` instead of `MapperFunc<,>`. Functionally, they're the same. However, in an upcoming blog post, I am going to demonstrate how to use Autofac to inject the delegate and resolve the delegate's type parameters at run-time. Using a specific delegate type instead of the generic `Func<,>` makes the injection code much easier to write. You'll see what I mean a little :)

Interestingly, after I started writing this entry but before I finished it, I came across a blog post by Steve Shogran about [modern dependency injection](http://deliberate-software.com/modern-di/). His post touches on some of the things I discussed in this post, and it also talks about some points that were outside of the scope of this article. It's worth a read.

And finally, the source code for this article is [available on Github](https://github.com/nelsonwellswku/blog-stuff/tree/master/using-delegates-for-loose-coupling-and-easy-unit-testing/DelegateExample). Feel free to check it out, run the unit tests, step through the code and whatever else you want to do.

Happy coding!
