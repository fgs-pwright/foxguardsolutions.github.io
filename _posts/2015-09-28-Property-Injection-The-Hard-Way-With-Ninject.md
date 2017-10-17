---
layout: post
title: Property Injection (The Hard Way) With Ninject
image: /images/2015-09-28-Property-Injection-The-Hard-Way-With-Ninject/BatmanSlap-PropertyInjectionByAttribute.jpg
author: Bruce Markham
excerpt: I find the marker attribute for property injection to be a bit distasteful. Maybe we can do better?
---
I was refactoring some PUMP portal code the other day, when I came across (something like) this in our base ASP.NET MVC controller: 

```csharp
    public abstract class BaseController : Controller
    {
        protected override ITempDataProvider CreateTempDataProvider()
        {
            return new AlternateTempDataProvider();
        }
    
        …
    }
```

My goal was to change this into something constructor injected, (so our DI container could start managing it), which would kind of look like this:

```csharp
    public abstract class BaseController : Controller
    {
        private ITempDataProviderFactory _tempDataProviderFactory;
    
        protected BaseController(ITempDataProviderFactory tempDataProviderFactory)
        {
            _tempDataProviderFactory = tempDataProviderFactory;
        }    
    
        protected override ITempDataProvider CreateTempDataProvider()
        {
            return _tempDataProviderFactory.CreateInstance();
        }
    
        …
    }
```

Unfortunately, this would mean modifying all of our derived controllers to include this constructor parameter, for a dependency that they wouldn't directly rely on.
It just wasn't clean enough for our standards, so I decided to put this option on the back-burner.

"Wait… I know, property injection! We use Ninject, so we can just do this:"

```csharp
    public abstract class BaseController : Controller
    {
        [Inject]    
        public ITempDataProviderFactory TempDataProviderFactory { get; set; }
    
        protected override ITempDataProvider CreateTempDataProvider()
        {
            return TempDataProviderFactory.CreateInstance();
        }
    
        …
    }
```

![Cover]({{ site.baseurl }}/images/2015-09-28-Property-Injection-The-Hard-Way-With-Ninject/BatmanSlap-PropertyInjectionByAttribute.jpg "Batman is tired of seeing `[Inject]` everywhere.") 

Unfortunately, I find the marker attribute for property injection to be a bit distasteful.
It couples your code to your dependency injection container's implementation.
This can be alleviated a little by using a custom marker attribute (other than `[Inject]`), but it still couples your code to the concept of a dependency injection container.
(Also, I had forgotten this was an option for Ninject.)
Maybe we can do better?

Usually, when working on the PUMP portal, if we are forced to do property injection (by a 3rd party component), we do it at the place where the binding is defined, in a Ninject module, like this:

```csharp
    public class SomeModule : NinjectModule
    {
        public override void Load()
        {
            Bind<ISomeContract>().To<SomeImplementation>()
                .WithPropertyValue("SomeProperty", ctx => ctx.Kernel.Get<SomePropertyType>());
            
            …
        }
    }
```

This gives us the benefit of having all of our construction logic kept to the scope of our dependency injection container's configuration.
However, you can't do this with a base controller type, because reasons.
(What would you bind your base controller to? Would you want to start manually binding/rebinding your controllers?)
What we need is something that behaves very similarly to the marker attribute for property injection, but based on specifying the property explicitly, like in a Ninject module.

