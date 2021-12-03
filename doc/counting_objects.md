## Counting Objects in C++

> by [Scott Meyers](http://www.awl.com/cseng/cgi-bin/cdquery.pl?name=smeyers)

Sometimes easy things are easy, but they're still subtle. For example, suppose you have a class `Widget`, and you'd like to have a way to find out at run time how many `Widget` objects exist. An approach that's both easy to implement and that gives the right answer is to create a _static counter_ in `Widget`, increment the counter each time a `Widget` __constructor__ is called, and decrement it whenever the `Widget` __destructor__ is called. You also need a static member function __howMany__ to report how many Widgets currently exist. If Widget did nothing but track how many of its type exist, it would look more or less like this: 

```cpp
    class Widget {
    public:
      Widget() { ++count; }
      Widget(const Widget&) { ++count; }
      ~Widget() { --count; }

      static size_t howMany()
      { return count; }

    private:
      static size_t count;
    };

    // obligatory definition of count. This
    // goes in an implementation file
    size_t Widget::count = 0;
```

This works fine. The only mildly tricky thing is to remember to implement the __copy constructor__, because a compiler-generated copy constructor for Widget wouldn't know to increment count.

If you had to do this only for Widget, you'd be done, but counting objects is something you might want to implement for several classes. Doing the same thing over and over gets tedious, and tedium leads to errors. To forestall such tedium, it would be best to somehow package the above object-counting code so it could be reused in any class that wanted it. The ideal package would:

- Be easy to use — require minimal work on the part of class authors who want to use it. Ideally, they shouldn't have to do more than one thing, that is, more than basically say "I want to count the objects of this type." 
- Be efficient — impose no unnecessary space or time penalties on client classes employing the package. 
- Be _foolproof_ — be next to impossible to accidentally yield a count that is incorrect. (We're not going to worry about malicious clients, ones who deliberately try to mess up the count. In C++, such clients can always find a way to do their dirty deeds.) 

Stop for a moment and think about how you'd implement a **reusable** object-counting package that satisfies the goals above. It's probably harder than you expect. If it were as easy as it seems like it should be, you wouldn't be reading an article about it in this magazine. 

## `new`, `delete`, and `Exceptions`

While you're mulling over your solution to the object-counting problem, allow me to switch to what seems like an unrelated topic. That topic is the relationship between **new** and **delete** when _constructors throw exceptions_. When you ask C++ to dynamically allocate an object, you use a `new` expression, as in: 

```cpp
class ABCD { ... };     // ABCD = "A Big Complex Datatype"
ABCD *p = new ABCD;     // "new ABCD" is a new expression
```

The new expression — whose meaning is built into the language and whose behavior you cannot change — does two things. First, it calls a memory allocation function called **operator new**. That function is responsible for finding enough memory to hold an **ABCD object**. If the call to **operator new** succeeds, the new expression then invokes an ABCD constructor on the memory that operator new found.

But suppose operator new throws a `std::bad_alloc` exception:
```cpp
    ABCD *p = new ABCD;     // suppose a std::bad_alloc is thrown here
```

Exceptions of this type indicate that an attempt to dynamically allocate memory has failed. In the new expression above, there are two functions that might give rise to that _exception_. The first is the invocation of operator new that is supposed to find enough memory to hold an ABCD object. The second is the subsequent invocation of the **ABCD constructor** that is supposed to turn the raw memory into a valid ABCD object.

If the exception came from the call to operator new, no memory was allocated. However, if the call to operator new succeeded and the invocation of the ABCD constructor led to the exception, it is important that the memory allocated by operator new be deallocated. If it's not, the program has a memory leak, because it's not possible for the client — the code requesting creation of the ABCD object — to determine which function gave rise to the exception.

For many years this was a hole in the _draft C++ language specification_, but in March 1995 the C++ Standards committee adopted the rule that if, during a new expression, the invocation of operator new succeeds and the subsequent constructor call throws an exception, the `runtime system` **must** automatically _deallocate the memory_ that operator new allocated. This de-allocation is performed by **operator delete**, the de-allocation analogue of `operator new`. (For details, see the sidebar on **placement new** and **placement delete**.)

It is this relationship between new expressions and operator delete that affects us in our attempt to automate the counting of object instantiations.

## Counting Objects

In all likelihood, your solution to the _object-counting_ problem involved the development of an object-counting class. Your class probably looks remarkably like — perhaps even exactly like — the `Widget` class I showed earlier:

```cpp
    class Counter {                   // see below for a discussion of
    public:                           // why this isn't quite right
      Counter() { ++count; }
      Counter(const Counter&) { ++count; }
      ~Counter() { --count; }

      static size_t howMany()
      { return count; }

    private:
      static size_t count;
    };

    // This still goes in an
    // implementation file
    size_t Counter::count = 0;
```

The idea here is that authors of classes that need to count the number of objects in existence simply use `Counter` to take care of the bookkeeping. There are two obvious ways to do this. One way is to define a `Counter` object as a class data member, like this:

```cpp
    // embed a Counter to count objects
    class Widget {
    public:
      ...                             // all the usual public Widget stuff

      static size_t howMany()
      { return Counter::howMany(); }
    private:
      ...                             // all the usual private Widget stuff
      Counter c;
    };
```

The other way is to declare Counter as a **base class**, like this:

```cpp
    // inherit from Counter to count objects
    class Widget: public Counter {
      ...                             // all the usual public Widget stuff
    private:
      ...                             // all the usual private Widget stuff
    };
```

Both approaches have advantages and disadvantages. But before we examine them, we need to observe that **neither** approach will work in its current form. The problem has to do with the static object count inside Counter. There's only one such object, but we need one for each class using Counter. For example, if we want to count both Widgets and ABCDs, we need two static size_t objects, not one. Making `Counter::count` nonstatic doesn't solve the problem, because we need one counter per class, not one counter per object.

We can get the behavior we want by employing one of the best-known but oddest-named tricks in all of C++: we turn Counter into a __template__, and each class using Counter instantiates the template with itself as the template argument.

Let me say that again. First, Counter becomes a template: 

```cpp
    template<typename T>
    class Counter {
    public:
      Counter() { ++count; }
      Counter(const Counter&) { ++count; }
      ~Counter() { --count; }

      static size_t howMany()
      { return count; }

    private:
      static size_t count;
    };

    template<typename T>
    size_t Counter<T>::count = 0;              // this may now go in a header file
```

The first `Widget` implementation choice then looks like this:

```cpp
    class Widget {                             // embed a Counter to count objects
    public:
      ...

      static size_t howMany()
      {return Counter<Widget>::howMany();}
    private:
      ...
      Counter<Widget> c;
    };
```

And the second choice now looks like:
```cpp
    class Widget: public Counter<Widget> {      // inherit from Counter
    public:                                     // to count objects
      ...
    };
```

Notice how in both cases we replace `Counter` with `Counter<Widget>`. As I said earlier, each class using Counter instantiates the template with itself as the argument.

The tactic of a class instantiating a template for its own use by passing itself as the template argument was first publicized by °Jim Coplien. He showed that it's used in many languages (not just C++) and he called it `"a curiously recurring template pattern"` [Reference 1]. I don't think Jim intended it, but his description of the pattern has pretty much become its name. That's too bad, because pattern names are important, and this one fails to convey information about what it does or how it's used.

The naming of patterns is as much art as anything else, and I'm not very good at it, but I'd probably call this pattern something like "Do It For Me." Basically, each class generated from Counter provides a service (it counts how many objects exist) for the class requesting the `Counter` instantiation. So the class `Counter<Widget>` counts Widgets, and the class `Counter<ABCD>` counts ABCDs.

Now that Counter is a template, both the embedding design and the inheritance design will work, so we're in a position to evaluate their comparative strengths and weaknesses. One of our design criteria was that object-counting functionality should be easy for clients to obtain, and the code above makes clear that the inheritance-based design is easier than the embedding-based design. That's because the former requires only the mentioning of Counter as a base class, whereas the latter requires that a Counter data member be defined and that howMany be re-implemented by clients to invoke Counter's howMany. (An alternative is to omit Widget::howMany and make clients call Counter<Widget>::howMany directly. For the purposes of this article, however, we'll assume we want howMany to be part of the Widget interface.) That's not a lot of additional work (client howManys are simple inline functions), but having to do one thing is easier than having to do two. So let's first turn our attention to the design employing inheritance. ¤

Using Public Inheritance ¤

The design based on inheritance works because C++ guarantees that each time a derived class object is constructed or destroyed, its base class part will be constructed first and destroyed last. Making Counter a base class thus ensures that a Counter constructor or destructor will be called each time a class inheriting from it has an object created or destroyed. ¤

Any time the subject of base classes comes up, however, so does the subject of virtual destructors. Should Counter have one? Well-established principles of object-oriented design for C++ dictate that it should. If it has no virtual destructor, deletion of a derived class object via a base class pointer yields undefined (and typically undesirable) results (Item E14): ¤

    class Widget: public Counter<Widget> { ... };

    Counter<Widget> *pw =              // get base class ptr to
      new Widget;                      // derived class object

    ...
    delete pw;                         // yields undefined results if Counter<Widget>
                                       // lacks a virtual destructor

Such behavior would violate our criterion that our object-counting design be essentially foolproof, because there's nothing unreasonable about the code above. That's a powerful argument for giving Counter a virtual destructor. ¤

Another criterion, however, was maximal efficiency (imposition of no unnecessary speed or space penalty for counting objects), and now we're in trouble. We're in trouble because the presence of a virtual destructor (or any virtual function) in Counter means each object of type Counter (or a class derived from Counter) will contain a (hidden) virtual pointer, and this will increase the size of such objects if they don't already support virtual functions. (For details, see Item M24.) That is, if Widget itself contains no virtual functions, objects of type Widget would increase in size if Widget started inheriting from Counter<Widget>. We don't want that. ¤

The only way to avoid it is to find a way to prevent clients from deleting derived class objects via base class pointers. It seems that a reasonable way to achieve this is to declare operator delete private in Counter: ¤

    template<typename T>
    class Counter {
    public:
      ...
    private:
      void operator delete(void*);
      ...
    };

Now the delete expression won't compile: ¤

    class Widget: public Counter<Widget> { ... };
    Counter<Widget> *pw = new Widget;
    ...
    delete pw;                          // error! Can't call private
    				    // operator delete

Unfortunately — and this is the really interesting part — the new expression shouldn't compile either! ¤

    Counter<Widget> *pw =               // this should not compile because
      new Widget;                       // operator delete is private!

Remember from my earlier discussion of new, delete, and exceptions that C++'s runtime system is responsible for de-allocating memory allocated by operator new if the subsequent constructor invocation fails. Recall also that operator delete is the function called to perform the deallocation. But we've declared operator delete private in Counter, which makes it invalid to create objects on the heap via new! ¤

Yes, this is counterintuitive, and don't be surprised if your compilers don't yet support this rule, but the behavior I've described is correct. Let us therefore consider a different way of preventing the deletion of derived class objects via Counter* pointers. Let us consider making Counter's destructor protected: ¤

    template<typename T>
    class Counter {
    ...
    protected:
      ~Counter();
      ...
    ...
    };

    class Widget: public Counter<Widget> { ... };
    Counter<Widget> *pw = new Widget;
    ...
    delete pw;                     // error! Can't call protected
                                   // destructor

This offers the behavior we want. We can still create Widget objects, and we can still destroy them, as long as we don't do it via Counter* pointers. When we destroy a static Widget or a stack-based Widget, the implicit call to the Counter destructor is attributed to the Widget destructor, and since the Widget destructor is a member function of a derived class, it has access to Counter's protected members. Similarly, when we delete a Widget through a Widget* pointer, the implicit call to the Counter destructor is attributed to the Widget destructor. ¤

Unfortunately, this design smells a bit of Bait and Switch. By having Widget publicly inherit from Counter, we're saying that every Widget object isa Counter object (Item E35). As a result, Widget* pointers are implicitly and automatically converted to Counter* pointers whenever it's necessary. Widget clients can thus be forgiven for expecting that they can delete Widgets via base class pointers; it's just one of the things public inheritance implies. Yet the above design disallows it. We get the behavior we want, yes, and that's worth a lot, but it just doesn't feel right. (The earlier design — the one involving a private operator delete — exhibits the same undesireable behavior.) ¤

Preventing deletion of derived class objects via public base class pointers looks unsatisfactory no matter how we slice it. I say we abandon this approach and turn our attention to using a Counter data member instead. ¤

Using a Data Member ¤

We've already seen that the design based on a Counter data member has one drawback: clients must both define a Counter data member and write an inline version of howMany that calls the Counter's howMany function. That's marginally more work than we'd like to impose on clients, but it's hardly unmanageable. There is another drawback, however. The addition of a Counter data member to a class will often increase the size of objects of that class type. ¤

At first blush, this is hardly a revelation. After all, how surprising is it that adding a data member to a class makes objects of that type bigger? But blush again. Look at the definition of Counter: ¤

    template<typename T>
    class Counter {
    public:
      Counter();
      Counter(const Counter&);
      ~Counter();

      static size_t howMany();
    private:
      static size_t count;
    };

Notice how it has no nonstatic data members. That means each object of type Counter contains nothing. Might we hope that objects of type Counter have size zero? We might, but it would do us no good. C++ is quite clear on this point. All objects have a size of at least one byte, even objects with no nonstatic data members. By definition, sizeof will yield some positive number for each class instantiated from the Counter template. So each client class containing a Counter object will contain more data than it would if it didn't contain the Counter. ¤

(Interestingly, this does not imply that the size of a class without a Counter will necessarily be bigger than the size of the same class containing a Counter. That's because alignment restrictions can enter into the matter. For example, if Widget is a class containing two bytes of data but that's required to be four-byte aligned, each object of type Widget will contain two bytes of padding, and sizeof(Widget) will return 4. If, as is common, compilers satisfy the requirement that no objects have zero size by inserting a char into Counter<Widget>, it's likely that sizeof(Widget) will still yield 4 even if Widget contains a Counter<Widget> object. That object will simply take the place of one of the bytes of padding that Widget already contained. This is not a terribly common scenario, however, and we certainly can't plan on it when designing a way to package object-counting capabilities.) ¤

I'm writing this at the very beginning of the Christmas season. (It is in fact Thanksgiving Day, 1997, which gives you some idea of how I celebrate major holidays...) Already I'm in a Bah Humbug mood. All I want to do is count objects, and I don't want to haul along any extra baggage to do it. There has got to be a way. ¤

Using Private Inheritance ¤

Look again at the inheritance-based code that led to the need to consider a virtual destructor in Counter: ¤

    class Widget: public Counter<Widget>
    { ... };

    Counter<Widget> *pw = new Widget;
    ...
    delete pw;               // yields undefined results if
    			 // Counter lacks a virtual destructor

Earlier we tried to prevent this sequence of operations by preventing the delete expression from compiling, but we discovered that that led to invalid or misleading code. There is something else we can prohibit. We can prohibit the implicit conversion from a Widget* pointer (which is what new returns) to a Counter<Widget>* pointer. In other words, we can prevent inheritance-based pointer conversions. All we have to do is replace the use of public inheritance with private inheritance: ¤

    class Widget:                                 // inheritance is
      private Counter<Widget> { ... };            // now private

    Counter<Widget> *pw =                         // error!  no implicit conversion
      new Widget;                                 // from Widget* to Counter<Widget>*

Furthermore, we're likely to find that the use of Counter as a base class does not increase the size of Widget compared to Widget's stand-alone size. Yes, I know, I just finished telling you that no class has zero size, but — well, that's not really what I said. What I said was that no objects have zero size. The C++ Standard makes clear that the base-class part of a more derived object may have zero size. In fact many compilers implement what has come to be known as the empty base optimization [Reference 2]. Thus, if a Widget contains a Counter, the size of the Widget must increase. The Counter data member is an object in its own right, hence it must have nonzero size. But if Widget inherits from Counter, compilers are allowed to keep Widget the same size it was before. This suggests an interesting rule of thumb for designs where space is tight and empty classes are involved: prefer private inheritance to containment when both will do. ¤

This last design is nearly perfect. It fulfills the efficiency criterion, provided your compilers implement the empty base optimization, because inheriting from Counter adds no per-object data to the inheriting class, and all Counter member functions are inline. It fulfills the foolproof criterion, because count manipulations are handled automatically by Counter member functions, those functions are automatically called by C++, and the use of private inheritance prevents implicit conversions that would allow derived-class objects to be manipulated as if they were base-class objects. (Okay, it's not totally foolproof: Widget's author might foolishly instantiate Counter with a type other than Widget, i.e., Widget could be made to inherit from Counter<Gidget>. I choose to ignore this possibility.) ¤

The design is certainly easy for clients to use, but some may grumble that it could be easier. The use of private inheritance means that howMany will become private in inheriting classes, so such classes must include a using declaration to make howMany public to their clients: ¤

    class Widget: private Counter<Widget> {
    public:
      using Counter<Widget>::howMany;           // make howMany public

      ...                                       // rest of Widget
    };                                          // is unchanged

    class ABCD: private Counter<ABCD> {
    public:
      using Counter<ABCD>::howMany;             // make howMany public

      ...                                       // rest of Widget
    };                                          // is unchanged

For compilers not supporting name spaces, the same thing is accomplished by replacing the using declaration with the older (now deprecated) access declaration: ¤

    class Widget: private Counter<Widget> {
    public:
      Counter<Widget>::howMany;                 // make howMany public

      ...                                       // rest of Widget
    };                                          // is unchanged

    class ABCD: private Counter<ABCD> {
    public:
      Counter<ABCD>::howMany;                   // make howMany public

      ...                                       // rest of ABCD
    };                                          // is unchanged

Hence, clients who want to count objects and who want to make that count available (as part of their class's interface) to their clients must do two things: declare Counter as a base class and make howMany accessible. (Simple variations on this design make it possible for Widget to use Counter<Widget> to count objects without making the count available to Widget clients, not even by calling Counter<Widget>::howMany directly. It is an interesting exercise to come up with one or more such variations.) ¤

The use of inheritance does, however, lead to two conditions that are worth noting. The first is ambiguity. Suppose we want to count Widgets, and we want to make the count available for general use. As shown above, we have Widget inherit from Counter<Widget> and we make howMany public in Widget. Now suppose we have a class SpecialWidget publicly inherit from Widget and we want to offer SpecialWidget clients the same functionality Widget clients enjoy. No problem, we just have SpecialWidget inherit from Counter<SpecialWidget>. ¤

But here is the ambiguity problem. Which howMany should be made available by SpecialWidget, the one it inherits from Widget or the one it inherits from Counter<SpecialWidget>? The one we want, naturally, is the one from Counter<SpecialWidget>, but there's no way to say that without actually writing SpecialWidget::howMany. Fortunately, it's a simple inline function: ¤

    class SpecialWidget: public Widget,
      private Counter<SpecialWidget> {
    public:
      ...
      static size_t howMany()
      { return Counter<SpecialWidget>::howMany(); }
      ...
    };

The second observation about our use of inheritance to count objects is that the value returned from Widget::howMany includes not just the number of Widget objects, it includes also objects of classes derived from Widget. If the only class derived from Widget is SpecialWidget and there are five stand-alone Widget objects and three stand-alone SpecialWidgets, Widget::howMany will return eight. After all, construction of each SpecialWidget also entails construction of the base Widget part. (For a more detailed examination of this topic, see Item M26's treatment of contexts for object construction.) ¤

Summary ¤

The following points are really all you need to remember: ¤

    Automating the counting of objects isn't hard, but it's not completely straightforward, either. Use of the "Do It For Me" pattern (Coplien's "curiously recurring template pattern") makes it possible to generate the correct number of counters. The use of private inheritance makes it possible to offer object-counting capabilities without increasing object sizes. ¤
    When clients have a choice between inheriting from an empty class or containing an object of such a class as a data member, inheritance is preferable, because it allows for more compact objects. ¤
    Because C++ endeavors to avoid memory leaks when construction fails for heap objects, code that requires access to operator new generally requires access to the corresponding operator delete too. ¤
    The Counter class template doesn't care whether you inherit from it or you contain an object of its type. It looks the same regardless. Hence, clients can freely choose inheritance or containment, even using different strategies in different parts of a single application or library. ¤ 

