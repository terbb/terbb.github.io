---
title: "The small fry's implementation of ECS in C++"
excerpt: "Only for the smallest of fries. Warning: terribly bad runtime complexity."
categories: Blog
tags:
- C++
- ECS
---

#### What the heck is ECS?

Short for *Entity-Component-System*, ECS is an architectural pattern typically used in game development. You can find a good explanation <a href="https://en.wikipedia.org/wiki/Entity%E2%80%93component%E2%80%93system">here</a>. I won't do the explaining here as I think other websites can do it better.

This article also assumes you have a basic understanding of C++, including template functions.

#### Why ECS?

I believe it provides a very flexible and modular way of thinking when developing games by primarily relying on composition instead of inheritance. Primarily, I love the idea of being able to use runtime composition to construct entities.

*An example detailing the utility of runtime composition:* We add a `CollisionComponent` to an object if we want to give it collision. If we want to remove collision from it, all we have to do is remove `CollisionComponent`. Being able to do this for every object seems extremely useful.

Plus, it was (apparently) once hot back in the day, so I believe it's worth investigating as a small fry developer.

#### Why a simple implementation?

If you search up ECS on Google, you'll get a huge smattering of implementations. The majority of them are complicated, reason being that ECS pairs well with a very specific implementation that gives significant performance improvement.

Furthermore, there are a lot of angry people debating on the internet about whether or not ECS is actually useful. This is not helpful when trying to figure out what the heck it is and how to actually try it.

Hence, I'm proposing a very basic implementation that anyone can follow and use.

#### The implementation:

Let's start with the `Component` class:

```c++
int const MAX_COMPONENTS = 1;
int const TRANSFORM_COMPONENT_TYPEID = 0;

class Component {
public:
  virtual int getTypeID() const = 0;
};
```

Note that `getTypeID()` is a pure virtual function, which means that we can't instantiate `Component`. We can only instantiate it's derived classes. Let's take a `TransformComponent` as an example below:

```c++
#include "component.hpp"

class TransformComponent : public Component
{
public:
  inline TransformComponent(int _x, int _y) : x(_x), y(_y) {};
  int x, y;
  static const int typeID = TRANSFORM_COMPONENT_TYPEID;
  inline virtual int getTypeID() const { return typeID; };
};

```

First, notice that we have a `static` variable that assigns all objects of the class a given ID. We'll be using this in our `Entity` class to retrieve and assign components by type.

Next, we've defined `getTypeID()` here, which is important for runtime inheritance. This way, we'll be able to call `getTypeID()` on a `Component` and figure out exactly what kind of `Component` it is.

We also have any other data that we need. In this case, the `TransformComponent` describes the position of the object, so we store the `x` and `y` coordinate as well.

#### The entity:

Now, our `Entity` class:

```c++
class Entity {
  public:
    Entity();
    int id;

    template <typename T>
    T* getComponent()
    {
      for (int i = 0; i < components.size(); i++) {
        if (components[i] != NULL && components[i]->getTypeID() == T::typeID) {
          return static_cast<T*>(components[i]);
        }
	  }
	  return nullptr;
	}

    template <typename T>
    void setComponent(T* component)
    {
      components[T::typeID] = component;
    }

  private:
    std::vector<BaseComponent*> components;
};

Entity::Entity()
{
  components = std::vector<Component*>(MAX_COMPONENTS, nullptr);
}
```

I'll go over this in detail, but in short, every `Entity` has a unique `id`, as well as a vector of `Component`. This makes it trivially easy for us to access components of an entity.

#### Spooky template functions:

Next, we have our template functions, which are not actually as spooky as they appear. We'll go over the first one:

```c++
template <typename T>
T* getComponent()
{
  for (int i = 0; i < components.size(); i++) {
    if (components[i] != NULL && components[i]->getTypeID() == T::typeID) {
      return static_cast<T*>(components[i]);
    }
  }
  return nullptr;
}
```

Here, in `getComponent()`, we iterate over all the components of the Entity in an attempt to find `T`, the given component we're looking for. Since `T` is a type, how can we find out whether any of our components in our vector is actually an object of that type?

Well, all we have to do is simply use our `getTypeID()` function to figure out the `typeID` of our component at runtime, and then use the *scope-resolution operator* to compare it to the `Component` type that we're passing in. Easy!

```c++
template <typename T>
void setComponent(T* component)
{
  components[T::typeID] = component;
}
```

Not much to say about this function: we simply use the *scope-resolution operator* to retrieve our component. 

#### The result:

All we have to do is create a global vector of `Entity`, and we can start creating and adding objects like so:

```c++
std::vector<Entity> entities; // global

Entity e;
TransformComponent transformComponent(10, 10);
e.addComponent(transformComponent);
entities.push_back(e);
```

All we have to do is define a `System` that takes in our entities, and acts on the corresponding entities!

For example,

```c++
void TransformSystem::moveEntities(std::vector<Entity> &entities)
{
  for (Entity &e : entities) {
    TransformComponent *transformComponent = e.getComponent<TransformComponent>();
    if (transformComponent != nullptr) {
      transformComponent->x += 1;
      transformComponent->y += 2;
    }
  }
}
```

This `System` moves the object's `x` by 1 and the object's `y` by 2 every loop. Note that it only acts on object's who actually have that `Component`, so if we want it to stop acting upon a given object, we just have to remove that `TransformComponent` by setting it to `nullptr`!

#### Pros:

The whole ECS architecture is extremely easy to build like this. We don't have to worry about having a seperate vector of component's for each `Component`, which is good, because then we don't have to keep track of which entities each component belongs to, which gets complicated, very quick.

#### Cons:

There are many.

Firstly, note that the method of creating unique ID's is awful as noted above. You'll only ever be able to create `2^31` entities. I would recommend researching a different way to use and recycle unique ID's eventually.

Secondly, note that we need to statically define each component's `typeID`. This can be annoying when creating a `Component`. Furthermore, this `typeID` has to refer to a correct position in the vector, which is extremely bad practice as it relies on correct outside input to avoid crashing.

Thirdly, the performance is awful. We need to iterate over all the component's in an entity (a majority of which can be empty) in order to find a component. A `System` also needs to do this for every single entity.

For a small game, this is fine. You can improve the performance by doing the following:
* Indexing directly in the vector using the `typeID`, which relies on the programmer inputting valid positions for the vector for each `typeID`.
* Using a map, which may or may not actually be faster than iterating through a vector (as iterating through arrays in cache is very fast).
* Using a bitmask for each `Entity` to identify which component's it has.

#### In conclusion

I believe that ECS is a merely an idea with it's own strengths and flaws. To create a "good" game engine, one has to take aspects from multiple ideas and adapt it to their own usage. 

Nevertheless, I think it's a good idea to experiment with ECS and understand it's good and bad points yourself so that you can understand how to make a "good" game engine.

I hope the implementation I provided above gives a good jumping point to play with ECS. If you have any questions or comments, please email me.

