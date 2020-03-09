# [EPAM] Вопросы для собеседования

# Introduction

Check connection with candidate. Name yourself. State purpose, length (1 hour), and structure of this "technical conversation". Structure: general talk, theoretical questions, English talk, candidate's questions.

## General talk

- [ ]  What was your role and responsibilities on your recent project?

## C#

- [ ]  Difference between for and foreach keywords.

    **Follow-up:** when using foreach, can you modify a collection that you are iterating through?

- [ ]  Difference between while and do...while loops.
- [ ]  What access modifiers do you know?
- [ ]  How are "break" and "continue" keywords used in loops?
- [ ]  How to handle exceptions in C# ?

    **Follow-up:** how does scheme with two catch blocks work?

- [ ]  What is result of (true || false && false) ?
- [ ]  Difference between overridden method and overloaded method.
- [ ]  Difference between static and non-static constructor.
- [ ]  Difference between "ref" and "out" parameter
modifiers.
- [ ]  How is "lock" keyword used?

    **Follow-up:** what should not be used as argument for "lock"?

- [ ]  How to call method or constructor of the parent class?
- [ ]  What is extension method?

    **Follow-up:** why can't you just use partial?

- [ ]  Difference between abstract class and interface.

    **Follow-up:** when to use what?

- [ ]  Difference among "is", "as" and casting (explicit conversion).
- [ ]  What can you tell about generic types, methods and delegates in C# ?

    **Follow-up**: how "where" keyword is used?

## .NET

- [ ]  What are the main principles of object-oriented programming (OOP) and how to apply them in .NET Framework?
- [ ]  What collection types do you know?

     *Array, List<>, Stack<>(LIFO), Queue<>(FIFO), Dictionary<K,V>*

- [ ]  Difference between value type and reference type.

    **Follow-up:** is String value type or reference
    type?

- [ ]  What did you use for parallel (multi-threading) programming or asynchronous programming?
- [ ]  What are unmanaged resources?

    **Follow-up:** how to prevent leak of unmanaged resources?

- [ ]  How does garbage collector work?

    **Follow-up:** give an example of how a memory leak can happen in .NET application.

## SQL Server

- [ ]  What are the authentication modes in SQL Server?
- [ ]  What is FOREIGN KEY?

    **Follow-up:** may a foreign key reference another column in the same table?

- [ ]  Difference between clustered and a non-clustered index.
- [ ]  How to implement one-to-one, one-to-many and many-to-many relationships while designing tables?
- [ ]  What are advantages and disadvantages of having multiple indexes? SQL
- [ ]  What is the difference between (INNER) JOIN and LEFT (OUTER) JOIN ?
- [ ]  What is the difference between UNION and UNION ALL ?
- [ ]  You have SELECT statement with 4 clauses: ORDER BY, HAVING, WHERE, GROUP BY. In which order should they appear in the statement?
- [ ]  How can subqueries be used (in which parts of the main query)?
- [ ]  What is SQL injection and how to avoid it?

## Entity Framework

- [ ]  Difference between database-first vs code-first approaches.

    **Follow-up:** which one do you personally prefer and why?

## LINQ

- [ ]  How to use "Include" method?
- [ ]  Which LINQ methods should you use to implement pagination?
- [ ]  What is AsEnumerable used for?

## [ASP.NET](http://asp.net/)

- [ ]  Difference between Windows Authentication and Forms Authentication.
- [ ]  Difference between authentication and authorization.
- [ ]  How is global.asax used?

## [ASP.NET](http://asp.net/) MVC

- [ ]  How does MVC pattern work?
- [ ]  What is strongly-typed view?
- [ ]  How do HTML helpers work? **Follow-up:** Did you create your own HTML helpers?
- [ ]  Where attributes can be used?

    **Follow-up:** Did you create your own attributes?

- [ ]  How to use ViewData or ViewBag?

    **Follow-up:** Can you use it together with passing a model?

- [ ]  What validation attributes do you remember?
- [ ]  What are partial views and how to use them?
- [ ]  Advantages and disadvantages of using entities as models for views.
- [ ]  How to implement setup correct caching of js/css files on production?

## JavaScript

- [ ]  Difference in declaring a variable with and without "var" keyword. **Follow-up:** How is "let" keyword different?
- [ ]  Difference between single quotes and double quotes for string literals.
- [ ]  If execution result of a function without return is assigned to a variable, what will it be?
- [ ]  How to implement namespaces in JavaScript?
- [ ]  Difference between "==" and "==="
- [ ]  What is the result of (1+1+"1") ?
- [ ]  What are data types in JavaScript? **Follow-up:** How JavaScript holds number values (using what type) ?
- [ ]  What built-in objects do you know?
- [ ]  How to implement a class in JavaScript?

## jQuery

- [ ]  What is the difference between using val method with and without argument?
- [ ]  What is method chaining in jQuery?
- [ ]  How can jQuery.each method be used? Follow-up: what are analogs of 'break' and 'continue' inside jQuery.each ?
- [ ]  What is the usage of methods jQuery.on and jQuery.off ?

## HTML

- [ ]  What is the difference between <div> and <span> default behavior?
- [ ]  What are two tags inside <html> element? **Follow-up:** what should be put into <head>
and <body> elements?
- [ ]  What is structure of <table> tag? **Follow-up:** how colspan and rowspan attributes are
used?
- [ ]  What are different approaches to create page layouts?

## CSS

- [ ]  What are three places where CSS can be defined? **Follow-up:** what is the restriction for CSS defined in 'style' attribute?
- [ ]  Difference between selectors "footer" , ".footer" and "#footer"
- [ ]  Difference between selectors "div span", "div > span" and "div, span"
- [ ]  What is CSS box model? **Follow-up:** is it true that some older browsers calculated content's width/height differently?
