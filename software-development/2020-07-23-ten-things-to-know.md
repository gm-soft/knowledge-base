# 10 Things Every Developer Should Learn

Translation in RU: [Habr](https://habr.com/ru/company/ruvds/blog/508440/)
Source in EN: [Medium](https://medium.com/better-programming/10-things-every-developer-should-learn-72697ed5d94a)

---

## Security Should Never Be an Afterthought

Application security should never be an afterthought or relegated to “we’ll add security later.” Strong application security requires it be a part of the conversation, and development pipeline, from Day 1 — not Day 300.

Leaving security until the last minute will actually increase your development time now that you have to go back and refactor to accommodate for it. Or worse, you have less time to work out any holes, and you end up shipping vulnerable code. Just ask companies like Yahoo how that worked out for them.

## Every Application Is Different and Has Different Needs, Choose According to the Application Needs, Not Political Pressure or Market Popularity

This really shouldn’t need to be said, but every application is different. There’s no one sacred set of rules that apply to all applications (these rules included). When starting a new application, it’s the application and its architecture that should dictate what technologies to use or what platforms to standardize on.

Deciding you’re going to use gRPC or Kubernetes before asking the question “What does my application need?” only does one thing: It puts roadblocks in your way before you write even a single line of code. This is how we get into ludicrous situations where companies like Canonical are offering Kubernetes for IoT devices. To quote Jeff Goldblum, “Your scientists were so preoccupied with whether or not they could, they didn’t stop to think if they should.”

## You Probably Don’t Need to “Do Microservices”

Microservices are sexy. I get it. The idea of being able to independently scale various bits and bobs in your application is exciting, and so is justifying your continued work on that particular codebase. But let’s be honest with ourselves — you probably don’t need to “do microservices.” Reasons like “I want to be able to decouple feature X code from feature Y code,” “keep bad code from creeping into other parts of the app,” or “minimize the blast radius should the app get compromised” aren’t reasons for moving to microservices; they’re symptoms of bad development practices (stay in your lane, touch only what’s necessary), code review standards that need to be more strict (bad code shouldn’t be merged, period), and a lack of fine-grained security controls, respectively.

Do you need to independently scale various parts of your application and are currently having capacity issues with one or more components, such as a login flow? Then you should probably explore breaking out those components that need to scale into separate services, if possible.

Are you running a virtual-server-based architecture and want to cut costs? Then you should not explore microservices. At best you’re going to break even. At worst you’re going to end up needing to spin up additional instances. Let’s say you have a monolith with five services in it that you then break apart into microservices. Now, you have five apps that either a) you spin up dedicated instances for, multiplying your initial footprint by five, or b) you use your existing footprint and now simply incur greater operational costs to manage it.

## Have a Standardized Development Environment

One of the most beneficial things you can do when working with more than one developer is to standardize your development environment across your team. Now, this doesn’t mean you have to hack together some container-based, virtual-development environment wizard magic. You can if you want, but something as simple as using the same language version can work wonders on your team’s sanity. Attempting to diagnose bugs on Go 1.12 while your coworker is writing code on Go 1.11 will only make you cry. Coordinating when to upgrade versions can be difficult, but if you get it right things can flow smoothly.

## Configuration Is Harder Than It Seems; Plan Accordingly

Contrary to what some popular websites say, configuration is a bit more complicated than “put everything in environment variables.” It’s my opinion that there should be no less than four ways to configure an app: in-code defaults, local config file, command-line flags, environment variables, remote configuration source (such as Hashicorp’s Consul). I would consider remote configuration optional; however, the other four are necessary. For development, relying on putting 27 different config values into environment variables just to run your app locally can be frustrating, to say the least.

Also, maybe you need better automation and a Makefile? Providing a way to have a local config source, like an application.yaml, allows you to set a default “dev” config. Additionally, having in-code defaults means that you only need to set the values you want to change from their defaults. Command-line flags are very useful when running an application via an init system like systemd, as it makes seeing the configuration easier when tracing a process. Environment variables are very useful when running in a container, yes, but there are some things they’re not suited for, like secrets.

You should never put secrets — things like passwords, authentication tokens, certificates, and generally anything you don’t want leaking out to the general public — into environment variables, as they are not secure and can be read by literally any process on the host machine. You should always use some sort of secrets manager for your secrets. My personal pick is Vault by Hashicorp, but use what works best for your app.

## Use Packages When You Need to, Not Just Because You Can