So as I was pondering this, I remembered that Ninject is architected "pretty well" for plugging in custom behavior (just kidding, dependency injection is scary and magical), but I groaned because I thought I was going to have to write a custom `IPlanningStrategy` to traverse Ninject's API delicately to make this happen.
(My previous exposure to this was trying a custom take on a transaction-applying interceptor planning strategy, like [this one](http://stackoverflow.com/a/6391216)).
My imagination was rushing with all of the arcane objects and collections I might have to poke at - but I managed to clench down and open up [`PropertyReflectionStrategy`](https://github.com/ninject/Ninject/blob/ff2e7b9c53f948ce405eaba8c3bebf2d1e48cb00/src/Ninject/Planning/Strategies/PropertyReflectionStrategy.cs) hoping to find something I could copy+paste as starter code for writing my own strategy.

This is what I found, (minus XML docs):

```csharp
    public class PropertyReflectionStrategy : NinjectComponent, IPlanningStrategy
    {
        public ISelector Selector { get; private set; }

        public IInjectorFactory InjectorFactory { get; set; }

        public PropertyReflectionStrategy(ISelector selector, IInjectorFactory injectorFactory)
        {
            Selector = selector;
            InjectorFactory = injectorFactory;
        }

        public void Execute(IPlan plan)
        {
            foreach (PropertyInfo property in Selector.SelectPropertiesForInjection(plan.Type))
                plan.Add(new PropertyInjectionDirective(property, InjectorFactory.Create(property)));
        }
    }
```

It was so incredibly clean, I actually groaned _louder_.
I didn't know what this `ISelector` was, but I dreaded the thought of having to re-implement it.
Clearly it was Beowulf, hidden in the eldritch depths of Ninject.
(As I came to see, my previous experiences with extending Ninject had biased me.)

So I dove deeper, finding Ninject's sole [implementation](https://github.com/ninject/Ninject/blob/f3dbc59afbb1f212608803eb50bc1f7ba0aa0702/src/Ninject/Selection/Selector.cs) of the interface, and my eyes immediately jumped to the method that was used back in `PropertyReflectionStrategy`:

```csharp
		public virtual IEnumerable<PropertyInfo> SelectPropertiesForInjection(Type type)
		{
		    List<PropertyInfo> properties = new List<PropertyInfo>();
		    
		    properties.AddRange(
		        type.GetRuntimeProperties().FilterPublic(Settings.InjectNonPublic)
		            .Select(p => p.GetPropertyFromDeclaredType(p))
		            .Where(p => this.InjectionHeuristics.Any(h => p != null && h.ShouldInject(p))));
		
		
		    if (this.Settings.InjectParentPrivateProperties)
		    {
		        for (Type parentType = type.GetTypeInfo().BaseType; parentType != null; parentType = parentType.GetTypeInfo().BaseType)
		        {
		            properties.AddRange(this.GetPrivateProperties(type.GetTypeInfo().BaseType));
		        }
		    }
		    
		    return properties;
		}
```

When I realized that Ninject was literally searching all properties, of all injected things, and looping through a quite possibly dynamic list of "heuristics" to filter said properties, my sigh of relief was audible. 
My random jaunt through the code actually led me to Ninject's implementation of the exact behavior I was trying to extend. Clearly `this.InjectionHeuristics`, a collection of `IInjectHeuristic` was what mattered. 
Once again, Ninject had only one [implementation](https://github.com/ninject/Ninject/blob/cc00946b1484db3c8d1c80c0c44e91beabc6b5be/src/Ninject/Selection/Heuristics/StandardInjectionHeuristic.cs), so I opened it up: 

```csharp
    public class StandardInjectionHeuristic : NinjectComponent, IInjectionHeuristic
    {
        public virtual bool ShouldInject(MemberInfo member)
        {
            var propertyInfo = member as PropertyInfo;

            if (propertyInfo != null)
            {
                bool injectNonPublic = Settings.InjectNonPublic;

                var setMethod = propertyInfo.SetMethod;
                if (setMethod != null && !injectNonPublic)
                {
                    if (!setMethod.IsPublic)
                        setMethod = null;
                }

                return member.HasAttribute(Settings.InjectAttribute) && setMethod != null;
            }

            return member.HasAttribute(Settings.InjectAttribute);
        }
    }
```

Seeing the usage of something called `Settings.InjectAttribute`, at this point I remembered that the marker attribute one uses for property injection _is configurable_ in Ninject.
I briefly reconsidered using marker attributes, (but using a custom one), and was still leaning toward marker attributes not being a good fit.
This would have meant introducing a low level concept to our architecture that would be inherently related to dependency injection, when right now our dependency injection sits at a high level, (hovering above like a puppeteer).

As my mind hovered around the thought of giving in to the marker attribute, something caught my eye: this thing derives from `NinjectComponent`.
I had seen parts of Ninject do this in the past, and I knew that Ninject managed these "components" in a way not unlike dependency injection.
I had a hunch - that I could write one of _these_ heuristic _things_, instead of a strategy, and register it with `IKernel.Components` directly, thereby introducing my _own_ injection heuristic.

I certainly couldn't write one that only cared about my `TempDataProviderFactory` property; I needed something more reusable, and wound up with this:

```csharp
    public class WellKnownMemberInjectionHeuristic : NinjectComponent, IInjectionHeuristic
    {
        private readonly List<MemberInfo> _wellKnownMembers = new List<MemberInfo>();

        public void Add(MemberInfo member)
        {
            if (!_wellKnownMembers.Contains(member))
                _wellKnownMembers.Add(member);
        }

        public void Remove(MemberInfo member)
        {
            if (_wellKnownMembers.Contains(member))
                _wellKnownMembers.Remove(member);
        }

        #region Implementation of IInjectionHeuristic

        public bool ShouldInject(MemberInfo member)
        {
            return _wellKnownMembers.Contains(member);
        }

        #endregion
    }
```

Then I just needed some helper methods to keep my colleagues from having to handle this thing raw, and keep my consuming Ninject module clean:

```csharp
    public static class KernelExtensions
    {
        public static void RegisterWellKnownMemberForInjection(this IKernel kernel, MemberInfo member)
        {
            var heuristic = kernel.GetOrAddWellKnownMemberInjectionHeuristic();
            heuristic.Add(member);
        }

        private static WellKnownMemberInjectionHeuristic GetOrAddWellKnownMemberInjectionHeuristic(this IKernel kernel)
        {
            var result = GetWellKnownMemberInjectionHeuristic(kernel);

            if (result == null)
            {
                kernel.Components.Add<IInjectionHeuristic, WellKnownMemberInjectionHeuristic>();
                result = GetWellKnownMemberInjectionHeuristic(kernel);
            }

            return result;
        }

        private static WellKnownMemberInjectionHeuristic GetWellKnownMemberInjectionHeuristic(IKernel kernel)
        {
            return kernel.Components.GetAll<IInjectionHeuristic>().OfType<WellKnownMemberInjectionHeuristic>().SingleOrDefault();
        }
    }
```

Finally, it was ready for use:

```csharp
    public class SomeModule : NinjectModule
    {
        public override void Load()
        {
            Kernel.RegisterWellKnownMemberForInjection(typeof(BaseController).GetProperty("TempDataProviderFactory"));
            
            …
        }
    }
```

And it actually works - with _far_ less code than what I initially surmised.

I wouldn't regard this as "the" optimal solution for everyone - but we are devotees of clean code here at FoxGuard Solutions.
If nothing else though, I think that this is a good lesson that sometimes, it is worth the time to browse the implementation of a 3rd party library in order to find your preferred way of bending it to your will.

I leave it as an exercise to the reader to implement an overload of `RegisterWellKnownMemberForInjection` that takes a lambda designating the target member, which would make this more resilient against refactors.

And kudos to the Ninject team for making an API that wasn't worth all the groans I started out with.