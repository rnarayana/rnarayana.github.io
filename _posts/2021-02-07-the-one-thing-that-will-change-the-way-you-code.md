---
title:  "Code Structure: The one thing that will change the way you code"
date: 2021-02-07
categories:
  - blog
tags:
  - code structure
  - design
  - monolith
  - modular
---

Well, not really :) In software engineering, like any other discipline, there isn't *one thing* that will change anything, instead there will be hundreds of such things which combinedly influence you. But I needed a catchy title, didn't I?

This week, I'll quickly touch upon why organizing your software projects in a particular way will keep your code more modular.

When you start with a new project, typically you start off with the code generators. Let's look at what Visual Studio gives you as a template.

![ProjStructure0](/assets/images/VS-NewProj.jpg)

Notice the `Controllers` folder with `WeatherForecastController.cs` in it, and a `WeatherForecast.cs`, the domain model class, outside of it. If you start off this way, and keep extending by adding your own controllers and models, you will end up with code that looks like this:

![ProjStructure1](/assets/images/VS-NewProj-Ext.jpg)

There is no help given by this style of organization when you try to read the code. Hardly would you need to see all the controllers or all the DAL classes in the project. Rather, you want to see a particular API, and how it works. WeatherForecastContoller, its associated services, domain objects, data classes and so on. So it makes no sense to organize code this way, even for small projects. We all know how small projects grow big over time.

The other problem with organizing code this way is that the interaction or dependencies between the classes does not "[scream](https://blog.cleancoder.com/uncle-bob/2011/09/30/Screaming-Architecture.html)" when you look at it.

Thirdly, to finish off the argument against this style, an API project is going to have controllers and models and data classes, why specify them explicitly. Do you create a folder called class and put all classes in it? Do you create a folder called interfaces and put all interfaces in it? If you do, DON'T!! You want to convey as much as possible about the *domain* of the project to the next maintainer or developer, not whether you have enums or contracts in your project.

It has been widely accepted that code that is modular and decoupled can be scaled and maintained well, especially if you are able to "split" them into smaller services as need arises. [Always start off with a monolith](https://codeopinion.com/start-with-a-monolith-not-microservices/) and organically move into micro / [mini](https://thenewstack.io/miniservices-a-realistic-alternative-to-microservices/) services as needed, especially if you are new to the domain.

So, what's a structure that aids this? Here's one that I follow:

![ProjStructure2](/assets/images/VS-NewProj-Ext2.jpg)

Now, the structure *screams* about the different domain objects. If you have separated this properly, it should be straight-forward to splice off one or more of the domains into services on their own. (What happens more frequently is this monolith splitting into two first, then three and so on.).

It is important to avoid doing this optimization earlier than needed. For example, if you want PredictorService to call WeatherService, do so. Do not split off Weather into another service unless needed. (We shall look at those reasons for splitting in a future blog post.) More importantly, do not add the functionality in PredictorService! Before adding each and every single feature, you need to sit and think where it belongs. Which classes modify Weather? Who creates and controls Weather? These are generally limited. But which classes use Weather? Could be many!

Another advantage I've seen with this structure is that, you are completely free to do things in whatever fashion within each of these domains. If you do not need a service, do not create one. If you need multiple "processor"s and "mapper"s, create for the domain as needed. There is no need to generalize for all domain objects. With the earlier structure, there is a tendency to think that all the domains *have* to follow an order, and there might be unnecessary conversions and generalizations done. *`Mapper<T>`* anyone?!

If you think about it, this is a kind of inversion of structure, so to speak. We all know what inversions do to our software quality!

Well, that's it. I hope it has been useful. Let me know your thoughts in the comments section below!
