---
category: human resources
tags: [teamlead, management, interview]
---

# What Great .NET Developers Ought To Know (More .NET Interview Questions)

## Everyone who writes code

- Describe the difference between a Thread and a Process?
- What is a Windows Service and how does its lifecycle differ from a "standard" EXE?
- What is the maximum amount of memory any single process on Windows can address? Is this different than the maximum virtual memory for the system? How would this affect a system design?
- What is the difference between an EXE and a DLL?
- What is strong-typing versus weak-typing? Which is preferred? Why?
- Corillian's product is a "Component Container." Name at least 3 component containers that ship now with the Windows Server Family.
- What is a PID? How is it useful when troubleshooting a system?
- How many processes can listen on a single TCP/IP port?
- What is the GAC? What problem does it solve?

## Mid-Level .NET Developer

- Describe the difference between Interface-oriented, Object-oriented and Aspect-oriented programming.
- Describe what an Interface is and how it’s different from a Class.
- What is Reflection?
- What is the difference between XML Web Services using ASMX and .NET Remoting using SOAP?
- Are the type system represented by XmlSchema and the CLS isomorphic?
- Conceptually, what is the difference between early-binding and late-binding?
- Is using Assembly.Load a static reference or dynamic reference?
- When would using Assembly.LoadFrom or Assembly.LoadFile be appropriate?
- What is an Asssembly Qualified Name? Is it a filename? How is it different?
- Is this valid? Assembly.Load("foo.dll");
- How is a strongly-named assembly different from one that isn’t strongly-named?
- Can DateTimes be null?
- What is the JIT? What is NGEN? What are limitations and benefits of each?
- How does the generational garbage collector in the .NET CLR manage object lifetime? What is non-deterministic finalization?
- What is the difference between Finalize() and Dispose()?
- How is the using() pattern useful? What is IDisposable? How does it support deterministic finalization?
- What does this useful command line do? tasklist /m "mscor*"
- What is the difference between in-proc and out-of-proc?
- What technology enables out-of-proc communication in .NET?
- When you’re running a component within ASP.NET, what process is it running within on Windows XP? Windows 2000? Windows 2003?

## Senior Developers/Architects

- What’s wrong with a line like this? DateTime.Parse(myString);
- What are PDBs? Where must they be located for debugging to work?
- What is cyclomatic complexity and why is it important?
- Write a standard lock() plus “double check” to create a critical section around a variable access.
- What is FullTrust? Do GAC’ed assemblies have FullTrust?
- What benefit does your code receive if you decorate it with attributes demanding specific Security permissions?
- What does this do? gacutil /l | find /i "Corillian"
- What does this do? sn -t foo.dll
- What ports must be open for DCOM over a firewall? What is the purpose of Port 135?
- Contrast OOP and SOA. What are tenets of each?
- How does the XmlSerializer work? What ACL permissions does a process using it require?
- Why is catch(Exception) almost always a bad idea?
- What is the difference between Debug.Write and Trace.Write? When should each be used?
- What is the difference between a Debug and Release build? Is there a significant speed difference? Why or why not?
- Does JITting occur per-assembly or per-method? How does this affect the working set?
- Contrast the use of an abstract base class against an interface?
- What is the difference between a.Equals(b) and a == b?
- In the context of a comparison, what is object identity versus object equivalence?
- How would one do a deep copy in .NET?
- Explain current thinking around IClonable.
- What is boxing?
- Is string a value type or a reference type?
- What is the significance of the "PropertySpecified" pattern used by the XmlSerializer? What problem does it attempt to solve?
- Why are out parameters a bad idea in .NET? Are they?
- Can attributes be placed on specific parameters to a method? Why is this useful?

## C# Component Developers

- Juxtapose the use of override with new. What is shadowing?
- Explain the use of virtual, sealed, override, and abstract.
- Explain the importance and use of each component of this string: Foo.Bar, Version=2.0.205.0, Culture=neutral, PublicKeyToken=593777ae2d274679d
- Explain the differences between public, protected, private and internal.
- What benefit do you get from using a Primary Interop Assembly (PIA)?
- By what mechanism does NUnit know what methods to test?
- What is the difference between: catch(Exception e){throw e;} and catch(Exception e){throw;}
- What is the difference between typeof(foo) and myFoo.GetType()?
- Explain what’s happening in the first constructor: public class c{ public c(string a) : this() {;}; public c() {;} } How is this construct useful?
- What is this? Can this be used within a static method?

## ASP.NET (UI) Developers

