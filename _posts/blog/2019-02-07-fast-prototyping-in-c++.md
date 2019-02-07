---
title: "Fast prototyping in C++"
excerpt: "An incredibly useful technique I've discovered recently. And it's language agnostic too, actually."
categories: Blog
tags:
- Tips
- C++
---

#### Fast prototyping?

It's just a name that I came up with on the spot. Basically, what I'm referring to is the act of creating small programs for debugging or even adding new features. It's easier to explain with a few examples.

Say you're working in C++, and you're trying to `push_back()` a reference of `Foo` you've passed into a `vector<Foo>`. But we're getting a gigantic compile error:

```
error: use of deleted function ‘std::unique_ptr<_Tp, _Dp>::unique_ptr(const std::unique_ptr<_Tp, _Dp>&) [with _Tp = int; _Dp = std::default_delete<int>]’
```

Let's assume that we have no idea what the heck this error is talking about (which happens often). Let's look at the class `Foo` first:

```
class Foo {
	public:
		Foo();
		~Foo();	
		void someRandomFunction();
		void anotherRandomFunction(int w);
		std::unique_ptr<int> bar;
	private: 
		int x;
		float y;
		bool z;
};
```

Okay, so `Foo`... has a ton of stuff. Let's assume that we've somewhat isolated our problem by heading to the block of code that's causing the compile error:

```
void passInReference(Foo &foo)
{
  vector<Foo> many_foos;
	vector<Foo> other_foos;
	foo.x = foo.x + 1;
	foo.someRandomFunction();
  many_foos.push_back(foo);  
}
```

What could be the problem here? We're passing in a reference to `foo`, doing some random stuff to it's members and call a random function. We shouldn't be modifying `foo` in any way - why is it complaining about using a deleted function of `unique_ptr`? What the heck?

Notice how it's difficult (if you're a C++ noob) to tell exactly what is causing the problem - our class `Foo` has a ton of random stuff in it, and we're doing a ton of random stuff in `passInReference()`. We could easily be led astray by all the random stuff that's obfuscating our problem.

And if you're running into this problem in a huge codebase, it can be an even bigger pain to debug this. Why? For reasons below:

1. We could be getting a bunch of other compile errors as a result of this single bug, which could easily lead us astray by causing us to think the problem is somewhere else.
2. Compiling and running the codebase could easily take 1-2 minutes, which is forever when you're trying to search for the bug. In fact, even when we find the bug, trying to test out ways to fix it still takes forever!

And, a reason why debugging this in a big project is bad:

3. We might end up ignoring the source of the problem and "fix" it by completely replacing the functionality.

This is clearly a sub-optimal choice, but sometimes a choice that we make because it's just so painfully difficult to sit through 2 minutes of compiling every time you want to try and test out something. And there's often *a lot* to test.

#### How this technique can help you debug

Well, unfortunately, this technique can't really help you with the 1st problem. You still have to find and isolate the bug that you're facing. But, on the bright side, once you've found it, you can easily use this technique to help figure out the problem.

In this case, we've already took out two blocks of code that we suspect are causing the problem. What can we do? Let's create a quick test program to start taking them apart in!

```
int main() {
	Foo foo;
	foo.x = 1;
	foo.bar = std::make_unique<int>(2);
	passInReference(foo);
	return 0;
}
```

It's as easy as that. Just open up a new file, call it `test.cpp` or whatever you like, and put the stuff you'd like to test in `main()`. We can just directly test out whatever hypotheses we might have without having to wait ten years to compile something.

Now, we're still going to run into the error - but guess what? It's way easier to digest. Instead of a billion errors popping up on your screen, you'll only get like, ten. And it takes at most a few seconds to compile.

Furthermore, we can just start taking out random pieces of code without having to `git commit -m "WIP"`, or worse, `ctrl + z` x100, after we've fixed the bug. Let's see a (fake) example of doing so:

```
void passInReference(Foo &foo)
{
  vector<Foo> many_foos;
	vector<Foo> other_foos;
	foo.x = foo.x + 1;
	foo.someRandomFunction();
}
```

