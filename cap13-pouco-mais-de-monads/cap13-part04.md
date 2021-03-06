Tasteful stateful computations
==============================

Haskell is a pure language and because of that, our programs are made of functions that can't change any global state or variables, they can only do some computations and return them results. This restriction actually makes it easier to think about our programs, as it frees us from worrying what every variable's value is at some point in time. However, some problems are inherently stateful in that they rely on some state that changes over time. While such problems aren't a problem for Haskell, they can be a bit tedious to model sometimes. That's why Haskell features a thing called the state monad, which makes dealing with stateful problems a breeze while still keeping everything nice and pure.

<a href="input-and-output#randomness">When we were dealing with random numbers</a>, we dealt with functions that took a random generator as a parameter and returned a random number and a new random generator. If we wanted to generate several random numbers, we always had to use the random generator that a previous function returned along with its result. When making a function that takes a [code]StdGen[/code] and tosses a coin three times based on that generator, we had to do this:

It took a generator [code]gen[/code] and then [code]random gen[/code] returned a [code]Bool[/code] value along with a new generator. To throw the second coin, we used the new generator, and so on. In most other languages, we wouldn't have to return a new generator along with a random number. We could just modify the existing one! But since Haskell is pure, we can't do that, so we had to take some state, make a result from it and a new state and then use that new state to generate new results.

You'd think that to avoid manually dealing with stateful computations in this way, we'd have to give up the purity of Haskell. Well, we don't have to, since there exist a special little monad called the state monad which handles all this state business for us and without giving up any of the purity that makes Haskell programming so cool.

So, to help us understand this concept of stateful computations better, let's go ahead and give them a type. We'll say that a stateful computation is a function that takes some state and returns a value along with some new state. That function would have the following type:

[code]s[/code] is the type of the state and [code]a[/code] the result of the stateful computations.

Assignment in most other languages could be thought of as a stateful computation. For instance, when we do [code]x = 5[/code] in an imperative language, it will usually assign the value [code]5[/code] to the variable [code]x[/code] and it will also have the value [code]5[/code] as an expression. If you look at that functionally, you could look at it as a function that takes a state (that is, all the variables that have been assigned previously) and returns a result (in this case [code]5[/code]) and a new state, which would be all the previous variable mappings plus the newly assigned variable.

This stateful computation, a function that takes a state and returns a result and a new state, can be thought of as a value with a context as well. The actual value is the result, whereas the context is that we have to provide some initial state to actually get that result and that apart from getting a result we also get a new state.


<h3>Stacks and stones</h3>

Say we want to model operating a stack. You have a stack of things one on top of another and you can either push stuff on top of that stack or you can take stuff off the top of the stack. When you're putting an item on top of the stack we say that you're pushing it to the stack and when you're taking stuff off the top we say that you're popping it. If you want to something that's at the bottom of the stack, you have to pop everything that's above it.

We'll use a list to represent our stack and the head of the list will be the top of the stack. To help us with our task, we'll make two functions: [code]pop[/code] and [code]push[/code]. [code]pop[/code] will take a stack, pop one item and return that item as the result and also return a new stack, without that item. [code]push[/code] will take an item and a stack and then push that item onto the stack. It will return [code]()[/code] as its result, along with a new stack. Here goes:

We used [code]()[/code] as the result when pushing to the stack because pushing an item onto the stack doesn't have any important result value, its main job is to change the stack. Notice how we just apply the first parameter of [code]push[/code], we get a stateful computation. [code]pop[/code] is already a stateful computation because of its type.

Let's write a small piece of code to simulate a stack using these functions. We'll take a stack, push [code]3[/code] to it and then pop two items, just for kicks. Here it is:

We take a [code]stack[/code] and then we do [code]push 3 stack[/code], which results in a tuple. The first part of the tuple is a [code]()[/code] and the second is a new stack and we call it [code]newStack1[/code]. Then, we pop a number from [code]newStack1[/code], which results in a number [code]a[/code] (which is the [code]3[/code]) that we pushed and a new stack which we call [code]newStack2[/code]. Then, we pop a number off [code]newStack2[/code] and we get a number that's [code]b[/code] and a [code]newStack3[/code]. We return a tuple with that number and that stack. Let's try it out:

Cool, the result is [code]5[/code] and the new stack is [code][8,2,1][/code]. Notice how [code]stackManip[/code] is itself a stateful computation. We've taken a bunch of stateful computations and we've sort of glued them together. Hmm, sounds familiar.

The above code for [code]stackManip[/code] is kind of tedious since we're manually giving the state to every stateful computation and storing it and then giving it to the next one. Wouldn't it be cooler if, instead of giving the stack manually to each function, we could write something like this:

Well, using the state monad will allow us to do exactly this. With it, we will be able to take stateful computations like these and use them without having to manage the state manually.

<h3>The State monad</h3>

The [code]Control.Monad.State[/code] module provides a [code]newtype[/code] that wraps stateful computations. Here's its definition:

A [code]State s a[/code] is a stateful computation that manipulates a state of type [code]s[/code] and has a result of type [code]a[/code].

Now that we've seen what stateful computations are about and how they can even be thought of as values with contexts, let's check out their [code]Monad[/code] instance:

