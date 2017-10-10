---
layout: post
title: "All I'd Wanted to Know about Ruby's Object Model Starting Out...and MOAR"
date: 2017-03-12
category:
  - tech
tags:
  - ruby
  - object oriented programming
  - internals
  - smalltalk
  - python
---

I aim to speak at [Red Dot Ruby Conf 2017](https://www.reddotrubyconf.com/)

## Abstract

One of the most fun yet confusing things about Ruby is its object model. It's something that can seem highly cryptic to beginners, and perhaps not even that well understood by experienced Rubyists. Not too far into my Ruby journey, I began to get a taste of metaprogramming, but even as I learnt and grew more familiar with common idioms, I always had a nagging feeling that my underlying mental model didn't quite cut it, so I decided to iron it out. The more I read, the more intrigued I became, until I ended up diving into the CRuby source itself! Here's the story of what I learnt about Ruby's object model, told in a way that's both digestible for beginner/intermediates and also insightful for the more experienced. It will also be the story of my journey from feeling like a newbie lacking confidence in my ability to understand something as complex as CRuby, to taking the plunge and learning how to fearlessly read the source!

## Outline

1. Motivations
When we pick up a new language, we tend to learn things by pattern recognition. As we interact more and more with the language we accumulate a certain vocabulary of idioms that we learn to recognize and eventually use. This was the case for me when I started seeing code with metaprogramming and using a bit of it myself - except there always comes a point when simple pattern recognition no longer cuts it, and there's a need to flesh out your underlying mental model of the language. That point came for me from a mounting frustration of feeling like I was not really 'groking it' and also from just a natural curiosity to dig a little deeper. I want to share this experience and talk about how, as beginner/intermediate-level programmers, even as our focus is on just learning the basic mechanics and figuring out how to get stuff done, it's important to sometimes take the time to do deep and deliberate learning. Hopefully I'll be able to share some tips which may help and encourage other beginner/intermediate-level programmers to go forth and dig deeper!

2. Eigenclass, Singleton class, Metaclass, Metametaclass...wha?

All I'd Wanted to Know Starting Out:
We will go through some common idioms which make use of singleton methods and classes (and which potentially confuse beginners, such as `class_eval` and `instance_eval` or the cryptic-at-first-glance `class << self`), and see how much we can figure out about the class hierarchy from Ruby alone (from introspection using methods like `class` and `superclass` and `ancestors`). I will then present a conceptual overview of the grand, infinitely cycling Ruby class-object hierarchy (told as an illustrated creation myth! As grand things should be :P), which will help make sense of what exactly singleton classes and methods (and class methods) are, and how they affect method dispatch and inheritance.

And Moooar!!!:
We will peek under the hood at the C structs that underlie Ruby objects and classes to see how the class hierarchy actually plays out (presented in a way that hopefully even programmers unfamiliar with C will be able to understand; also illustrated!). We'll see where Ruby's class hierarchy is bootstrapped, where the built-in classes are initialized, what happens when we create singleton methods (and classes), and iron out the terminology a little. (Metaclasses are a kind of singleton class. But they refer specifically to singleton classes of class objects, and they are handled differently from normal singleton classes in the underlying C code)

3. Mixin Modules and Include Classes

All I'd Wanted to Know Starting Out:
Again, we will look at some common idioms using Ruby modules (like class extensions, or an alternative to the `alias_method` pattern), and see how mixing in modules (whether through `include`, `prepend` or `extend`) affects our class hierarchy as seen from Ruby (again, using `ancestors`). We'll also look at gotchas, like: what happens when you include M1 in class A, then include M2 in M1?

And Moooar!!!:
Again, let's see how it all works under the hood! We'll see how Ruby injects modules into our class hierarchy using hidden include classes, and how this gives us a clearer picture of the differences between `include`, `prepend` and `extend`. With this, we can finally return to the grand overall picture of Ruby's object model and wrap up our creation myth.

AND MOOARR!!!
4. (Tentative section; this will be contingent on how well I can comfortably fit the above into 25min) OO Elsewhere?
Finally, it'd be fun to check out other OO languages and see how their object models compare. We will take a brief look at the object models of Python and Smalltalk to get a bigger picture of things, and maybe conclude that there isn't really anything as fun as Ruby's grand, never-ending inheritance chain, and metametametametametametametametametametametametametametametameta...................classes :)

## Script

[slide 1]
Hello everybody! I'm Jun Qi, I'm a final year student at the Singapore University of Technology and Design, and I was introduced to the world of Ruby, and web development through Rails, about one and half years ago.

[slide 2]
So by way of a very short self-introduction, this is my exceedingly silly github/slack handle, and
[slide 3]
this may or may not be my typical facial expression IRL

[slide 4]
So my talk today is called "All I'd Wanted to Know About Ruby's Object Model Starting Out"
[slide 5]
...And Moooarrr!!!!

[slide 6](waii)
But before diving into the topic proper, I want to talk a little bit about the motivations behind this. So why this talk, and why this topic.

[slide 7](yaee tickets)
I mean, if you ask me why this talk the first thing I'm going to say to you is "FREE RDRC TICKETS YAYEEEY!!!!". But I also want to approach this as someone who still has a lot to learn, about the language, and about programming in general.

[slide 10]
So, I have this question: "What is it like to be a beginner learning Ruby?" And I'm going to give a very quick outline of my personal experience.

(1:15)

[slide 11](doge; such ruby, so beauty)
So at the beginning everything feels really beautiful and great, right. The language is clean, clutter-free, expressive,
[slide 12](while life, stuff.sample)
sometimes even reads like English. Blocks, procs and enumerators are slightly strange, but you get the hang of it after a while.
[slide 13](class DangerousThings, Cupcake subclass, include Batter, Icing)
The mechanics of the basic object-oriented paradigm are not too hard to pick up, and pretty soon you're fairly comfortable writing your own classes, subclassing them, and mixing in modules.
[slide 14](Ruby hugggs; we are friends nao!; grows tentacles)
And so you're in love. But if you hang around with Ruby long enough, you start seeing fun stuff like this (class_eval), and this(`class << self`), and you hear stuff like singleton methods, metaclass, extending modules instead of including them, and a lot of head-scratching ensues.

[slide 15](all the jargons; singleton method, receiver, class_eval, instance_eval, class << self, include, extend)
And maybe you keep reading and re-reading Ruby docs and various blogposts, but you still keep forgetting the difference between `class_eval` and `instance_eval`, or you still don't quite understand what that `self.included` hook does down there, and you realize you should be able to do better.

[slide 16]
You should be able to build a mental model that has the explanatory power of unifying all these strange and disparate bits of Ruby.

[slide 17](brain with words)
So that's what I set out to do, and that's basically All I'd Ever Wanted To Know About Ruby's Object Model Starting Out.

[slide 18]
But because I also have slight completionist tendencies, I actually decided to dive into the CRuby source itself, and that's where the MOAAAR comes in.

(3:00)

~~[slide 19]
So I'm just going to do a quick aside here, for the less experienced amongst us, about reading source code. I think other than working on projects or doing programming kata, reading source code is really one of the biggest things that can help you grow your competency as a programmer.
[slide 20]
And like most people I was at the beginning really apprehensive about reading source code; I treated it like magic incantations from programmer wizards living on a mountain somewhere. But one day I just tried to do it. I dove into bits of Rails source. And then some other gems I'd used before. And it gradually got easier with time, although it's still challenging.
[slide 21]
And so, while this could be a whole talk in and of itself, I really encourage you to do some deliberate digging into source code, once in a while, and other than basic knowledge of the language you really only need two things: 1. A good code navigation environment; and 2. Motivation through some form of commitment; so, maybe something like a study group, or if you're a brazen idiot like me, signing up to do a talk in front of 300 people.~~

(3:30)

------------------------------

[slide 22]
(So, now you know why this talk. And) So without further ado, All I'd Wanted to Know About Ruby's Object Model Starting Out, as a story.

[slide 23]
In the beginning, there was C[haos]. But soon from that primordial soup of procedural code there sprung forth typedefs and macros and these gradually coalesced into the finest of all rubies.

[slide 24]
And Ruby said, "Let Us make objects, but not in Our image nor likeness, for I am a jealous Ruby and I want to be the shiniest Ruby there is."

[slide 25]
And so was wrought the plainest of all objects, BasicObject. And BasicObject knew Kernel, and unto them was conceived and born Object; and Object begot Module; and Module begot Class.

[slide 26]
Now Ruby had given her creations dominion across the land, and so Object set forth and begot many other classes (prolific and unique form of parthogenesis), and soon enough these classes had materialized their multitudes of concrete instances that spread across the code of the very many Ruby programmers in this world.

[slide 27]
And this was the world that all the objects knew, and it was great and happy.

[slide 28]
But Ruby had also furnished her creations with a very special power, which was soon to precipitate a great existential crisis. It was the power of introspection.

[slide 29] (shiba dog)
And so it was that one of the first objects, doge, began to ask: "What am I?"

[slide 30]
And he discovered that he could simply call the method `class`, and the answer was as plain as day, and the existential crisis passed.

[slide 31] (show class hierarchy, fading in the labels)
But even as doge was content with the answer to that pressing question, it was not long before Dog thought to ask the same. And so Dog called the method `class`, and discovered that what he was, was a Class. But he discovered that he could also call `superclass`, and remembered that his parent was Object. And Dog knew what he was, and where he came from, and so he was content.

[slide 32] (fade in arrows and labels)
Yet this was not the end. Soon even the most ancient of objects began to question their own existence. BasicObject, Object, Module, Class, all of them asked, "What am I?" And it turned out that all of them were Classes, and they remembered whom begot whom. Kernel asked herself the same question, and discovered that she was a Module, and remembered that she had no parent to speak of. And so this was the world that all the objects knew, and some thought the arrows were getting a little bit messed up, but they lived with it.

(4:00)

[slide 33]
Alas, the first wave of the crisis was over, but now doge began agonizing again. He looked at Dog and complained, "You say that as a Dog, I should only be able to bark, and wag my tail, and be stroked on my belly. I know that I am different from the other dog instances; I weigh different and so on. But truly I want more individuality than that! I want the means and the methods for manifesting my singly doge-nature. Not just to bark but to go, 'So Doge!' and 'Such Wow!'" And Dog looked at him and shook his head, for he knew not what he could do.

[slide 34]
But in the night, doge was visited by Ruby herself, who was full of sympathy for the poor animal, and she spoke: "Thusly I do grant you the power to be the doge you want to be. No longer shall you be a Dog, but you shall be a Singleton Doge. Yet to keep the peace, I cannot make this obvious, for if I do Dog will be jealous." And so was created a new class, a singleton class of doge, and it was such that the answer to doge's "What am I?" should have been this singleton class, but Ruby made all the objects half-blind to this. So that if doge called the method `class`, he knew himself still as a Dog, and if he called the method `singleton_class`, he knew where it was that he got his uniquely doge abilities.(next slide with klass and super) Only Ruby knew that, deep in the primordial C[haos], doge's true klass, with the k, was its singleton class.

[slide 35] (sad Dog, and class method snippets)
As it happened, naturally, Dog, the class, now had a problem. He prayed to Ruby, "The programmers want me to keep track of all my Dog instances and find them by their name. But I can't do that with normal instance methods! I need class methods!" And Ruby, remembering what she had granted doge, saw a similar solution here, and spoke, "Thusly I do grant you the power to have methods of your own, and not just methods that all classes possess".

[slide 36] (show diagram with metaclass; fade in metaclasses of class, module, object, basicobject)
And so she wrought a singleton class for Dog, and because Dog himself was a class, this singleton class was like a class of a class, and Ruby christened it a metaclass, to set it apart from the singleton classes of normal objects like doge. And in creating the Dog metaclass, Ruby had to create the Class metaclass, and Module metaclass, and Object metaclass, and BasicObject metaclass, and she decided to mirror the original genealogy, so that it was as if the BasicObject metaclass had begot the Object metaclass, and so on. And because it's really fun to have arrows flying all over the place, she made it such that Class begot the BasicObject metaclass.

[slide 37]
And so it was that the objects and their singleton classes and their metaclasses were finally happy, though now their world had been very much complicated, and every so often one of the metaclasses was wont to cause mischief, demanding that they have their own metaclass. And that metametaclass would be even more mischievious, and demand its own metaclass, and on and on would these metametametaclasses go, till no one could even see where the whole thing ended, and the ordinary objects and classes just rolled their eyes and made do with their first level singleton classes and metaclasses, and carried on with their day-to-day, oblivious to all the meta-madness.



(9:00)
(8:15)

------------------------------------------------------

(Start from 0:00)

[slide 38] (campy picture of happy objects)
And this is where my Ruby Creation Story ends. So hopefully that was able to give you a sort of grand, schematic overview of Ruby's object model, in a fun way.

(6:00)

[slide 39] (pandora's box)
And now...DUN DUN DUN, it's time for MOAAAR! It's time to open Pandora's Box and dig into that C code. Time to retell the creation myth through the lens of CRuby source

[slide 39] (pacman object eating data)
So, as they always say, in Ruby all data is represented as objects. Doge is an object, and Dog, which is a class, is also an object.

[slide 40] (VALUE obj pointing to a struct (comment that built-in types are handled differently))
So when I say that all data in Ruby is an object, what this translates to, in the C source, is every bit of ruby data being represented as pointers to structs. Now if you don't really know C or anything about pointers and structs just imagine a pointer literally being this arrow over here, a reference; and a struct you can think of as a really barebones class, no methods, just a bag of attributes, or members. Now the question is, what's actually in the struct?

[slide 41] (C code, RObject)
And this is where I'm going to show you some actual C code. And just to note also that all the code I'm showing is from the ruby 2.4 branch.

[slide 42]
So, objects are represented as structs. And there are 3 structs that we'll primarily be interested in. The first is the struct that represents normal instance objects, RObject, and the second is the struct that represents class objects, RClass. And you notice that both these structs actually store another kind of struct in them, which is RBasic.

[slide 43] (C code, RBasic, T_CLASS, T_OBJECT, FL_SINGLETON)
Let's look at RBasic first, which stores basic information that every object has.

So you see that we're storing some flags, which is kinda like metadata of the object. So whether this object is a normal instance object, or is a class, or module, we actually store that in flags. And also whether or not my object is a singleton class, we store that in flags.

And then more importantly, for us, is this klass, klass with the k, member. This is how we actually keep track of the 'true' class of our object, and this is one major thing we care about in figuring out our object model.

[slide 44] (highlight bits of code)
Now RObject isn't really much more. It's just RBasic plus this union thing here, which suffice to say is for storing our instance variables ~~[either on the heap, or embedded directly in an array in the struct, depending on how many instance variables we have to handle]~~.

[slide 43] (C code, RClass and rb_classext_struct; class instance variables)
And finall, RClass. Again, it also stores an RBasic struct, so it has flags and klass. It also very importantly stores super, and that's the reference to the superclass of this class. Here there's a pointer to a ruby class extension struct, which we won't worry about (but I should mention, since classes are also objects, so they also have instance variables, and those are stored by this extension struct), ~~[but it's worth nothing that among other things this class extension struct stores pointers to a table of class instance variables, meaning variables of a class instance, not variables of an instance of a class; so what you get when you define an instance variable in a class body like this, instead of inside an instance method. And the class extension struct also stores a pointer to a table of class constants. But we won't worry too much about that]~~ and here is a pointer to a method table, and this is the table that's looked up during method dispatch.

[slide 43.5] (search_method and super chain)
So, this is the core of method dispatch right here, right. Basically keep searching the method tables of your class and then your super classes until you find something. So if doge called the method 'class', this is the chain of RClass structs whose method tables we would be searching. Except not quite. There's a bit of a complication here from Kernel, which is a module included in Object and not its superclass, but somehow it still sort of finds its way into this super chain here. And we'll actually see why, more clearly, in a later part of this talk.

[slide 44] (where does it all begin?)
So we've laid the basic groundwork, we've seen the C structs that Ruby uses to represent our data. So now the question is where does it all begin? Where does our creation myth start?

(3:30)
(2:50)

----------------------------

(start from 0:00)

[slide 45] (InitVM_Object, init_class_hierarchy)
And you'll find it in the `InitVM_Object` function in `object.c`. And you see that it's calling `init_class_hierarchy` here, and this function, is actually where our class hierarchy gets bootstrapped. So after these `boot_defclass` functions get called, our class hierarchy grows like this. And here with `RBASIC_SET_CLASS` we're saying, the class of BasicObject, Object, Module, and Class, is Class, and we get all these nice arrows.

And you may be wondering, what about `Kernel`? So if we go back to `InitVM_Object`, and this is a very long method, right, because it's basically defining all the built-in methods of our built-in classes. But you see here that we define the module `Kernel` and include it in `Object`. And then we go on to define the methods in `Kernel`.

[slide 46] (basic schematic from myth (first 5 objects))
And what you get after this is basically this schematic that you saw from the creation myth before; in addition to this genealogy of ancients (laser pointer?) Ruby also initializes all the built-in classes like Nil, and String and so on. Now the next interesting question is, what happens when I define my own class?

(1:00)


[slide 47] (rb_define_class_id)
So let's say you define a class, Dog. This will end up calling the C function `rb_define_class_id`, which you see here. Now it turns out that this `id` parameter doesn't actually do anything, probably got obsoleted after refactoring, so just ignore that. So there are 3 things going on here: first, if the superclass wasn't explicitly specified, set super to `Object`. Second, actually initialize the class with the given super. Importantly, this sets up the superclass of our class, as well as the klass of our class, which is Class with the capital C. And thirdly, we actually create the metaclass of our class straightaway.

[slide 47.5] (schematic before metaclass creation)
Now before jumping into any of that metaclass stuff, what you have is basically something like this. Seems simple enough, but this is like the breather before we jump into metaclasses.

(2:00)

---------------------------------

(start from 0:00)

[slide 48] (Venn diagram)
Now if you're still confused about terminology, like is it metaclass or singleton class, or why do I seem to use one not the other. Basically a metaclass *is* a kind of singleton class. All objects can have singleton classes, right. But there are a special group of objects that are also *classes*, like our Dog here, or like Class, or Module. And to differentiate between the singleton classes of normal objects like doge, and the singleton classes of class objects like Dog, we call the singleton classes of class objects metaclasses, because they are basically classes of classes. And you can see where this is going to lead, because singleton classes are themselves class objects, and you can have singleton classes of singleton classes. So you can have metametaclasses, metametametaclasses, in general meta^(n)classes, although pretty much any n greater than 1 is not practically useful at all. It's just fun to realize that Ruby's object model actually allows you to do this.

(1:00)

[slide 49] (rb_make_metaclass)
Ok, so get ready, because this is where we jump into the metaclass stuff. And this make_metaclass method here looks crazy, but I'm going to break it down for you conceptually.

(Show Dog RClass struct; highlight code and show changes in diagram along the way)
Step one: we create a new RClass struct, and set its singleton flag. Step two: we actually set the metaclass of Dog, to this newly created RClass struct. So we have this new metaclass, and new arrow, on our diagram. Step three: Now here is where things get a little crazy: we not only want to set the metaclass of Dog, but we want to set the metaclass of the metaclass of Dog, and what would that be? It turns out that we end up having the create the metaclass of Class, and if we do that we also have to worry about setting the metaclass of the metaclass of Class! But in this case, just like how Class before had that arrow pointing back onto itself, so now we have that arrow of Class's metaclass, point back to itself.

And then what? Are we done? Not quite! We still have to set the superclass of our new metaclasses. So what's the superclass of Class's metaclass? Now this is going to sound like a tongue twister, so I'll say it slowly: The superclass of Class's metaclass, is the metaclass of Class's superclass. If you didn't understand the sentence, you should have understood the diagram. So this means we have to create the Module metaclass. And if we do that we also have to set its metaclass, which is now Class's metaclass. And we also have to set its superclass, which, guess what, is the metaclass of Object. So we've set up a chain reaction of sorts. And where does it end? With the metaclass of BasicObject. And what is the superclass of BasicObject's metaclass? As it turns out, it's Class, and so we have this beautiful cycle over here.

[slide 50] (all that just to create the metaclass of Class!)
And that was alot of work, wasn't it, just to create the metaclass of Class. And our original goal was to create the metaclass of Dog, and we're still not quite done: we haven't set its superclass, which is step four. But it turns out that the superclass of Dog's metaclass, is just the metaclass of Object.

(3:10)

[slide 51]
And congratulations, we are done! All we wanted was to create a new Dog class, but we ended up spawning like 5 metaclasses and a gazillion arrows along the way. Boromir was right.

[slide 52] (make_singleton_class)
Now after all that metaclass madness, normal singleton classes are really a piece of cake. Say we've instantiated doge, and we want to create its singleton class. Again, we create a new RClass struct, whose superclass is our original Dog class, and set its singleton flag. Next, we simply set the klass member of doge's RBasic struct, to the newly created RClass struct. And we also want to set the metaclass of our singleton class, which turns out, is the metaclass of Class.

[slide 53] (big diagram)
So we're back at this diagram, which we've actually arrived at twice; first with the story, now with the C code. So hopefully it's pretty much burned into your brain right now, and there's nothing cryptic about Ruby's object model anymore.

(4:00)

---------------------------------

[slide 54] (include Saberteeth, include Sunglasses)
But, you realize that we're still missing one big thing, and that is modules. So how does including modules work? And the answer is that Ruby finds a pretty clever way of sneaking our modules into our inheritance chain, which is how they pop up when we call the `ancestors` method on our class. Ruby does this with *include classes*.

[slide 55] (diagram, and include_modules_at)
What happens when you include the Saberteeth module into Dog, is that Ruby creates a special kind of class called an include class, with the same method table as the module itself, and basically inserts it into our inheritance chain here; so the superclass of Dog is now this include class here, though we can't actually see this from Ruby. And in this way method dispatch just works like normal, we just go up the inheritance chain.

Now the bulk of this module inclusion logic, is actually in this `include_modules_at` function, and this is a really hefty function, but this is the core of it, really, inserting the include class into our inheritance chain. And what you should notice from this diagram is that, our include classes and our modules, are also represented by RClass structs, which mean they also have a `klass` pointer and a `super` pointer. And we know the `klass` of our Saberteeth module, it's just Module. But the klass of our Saberteeth include class, is actually Saberteeth, and that's how the include class keeps track of the module it's created from.

So you can see that here's the function where the magic happens. It's so long that I'm not even going to bother showing you all of it, just this bit here where you see the include class being initialized and the super pointers being set.

[slide 56] (doge with saberteeth and sunglasses)
And what you should realize is that this function is called everytime an include statement is evaluated, meaning, if you have multiple includes, based on how our include class is being inserted over here, the second include class will be lower down the chain.

So saberteeth alone wasn't kickass enough. We know that all cool people, except me, wear sunglasses as well, so let's give that to doge over here. And this is basically how our diagram looks like.

(1:30)

[slide 57] (saberteeth sunglasses)
Now let's complicate things abit. What about including modules in modules? So instead of a doge with saberteeth and sunglasses, I want a doge with saberteeth sunglasses. So we include the Saberteeth module in our Sunglasses module. What happens here? Remember I said that modules are just RClass structs, so they also have a `super` pointer. Now, for normal modules, this pointer is just NULL. And from the Ruby side of things, there really isn't such a thing as a superclass of a module, right? It doesn't make any sense. But under the hood, this `super` pointer comes in handy when we include modules in modules. This results in the creation of a Saberteeth include class, which the `super` of our Sunglasses module now points to. And now when you finally include Sunglasses into Dog, you get two include classes snuck into the inheritance chain, just like this.

(2:30)

[slide 58] (quizz time!!! (yaey rdrc illustration))
~~So time for a bonus question. We've seen what happens when we include Saberteeth into Sunglasses, and then include Sunglasses into Dog. What about when we include Sunglasses into Dog, and then include Saberteeth into Sunglasses? Is there a difference? Aaaand I'm going to be annoying and not answer that question and let it bug you for the rest of the day.~~

So awesome, I am approaching the end of my talk, and we've covered a lot of ground. We talked about objects, classes, superclasses, singleton classes, metaclasses, modules, all the core bits of Ruby's object model. But before I start wrapping up, I would be remiss not to clarify this final point: the differences between the C klass and super pointers, and the Ruby class and superclass methods.

(3:00)

-----------------------------------------------------

(start from 0:00)

[slide 59] (but what's it all for? (waii illustration))
So now I'm approaching the end of my talk, and maybe at this point you're like, uuhhh yea all this under the hood stuff, it's fun to know, but it doesn't really help be become a better Ruby programmer. And well, yea, maybe that's true to a certain extent; knowing that I can create metametametametametametaclasses really isn't going to help me or you in anyway, in the day-to-day. But I'd like to think that having a really solid mental model of Ruby's object model will help you reason more clearly about your code, especially when you're doing some confusing metaprogramming stuff.

[slide 60] (class eval, instance eval; Ruby and C code)
So with that in mind, let's look at some metaprogramming stuff that is usually really confusing to newcomers but which is just totally clear once you understand Ruby's object model. So the infamous instance_eval and class_eval distinction. In both cases `self` in your block, or string, refers to the receiver of the instance_eval or class_eval method. The only difference, which you can very clearly see in the C source, is that class_eval evaluates the code inside the receiver, which is a class or a module, so that these methods here become instance methods; while instance_eval evaluates the code inside the singleton class. So if you're calling instance eval on a normal instance object, the methods that get defined are singleton methods. And if you're calling it on a class object, you get singleton methods on the class object, meaning, class methods.

[slide 61] (include and extend)
And it's the exact same dichotomy with including versus extending modules. When you include a module in a class, you're sneaking in that include class as the superclass of your class. And you get those module methods and instance methods. When you extend an object, you're sneaking in that include class as the superclass of your object's singleton class. And when your object happens to be a class object, you're sneaking in that include class as the superclass of your metaclass. So now you get those module methods as class methods.

[slide 62]
And I could probably stand here all day giving and explaining examples. But I really don't need to do that anymore. Y'now why? Because *you* now have an awesome mental model to figure it out by yourself. The Mental Model I Wish I Had Starting Out...AND MOOAR!!! So thank you.

(2:34)

--------------

**Walking through the method in detail like this takes too damn long and is probably really dry**

[slide 49] (rb_make_metaclass)
So I hope it's clear now, what metaclasses and singleton classes are, if you've ever been confused by the terminology, as I know is all too easy to be, because it's not always used very consistently, even in the C source itself. And I'm going to show you what I mean, because we're going to dig a little bit more into that `rb_make_metaclass` function we saw just now; this is where it gets both exciting and confusing. We see that despite its name, `rb_make_metaclass`, actually makes either a metaclass, if the object passed to it is a class object, or a normal singleton class, if the object passed to it is a normal object.

[slide 50] (make_metaclass; separate slides for each bit)
Since we were in the midst of defining our Dog class, which is a class object, let's see what `make_metaclass` does. And this can seem slightly overwhelming, but let's take it bit by bit. You see here we initialize the metaclass, and set the singleton flag on it. Now, this `if-else` block. The `if` condition evaluates this `META_CLASS_OF_CLASS_CLASS_P` (for predicate) macro. This is a bit of a mouthful, but basically it's checking, is my klass Class? Or is it a metaclass of Class? Or is it in general a meta(^n)class of Class? So here our klass with the k is Dog, so it doesn't satisfy this condition and we go to the `else` block.

Now there's actually two key things going on here. The first is straightforward, which is, we set the metaclass of Dog, to the metaclass that we created over here. And what this does is that, remember that Dog is actually represented by an RClass struct, and that RClass struct stores an RBasic struct with a klass member? What this does is set the klass member of Dog's RBasic struct to the new metaclass we created over here. So that's the first thing that's going on.

(18:40)

The second thing is slightly more tricky. Not only do we want to set the metaclass for Dog, we also want to set the metaclass for Dog's metaclass. So Dog's metaclass is also represented by an RClass, and we also want to set the klass member of its RBasic struct. And you see that we are setting the metaclass of Dog's metaclass, to this `ENSURE_EIGENCLASS(tmp)` thing over here. First off, what is `tmp`? And we see that `tmp` is whatever was initially stored in Dog's RBasic `klass` member. In other words, what is the class of our Dog class? And remember that I said that before all this metaclass stuff happened, the class of our Dog class is Class with the capital C [So let's backtrack a bit to where we originally started, which was this `rb_define_class_id` method. And before `rb_make_metaclass` was called we called `rb_class_new` to actually initialize our Dog class. And suffice to say that somewhere in there, the RBasic klass member of our Dog class object, was set to capital C Class.] Meaning, going back to the `make_metaclass` method, `tmp` is basically Class with the capital C. And what the ENSURE_EIGENCLASS macro does is, it tries to get the metaclass of tmp, or create it if it doesn't already exist. (Now at this point I should probably note that this is another reason why the terminology can be confusing, because before it stabilized on singleton class and metaclass, eigenclass was also being used to refer to the same concept. So anyway eigenclass is basically the same as singleton class.) So, ENSURE_EIGENCLASS(tmp) gives us the metaclass of tmp, so, the metaclass of Class. But that's not created yet. (show schematic with dotted box)

So let's make it! And guess what function gets called? `make_metaclass`, again! But this time, we're passing in Class. And this time, our `META_CLASS_OF_CLASS_CLASS_P` condition returns true. So we enter this branch, we set the metaclass of Class to our newly created metaclass. Originally, remember the klass member in the RBasic struct of Class, pointed back to Class itself. So now it points to its metaclass. But now we also want to set the metaclass of Class's metaclass. But we're not going to create another metaclass, or it's never going to end. Instead, like how Class originally pointed back to itself. Class's metaclass now points back to itself. And our beautiful schematic now looks like this.

(20:40)

[slide 51] (just a little more!)
And by now you're probably looking at me, and thinking, that felt like an awful lot of work just to get a few arrows up on the screen. And I'm like, well, yeah, but we're not done yet, *but* we are almost there.

[slide 52] (make_metaclass)
There's just this last bit to the `make_metaclass` function, which is where we set the superclass of our metaclass; and this sets off a bit of a chain reaction here, and you'll see what I mean soon. And we're still talking about the case where our klass with the k here is Class with the capital C, not Dog, we haven't returned from this function call yet. So what's the superclass of Class's metaclass? The first thing we do, is get the superclass of Class itself. Which is Module. Then, we check to make sure the superclass we got isn't an *include class*, with type T_ICLASS over here. What, you ask, is an include class? We'll talk about it when we talk about including modules, but suffice to say, for now, we want to skip over them. So super is set to Module, and now we set the super class of Class's metaclass to ENSURE_EIGENCLASS(super). In other words, the superclass of Class's metaclass is the metaclass of Class's superclass. And that probably didn't make any sense at all, so let's go back to the diagram, and this is basically what I mean. The superclass of Class's metaclass is Module's metaclass. But when we create Module's metaclass, we're calling `make_metaclass` again, right, so we also have to set *it's* superclass, and now you see what I mean by the chain reaction. This means that I have to create Object's metaclass, so that it can be a superclass to Module's metaclass. And I have to create BasicObject's metaclass, so that it can be a superclass to Object's metaclass. (show callgraph) And now the obvious question is, who's going to be the superclass of BasicObject's metaclass? Aaand if you look at this `RCLASS_SET_SUPER` line over here, you notice that we don't always call ENSURE_EIGENCLASS(super), but we actually have this ternary operator, to check if super actually exists. And when we call make_metaclass on BasicObject, the super of BasicObject is `NULL`, so instead of executing the ENSURE_EIGENCLASS(super) branch of the ternary, because there's no super to be found, we actually set the superclass of BasicObject's metaclass to Class. So what we have is this nice cycle over here.

**This is too damn long**

[slide 53] (schematic)
And congratulations, all we set out to do was to create our Dog class, but we ended up with this monstrosity instead

## Illustrations

* Ancients family tree (BasicObject till Module) DONE
* Object vomitting children DONE
* Family tree with built-in classes and Dog DONE
* Doge derps DONE
* Doge "what am I" DONE
* Doge "I am a Dog!" DONE
* Family tree with doge klass DONE
* Dog "What am I" DONE
* Dog "Where did I come from?" DONE
* Family tree with Dog klass and super DONE
* "WHAT AM I?" (giant thought bubble) DONE
* Family tree with ancients klass, then super DONE
* Family tree with Kernel klass DONE

----------------

* Doge "make me a unique pretty snowflake woof woof woof" DONE
* Ruby + doge + code DONE
* Family tree with singleton class; fade in `class`, `klass` DONE
* Dog "gimme class methods leh"; Ruby "okay can" DONE
* Family tree with singleton classes; fade in metaclasses of Class, Module, Object, BasicObject, with arrows (one by one) DONE
* Family tree with second level metaclasses DONE
* Family tree with third level metaclasses DONE
* Family tree with many levels of metaclasses (fade out at either end) DONE

---------------

* And they all lived happily ever after (campy) DONE
* MOAAR!
* Pacman Object eating data DONE
* Slides slides code slides...
* Doge, Dog and mtable (fade in arrows)

--------------

*
