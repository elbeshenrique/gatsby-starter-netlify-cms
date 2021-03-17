---
templateKey: blog-post
title: Why I built an ORM?
date: 2021-03-17T21:43:48.617Z
description: This is the story of why (maybe a little bit of how) I built a
  proprietary ORM. I aim to share with you a few choices and experiences that me
  and my team has on the mission of building that.
featuredpost: true
featuredimage: https://miro.medium.com/max/1844/1*vK7NzagpDws_lSJYeKV8Yw.png
---
<!--StartFragment-->

![](https://miro.medium.com/max/1844/1*vK7NzagpDws_lSJYeKV8Yw.png "an ORM’s diagram representation")



**Object-relational Mapping**

ORM stands for **[Object-relational Mapping](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping)** and is a programming technique to deal with the conversion of data between objects (from object oriented languages for example) and relational databases structures, such as tables and views. This technique allows the developer to define and manipulate relational data as object classes, using the application code instead of **pure SQL**.

The main benefit of using an ORM is that it simplifies the reading, inserting, updating and deleting processes for the developer, freeing more time to focus on the core business logic and application features.

**A Little of the Backstory**

In the year of 2013, I was a beginner programmer participating of a project that has the objective of developing a core web application and it’s secondary applications using .NET technologies and C# as the main programming language.

At the time, the team and I had the task of choosing the best way to approach data access and manipulation in the database. We were using mainly two database systems: **[SQL Server](https://en.wikipedia.org/wiki/Microsoft_SQL_Server)** on the primary application, which stored the core data and **[MySQL](https://en.wikipedia.org/wiki/MySQL)** on the secondary applications, which needed to store temporary local data to be synchronized with the main server data.

**The Why**

We were planning to use in both applications one of the existing ORMs on the market that could use multiple database providers, which would bring the main benefit of being able to access data similarly on the primary and secondaries applications and would smooth the learning curve as we would need to learn only one way to work with data (at least at the beginning).

After some meetings and discussions we decided that besides **[NHibernate](https://nhibernate.info/)** (an ORM built with C# too) was a good option, it had enough critical conditions for us to become a no go and they had to do with performance and initial load time. Unfortunately **[Microsoft Entity Framework (EF)](https://docs.microsoft.com/en-us/ef/)** was not traveling calm waters at the time (before it became open source) and we didn’t know that **[Dapper](https://github.com/StackExchange/Dapper)** existed back then. We needed to make a fast decision. The clock was ticking.

Because the performance of NHibernate was not satisfying our needs based on our experience with other applications that we had worked on and we hadn’t seen any other suitable options on the market at the time, we ended up deciding to build our own **lightweight ORM** based on NHibernate mapping capabilities, which should be flexible enough to support any relational database we would need on the way.

The responsibility of coding this library was given to me. A solo coding mission but I had massive help from the team’s greatest brains, no doubt. Thank you guys!

**Getting the Hands Dirty (on the good sense)**

Here I started my journey of coding this ORM. I knew we needed some basic features to get the things the way we planned, so basically this were the main features that I needed to aim for:

* **Fast initial loading time;**
* One table or view equals to one entity;
* **Easy way to map database entities using [attributes/annotations/decorators](https://docs.microsoft.com/pt-br/dotnet/csharp/tutorials/attributes);**
* The ability to read data from the database, choose columns (or use all, yes [I’m aware that * is bad](https://dzone.com/articles/why-you-should-not-use-select-in-sql-query-1)), set filter criteria, sort, paginate and transform that data in entity instances (SELECT, WHERE, ORDER BY, TOP/OFFSET/LIMIT);
* The ability to use One-to-One entity relationships (INNER and LEFT JOIN);
* The ability to write new data from new entities to the database and choose which columns (or all) that would be saved (INSERT, VALUES);
* The ability to update existing data from entities to the database, choose which columns (or all) that would be updated and set a filter criteria to decide which rows would be affected (UPDATE, SET, WHERE);
* The ability to delete data based on a filter criteria (DELETE, WHERE);
* Support composite keys on the entity mapping, they being made of primitive types or entity classes ([I regret a lot of making this decision](https://medium.com/@pablodalloglio/7-reasons-not-to-use-composite-keys-1d5efd5ec2e5));
* Support multiple relational databases based on the appropriate provider implementation;
* **Being able to write queries and statements without using string text written queries.**

Well, it seems like I’ve got a lot of work ahead. Let’s get to it.

\*\** Sadly I cannot show any pieces of the real code in this article because this is a proprietary solution.

**The Entity Mapping**

I started by implementing the entity mapping features that we needed to be able to write entities that would represent the database tables and views in the codebase.

Basically one class would represent one table or view and it’s public properties that has the appropriate attribute/annotation would represent one or more columns, as one column could be represented by primitive data type or by another entity class, properly mapped, with one or more columns as primary key. The second case would result in a one-to-one relationship and its foreign keys.

Having the mapping attributes implemented, it was time to find a way to load the mapping information into some data structure that would be both fast to access and fast to load at the application’s startup.

I decided to use **[dictionaries](https://en.wikibooks.org/wiki/A-level_Computing/AQA/Paper_1/Fundamentals_of_data_structures/Dictionaries)** to implement the “fast to access” problem. A dictionary is an associative array data structure that is composed by a collection of key-value pairs where the key can never occur more than one time, as in a **[JSON](https://en.wikipedia.org/wiki/JSON) object**. Usually the time taken to do a lookup in these types of data structures is **constant (O(1))** and the space required is **linear (O(n))**. These characteristics make them the perfect fit for this scenario when you need to access information quickly and don’t spend absurd amounts of memory. It’s important to also take note that this structure was built only once, at the application startup, so the costs of insertion would only take effect one time and the size was previously known.

So, by using **[reflection](https://en.wikipedia.org/wiki/Reflective_programming)**, the metadata of the classes could be read and stored in these dictionaries, but as I knew, reflection operations are very performance expensive and this caused a longer initial loading time that I was hoping for.

The main issue there had to do with the fact that on every application’s startup the entity classes were iterated using reflection and loaded into the memory (dictionaries), this caused a very negative performance impact at the server’s first request which was resulting in a long waiting everytime the server was initialized and so, on every deploy.

The solution for that problem was to make the use of a **cache implementation** that would generate at compile time a binary file containing the entitie’s mapping data. Then this file could be read and loaded into the dictionaries (memory) faster than using reflection at runtime. It did the job and the initial loading time was no longer a problem.

Entity mapping capabilities and fast initial loading, check.

This is an example of one entity mapping class:

![Image 1: Example of an entity mapping class.](https://miro.medium.com/max/902/0*nFNOOtL9MxPazShM "Image 1: Example of an entity mapping class.")



**Queries and Statements Without String Texts**

The team had used NHibernate before in some smaller projects and we have dealt with a lot of time waste related to **refactoring queries**. So based on that experience we decided that the ability to write queries without using string text (to name fields, write filters, choose columns, etc) was essential for future maintainability and “easier to write” queries (more than pure SQL), with the exception of too complex queries of course.

That had to do primarily with the fact that when writing queries we could use the power of modern [IDEs](https://en.wikipedia.org/wiki/Integrated_development_environment) and their **code completion helper tools** to find the fields in an easier and faster way, reducing typos and miswritten columns. This also would help a lot with the renaming and refactoring of the queries and entities, seeing that the best IDEs have this kind of tools built-in.

We were inspired by Microsoft Entity Framework on this one (yes, the one we didn’t want to use back then).

So, as the main goal here was to be able to write queries without the use of string texts — with the exception of cases that we would want to use texts like in “where” comparisons for example — we decided to use the power of **[lambda expressions](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/lambda-expressions)** and .NET ability’s to access it’s metadata information at runtime.

This would give us the power to write from column selection to filter criteria **without using any plain text**. This looked awesome to use in comparison with other ways we had done until then.

The final result looked something like this:

![Image 2: Example of a query written using the ORM.](https://miro.medium.com/max/950/0*WdwjdUSMk0LoHrA4 "Image 2: Example of a query written using the ORM.")

This piece of code would result in a list of the type “TableExample” filled with 10 instances at max, each populated with data of the selected fields listed in the “Select” call, corresponding to the rows that satisfacts the filter criteria written in the “Where” call, sorted descrescently by the field in the “OrderByDesc” call, skipping the first 10 rows respecting the sort rule. The fields not listed in the “Select” call would have the default value of their respective type (int would have 0 as value and entity classes would be null).

Basically, an SQL query written without the use of any string texts.

The resulting SQL query would look something like this:

![Image 3: Example of the SQL query generated by the ORM.](https://miro.medium.com/max/854/0*iNKgBzPuj7513rk4 "Image 3: Example of the SQL query generated by the ORM.")

Note that in the where criteria it was made the use of query parameters to avoid some security problems as SQL Injection. Each database provider could result in a different query, based on each database SQL syntax. The real query had a little bit more verbosity than the one of the Image 3 because of each database’s different SQL writing rules, but this was the gist.

To achieve this result, I had some **roadblocks and a lot of mistakes**, the majority of them related to bad use of lambda expressions, which are very expensive to evaluate at runtime, but I will leave these details to a part 2.

Similarly to queries, the ORM supported insert, update and delete operations much like the select example of the Image 2, all with the same writing style.

The insert operations could save entities with all or only the given fields and retrieve their respective ids, supporting operations with entities lists, which was not ideal. When executing batch operations I like to recommend pure SQL or bulk operations depending on the case, if supported.

The update operations had similar characteristics as the insert and could update only given fields of rows that match the filtering criteria, as well as, update a single entity based on the primary key.

The delete operations had only an optional filtering criteria and worked a lot like a delete statement, being able to delete many rows at once or delete a single entity row based on the primary key as well.

I can write more about them with more depth in a future article.

**Wrapping Up**

The features presented in this article sums up the very core and the main benefits of this tool. We made the use of it for **at least 5 years** until we felt the need to move on to other technologies that were evolving in the market and that decision was taken because we’re starting to need some advanced features that would improve a lot our productivity as a team and also improve the overall performance of the applications, such as, migrations, query caching and lambda expression processing optimizations at runtime.

**Other ORMs**

As mentioned, many features that are present in the Microsoft Entity Framework (and EFCore) today had made a huge impact on our decision for the next ORMs to be used.

Now I can say that the EF has great performance results and acceptable initial loading time, fitting our needs and if it’s state was like this at the time, I would have certainly chosen it.

Dapper has its own merits and today it’s one of my favorite ORM (micro ORM) of the market, being the fastest and simplest to use that I’ve ever seen. It certainly has the focus on performance over tons of features that EF has but it does the job and does it well, kudos to StackExchange’s team. Enough talking about these other ORMs, noting that this could give birth to another article of its own, they are awesome.

**The Aftermath**

As the scene of modern ORMs had been evolving and improving a lot as the years passed by, our own ORM **would demand a lot of time**, research and work to satisfy our needs forward on when compared to using other solutions currently available in the market.

Considering that, we moved on to the use of other ORMs (EF and Dapper) in the new parts of the application and in the parts that would demand refactoring. And as I mentioned at the beginning: **to focus our time on business logic and application features**.

Looking back, I can’t say for sure that I would have chosen a different path, but now I know, after all the work done, that the implementation of a satisfactory ORM is a very difficult and time consuming task. I will leave that task for the other respectable existing projects and teams that are focused specifically on this type of libraries and frameworks all around the world.

**The Use of ORMs**

It’s important to note that ORMs **are not suitable for all types of projects**, depending a lot on the nature of the application, the platform that it will be executed on and the quantity of entities mapped. On mobile apps for instance, it could make the user experience not smooth at all on more weak devices.

Yes, it’s true, **they make a programmer’s life much easier** when we talk about data handling and it’s a breeze using them if compared to pure SQL, but they have their costs, primarily related to performance, since they will need more memory to be used and add an extra layer between the application and the actual datasource.

It’s worth mentioning that the SQL query generated by ORMs could have a big negative impact on **how the SQL will use indexes** and can easily generate some very poor performant queries. Prepare for bottlenecks if you use it indiscriminately.

ORMs have their value for sure, but use with them caution and if it is really worth the time and the benefits.

**Conclusion**

Overall I can say that this project has served its purpose, for many years since 2013, delivering the expected performance and helping us to write and refactor queries with ease.

The journey of implementing it from the ground up was a **lot of fun, very challenging and I learned a lot** about coding and data handling, making me very proud of that accomplishment.

I hope this article about some of my experiences helps you on how to approach the decision of choosing the way you access and handle data on your own projects, and maybe even so, on the decision of if you should use an existing solution or code it yourselves.

Decide with care.

I understand that we shouldn’t reinvent the wheel, **but do not be afraid to step up and code something by yourselves** if you think that it’s the right call, even if similar things already exists.

Great solutions and ideas were born like that.

.

.

.

Thank you for reading this, I am looking forward to hearing your opinion!

See you next time.

<!--EndFragment-->