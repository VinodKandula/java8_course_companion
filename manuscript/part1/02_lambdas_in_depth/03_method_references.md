# λ In-depth

## Links

[Article I refer to in my blog post](http://java.amitph.com/2012/08/at-first-sight-with-closures-in-java.html)
[Nice write up](http://blog.informatech.cr/2013/03/15/java-lambda-expressions-vs-method-references/)
[Another write up](http://javapapers.com/core-java/java-method-reference/)

## Method References

[slide divider] Ok, lets have a look at method references.

I mentioned earlier that method references are kind of like shortcuts to lambdas. They're a compact and convenient way to point to a method and _allow_ that method to be used anywhere a lambda would be used.

When you create a lambda, you create an anonymous function and supply the method body. When you use a method reference as a lambda, it's actually pointing to a _named_ method that already exists; it already has a body.

You can think of them as _transforming_ a regular method into a functional interface.

The basic syntax looks like this.

    Class::method

or, a more concrete example

    String::valueOf


[add call out arrows in ScreenFlow]

The part preceding the double colon is the target reference and after, the method name. So, in this case, we're targeting the `String` class and looking for a method called `valueOf`; we're referring to the static method on `String`.

    public static String valueOf(Object obj) { ... }


The double colon is called the delimiter, when we use it, we're not invoking the method, just _referencing_ it. So remember not to add brackets on the end.

    String::valueOf(); // <-- errors




That's neat but once we have a method reference, what can we do it?

You can't invoke them directly, they can only be used in-lieu of a lambda. So anywhere a lambda is used, you can use a method reference.

[demo]

So, this statement on it's own won't compile.

    public static void main(String... args) {
        String::valueOf;
    }

That's because the method reference can't be transformed into a lambda as there's no context for the compiler to infer what type of lambda to create.

_We_ happen to know that this reference is equivalent to

    (x) -> String.valueOf(x)

but the compiler doesn't know that. Yet. It _can_ tell some things though. It knows, that as a lambda, the return value should be of type `String` because all methods called `valueOf` in `String` return a string. But it has no idea what to supply as a argument. We need to give it a little help and give it some more context.

So we'll create a functional interface called `Conversion` that takes an integer and returns a `String`. This is going to be the target type of our lambda.

    @FunctionalInterface
    interface Conversion {
        String convert(Integer number);
    }

Next, we need to create a scenario where we use this as a lambda. So we create a little method to take in a functional interface and apply an interger it.

    public static String convert(Integer number, Conversion function) {  // <-- notice the functional interface argument
        return function.convert(number);
    }

Now. Here's the thing. We've just given the compiler enough information to transform a method reference into the equivalent lambda.

When we call `convert` method, we can do so with a lambda.

    convert(100, (number) -> String.valueOf(number));

And we can literally replace the lambda with a reference to the `valueOf` method. The compiler now knows we need a lambda that returns a `String` and takes an integer. It now knows that the `valueOf` method "fits" and can substitute the integer argument.

    convert(100, String::valueOf);


Another way to give the compiler the information it needs is just to assign the reference to a type.

    Conversion b = (number) -> String.valueOf(number);

and as a method reference;

    Conversion a = String::valueOf;


the "shapes" fit so it can be assigned.

Interestingly, we can assign the same lambda to any interface that requires the same "shape". For example, if we have another functional interface with the same "shape",

Here `Example` returns a `String` and takes an `Object` so it has the same signature shape as `valueOf`.

    interface Example {
        String theNameIsUnimportant(Object object);
    }

we can still assign the method reference (or lambda) to it.

    Example a = String::valueOf;


[end-demo]



There are four types of method reference; constructor references, static method references and two types of instance method references.

Now, these last two... the first is a method reference of a particular object and the second is a method reference of an _arbitrary_ object _but_ of a particular type.

It's a bit confusing as to the difference but it's about how you want to use the method and if you _have_ the instance ahead of time or not. Hopefully, everything will become clear as we go on.


Firstly then, lets have a look at constrictor references.


#### Constructor reference

The basic syntax looks like this,

    String::new

A _target_ type following by the double colon, followed by the `new` keyword. It's going to create a lambda that will call the zero argument constructor of the `String` class.

It's equivalent to this lambda

    () -> new String()


Remember that method references never have the parentheses; they're not invoking methods, just referencing one. This example is referring to the constructor of the `String` class but _not_ instantiating a `String`.


[demo - usage of constructor references]

Lets have a look at how we might actually use a constructor reference.

If we create a list of objects we might want to populate that list say ten items. So we could create a loop and add a new object ten times.

	public void usage() {
		List<Object> list = new ArrayList<>();
		for (int i = 0; i < 10; i++) {
			list.add(new Object());
		}
	}

but if we want to be able to reuse that initialising function, we could extract the code to a new method called `initialise` and then use a factory to create the object.

    public void usage() {
		List<Object> list = new ArrayList<>();
		initialise(list, Object::new);
	}

	private void initialise(List<Object> list, Factory<Object> factory) {
		for (int i = 0; i < 10; i++) {
			list.add(factory.create());
		}
	}

So we pass in a parameter called `Factory` here.

`Factory` here is actually a functional interface with a method called `create` that returns some object. We can then add that object to the list.

Now when we go back to where we call the initialise function. Because it's a functional interface, we can use a lambda to implement the factory. Or we could swap in a constructor reference.

There's a couple of other things we could do here. If we add some generics to the `initialise` method we can reuse it when initialising lists of any type. For example, we can go back and change the type of the list to be `String` and use a constructor reference to initialise it.

[end-demo]


[demo - multiple argument constructors]

We've seen how it works for zero argument constructors, but what about the case when classes have multiple argument constructors?

When there are multiple constructors, you use the same syntax but the compiler figures out which constructor would be the best match. It does this based on the _target_ type and inferring functional interfaces that it can use to create that type.

Let's take the example of a `Person` class, it looks like this and you can see the constructor takes a bunch of arguments.

    class Person {
        public Person(String forename, String surname, LocalDate birthday, Sex gender, String emailAddress, int age) {
            // ...
        }
    }

Going back to our example from earlier and looking at the general purpose `initialise` method, we could use a lambda like this

    initialise(people, () -> new Person(forename, surname, birthday, gender, email, age));


where I use some constants defined elsewhere for the constructor arguments.

but to be able to use a constructor reference, we actually need a lambda _with variable arguments_ and that looks like this

    (a, b, c, d, e, f) -> new Person(a, b, c, d, e, f);


but this doesn't translate to a constructor reference directly. If we were to try and use

    Person::new;

it won't compile as it doesn't know anything about the parameters. If you try and compile it, the error says you've created an invalid constructor reference that cannot be applied to the given types; it found no arguments. [compile and show error]

I'll just get ride of this for a moment.

Instead, we have to introduce some indirection to give the compiler enough information to find an appropriate constructor.

We can create something that can be used as a functional interface _and_ has the right types to slot into the appropriate constructor;

I'll just comment this out and go ahead and create a new functional interface called `PersonFactory`.

[comment out previous failing bit]

    @FunctionalInterface
    interface PersonFactory {
        Person create(String forename, String surname, LocalDate birthday, Sex gender, String emailAddress, int age);
    }


Here, the arguments from `PersonFactory` match the available constructor on `Person`. Magically, this means we can go back and use it with a constructor reference of `Person`.

    public void example() {
        List<Person> list = new ArrayList<>();
        // initialise(list, Person::new);
        PersonFactory factory = Person::new;
    }

Notice I'm using the constructor reference from Person.

The thing to note here is that a constructor reference can be assigned to a target functional interface even though we don't yet know the arguments.

It probably seems a strange that the type of the method reference is `PersonFactory` and not `Person`. This extra target type information helps the compiler to know it has to go via `PersonFactory` to create a `Person`. With this extra hint, the compiler is able to create a lambda based on the factory interface that will _later_ create a `Person`.

Writing it out long hand, the compiler would generate this.

    public void example() {
        PersonFactory factory = (a, b, c, d, e, f) -> new Person(a, b, c, d, e, f);
    }

which could be used later like this;

    public void example() {
        PersonFactory factory = (a, b, c, d, e, f) -> new Person(a, b, c, d, e, f);
        Person person = factory.create(forename, surname, birthday, gender, email, age);
    }

Fortunately, the compiler can do this for us once we've introduced the indirection.

It understands the target type to use is `PersonFactory` and it understands that it's single abstract method can be used in lieu of a constructor. It's kind of like a two step process, firstly, to work out that the abstract method has the same argument list as a constructor and that it returns the right type, then apply it with colon colon `new` syntax.

To finish off the example, we need to tweak our initialise method though...

Add the type information.

Add the parameters.

and call the factory.

    private void initialise(List<Person> list, PersonFactory factory, String forename, String surname, LocalDate birthday, Sex gender, String emailAddress, int age) {
        for (int i = 0; i < 10; i++) {
            list.add(factory.create(forename, surname, birthday, gender, emailAddress, age));
        }
    }

and then we can use it like this

        initialise2(people, Person::new, a, b, c, d, e, f);


So there we go. Calling multiple argument constructors via constructor references.

[end-demo]


#### Static method reference

A method reference can point directly to a static method.

For example,

    String::valueOf

This time, the left hand side refers to the type where a static method, in this case `valueOf` can be found.

It's equivalent to this lambda

    x -> String.valueOf(x))


A more extended example would be where we sort a collection using a reference to a static method on the class `Comparators`.

    Collections.sort(Arrays.asList(1, 2, 54, 12, 14), Comparators::ascending);

    // equivalent to
    Collections.sort(Arrays.asList(1, 2, 54, 12, 14), (a, b) -> Comparators.ascending(a, b));

where, the static method `ascending` might be defined like this.

    public static class Comparators {
        public static Integer ascending(Integer first, Integer second) {
            return first.compareTo(second);
        }
    }



#### Instance method reference of particular object (in this case, a closure)

Here's an example of an instance method reference of a specific instance.

    x::toString

The `x` is a specific instance that we want to get at. It's lambda equivalent looks like this;

    () -> x.toString()


The ability to reference the method of a specific instance also gives us a convenient way to convert between different functional interface types. For example;

    Callable<String> c = () -> "Hello";

`Callable`'s functional method is `call`, when it's invoked the lambda will return "Hello".

If we have another functional interface, `Factory`, we can convert the `Callable` using a method reference.

    Factory<String> f = c::call;


We could have just re-created the lambda but this trick is a useful way to get reuse out of predefined lambdas. Assign them to variables and reuse them to avoid duplication.


[demo - usage of instance method reference #1 (closure)]

Here's an example of it in use.

    public void example() {
        String x = "hello";
        function(x::toString);
    }

This is an example where the method reference is using a closure. It creates a lambda that will call the `toString` method on the instance `x`.

The signature of `function` looks like this

    public static String function(Supplier<String> supplier) {
        return supplier.get();
    }

The `Supplier` interface must provide a string value (the `get` call) and the only way it can do that is if it's been supplied to it on construction. It's equivalent to

    public void example() {
        String x = "";
        function(() -> x.toString());
    }

Notice here that the lambda has no arguments (it uses the 'hamburger' [emphasis this with a screenflow callout]). This shows that the value of `x` isn't available in the lambda's local scope and so can only be available from outside it's scope. It's a closure because must close over `x`.

If you're interested in seeing the long hand, anonymous class equivalent, it'll look like this. Notice again how `x` must be passed in.

    public void example() {
        String x = "";
        function(new Supplier<String>() {
            @Override
            public String get() {
                return x.toString(); // <- closes over 'x'
            }
        });
    }


All three of these are equivalent. Compare this to the lambda variation of an instance method reference where it doesn't have it's argument explicitly passed in from an outside scope.

[end-demo]


#### Instance method reference of a arbitrary object who's instance is supplied later (lambda)

The last case is for a method reference that points to an arbitrary object referred to by its type.

    Object::toString

So in this case, although it looks like the left hand side is pointing to a class, it's actually pointing to an instance. `toString` is an instance method on `Object`, not a static method. The reason why you might not use the regular instance method syntax is because you may not yet have an instance to refer to.

So before, when we call `x` colon colon `toString`, we know the value of `x`. There are some situations where you don't have a value of `x` and in these cases, you can still pass around a reference to the method but supply a value later using this syntax.

For example, the lambda equivalent doesn't have a bound value for `x`.

    (x) -> x.toString()

There difference between the two types of instance method reference is basically academic. Sometimes, you'll need to pass something in, other times, the usage of the lambda will supply it for you.


[demo - usage of instance method reference #2 (lambda)]

The example is similar to the regular method reference; it calls the `toString` method of a string only this time, the string is supplied to the function that's making use of the lambda and not passed in from an outside scope.

    public void lambdaExample() {
        function("value", String::toString);
    }

The `String` part looks like it's referring to a class but it's actually referencing an instance. It's confusing, I know but to see things more clearly, we need to see the function that's making use of the lambda. It looks like this.

    public static String function(String value, Function<String, String> function) {
        return function.apply(value);
    }

So, the string value is passed directly to the function, it would look like this as a fully qualified lambda.

    public void lambdaExample() {
        function("value", x -> x.toString());
    }

which Java can shortcut to look like `String::toString`; it's saying "supply the object instance" at runtime.

If you expand it fully to an anonymous interface, it looks like this. The `x` parameter is made available and not closed over. Hence it being a lambda rather than a closure.

    public void lambdaExample() {
        function("value", new Function<String, String>() {
          @Override
          public String apply(String x) {   // <- takes the argument as a parameter, doesn't need to close over it
            return x.toString();
          }
        });
    }

[end-demo]



#### Summary

Let's have a look at the summary of the syntax for the four types of method reference.

Note that the syntax for a static method reference looks very similar to a reference to an instance method of a class. The compiler determines which to use by going through each applicable static method and each applicable instance method. If it were to find a match for both, the result would be a compiler error.


You can think of the whole thing as a transformation from a method reference to a lambda. The compiler provides the _transformation_ function that takes a method reference and target typing and can derive a lambda.

[transformation ( method\,reference , target\,typing ) \rightarrow \lambda](http://www.codecogs.com/latex/eqneditor.php)



#### Extended Example

http://java.amitph.com/2012/08/at-first-sight-with-closures-in-java.html