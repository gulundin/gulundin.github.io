---
layout: post
title: Dependency injection is dynamic scoping in disguise
---
Once upon time programmer used to debate whether programming
languages should be lexically or dynamically scoped. That might sound like gibberish to you
but I think there's an important lesson about modern software engineering
to be learnt here, so allow me to explain.

## Scoping: lexical or dynamic?
The question of lexical or dynamic scoping essentially boils down to this:
what should `greet()` print?
{% highlight javascript %}
const greeting = "Hello!";

function printGreeting() {
    console.log(greeting);
}

function greet() {
    const greeting = "Fuck off!";
    printGreeting();
}
{% endhighlight %}

The name `greeting` is not defined inside the function `printGreeting`, so
we have to look into the outside environment to determine its value.
* In a lexically scoped language we will look into the environment in which `printGreeting`
is defined and see that `greeting` has been assigned the value `"Hello!"`.
* In a dynamically scoped language we will look into the environment in which `printGreeting`
is executing and see that `greeting` has been assigned the value `"Fuck off!"`.

If dynamic scoping seems very weird to you, it likely is partly because you've never seen it before,
and partly because it actually is very weird. Almost all modern programming languages are lexically
scoped[^dynamic-languages]. Dynamic scoping makes it hard to figure what our program actually does, without executing it,
and that's not a quality we want our programs to exhibit.

Dynamic scoping might seem dubious at first glance, but I'm going to argue that one of the biggest
advances in software engineering these past 20 years[^debatable] essentially boils down emulating dynamic scoping
in lexically scoped languages. I'm talking about dependency injection.

## Dependency injection
Let's switch example language to java.

Consider the following piece of hypothetical code which handles an order in a web shop:
{% highlight java %}
public static void acceptOrder(Customer customer, List<OrderItem> items) {
    var totalCost = 0;
    for (var item : items) {
        Supply.reduce(item.id, item.amount);
        totalCost += item.cost * item.amount;
    }
    Bank.chargeMoney(customer, totalCost);
    var mailContents = formulateOrderAcceptedMail(customer, items);
    Mailer.sendMail(customer, mailContents);
}
{% endhighlight %}

Our `acceptOrder` function above has some problems when it comes to testability.
We simply cannot call it without incurring severe side effects on the outside world;
charging money and sending mails are not stuff we want to do in test.

Commonly accepted wisdom tells us that the above problem can be solved by
introducing objects and accepting our dependencies in the constructor:
{% highlight java %}
public final class OrderService {

    private final SupplyService supply;
    private final BankService bank;
    private final MailService mailer;

    public OrderService(
        SupplyService supply,
        BankService bank,
        MailService mailer
    ) {
        this.supply = supply;
        this.bank = bank;
        this.mailer = mailer;
    }

    public void acceptOrder(Customer customer, List<OrderItem> items) {
        var totalCost = 0;
        for (var item : items) {
            supply.reduce(item.id, item.amount);
            totalCost += item.cost * item.amount;
        }
        bank.chargeMoney(customer, totalCost);
        var mailContents = formulateOrderAcceptedMail(customer, items);
        mailer.sendMail(customer, mailContents);
    }
}
{% endhighlight %}

Now two things have happened. 

First of all, our code got much longer. However, 
this is mostly Java's fault and not something inherently wrong with objects 
and dependency injection. 

Second, we can now pass in different implementations 
of our dependencies when executing in test[^testability]. This is very good, but let me rephrase
that in more general terms: the value of certain names are now dependent on the environment
in which we are executing. This should sound very familiar,
dependency injection is just dynamic binding in disguise.

## Dependency injection is not trivial
The problem with objects is that they have to be constructed somewhere. To
build an `OrderService` we also need to build a `SupplyService`, and whatever
that `SupplyService` depend upon and so forth. This can easily turn into a huge
hassle where shared dependencies lead to code duplication and just thinking of
having to add a new dependency to an object makes you groan in annoyance.

