Infector manages 2 kind of objects:
  * Shared Objects (they will be injected using a **shared\_ptr**, only 1 instance of each object is created)
  * Unique Objects (they will be injected using a **unique\_ptr**, many instances of each object are created, but every instance is owned only by 1 other object)

Shared Pointers are nice, but if heavily used/misused can cause more troubles (each reference counter need to be allocated, it is easy to have circular references by accidentally passing around a shared\_ptr and then this will cause a memory leak), as far as I know, Infector is the first IoC Container that has some real use for unique\_ptr (this is a bit strange since unique\_ptr is great). Most of troubles are solved by simply using "unique\_ptr", it can't be accidentally passed around (requires a explicit "std::move") it does not require to allocate a reference counter, and so is almost the best way to inject dependencies on something that is not shared.

| **Number of instances ->** | **Single Instance** | **Many Instances** |
|:--------------------------|:-------------------|:------------------|
| Smart pointer type        | std::shared\_ptr   | std::unique\_ptr  |
| Concrete binding          | bindSingleAsNothing | bindAsNothing     |
| Interface binding         | bindSingleAs       | bindAs            |
| Owners                    | many               | one               |
| Created by                | buildSingle< T>    | build< T>         |
| Behaviour                 | create once and re-use same instance during container lifetime | create new instance each time is called |

## Interface Binding ##
Registering classes to be instantiated is done by calling "bind...", this is part of a 2-steps process necessary to do interface binding.
How that works? That's simple don't warry.

If you have "CFoo" and you want to inject it using its interface "IFoo", you have just to Bind "CFoo" as "IFoo" (so later you can forgot about CFoo and request for IFoo only instead).

this is done by:
```
ioc.bindAs<CFoo, IFoo>(); //bind CFoo as IFoo
//later you can do
auto myFoo = ioc.build<IFoo>(); //you are no longer dependent on CFoo
```

**Infector is TYPESAFE (if CFoo is not inheriting from IFoo you will get a compile-time error).**

You are not forced to do interface binding, but since interface binding is usually preferred, if you want to skip that, you have to type a bit more.

this is done by:
```
ioc.bindAsNothing<CFoo>(); //bind CFoo as nothing
//later you can do
auto myFoo = ioc.build<CFoo>(); //you are still dependent on CFoo
```

IMPORTANT!
  * "bindAs" allows to bind only 1 interface
  * Each interface can only have 1 implementation

## Single Instances ##
You can also create Objects that have to be shared, (by the way, Infector++ is not a service locator). In that case only 1 instance will be created for each object.

Since shared objects are.. shared.. it is allowed to bind them to multiple interfaces (Note: each interface is still limited to **1 implementation**). This helps a bit more to specialize shared objects. If you have "FoodFactory" that implements multiple interfaces "ITunaFactory" and "IBeansFactory", you can do:
```
ioc.bindSingleAs<FoodFactory, ITunaFactory, IBeansFactory>(); //bind FoodFactory as ITunaFactory and as IBeansFactory
//later you can do
auto myTunaFactory = ioc.buildSingle<ITunaFactory>(); //if you need only ITunaFactory
auto myBeansFactory = ioc.buildSingle<IBeansFactory>(); //if you need only IBeansFactory
```


You can still bind shared objects to no interfaces:
```
ioc.bindSingleAsNothing<FoodFactory>(); //bind singleInstance of FoodFactory as Nothing
//later you can do
auto myTunaFactory = ioc.buildSingle<FoodFactory>(); //bad you can still call methods of "BeansFactory"
auto myBeansFactory = ioc.buildSingle<FoodFactory>();
```

## Wiring ##
Wiring is the 2nd step necessary to inject dependencies.
by Binding CFoo as IFoo, what I'm really telling to computer is
"When I ask for IFoo, you must create CFoo by calling its constructor, then return me CFoo by casting it to IFoo"
This is a bit of work.. but you still need the constructor, so you have to "wire" it.

Wiring is very simple, basically you have to list all the dependencies, everything else is done by Infector++:
```
//your class
class JustAClass:public virtual IInterfaceOfClass{
public:
    JustAClass(std::unique_ptr<dependency1> p1, //unique
               std::shared_ptr<dependency2> p2, //shared
               std::uinque_ptr<dependency3> p3); //what you want
};

//...

// the wiring
ioc.wire<JustAClass, dependency1, dependency2, dependency3>(); 
               //you get compile time error if no constructor match
```
As you see there's no limit to number of constructor arguments.. As long as they are passed as "unique\_ptr" or "shared\_ptr".
Of course everything must be bound first. In case you miss something you will get an exception. If you want to call Default Constructor this is the way:
```
class AnotherClass{
public:
    AnotherClass(); //default ctor
};

//...

//the wiring
ioc.wire<AnotherClass>(); //by not providing further parameters we are telling the compiler to use default constructor 
                          //(If there is no default constructor you will get compile-time error.)
```

## Building ##

With Infector you can basically create 2 type of objects, **Shared Objects** and **Unique Objects**.

**Shared Objects** are services and managers that are created only once and used in multiple places of the code. Instead of using Singletons or Service Locators, Infector inject them using std::shared\_ptr. This is good for several reasons:

  1. Advantages over Singletons: You can mock instances by providing a different implementation (provided you use them through interfaces). You can test easily your code, and creation of objects is not dependent on objects themselves.
  1. Advantages over Service Locators: Client code can't accidentally use services that are not injected. You will NOT get strange bugs because of unexpected dependencies, every dependency that you use goes through wiring of constructors.

```
//return the shared instance to Audio Manager or create it
auto manager = ioc.buildSingle<IAudioManager>(); //shared_ptr
```

**Unique Objects** are objects that need to be instantiated many times during application lifetime. Infectorpp acts like a basic factory: it provides needed wiring of dependencies also to unique objects allowing faster iteration over your code. I'm considering adding in a future release the explicit injection of "basicFactoryOf< T>" (wich would basically be a wrap around "build< T>".


```
//create a new car
auto mycar = ioc.build<ICar>();  //unique_ptr
```

## Proper way to do things ##

Keep "Binding" "Wiring" and "Building" separed (you cannot wire something that is not bound, and you cannot build something that is not wired)

```

void compositionRoot(Infector::Container & ioc){
    //BINDINGS
    ioc.bindSingleAs<FoodFactory, ITunaFactory, IBeansFactory>();
    ioc.bindSingleAs<MainLoop, IApplication>(); 
    //...

    //WIRING
    ioc.wire<FoodFactory>(); //default ctor for FoodFactory
    ioc.wire<MainLoop, ITunaFactory, IBeansFactory>(); // assume main loop takes some factories as parameter.
    //... 
}

int main(){
    Infector::Container ioc;
    compositionRoot(ioc);
    //BUILDING
    auto app=ioc.buildSingle<IApplication>(); //creates MainLoop, it has some factories
    app->run(); //run the application (can creates objects thanks to factories).

    return 0;
}
```

Composition Root is a common and great concept behind IoC Containers, usually there will be only 1 composition root in most applications. You are of course not limited to only 1. Interaction between multiple containers will be provided in a future release. (I saw it designed in many ways in other languages and still searching for best way to design it in C++)

## Split work, and reduce compile time ##
To reduce compile time you can now split binding and wiring across multiple methods/functions in different compile units (different .cpp files).
Splitting things makes code a little less "centric" (you have to navigate across multiple files instead of only 1 to find bindings), but may be worth to put part of the team working only on certain aspects of the project without touchin the rest (especially considering that they will access needed functionalities through interfaces so they should not bother with implementation)
