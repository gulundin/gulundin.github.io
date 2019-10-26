---
layout: post
title: Dependency rejection does not replace dependency injection
---

## Dependency rejection
Dependency rejection is a simple idea:
* Instead of asking for a service that can produce a value, you should ask for the value directly.
* Instead of asking for a service that can consume a value, you should return the value.

Let's rewrite our original example function in this style:
{% highlight java %}
public static class Result {
    public final long totalCost;
    public final String mailContents;

    public Result(long totalCost, String mailContents) {
        this.totalCost = totalCost;
        this.mailContents = mailContents;
    }
}

public static Result acceptOrder(Customer customer, List<OrderItem> items) {
    var totalCost = 0;
    for (var item : items) {
        totalCost += item.cost * item.amount;
    }
    var mailContents = formulateOrderAcceptedMail(customer, items);
    return new Result(totalCost, mailContents);
}
{% endhighlight %}

Now our function looks very awkward. Why have we entangled the cost calculations
with the formulation of the order-accepted mail we're going to send to the customer?
Clearly those two should have been separate functions.

Dependency rejection pushes interactions with the outside world towards the edge of the
program. This lets you isolate complicated calculations and algorithmic logic from IO, which makes
them easy to test and reuse.

However, in many programs complicated calculations and logic are scarce. Shuffling data
around is the point of the program, not some ugly thing you can sweep under the carpet.
Proper dependency rejection leads to many tiny piece of logic interwoven with lots of IO.
Many of those pieces are going to be so simple that unit testing them in isolation would be pointless because they are obviously
correct (this is a very good thing). Instead, the problem becomes verifying that everything integrates correctly.

Some things that need to integrate are obviously external services. To test that you integrate correctly with them
you will need proper integration tests where servers and databases are spun up. However, many of the things that
need to integrate will also be regular functions and objects in your own codebase. To test that those
integrate with each other we want to be able to replace the external services with test doubles.

At this point you might ask why we need internal integration tests if we are going to run proper integration
tests with external services anyway? The answer is that there's generally branching logic and state dependencies
which makes achieving good coverage of your own codebase much harder than verifying that it's using external services correctly.

Thus dependency rejection does not eliminate the need for dependency injection, but it is still a good design principle.
