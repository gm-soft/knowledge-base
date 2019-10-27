---
category: .net
tags: [.net, async, copypaste]
---

# My two cents on the yield keyword in C#

Source: [infoworld.com](https://www.infoworld.com/article/3122592/my-two-cents-on-the-yield-keyword-in-c.html).

Author: [Joydip Kanjilal](https://www.infoworld.com/author/Joydip-Kanjilal/)

---

The yield keyword performs custom and stateful iteration and returns each element of a collection one at a time sans the need of creating temporary collections

The yield keyword, first introduced in C# 2.0, T returns an object that implements the IEnumerable interface. The IEnumerable interface exposes an IEnumerator that can used to iterate a non-generic collection using a foreach loop in C#. You can use the yield keyword to indicate that the method or a get accessor in which it has been used is an iterator.

There are two ways in which you can use the yield keyword: Using the "yield return" and the "yield break" statements. The syntax of both is shown below.

```csharp

yield return <expression>;

yield break;

```


## Why should I use the yield keyword?

The yield keyword can do a state-full iteration sans the need of creating a temporary collection. In other words, when using the "yield return" statement inside an iterator, you need not create a temporary collection to store data before it returned. You can take advantage of the yield return statement to return each element in the collection one at a time, and you can use the "yield return" statement with iterators in a method or a get accessor.

Note that the control is returned to the caller each time the "yield return" statement is encountered and executed. Most importantly, with each such call, the callee's state information is preserved so that execution can continue immediately after the yield statement when the control returns.

Let’s look at an example. The following code snippet illustrates how the yield keyword can be used to return a Fibonacci number. The method accepts an integer as argument that represents the count of the Fibonacci numbers to generate.

```csharp

static IEnumerable<int> GenerateFibonacciNumbers(int n)
{
    for (int i = 0, j = 0, k = 1; i < n; i++)
    {
        yield return j;

        int temp = j + k;
        j = k;
        k = temp;
    }
}

```

As shown in the code snippet above, the statement “yield return j;” returns Fibonacci numbers one by one without exiting the “for” loop. In other words, the state information is preserved. Here's how the GenerateFibonacciNumbers method can be called.

```csharp

foreach (int x in GenerateFibonacciNumbers(10))
{
    Console.WriteLine(x);
}

```

As you can notice, there is no need of creating an intermediate list or array to hold the fibonacci numbers that need to be generated and returned to the caller.

Note that under the covers the yield keyword creates a state machine to maintain state information. The MSDN states: "When a yield return statement is reached in the iterator method, expression is returned, and the current location in code is retained. Execution is restarted from that location the next time that the iterator function is called."

Another advantage of using the yield keyword is that the items that are returned are created only on demand. As an example, the following get accessor returns the even numbers between 1 and 10.

```csharp

public static IEnumerable<int> EvenNumbers
{
    get
    {
        for (int i = 1; i <= 10; i++)
        {
            if((i % 2) ==0)
                yield return i;
        }
    }
}

```

You can access the EvenNumbers static property to display the even numbers between 1 and 10 at the console window using the code snippet given below.

```csharp

foreach (int i in EvenNumbers)
{
    Console.WriteLine(i);
}

```

You can use the "yield break" statement within an iterator when there are no more values to be returned. The "yield break" statement is used to terminate the enumeration.

```csharp

public IEnumerable<T> GetData<T>(IEnumerable<T> items)
{
   if (null == items)
       yield break;

   foreach (T item in items)
       yield return item;
}

```

Refer to the code listing above. Note how a check is made to see if the "items" parameter is null. When you invoke the GetData() method within an iterator and with null as parameter, the control just returns back to the caller without any value being returned.

## Points to remember

When working with the yield keyword, you should keep these points in mind:

- You cannot have the yield return statement in a try-catch block though you can have it inside a try-finally block
- You cannot have the yield break statement inside a finally block
- The return type of the method where yield has been used, should be `IEnumerable`, `IEnumerable<T>`, `IEnumerator`, or `IEnumerator<T>`
- You cannot have a ref or out parameter in your method in which yield has been used
- You cannot use the `yield return` or the `yield break` statements inside anonymous methods
- You cannot use the `yield return` or the `yield break` statements inside "unsafe" methods, i.e., methods that are marked with the "unsafe" keyword to denote an unsafe context