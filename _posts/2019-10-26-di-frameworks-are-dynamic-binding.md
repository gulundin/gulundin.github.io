---
layout: post
title: Dynamic binding is the simplest form of dependency injection
---
Once upon a time programmers used to debate whether programming
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

Dynamic scoping might seem dubious at first glance, but I'm going to argue that
it can be tamed, and that it can then can be used to accomplish dependency injection
with fewer drawbacks than other methods. First, let's survey those other methods.

## Dependency injection via objects
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
this is mostly Java's fault and not something inherently wrong with objects. 

Second, we can now pass in different implementations 
of our dependencies when executing in test[^testability]. This is very good, but let me rephrase
that in more general terms: the values associated with certain names are now dependent on the environment
in which we are executing. This should sound very familiar.

## Object construction is hard
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

## Env passing
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
This again should sound very familiar.

You might think that this results in an entangled mess where every module depend upon every other module.
However, this can be tamed by having each module declare its own `Env` interface and having the real
`Env` implement them all.

The real problem with this style of programming is that we have to pass the `Env` around everywhere.

## Reader monads
There is a way of implicitly passing around an environment of dependencies that's
called using a reader monad. Here follows an example of how to use it in Haskell:

{% highlight haskell %}
import Control.Monad.Reader

-- The Env holds the services we want to replace
data Env = Env { sendNukes :: IO () }

-- Set up a service we don't want called in test
sendNukesProd :: IO ()
sendNukesProd = do putStr "Very Bad"

-- And a test double for it
sendNukesTest :: IO ()
sendNukesTest = do putStr "No problem"

-- Uses our replaceable dependency to do stuff
doStuff :: String -> ReaderT Env IO ()
doStuff someVal = do
    -- First do some stuff on our own
    liftIO $ putStr someVal
    -- Looks up the implicit env
    env <- ask
    -- Extract and use the particular service we're looking for
    liftIO $ sendNukes env

doMoreStuff :: String -> ReaderT Env IO ()
doMoreStuff someVal = do
    -- Now we don't have to pass the env along when making use of doStuff,
    -- but it still appears in our type signature.
    liftIO $ putStr "("
    doStuff someVal
    liftIO $ putStrLn ")"

main :: IO ()
main = do
    -- At the top level we do have to supply the actual env
    let prodEnv = Env sendNukesProd
    -- prints "(In prod: Very bad)"
    runReaderT (doMoreStuff "In prod: ") prodEnv

    let testEnv = Env sendNukesTest
    -- prints "(In Test: No problem)"
    runReaderT (doMoreStuff "In test: ") testEnv
{% endhighlight %}

By using reader monads we have avoided having to pass around the `Env` explicitly,
however, this came at the price of having to use monad transformers. Honestly,
the reader monad solution does have a lot of nice properties, but I don't think they are worth
having to struggle to get my code to compile. I only have around 6 months of full time experience
with Haskell and I fully expect that this would turn into a non-problem after a few years more.
But do I want to put in that time? Can I get my friends to put in that time?

## Explicit dynamic scoping
Some lexically scoped programming languages, like Perl and most Lisps, allow us to explicitly
declare that a certain name should be dynamically scoped. Let's explore how that would be done
in Clojure[^gotcha][^redef].

First let's define a dangerous function that we don't want called in test:
{% highlight clojure %}
(ns myproject.lib)

; This defines send-nukes as dynamically scoped.
(defn ^:dynamic send-nukes
  "Performs some side effect on the outside world. Should not be called in test."
  []
  (println "This was bad. Really bad."))
{% endhighlight %}

Then let's define a test double:
{% highlight clojure %}
(ns myproject.fakes)

(defn send-nukes-test-double []
  (println "No problem"))
{% endhighlight %}

Finally, let's use the function:
{% highlight clojure %}
(ns myproject.core
  (:require [myproject.lib :as lib]
            [myproject.fakes :as fakes]))

(defn do-stuff [x]
  ; Do dangerous stuff
  (lib/send-nukes)
  ; Return a value
  x)

; Normal usage, prints "This was bad. Really bad."
(do-stuff 0)
; Safe testing, prints "No problem"
(binding [lib/send-nukes fakes/send-nukes-test-double]
  (do-stuff 0))
{% endhighlight %}
Take note of how `send-nukes` is namespaced. This ensures that we only swap out the
function we intend to, while others with the same name are left alone -- even if they
too are defined as `^:dynamic`. Namespaces are the magic sauce that make dynamic scoping behave in a sane way.

Selective dynamic scoping allows us to do exactly what we want: write testable code without harming
our design[^dhh]. It's a shame that this feature it isn't available in more languages.

## Discussion
[Hacker news](https://news.ycombinator.com/item?id=21405962)

[^dynamic-languages]:
    Some examples of languages that use dynamic scoping by default are APL,
    Latex and Emacs Lisp.

[^testability]:
    There are reasons other than testability for wanting to swap out one implementation
    for another. However, testability is the most common reason and the solutions are the
    same regardless of our reasons.

[^gotcha]: 
    There is a minor gotcha with threading though. Before spawning a new thread
    you have to wrap its main function using `bound-fn`, or else it will use the default bindings.

[^redef]:
    Clojure also has `with-redefs` which lets you temporarily swap out lexically scoped functions.
    However, this is not thread-safe.

[^dhh]:
    [Test induced design damage](https://dhh.dk/2014/test-induced-design-damage.html), a very interesting
    rant by the creator of Rails.