# You've only added two lines - why did that take two days

Source in EN: [Matt Lacey](https://www.mrlacey.com/2020/07/youve-only-added-two-lines-why-did-that.html)
Translation in RU: [Habr.com](https://habr.com/ru/post/511044/)

---

It might seem a reasonable question, but it makes some terrible assumptions:

- lines of code = effort
- lines of code = value
- all lines of code are equal

None of those are true.

Why did a fix that seems so simple when looking at the changes made take two days to complete?

1. Because the issue was reported with a vague description of how to recreate it. It took me several hours to get to a reliable reproduction of the item. Some developers would have immediately gone back to the person reporting the problem and required more information before investigating. I try and do as much as I can with the information provided. I know some developers don't like having to fix bugs, and so do whatever they can to get out of it. Claiming there isn't enough is a great way to look like you're trying to help but not have to do anything. I know that reporting errors can be hard, and I'm grateful for anyone who does. I want to show appreciation for error reports by trying to do as much as possible with the information provided before asking for more details.

2. Because the reported issue was related to functionality, I'm not familiar with. The feature it was to do with was something I rarely use and is not something I've ever used in great detail. This meant it took me longer than it might to understand how to use it and the nuances of how it interacts with the software with the bug.

3. Because I took the time to investigate the real cause of the issue, not just looking at the symptoms. If some code is throwing an error, you could just wrap it in a try..catch statement and suppress the error. No error, no problem. Right? Sorry, for me, making the problem invisible isn't the same as fixing it. "Swallowing" an error can easily lead to other unexpected side-effects. I don't want to have to deal with them at a point in the future.

4. Because I investigated if there were other ways of getting to the same problem, not just the reported reproduction steps. One set of reproduction steps can easily make the error appear to be in one place when it may actually be more deep-seated. Finding the exact cause of a problem, and looking at all the ways to get there can provide valuable insights. Insights such as how the code is actually used, where there might be other places with possible (other?) problems that might need addressing, or it may show inconsistencies in the code that mean an error is caused (or handled) in one code path but not another.

5. Because I took the time to verify if there were other parts of the code that might be affected in similar ways. If a mistake led to the bug, the same error could have also been made elsewhere in the code-base. Now's a great time to check.

6. Because when I found the cause of the issue, I looked to find the simplest way of fixing it that would have minimal risk of introducing side-effects. I don't want the quickest possible fix. I want a fix that isn't likely to cause confusion or other problems in the future.

7. Because I tested the change thoroughly and verified that it addressed the problem for all the different code paths that were affected. I don't want to rely on someone else to have to test that what I've done is correct. I don't want a bug to be found in the future and for me to have to come back to this code when I've mentally moved on. Context switching is expensive and frustrating. Having a dedicated tester have to look at the "same" change again is something I want to avoid whenever possible.

I don't like having to fix bugs. Partly because they can feel like the result of a previous failure on my part. The other reason I don't like fixing bugs is that I'd prefer to be working on new things.

What's worse than having to fix a bug? Having to fix the same bug repeatedly.

I take the time to make sure any bug is totally fixed any time it is encountered so that it doesn't need to be faced, investigated, fixed, and tested more than once.