We all know about the left-pad apocalypse right? How one 11-line NPM package getting removed from the repository broke, well, everything on the internet? Don’t do that. Packages should be used when there’s a legitimate need to import one, like a vendor-specific SDK (AWS’ SDK, for example), an abstraction on a highly verbose set of standard libraries (that’s why people like using Python’s Requests instead of urllib), or a widely used framework such as Go’s Echo HTTP server or Python’s Flask WSGI server. Some convenience libraries are OK too, like JavaScript’s Lodash that provides some common functionality and extras not found in the standard library. These external dependencies should make developing easier and not require you to write boilerplate or integration code by hand — this is the benefit of package systems. But, like left-pad, it’s easy to fall into the trap of “oh, there’s a library for this, I’ll just use that.” For every dependency you import, you increase the risk that dependency will introduce instability, insecurity, or just simply not be maintained.

The risk also increases with every package your new dependency itself imports — also known as a transitive dependency. If you import a single package that, in turn, imports five packages, you have now inherited those five dependencies and all of the risk and hazards that come with them. I and many others in this industry would argue that packages shouldn’t introduce transitive dependencies. This isn’t always possible, but at the very least packages should be as lean as possible and, if greater functionality is desired, provide a way for the user to explicitly extend it.

A simple rule that I try to follow with importing libraries is if I can write it on my own in about 10–15 minutes, then I do. Otherwise, I’ll use an external library for it if one is available. Developing with this rule in mind will save you from importing unnecessary cruft into your application but is lenient enough where you’re not expected to write a new HTTP server from scratch every time you want to serve an API.

## You Don’t Need to Abstract That Bit of Code Until You Do

One big pitfall that’s super easy to fall in to is the “abstract everything” hole. The thought of “oh, but I might need to reuse this later” can lead down some dark and scary object-oriented paths. And I get why. The DRY principle (don’t repeat yourself) is drilled into our heads, and with good reason. But there’s a limit to where you’re spending too much time abstracting and not enough time writing logic. Just write your code! If you find you need to implement a method or a function that’s similar to something else you’ve already done, then you can go back and abstract it — but again, with limits. A personal rule that I tend to go by is if it’s a simple three-line function before abstraction, then maybe it’s fine to leave it as is and just repeat it. And if it is just 3 lines, maybe ask if it needs to be a function at all.

## You Should “Phoenix” Your Project From Time to Time

This one is the scariest of all. It makes managers nervous, it makes product owners cranky, and it makes developers angry, but you absolutely need to do it. Starting over from the beginning every once in a while is a good thing. It allows you to remove the crap from your code, implement new ideas without retrofitting half the existing codebase, and forces everyone to re-evaluate the project.

Think of your project as a forest, each line of code a mighty pine tree in a vast forest stretching for miles. As the forest ages, it becomes littered with undergrowth, cast-off pine needles, pine cones, dead branches, and a host of other debris. This is your cruft; your technical debt. It piles up and up, never stopping until some radical change is affected. For the forest that change comes in the form of a wildfire. The fire sweeps through the forest burning up the unwanted grift that has accumulated. The trees whose bark is thick enough remain, with the immature or underdeveloped trees being consumed in the fire.

While this might seem like an end for the forest, it holds a great secret: It was waiting for the fire. Patiently, for years, the forest has been hoping for a fire to cleanse it and bring about change, because as the fire rages below the canopy, the next generation of towering trees is germinating in their pine cones. As the fire marches across the forest floor, it makes way for tender vulnerable saplings to emerge and take their place next to the fire-blackened survivors. This is how your application must be: the resilient, well-written parts survive the purge while the rest make way for new ideas and new code to emerge from the charred remains — like a phoenix rising from the ashes.

## You’re Not Google

Unless you are. But if that’s the case then why are you reading this? The point is that you’re most likely not Google, or Microsoft, or Amazon, or Twitter, or Facebook. You don’t have a need to orchestrate 150,000 containers across 10,000 bare-metal servers in 17 different data centers around the world. Your problems aren’t typically impacts-every-single-person-in-the-world big. Why are we talking about this? Because your scale should determine your operational platform.

If you’re running a few hundred containers, do you really need Kubernetes? And do you really need to run it yourself, or are you just trying to add it to your résumé? Something like HashiCorp Nomad is perfect for those small-to-mid scale problems: It’s easy to set up, requires little to no maintenance, is well documented, and is really easy to transition your applications to since it works with containers as well as system processes and JVM applications natively. And if you really want to run Kubernetes, why not do it under the management of something like Rancher that abstracts all the messy crap away?

The headache and overhead of running a complex system like Kubernetes, that was designed at and for companies like Google, is too much for a single team to manage. I would even flat out say that startups should avoid it at all costs unless their product is specifically targeted to Kubernetes. A caveat is using managed services like those offered by Google, Amazon, and Microsoft in their respective cloud offerings. Because they manage all of the ugly, a lot of that overhead isn’t there for you to take on yourself. But never, ever, ever let me catch you using Kubernetes on IoT devices. Please, just don’t.

## Don’t Form Your Development Philosophy Entirely From Strangers on the Internet

You should make up your own mind about what rules apply to your application and to your development style. Even these ten things are up for debate. I’m just a guy on the internet.
