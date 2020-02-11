# Expression trees for dummies. StackOverflow

Источник - [ответ на StackOverflow](https://stackoverflow.com/a/20766203), который сам является пересказом [статьи](https://blogs.msdn.microsoft.com/charlie/2008/01/31/expression-tree-basics/)

---

An expression tree represents what you want to do, not how you want to do it.

Consider the following very simple lambda expression:

```csharp

Func<int, int, int> function = (a, b) => a + b;

```

This statement consists of three sections:

A declaration: `Func<int, int, int> function`
An equals operator: `=`
A lambda expression: `(a, b) => a + b;`
The variable function points at raw executable code that knows how to add two numbers.

This is the most important difference between delegates and expressions. You call function (a `Func<int, int, int>`) without ever knowing what it will do with the two integers you passed. It takes two and returns one, that's the most your code can know.

In the previous section, you saw how to declare a variable that points at raw executable code. Expression trees are not executable code, they are a form of data structure.

Now, unlike delegates, your code can know what an expression tree is meant to do.

LINQ provides a simple syntax for translating code into a data structure called an expression tree. The first step is to add a using statement to introduce the Linq.Expressions namespace:

`using System.Linq.Expressions;`

Now we can create an expression tree:
`Expression<Func<int, int, int>> expression = (a, b) => a + b;`

The identical lambda expression shown in the previous example is converted into an expression tree declared to be of type Expression<T>. The identifier expression is not executable code; it is a data structure called an expression tree.

That means you can't just invoke an expression tree like you could invoke a delegate, but you can analyze it. So what can your code understand by analyzing the variable expression?

```csharp

// `expression.NodeType` returns NodeType.Lambda.
// `expression.Type` returns Func<int, int, int>.
// `expression.ReturnType` returns Int32.

var body = expression.Body;
// `body.NodeType` returns ExpressionType.Add.
// `body.Type` returns System.Int32.

var parameters = expression.Parameters;
// `parameters.Count` returns 2.

var firstParam = parameters[0];
// `firstParam.Name` returns "a".
// `firstParam.Type` returns System.Int32.

var secondParam = parameters[1].
// `secondParam.Name` returns "b".
// `secondParam.Type` returns System.Int32.

```

Here we see that there is a great deal of information we can get from an expression.

But why would we need that?

You have learned that an expression tree is a data structure that represents executable code. But so far we have not answered the central question of why one would want to make such a conversion. This is the question we asked at the beginning of this post, and it is now time to answer it.

A LINQ to SQL query is not executed inside your C# program. Instead, it is translated into SQL, sent across a wire, and executed on a database server. In other words, the following code is never actually executed inside your program:
var query = from c in db.Customers where c.City == "Nantes" select new { c.City, c.CompanyName };

It is first translated into the following SQL statement and then executed on a server:

`SELECT [t0].[City], [t0].[CompanyName] FROM [dbo].[Customers] AS [t0] WHERE [t0].[City] = @p0`

The code found in a query expression has to be translated into a SQL query that can be sent to another process as a string. In this case that process happens to be a SQL server database. It is obviously going to be much easier to translate a data structure such as an expression tree into SQL than it is to translate raw IL or executable code into SQL. To exaggerate the difficulty of the problem somewhat, just imagine trying to translate a series of zeros and ones into SQL!

When it is time to translate your query expression into SQL, the expression tree representing your query is taken apart and analyzed, just as we took apart our simple lambda expression tree in the previous section. Granted, the algorithm for parsing the LINQ to SQL expression tree is much more sophisticated than the one we used, but the principle is the same. Once it has analyzed the parts of the expression tree, then LINQ mulls them over and decides the best way to write a SQL statement that will return the requested data.

Expression trees were created in order to make the task of converting code such as a query expression into a string that can be passed to some other process and executed there. It is that simple. There is no great mystery here, no magic wand that needs to be waved. One simply takes code, converts it into data, and then analyzes the data to find the constituent parts that will be translated into a string that can be passed to another process.

Because the query comes to the compiler encapsulated in such an abstract data structure, the compiler is free to interpret it in almost any way it wants. It is not forced to execute the query in a particular order, or in a particular way. Instead, it can analyze the expression tree, discover what you want done, and then decide how to do it. At least in theory, it has the freedom to consider any number of factors, such as the current network traffic, the load on the database, the current results sets it has available, etc. In practice LINQ to SQL does not consider all these factors, but it is free in theory to do pretty much what it wants. Furthermore, one could pass this expression tree to some custom code you write by hand which could analyze it and translate it into something very different from what is produced by LINQ to SQL.

Once again, we see that the expression trees allow us to represent (express?) what we want to do. And we use translators that decide how our expressions are getting used.