We've removed our `many_foos.push_back(foo)` call, and wow, suddenly it compiles just fine! So we can see that the problem has to do with vectors. And indeed, the reason is because when `push_back` is called, it copies the argument into the vector. And as you may know, copying `unique_ptr` is forbidden, because the whole point of `unique_ptr` is that we can only have 1 copy of it.

So if we want to actually "move" our argument into the vector, we're going to have to call: `many_foos.push_back(move(foo))`. And indeed, this works! We're no longer copying something!

Now, in this case, I've miraculously identified the bug extremely quick because I already know what the bug is. But, even if you don't know what it is, it's still incredibly quick to identify one because you can just arbitrarily take out lines and compile and see if it works - something that might take a really long time in a project.

Ever since I've discovered this technique while working on a school project, I've been abusing it like mad to debug some stickier problems. And it's amazing!

#### How this technique can help you prototype

But that's not all. This technique can also help you prototype new features!

Often times, when I'm trying to think of how to implement new features, I learn best by doing. The problem with doing, is that when you're doing, you can end up inadvertently introducing a ton of new bugs. And while you're experimenting, you have to fix these bugs, revert changes, switch implementations.. it's a whole lot of work.

But the great thing about creating a new program to prototype your new feature, is that you don't have to worry about any of that! You can still keep the benefits of experimentation, while not ruining your code base! For an easier to read format, let's summarize the benefits:

1. Don't have to worry about a ton of random, irrelevant code to your feature. Often times, if you want to compile something, you have to modify a whole bunch of classes, which ends up breaking some more stuff, which you then have to fix...
2. Don't have to worry about introducing new bugs into your code, from forgetting to revert old implementations.
3. Don't have to wait ten years to compile and test out your feature!

In other words, it saves a lot of time.

Here's an example of how I used prototyping to implement checking components in an ECS system using bitmasks:

```
const int max_size = 8;
const int transform = 0;
const int color = 1;

bitset<max_size> construct_bitset(vector<int> components) {
  bitset<max_size> temp;
  for (int i : components) {
    temp.set(i, true);  
  }
  return temp;
}

int main() {
  vector<int> components {transform, color};
  bitset<max_size> check = construct_bitset(components);
  bitset<max_size> has = construct_bitset(components);

  if ((check&has) == has) cout << "has matches check" << endl;

  for (int i = max_size; i >= 0; i--) {
    cout << check[i];
  }
  cout << endl;
  return 0;
}
```

Instead of actually trying to test out my implementation directly in my code, and fumbling across multiple files, I just directly tested it here. While it looks small, I actually learned a lot by writing this:

1. I learned I could specify the size of a bitset using some globally defined `const`. (I suspected I could already do this, but wanted to make sure.)
2. I learned that I could just create an arbitrary `vector<int>` using uniform initialization and putting in globally defined `const` variables, which is a mock-up of how our component ID's are actually set.
3. I figured out a quick way to construct a bitset that would be convenient for us (previously, I tested other implementations, like a function that took in a single position of the bit to set. Needless to say, it was inefficient.)
4. I was able to quickly test out the syntax of using `&` and `==`, bitwise AND and bitwise EQUALS respectively, with `bitset`. I wasn't sure if it worked with `bitset` as well.

And just like that, in ~20 minutes, I was able to test out all these hypotheses and easily and smoothly slot this into our project.

#### In conclusion

This technique is awesome. Honestly, it's saved me so much time so far. Googling and StackOverflow helps when debugging or trying new implementations, but nothing helps as much as actually testing out your hypotheses in a closed environment.

I've even used this technique to better understand how C++ works. For example, I've written a test program to help me figure out how to use `shared_ptr`.What did I learn? As I thought, it's sufficient to set whatever is holding onto your `shared_ptr` to `nullptr` - it will then de-allocate itself. Sounds simple, but you never really know until you actually try it.

As a side note, this technique is especially helpful for me as I'm programming everything on the terminal. So it's trivially easy for me to open up a new tab, type `vim testProgram.cpp`, type some stuff, open a new tab, and then run `g++ -std=c++14 -o testProgram testProgram.cpp`. Still, I believe this technique has it's benefits.

This technique isn't just limited to C++ either - try it in any language you want! All in all, I hope this saves you guys some time and pain. If you have any questions, feel free to email me at **trevinwong@gmail.com**.




