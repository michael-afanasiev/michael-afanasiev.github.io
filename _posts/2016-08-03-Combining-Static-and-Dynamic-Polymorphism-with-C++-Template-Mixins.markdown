---
layout: post
author: Michael Afanasiev
title: Combining Static and Dynamic Polymorphism with C++ Mixin classes
---

This is the first in a series of blog posts which describe an interesting way to combine both dynamic and static polymorphism. My day job is as a PhD student in Geophysics, focusing on Full Waveform Inversion, and the application space will be one I’m personally familiar with: numerical analysis using the finite element method. This being said, the techniques presented can be applied to a range of problems, numerical or otherwise. Please read on for a (hopefully) informative and (maybe even) entertaining look at the `C++` Template Mixin Pattern.

## Part 1: Problem statement
The struggle is real. We often want to provide users with dynamic polymorphic behaviour, but would also like to leverage the speed and compile-time checking available with template based static polymorphism. Right off the bat, suggesting that we combine the two seems almost oxymoronic. I mean “dynamic” and “static” are the names of the concepts themselves right? Well, yes, and when we get to the end you’ll notice that a small compromise was made. However, I hope to convince you that this compromise is negligible in relation to what you gain.

Before we get to finite elements, I’d like to introduce mixin templates in a friendlier way. Say we have a cat and a dog, and we want to make them do what they do best: occasionally make sounds. Easy.

{% highlight cpp %}

#include <iostream>
#include <vector>
#include <memory>
 
class Animal {
public:
  virtual void vocalize() = 0;
};
 
class Dog: public Animal {
public:
  void vocalize() { std::cout << "Woof." << std::endl; }
};
 
class Cat: public Animal {
public:
  void vocalize() { std::cout << "Meow." << std::endl; }
};
 
int main(int argc, char const *argv[]) {
 
  std::vector<std::unique_ptr<Animal>> animals;
  animals.emplace_back(new Dog());
  animals.emplace_back(new Cat());
 
  for (auto &animal: animals) { animal->vocalize(); }
  return 0;
 
}

{% endhighlight %}

If you compile and run this (ensuring to use `-std=c++11`) you’ll get a simple conversation: 

`Woof. Meow.` 

This is about as simple as you can get. If you’d like, you can easily make this into something that’s polymorphic at runtime by adding a factory function that returns one class or the other based on some command line arguments. We also find no problem in adding an arbitrary number of animals and operating on them one at a time in our vector.

Now, say that you wanted to expand your pets’ minds. Maybe you sat down with them and watched David Attenborough wax poetic about the Arctic night, and now want to have a conversation about it. This will require modifying their original responses to include their thoughts on BBC: Frozen Planet. Below I’ll jump straight into using mixins to accomplish this, even though it can be accomplished (at this stage) by class pointers. Once I get to the finite-element example, we’ll revisit that decision.

## Part 2: Compile time polymorphism with template mixins

After the show, you guys might have a conversation like this:

{% highlight cpp %}

#include <iostream>
#include <vector>
#include <memory>
 
class Animal {
public:
  virtual void vocalize() = 0;
};
 
class NegativeVibe {
public:
  std::string intelligentComment() {
    return "Am I glad I don't live in the Arctic! ";
  }
};
 
class PositiveVibe {
public:
  std::string intelligentComment() {
    return "I definitely wouldn't mind catching the Aurora Borealis one day! ";
  }
};
 
template <typename T>
class Dog: public T {
public:
  void vocalize() { std::cout << "Woof. " << T::intelligentComment(); }
};
 
template <typename T>
class Cat: public T {
public:
  void vocalize() { std::cout << "Meow. " << T::intelligentComment(); }
};
 
int main(int argc, char const *argv[]) {
 
  Cat<PositiveVibe> intelligentCat;
  Dog<NegativeVibe> intelligentDog;
 
  intelligentDog.vocalize();
  intelligentCat.vocalize();
 
  return 0;
 
}

{% endhighlight %}

This prints out: 

`Woof. Am I glad I don't live in the Arctic! Meow. I definitely wouldn't mind catching the Aurora Borealis one day!`. 

See what’s going on here? Cat and Dog now inherit from some generic template parameter T. T can really be any well-formed class, as long as it provides a function intelligentComment(). What’s also great is that we haven’t introduced any sort of hierarchy tree. We could write any number of purely static Vibe classes, and could attach any of them either Dog or Cat (or both) at will. Really consider the implications of this, because they are pretty amazing. Try switching the Vibe template arguments of Cat and Dog. Still works, except they’ve changed their minds.

For example, you can easily nest things to an arbitrary depth. Perhaps your cat wasn’t that impressed by the next episode, but your dog had a change of heart, and also felt like complimenting Sir Attenborough. No problem to switch it up, and add statements:

{% highlight cpp %}
#include <iostream>
#include <vector>
#include <memory>
 
class Animal {
public:
  virtual void vocalize() = 0;
};
 
class NegativeVibe {
public:
  std::string intelligentComment() {
    return "Am I glad I don't live in the Arctic! ";
  }
};
 
class PositiveVibe {
public:
  std::string intelligentComment() {
    return "I definitely wouldn't mind catching the Aurora Borealis one day! ";
  }
};
 
template <typename T>
class Compliment: public T {
public:
  std::string intelligentComment() {
    return "That David sure is smart. " + T::intelligentComment();
  }
};
 
template <typename T>
class Cutoff: public T {
public:
  std::string intelligentComment() {
    return "";
  }
};
 
template <typename T>
class Dog: public T {
public:
  void vocalize() { std::cout << "Woof. " << T::intelligentComment(); }
};
 