One solution to this problem is to write a factory to construct our objects for us.
This can remove all code duplication which makes changing dependencies easy again,
but it still leads to lots of brain dead code that we'd rather not write. In practice many
organizations end up forgoing their factories in favor of more dynamic dependency
injection frameworks that construct objects using reflection, like Spring or Guice.

## Reader monads
One alternative way to solve the testability problem would be to introduce a 
container for all things we want to change when testing:

{% highlight java %}
public final class Env {
    public final SupplyService supply;
    public final BankService bank;
    public final MailerService mailer;

    public Env(
        SupplyService supply,
        BankService bank,
        MailService mailer
    ) {
        this.supply = supply;
        this.bank = bank;
        this.mailer = mailer;
    }
}
{% endhighlight %}

{% highlight java %}
public static void acceptOrder(Env env, Customer customer, List<OrderItem> items) {
    var totalCost = 0;
    for (var item : items) {
        env.supply.reduce(env, item.id, item.amount);
        totalCost += item.cost * item.amount;
    }
    env.bank.chargeMoney(env, customer, totalCost);
    var mailContents = formulateOrderAcceptedMail(customer, items);
    env.mailer.sendMail(env, customer, mailContents);
}
{% endhighlight %}

Now we only had to construct objects for those services that we want to change
when testing. In this small example it didn't have much effect on the code size
but in a large project it could remove a lot of useless objects. If we preferred to, we could also put
regular function references in the `Env`, instead of service objects.
Finally, we could also remove the `Env` entirely and pass around the services one by one, but
this quickly leads to an unmaintainable mess.

Here we are looking up names in an environment which we got passed to us by our caller.
It should be even more obvious than with dependency injection that this is just dynamic scoping in disguise.

The problem with this style of programming is of course that we have to pass the `Env` around everywhere.
However, some programming languages --like Haskell-- has syntactic sugar that lets us implicitly pass
the `Env` around: this is called using a reader monad.

## Explicit dynamic scoping
Some lexically scoped programming languages, like Perl and most Lisps, allow us to explicitly
declare that a certain name should be dynamically scoped. Let's explore how that would be done
in Clojure[^redef].

First let's define a dangerous function that we don't want called in test:
{% highlight clojure %}
(ns myproject.lib)

; This defines send-nukes as dynamically scoped.
(defn ^:dynamic send-nukes
  "Performs some side effect on the outside world. Should not be called in test."
  []
  (println "This was bad. Really bad."))
{% endhighlight %}

Then let's use it in two different ways:
{% highlight clojure %}
(ns myproject.core
  (:require [myproject.lib :as lib]))

(defn do-stuff [x]
  ; Do dangerous stuff
  (lib/send-nukes)
  ; Return a value
  x)

; Normal usage, prints "This was bad. Really bad."
(do-stuff 0)
; Safe testing, prints "No problem"
(binding [lib/send-nukes (fn [] (println "No problem"))]
  (do-stuff 0))
{% endhighlight %}
Take note of how `send-nukes` is namespaced. This ensures that we only swap out the
function we intend to, while others with the same name are left alone -- even if they
too are defined as `^:dynamic`. This is what makes dynamic scoping sane.

In my opinion this style of dependency injection is the simplest one[^gotcha]. 
It's a shame that it isn't available in more languages.

[^dynamic-languages]:
    Some examples of languages that use dynamic scoping by default are APL, Bash,
    Latex and Emacs Lisp.

[^debatable]:
    Some will undoubtedly disagree with me about the importance of dependency injection.

[^testability]:
    There are reasons other than testability for wanting to swap out one implementation
    for another. However, testability is the most common reason and the solutions are the
    same regardless of our reasons.

[^redef]:
    Clojure also has `with-redefs` which lets you temporarily swap out lexically scoped functions.
    However, this is not thread-safe.

[^gotcha]: 
    There is a minor gotcha with threading though. Before spawning a new thread
    you have to wrap its main function using `bound-fn`, or else it will use the default bindings.