+++
title = 'Python groupby has surprising behavior'
date = 2026-09-13T15:00:00-05:00
tags = ['programming', 'python', 'csharp', 'javascript']
draft = true
+++
Python's [groupby from itertools](https://docs.python.org/3/library/itertools.html#itertools.groupby) has surprising behavior if you compare it to other languages' `group by` implementations.

Consider this Python snippet that groups objects by a key and then prints each key with its group.

    friends = [
        {"couple_id": 3, "name": "John"},
        {"couple_id": 1, "name": "Thomas"},
        {"couple_id": 2, "name": "Eugene"},
        {"couple_id": 2, "name": "Beatrice"},
        {"couple_id": 1, "name": "Imani"},
        {"couple_id": 3, "name": "Mary"}
    ]

    friends_groups = groupby(friends, lambda x: x["couple_id"])

    for key, grouping in friends_groups:
        print(key, " --> ", " loves ".join([obj["name"] for obj in grouping]))

<!--more-->
## The problem
If you are used to other languages' `group by` implementations, this output may be surprising.

    $ uv run main.py
    3 --> John
    1 --> Thomas
    2 --> Eugene loves Beatrice
    1 --> Imani
    3 --> Mary

Despite grouping by the `couple_id` property in our list of dictionaries, John and Mary (`couple_id` of 3) and Thomas and Imani (`couple_id` of 1) were not paired up. This is because the `groupby` function from itertools only groups consecutive items with the same key. When grouping by non-consecutive keys, it will yield duplicate keys instead of properly grouping data under the same key. In my experience, this is rarely desired.

This has bitten both me and my colleagues more than once. Often the data I'm grouping isn't naturally ordered, and I usually don't care about the order of the output. Getting the grouping right, however, is important.

## The solution
The solution is to sort your collection by the grouping key first.

    sorted_friends = sorted(friends, key=lambda x: x["couple_id"])
    sorted_friends_groups = groupby(sorted_friends, lambda x: x["couple_id"])

    for key, grouping in sorted_friends_groups:
        print(key, "-->", " loves ".join([obj["name"] for obj in grouping]))

This yields:

    1 --> Thomas loves Imani
    2 --> Eugene loves Beatrice
    3 --> John loves Mary

Once sorted, our couples have properly found one another.

## Why is this a surprise?
Besides Python, I primarily use TypeScript and C#, and neither of them has the consecutive key requirement. SQL, a set-based language, does not have that requirement when using `group by` to aggregate rows. For reference, here are analogous JavaScript and C# snippets.

**JavaScript**

    const friends = [
      { coupleId: 3, name: "John" },
      { coupleId: 1, name: "Thomas" },
      { coupleId: 2, name: "Eugene" },
      { coupleId: 2, name: "Beatrice" },
      { coupleId: 1, name: "Imani" },
      { coupleId: 3, name: "Mary" },
    ];

    const friendsGroups = Map.groupBy(friends, (x) => x.coupleId);
    for (const [key, grouping] of friendsGroups) {
      console.log(key, "-->", grouping.map((obj) => obj.name).join(" loves "));
    }

**C#**

    List<Person> friends = [
        new Person {CoupleId = 3, Name = "John"},
        new Person {CoupleId = 1, Name = "Thomas"},
        new Person {CoupleId = 2, Name = "Eugene"},
        new Person {CoupleId = 2, Name = "Beatrice"},
        new Person {CoupleId = 1, Name = "Imani"},
        new Person {CoupleId = 3, Name = "Mary"}
    ];

    var groupedFriends = friends.GroupBy(x => x.CoupleId);
    foreach (var group in groupedFriends)
    {
        var coupleId = group.Key;
        var names = string.Join(" loves ", group.Select(x => x.Name));
        Console.WriteLine($"{coupleId} --> {names}");
    }

You can find the complete code samples for each language in [GitHub](..).

Happy coding!