- Describe how a browser-based Form POST becomes a Server-Side event like Button1_OnClick.
- What is a PostBack?
- What is ViewState? How is it encoded? Is it encrypted? Who uses ViewState?
- What is the `<machinekey>` element and what two ASP.NET technologies is it used for?
- What three Session State providers are available in ASP.NET 1.1? What are the pros and cons of each?
- What is Web Gardening? How would using it affect a design?
- Given one ASP.NET application, how many application objects does it have on a single proc box? A dual? A dual with Web Gardening enabled? How would this affect a design?
- Are threads reused in ASP.NET between reqeusts? Does every HttpRequest get its own thread? Should you use Thread Local storage with ASP.NET?
- Is the [ThreadStatic] attribute useful in ASP.NET? Are there side effects? Good or bad?
- Give an example of how using an HttpHandler could simplify an existing design that serves Check Images from an .aspx page.
- What kinds of events can an HttpModule subscribe to? What influence can they have on an implementation? What can be done without recompiling the ASP.NET Application?
- Describe ways to present an arbitrary endpoint (URL) and route requests to that endpoint to ASP.NET.
- Explain how cookies work. Give an example of Cookie abuse.
- Explain the importance of HttpRequest.ValidateInput()?
- What kind of data is passed via HTTP Headers?
- Juxtapose the HTTP verbs GET and POST. What is HEAD?
- Name and describe at least a half dozen HTTP Status Codes and what they express to the requesting client.
- How does if-not-modified-since work? How can it be programmatically implemented with ASP.NET?
- Explain <@OutputCache%> and the usage of VaryByParam, VaryByHeader.
- How does VaryByCustom work?
- How would one implement ASP.NET HTML output caching, caching outgoing versions of pages generated via all values of q= except where q=5 (as in http://localhost/page.aspx?q=5)?

## Developers using XML

- What is the purpose of XML Namespaces?
- When is the DOM appropriate for use? When is it not? Are there size limitations?
- What is the WS-I Basic Profile and why is it important?
- Write a small XML document that uses a default namespace and a qualified (prefixed) namespace. Include elements from both namespace.
- What is the one fundamental difference between Elements and Attributes?
- What is the difference between Well-Formed XML and Valid XML?
- How would you validate XML using .NET?
- Why is this almost always a bad idea? When is it a good idea? myXmlDocument.SelectNodes("//mynode");
- Describe the difference between pull-style parsers (XmlReader) and eventing-readers (Sax)
- What is the difference between XPathDocument and XmlDocument? Describe situations where one should be used over the other.
- What is the difference between an XML "Fragment" and an XML "Document."
- What does it meant to say “the canonical” form of XML?
- Why is the XML InfoSet specification different from the Xml DOM? What does the InfoSet attempt to solve?
- Contrast DTDs versus XSDs. What are their similarities and differences? Which is preferred and why?
- Does System.Xml support DTDs? How?
- Can any XML Schema be represented as an object graph? Vice versa?

## New Interview Questions for Senior Software Engineers

- What is something substantive that you've done to improve as a developer in your career?
- Would you call yourself a craftsman (craftsperson) and what does that word mean to you?
- Implement a <basic data structure> using <some language> on <paper|whiteboard|notepad>.
- What is SOLID?
- Why is the Single Responsibility Principle important?
- What is Inversion of Control? How does that relate to dependency injection?
- How does a 3 tier application differ from a 2 tier one?
- Why are interfaces important?
- What is the Repository pattern? The Factory Pattern? Why are patterns important?
- What are some examples of anti-patterns?
- Who are the Gang of Four? Why should you care?
- How do the MVP, MVC, and MVVM patterns relate? When are they appropriate?
- Explain the concept of Separation of Concerns and it's pros and cons.
- Name three primary attributes of object-oriented design. Describe what they mean and why they're important.
- Describe a pattern that is NOT the Factory Pattern? How is it used and when?
- You have just been put in charge of a legacy code project with maintainability problems. What kind of things would you look to improve to get the project on a stable footing?
- Show me a portfolio of all the applications you worked on, and tell me how you contributed to design them.
- What are some alternate ways to store data other than a relational database? Why would you do that, and what are the trade-offs?
- Explain the concept of convention over configuration, and talk about an example of convention over configuration you have seen in the wild.
- Explain the differences between stateless and stateful systems, and impacts of state on parallelism.
- Discuss the differences between Mocks and Stubs/Fakes and where you might use them (answers aren't that important here, just the discussion that would ensue).
- Discuss the concept of YAGNI and explain something you did recently that adhered to this practice.
- Explain what is meant by a sandbox, why you would use one, and identify examples of sandboxes in the wild.
- Concurrency
    - What's the difference between Locking and Lockless (Optimistic and Pessimistic) concurrency models?
    - What kinds of problems can you hit with locking model? And a lockless model?
    - What trade offs do you have for resource contention?
    - How might a task-based model differ from a threaded model?
    - What's the difference between asynchrony and concurrency?
- Are you still writing code? Do you love it?
- You've just been assigned to a project in a new technology how would you get started?
- How does the addition of Service Orientation change systems? When is it appropriate to use?
- What do you do to stay abreast of the latest technologies and tools?
- What is the difference between "set" logic, and "procedural" logic. When would you use each one and why?
- What Source Control systems have you worked with?
- What is Continuous Integration?  Have you used it and why is it important?
- Describe a software development life cycle that you've managed.
- How do you react to people criticizing your code/documents?
- Whose blogs or podcasts do you follow? Do you blog or podcast?
- Tell me about some of your hobby projects that you've written in your off time.
- What is the last programming book you read?
- Describe, in as much detail as you think is relevant, as deeply as you can, what happens when I type "cnn.com" into a browser and press "Go".
Describe the structure and contents of a design document, or a set of design documents, for a multi-tiered web application.
- What's so great about <cool web technology of the day>?
- How can you stop your DBA from making off with a list of your users’ passwords?
- What do you do when you get stuck with a problem you can't solve?
- If your database was under a lot of strain, what are the first few things you might consider to speed it up?
- What is SQL injection?
- What's the difference between unit test and integration test?
- Tell me about 3 times you failed.
- What is Refactoring ? Have you used it and it is important? Name three common refactorings.
- You have two computers, and you want to get data from one to the other. How could you do it?
- Left to your own devices, what would you create?
- Given Time, Cost, Client satisfaction and Best Practices, how will you prioritize them for a project you are working on? Explain why.
- What's the difference between a web server, web farm and web garden? How would your web application need to change for each?
- What value do daily builds, automated testing, and peer reviews add to a project? What disadvantages are there?
- What elements of OO design are most prone to abuse? How would you mitigate that?
- When do you know your code is ready for production?
- What's YAGNI? Is this list of questions an example?
- Describe to me some bad code you've read or inherited lately.
