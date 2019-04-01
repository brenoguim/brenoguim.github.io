---
layout: post
title: Exploring Hash Tables in C++
---

This will be a journey on hash tables in C++. The intent of this series of posts is to describe different ways of implementing hash tables. The most important thing: There will be no search for the best hash table. Every table has its own charm.

### A Simple Hash Table

(post is not ready - I still have to test the code :) )

A hash table is a container of objects that offers fast queries. To offer that, one example of API is the following:

* `bool contains(const T& object) const`
    * Returns true if the object exists in the table, false otherwise.
* `bool insert(const T& object)`
    * Attempts to insert the object in the table. If the object already existed in the table, returns false and does nothing. Otherwise, inserts the object in the table and returns true.

In this API, `T` is the type of the object being stored in the hash table.

Let's start implementing that API with a std::list and work the implementation towards a proper hash table.

We want a class template called `Ht`.
```cpp
template<class T>
class Ht
{
```

With the public API we are going to implement:
```cpp
  public:
    bool contains(const T& obj) const;
    bool insert(const T& obj);
```

And a private member `std::list<T>` to serve as storage for the objects in this hash table:
```cpp
  private:
    std::list<T> m_objects;
};
```

Resulting in:
```cpp
template<class T>
class Ht
{
  public:
    bool contains(const T& obj) const;
    bool insert(const T& obj);
  private:
    std::list<T> m_objects;
};
```

Now, for the `contains` API. To know if the element exists in the table, we just search over the elements of the list using the STL algorithm `std::find` provided by `#include <algorithm>`:

```cpp
bool contains(const T& obj) const
{
    return std::find(m_objects.begin(), m_objects.end(), obj) != m_objects.end(); 
}
```

This is not a hash table. It uses a linear search `O(n)` over the elements to check if the object is in the container. The hash table is supposed to offer `O(1)` access.

But wait a minute, what if we had two lists instead of one? Let's call them 'buckets' from now on.

```cpp
bool contains(const T& obj) const
{
    if (objectShouldBeInFirstBucket(obj))
        return std::find(m_bucket1.begin(), m_bucket1.end(), obj) != m_bucket1.end();
    else
        return std::find(m_bucket2.begin(), m_bucket2.end(), obj) != m_bucket2.end();
}
```

Now we only have to search over half the elements (assuming they are equally distributed over the two buckets). That's a lot better.
But how to implement `objectShouldBeInFirstBucket` ?

---
#### Hash function

A hash function (in our world) is a function that can convert a object of type `T` into a `std::size_t` number. Of course, `T` can be a huge object, and not uniquely representable just in a `std::size_t`, but that is fine as long as the following is respected:

- hash(obj1) == hash(obj2) if obj1 == obj2
---

That is very useful because we can use this function to implement `objectShouldBeInFirstBucket`. For our implementation, let's use `std::hash` as a hash function:

```cpp
bool objectShouldBeInFirstBucket(const T& obj)
{
    return std::hash<T>()(obj) % 2; // If the hash number is even, then put in the first bucket.
}
```

Now, if the hash function distributes numbers evenly, then half should be even and the other half should be odd. That will evenly distribute the objects across `m_bucket1` and `m_bucket2`.
Our `contains` method now is twice as fast because it searches only half the elements in the container. :)

Let's make it 100 buckets:

```cpp
template<class T>
class Ht
{
    using Bucket = std::list<T>;
  public:
    bool contains(const T& obj) const;
    bool insert(const T& obj);
  private:
    std::vector<Bucket> m_buckets {100};
};
```

Then we can implement `contains` by using the hash and converting it to a number between 0 and 99 using the module operator:

```cpp
bool contains(const T& obj) const
{
    std::size_t bucketId = std::hash<T>()(obj) % 100;
    auto& bucket = m_buckets[bucketId];
    return std::find(bucket.begin(), bucket.end(), obj) != bucket.end();
}
```

Now, we have to search 100 times less elements than the original list since the objects will be evenly distributed across the 100 buckets.


We can make it faster if we add more and more buckets, right? Particularly, if there are `N` buckets where `N` is the number of elements in the container, then on average there will be only 1 element on each bucket - thus making our search `O(1)`.