Let's take a gander at [code]return[/code] first. Our aim with [code]return[/code] is to take a value and make a stateful computation that always has that value as its result. That's why we just make a lambda [code]\s -&gt; (x,s)[/code]. We always present [code]x[/code] as the result of the stateful computation and the state is kept unchanged, because [code]return[/code] has to put a value in a minimal context. So [code]return[/code] will make a stateful computation that presents a certain value as the result and keeps the state unchanged.

What about [code]&gt;&gt;=[/code]? Well, the result of feeding a stateful computation to a function with [code]&gt;&gt;=[/code] has to be a stateful computation, right? So we start off with the [code]State[/code] [code]newtype[/code] wrapper and then we type out a lambda. This lambda will be our new stateful computation. But what goes on in it? Well, we somehow have to extract the result value from the first stateful computation. Because we're in a stateful computation right now, we can give the stateful computation [code]h[/code] our current state [code]s[/code], which results in a pair of result and a new state: 

[code](a, newState)[/code]. Every time so far when we were implementing [code]&gt;&gt;=[/code], once we had the extracted the result from the monadic value, we applied the function [code]f[/code] to it to get the new monadic value. In [code]Writer[/code], after doing that and getting the new monadic value, we still had to make sure that the context was taken care of by [code]mappend[/code]ing the old monoid value with the new one. Here, we do [code]f a[/code] and we get a new stateful computation [code]g[/code]. Now that we have a new stateful computation and a new state (goes by the name of [code]newState[/code]) we just apply that stateful computation [code]g[/code] to the [code]newState[/code]. The result is a tuple of final result and final state!

So with [code]&gt;&gt;=[/code], we kind of glue two stateful computations together, only the second one is hidden inside a function that takes the previous one's result. Because [code]pop[/code] and [code]push[/code] are already stateful computations, it's easy to wrap them into a [code]State[/code] wrapper. Watch:

[code]pop[/code] is already a stateful computation and [code]push[/code] takes an [code]Int[/code] and returns a stateful computation. Now we can rewrite our previous example of pushing [code]3[/code] onto the stack and then popping two numbers off like this:

See how we've glued a push and two pops into one stateful computation? When we unwrap it from its [code]newtype[/code] wrapper we get a function to which we can provide some initial state:

We didn't have to bind the second [code]pop[/code] to [code]a[/code] because wedidn't use that [code]a[/code] at all. So we could have written it like this:

Pretty cool. But what if we want to do this: pop one number off the stack and then if that number is [code]5[/code] we just put it back onto the stack and stop but if it isn't [code]5[/code], we push [code]3[/code] and [code]8[/code] back on? Well, here's the code:

This is quite straightforward. Let's run it with an initial stack.

Remember, [code]do[/code] expressions result in monadic values and with the [code]State[/code] monad, a single [code]do[/code] expression is also a stateful function. Because [code]stackManip[/code] and [code]stackStuff[/code] are ordinary stateful computations, we can glue them together to produce further stateful computations.

If the result of [code]stackManip[/code] on the current stack is [code]100[/code], we run [code]stackStuff[/code],otherwise we do nothing. [code]return ()[/code] just keeps the state as it is and does nothing.

The [code]Control.Monad.State[/code] module provides a type class that's called [code]MonadState[/code] and it features two pretty useful functions, namely [code]get[/code] and [code]put[/code]. For [code]State[/code], the [code]get[/code] function is implemented like this:

So it just takes the current state and presents it as the result. The [code]put[/code] function takes some state and makes a stateful function that replaces the current state with it:

So with these, we can see what the current stack is or we can replace it with a whole other stack. Like so:

It's worth examining what the type of [code]&gt;&gt;=[/code] would be if it only worked for [code]State[/code] values:

See how the type of the state [code]s[/code] stays the same but the type of the result can change from [code]a[/code] to [code]b[/code]? This means that we can glue together several stateful computations whose results are of different types but the type of the state has to stay the same. Now why is that? Well, for instance, for [code]Maybe[/code], [code]&gt;&gt;=[/code] has this type:

It makes sense that the monad itself, [code]Maybe[/code], doesn't change. It wouldn't make sense to use [code]&gt;&gt;=[/code] between two different monads. Well, for the state monad, the monad is actually [code]State s[/code], so if that [code]s[/code] was different, we'd be using [code]&gt;&gt;=[/code] between two different monads.


<h3>Randomness and the state monad</h3>

At the beginning of this section, we saw how generating numbers can sometimes be awkward because every random function takes a generator and returns a random number along with a new generator, which must then be used instead of the old one if we want to generate another random number. The state monad makes dealing with this a lot easier.

The [code]random[/code] function from [code]System.Random[/code] has the following type:

Meaning it takes a random generator and produces a random number along with a new generator. We can see that it's a stateful computation, so we can wrap it in the [code]State[/code] [code]newtype[/code] constructor and then use it as a monadic value so that passing of the state gets handled for us:

So now if we want to throw three coins ([code]True[/code] is tails, [code]False[/code] is heads) we just do the following:

[code]threeCoins[/code] is now a stateful computations and after taking an initial random generator, it passes it to the first [code]randomSt[/code], which produces a number and a new generator, which gets passed to the next one and so on. We use [code]return (a,b,c)[/code] to present [code](a,b,c)[/code] as the result without changing the most recent generator. Let's give this a go:

Nice. Doing these sort of things that require some state to be kept in between steps just became much less of a hassle!