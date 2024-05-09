+++
title = 'Autofixture Do() not working? You forgot Without()'
date = 2016-07-01T15:00:00-05:00
tags = ['programming', 'csharp', 'unit testing']
draft = false
+++
I use [Autofixture](https://github.com/AutoFixture/AutoFixture) for generating test data in my unit tests. AutoFixture has served me well, but the documentation doesn't give great examples of how to use it. The "cheat sheet" has links to the authors blogs, and if you're not careful you won't find everything you need. It's sometimes easy to miss those links.

The other day, I was trying to generate a POCO that had a string property, and I wanted that string property to be constrained to a list of valid values. Since I obviously didn't want to customize the string type across my entire fixture, I instead customized the parent type with the Do method. I wanted to pass in an action that would randomly select from the list pre-defined values and then assign it to the string property in my class instance. The code was straight-forward, but the string was still being populated with the default AutoFixture value.

<!--more-->
## The First Attempt
    var _names = new HashSet<string>;
    {
        "Peter",
        "Paul",
        "Mary",
        "John"
    };

    var fixture = new Fixture();
    fixture.Customize<Employee>(ob => ob
        .Do(e => 
            employee.Name = _names
                .OrderBy(x => Guid.NewGuid())
                .First();
        )
    );

    var employee = fixture.Create<Employee>();

    // employee.Name is still Name8fd70865-0d1e-4dfd-9fde-956028e6be7a

After generating an instance of my class, the Name property was still being generated with the default value for strings, {propertyName} + {guid}. My assignment was being hit in the debugger, it didn't seem to have any effect. The problem was that I assumed incorrectly that the Do() method would be ran at the end of the the object generation pipeline. Instead, fixture customizations happen in the order specified by the developer, and _then_ the default rules are ran. The value assigned in the Do() method was being overwritten by the default rules.

## The fix
The fix is simple. Be sure to tell AutoFixture to ignore properties that you're going to set manually with Do(). You can use the Without() method.

    var fixture = new Fixture();
    fixture.Customize<Employee>(ob => ob
        .Without(e => e.Name)
        .Do(e => 
            employee.Name = _names
                .OrderBy(x => Guid.NewGuid())
                .First();
           )
        );

This will prevent AutoFixture from generating Name at all and leave you responsible for doing it.

As always, a project is available on [Github](https://github.com/nelsonwellswku/blog-stuff/tree/master/autofixture-do-not-working-you-forgot-without) so that you can see this code in action.