But that introduces a problem: The number of buckets is now dynamic - it depends on the number of items in the hash table.
Let's not worry about that right now, and see the full implementation with `N` buckets:


```cpp
template<class T>
class Ht
{
    using Bucket = std::list<T>;
  public:
    bool contains(const T& obj) const
    {
        // Now we use the modulo operator on the size of the buckets container, whatever that size is.
        std::size_t bucketId = std::hash<T>()(obj) % m_buckets.size();
        auto& bucket = m_buckets[bucketId];
        return std::find(bucket.begin(), bucket.end(), obj) != bucket.end();
    }
    bool insert(const T& obj)
    {
        // The insert implementation checks if the container has the element already.
        // If not, search for the proper bucket and insert it.
        if (!contains(obj))
        {
 
            std::size_t bucketId = std::hash<T>()(obj) % m_buckets.size();
            auto& bucket = m_buckets[bucketId];
            bucket.push_back(obj);
            return true;
        }
        return false;
    }
  private:
    std::vector<Bucket> m_buckets {1}; // Start with one bucket
};
```

Now, whenever we insert a new element on the hash table we run the risk of having a bad ratio of elements to buckets. If we have 100 buckets and 1000 elements, then we will have an average of 10 elements per bucket.

#### Rehashing

We want to make that search really fast, so we will increase the number of buckets whenever we start having too many elements. To do that, we call the method `rehashIfNecessary` everytime we insert. In that method we will check if we are getting to a bad ratio and increase the number of buckets if necessary:


```cpp
template<class T>
class Ht
{
    using Bucket = std::list<T>;
  public:
    bool contains(const T& obj) const { ... }
    bool insert(const T& obj)
    {
        rehashIfNecessary();
        ...
    }
  private:
    void rehashIfNecessary()
    {
        if (m_numberOfElements < m_buckets.size())
        {
            // We are fine in number of buckets
            return;
        }
        
        // Create a new hash table
        Ht newHt;

        // Make sure it has twice as many buckets (the +1 will be explained later)
        newHt.m_buckets.resize(2*m_buckets.size() + 1);

        // Iterate over all elements and add them to the new hash table
        for (auto& bucket : m_buckets)
            for (auto& element : bucket)
                newHt.insert(element);
        
        // Assign the new hash table to myself
        *this = std::move(newHt);
    }
    
    std::size_t m_numberOfElements {0};
    std::vector<Bucket> m_buckets {1};
};
```

Great, now we increase the number of buckets everytime we are running low and make sure we always have a good ratio of number of elements and buckets.

The full code looks like this:

```cpp
#include <list>
#include <vector>

template<class T>
class Ht
{
    using Bucket = std::list<T>;
  public:
    bool contains(const T& obj) const
    {
        std::size_t bucketId = std::hash<T>()(obj) % m_buckets.size();
        auto& bucket = m_buckets[bucketId];
        return std::find(bucket.begin(), bucket.end(), obj) != bucket.end();
    }
    
    bool insert(const T& obj)
    {
        rehashIfNecessary();
        if (!contains(obj))
        {
            std::size_t bucketId = std::hash<T>()(obj) % m_buckets.size();
            auto& bucket = m_buckets[bucketId];
            bucket.push_back(obj);
            ++m_numberOfElements;
            return true;
        }
        return false;
    }
  private:
    void rehashIfNecessary()
    {
        if (m_numberOfElements < m_buckets.size())
        {
            // We are fine in number of buckets
            return;
        }
        
        // Create a new hash table
        Ht newHt;

        // Make sure it has twice as many buckets (the +1 will be explained later)
        newHt.m_buckets.resize(2*m_buckets.size() + 1);

        // Iterate over all elements and add them to the new hash table
        for (auto& bucket : m_buckets)
            for (auto& element : bucket)
                newHt.insert(element);
        
        // Assign the new hash table to myself
        *this = std::move(newHt);
    }
    
    std::size_t m_numberOfElements {0};
    std::vector<Bucket> m_buckets {1};
};
```

Here is a sample code using our new hash table:
```cpp
todo
```

For the next posts, I will discuss a bunch of issues with this implementation:

- Magic `2*m_buckets.size() + 1` formula for increasing the number of buckets
- Two lookups being done for every insertion
- No iterator interface (`begin()`, `end()`)