template <typename T>
class Cat: public T {
public:
  void vocalize() { std::cout << "Meow. " << T::intelligentComment(); }
};
 
int main(int argc, char const *argv[]) {
 
  Dog<Compliment<PositiveVibe>> intelligentCat;
  Cat<Cutoff    <NegativeVibe>> intelligentDog;
 
  intelligentDog.vocalize();
  intelligentCat.vocalize();
 
  return 0;
 
}
{% endhighlight %}

which results in: 

`Meow. Woof. That David sure is smart. I definitely wouldn't mind catching the Aurora Borealis one day!`. 

The important thing here is that you can switch PositiveVibe and NegativeVibe, Cutoff and Compliment, and Cat and Dog, and never have to worry about how the functions are being called internally. As long as you write a class that provides the intelligentComment() function, you can write new functionality and chain it arbitrarily deep.

You’ll notice here, though, that we lost one important thing: the ability to store our classes in a polymorphic container. In fact, we’re not related to Animal at all anymore. We’ll quickly fix this in the next section

## Part 3: Using mixins in polymorphic containers

There’s just one easy step we have to make to make our program truly polymorphic:
{% highlight cpp %}
#include <iostream>
#include <vector>
#include <memory>
 
class Animal {
public:
  virtual void vocalize() = 0;
};
 
class NegativeVibe {
public:
  std::string intelligentComment() {
    return "Am I glad I don't live in the Arctic! ";
  }
};
 
class PositiveVibe {
public:
  std::string intelligentComment() {
    return "I definitely wouldn't mind catching the Aurora Borealis one day! ";
  }
};
 
template <typename T>
class Compliment: public T {
public:
  std::string intelligentComment() {
    return "That David sure is smart. " + T::intelligentComment();
  }
};
 
template <typename T>
class Cutoff: public T {
public:
  std::string intelligentComment() {
    return "";
  }
};
 
template <typename T>
class Dog: public T, public Animal {
public:
  void vocalize() { std::cout << "Woof. " << T::intelligentComment(); }
};
 
template <typename T>
class Cat: public T, public Animal {
public:
  void vocalize() { std::cout << "Meow. " << T::intelligentComment(); }
};
 
int main(int argc, char const *argv[]) {
 
  std::vector<std::unique_ptr<Animal>> animals;
 
  animals.emplace_back(new Dog<Cutoff    <PositiveVibe>>());
  animals.emplace_back(new Cat<Compliment<NegativeVibe>>());
  for (auto &animal: animals) { animal->vocalize(); }
 
  return 0;
 
}
{% endhighlight %}

Here we’ve just added one level of multiple inheritance on the outer-most class. Now that we also inherit from Animal, we can store generic pointers to all our different Mixin classes in a single stl container. What a lovely thing that is! All of these different operations are valid:
{% highlight cpp %}
animals.emplace_back(new Cat<NegativeVibe>());
animals.emplace_back(new Dog<NegativeVibe>());
animals.emplace_back(new Cat<PositiveVibe>());
animals.emplace_back(new Dog<PositiveVibe>());
animals.emplace_back(new Cat<Cutoff<NegativeVibe>>());
animals.emplace_back(new Dog<Cutoff<NegativeVibe>>());
animals.emplace_back(new Cat<Cutoff<PositiveVibe>>());
animals.emplace_back(new Dog<Cutoff<PositiveVibe>>());
animals.emplace_back(new Cat<Compliment<NegativeVibe>>());
animals.emplace_back(new Dog<Compliment<NegativeVibe>>());
animals.emplace_back(new Cat<Compliment<PositiveVibe>>());
animals.emplace_back(new Dog<Compliment<PositiveVibe>>());
{% endhighlight %}

In 45 lines of code, we’ve generated 12 separate classes (or types), that can all be considered Animals. What’s more, we put them all into a dynamically polymorphic container, and called them all with only one virtual function call! That’s incredible. This one virtual call is the compromise I referred to in the introduction, but as you can see its still quite a small one.

This has been a trivial example, but I hope it gets the main point across. With `C++` template mixins, we can combine dynamic and static polymorphism under one roof. This is especially useful in designing incredibly complex abstract class hierarchies, where most of the complexity is resolved at compile time, and then conveniently operating on these classes using stl containers.

I’ve made extensive use of this pattern in a code called Salvus, which is a package designed for full waveform modelling and inversion. If you’re interested in seeing just how these patterns might work in the real world, check out [these](https://github.com/SalvusHub/salvus/blob/master/src/cxx/Element/Element.cpp#L122-L242) lines, which define a factory function. Here, instead of Dog and Cat, we’re mixing in dynamic physical equations with different types of finite elements. They all get pushed back to a polymorphic container, and get operated on [here](https://github.com/SalvusHub/salvus/blob/master/src/cxx/Problem/Problem.cpp#L177-L200). The package is a work in progress (isn’t it always), but the linked functionality is key to the design.

Next time (next week?) I’ll go over how we manipulate the mixins to allow for some very cool unit tests. Again, this should be applicable to generic problems, but the main example will be based on finite elements. Further down the road, I’ll delve into the highs and lows of shipping mixin classes off to CUDA capable GPUs. And if all goes well, I just might wax poetic about the Arctic night myself.

If you’re interested in any of this stuff, please don’t hesitate to send an email.

While learning about `C++` mixins, I found these blog posts very helpful: [Dr. Dobbs](http://www.drdobbs.com/cpp/mixin-based-programming-in-c/184404445), [ThinkBottomUp](http://www.thinkbottomup.com.au/site/blog/C%20%20_Mixins_-_Reuse_through_inheritance_is_good).
