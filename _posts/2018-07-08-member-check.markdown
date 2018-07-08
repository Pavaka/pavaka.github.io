---
layout: post
title: Type trait for member detection
date: 2018-07-08 00:00:00 +0300

tags: [template, C++, metaprogramming]
---
In this post I will show you how to write a type trait
that checks if a given struct/class contains member with
some name.

We will use the following structs.
```c++
struct Warrior
{
    int Courage;
};

struct Troll
{
    int Smell;
};

struct Viking : public Warrior
{};
```

We want to achieve a type trait that we can query like this
```c++
static_assert(HasMember_Courage<Warrior>::Value, "Warrior does not contain Courage member")
```

To achieve this we need a struct like this
```c++

template<typename T>
class HasMember_Courage
{
public:
    static const bool Value = MAGIC;
};
```

In order to compute the MAGIC we need some form of compile time resolution, that looks like this.
```c++
template<typename T>
class HasMember_Courage
{
    template<typename U>
    static std::false_type Test(PTR_TO_SOME_COPMILE_TIME_SFINAE_CHECK_STRUCT);

    template<typename U>
    static std::true_type Test(...);
public:
    static const bool Value = decltype(Test<TO_BE_DEFINED_TYPE>(nullptr))::value;
};
```
We have defined two static member function
template overloads, which return either std::true/false_type and we invoke the overloaded
function and see which of the overloads is matched. We use the decltype to extract the
return type of the overloaded funciton and thus we get the result.

The first overload has compile time check that will *succeed* if the type **T**
DOES NOT have a member named Courage. By succeeding we mean that no substitution
failure will occur and thus this will be a valid existing function.

Since the ellipsis(...) operator is a last resort
in function resolution if the first overload does not vanish due to SFINAE,
the nullptr will match the first overload, otherwise it will match Test(...) function.

The last step is to see what is the actual PTR_TO_SOME_COPMILE_TIME_SFINAE_CHECK_STRUCT
and what is the TO_BE_DEFINED_TYPE. The complete typetrait looks like this

```c++
template<typename T>
class HasMember_Courage
{
    struct Fallback { int Courage; };
    struct Derived : T, Fallback {};

    template<typename U, U> struct Check;

    template<typename U>
    static std::false_type Test(Check<int Fallback::*, &U::Courage>*);

    template<typename U>
    static std::true_type Test(...);
public:
    static const bool Value = decltype(Test<Derived>(nullptr))::value;
};
```
We define a Fallback struct that contains a member with the same name as
the tested(Courage). Then we define a struct that inherits both from T and Fallback.
As you know in C++ multiple inheritance is allowed and both parent classes/structs
can have the same named member. The only catch is that you have to explicitly
specify which one you use in order to not be ambiguous.
```c++
Derived d;
d.Fallback::Courage = 5; // OK
d.T::Courage = 5; // OK
d.Courage = 5; // Ambiguous
```

We define a local struct Check that be used to perform the check.
As you can see the test overload is
```c++
template<typename U>
static std::false_type Test(Check<int Fallback::*, &U::Courage>*);
```
The funtion takes as an argument a **pointer** to the Check struct so
if the Check struct does not fail at subtitution of template arguments
this will be a valid Test overload that takes poiner to Check.

The first template argument of Check is *int Fallback::\** this means
pointer to member of Fallback of type int. The second is
*&U::Courage* this is a pointer to a member of U with name Courage.

As you might have noticed the TO_BE_DEFINED_TYPE is Derived
which contains both members of **T** and Fallback. If **T** contains member
Courage the call to U::Courage will be ambiguous because Derived
has the same named member from both of the parents. Thus this will cause
a substitution failure and the whole overload will be gone.

So if type **T** has member Courage the first overload will be gone
and only the ellipsis which returns std::true_type will be matched. In all
other cases the first overload will be vaild and the result will
be obtained from the std::false_type.

Note that this solution works even if **T** has the member defined
in some of it's parents, while some other solutions I've seen online
work only if it is in this exact type.

As you probably want this type trait to be easily computable for different types
I provide a simple MACRO that wraps this type trait.
```c++
#define GENERATE_MEMBER_CHECK(MemberName)                                 \
template<typename T>                                                      \
class HasMember_##MemberName                                              \
{                                                                         \
    struct Fallback { int MemberName; };                                  \
    struct Derived : T, Fallback {};                                      \
                                                                          \
    template<typename U, U> struct Check;                                 \
                                                                          \
    template<typename U>                                                  \
    static std::false_type Test(Check<int Fallback::*, &U::MemberName>*); \
                                                                          \
    template<typename U>                                                  \
    static std::true_type Test(...);                                      \
public:                                                                   \
    static const bool Value = decltype(Test<Derived>(nullptr))::value;    \
};

GENERATE_MEMBER_CHECK(Courage)

static_assert(HasMember_Courage<Warrior>::Value);
static_assert(HasMember_Courage<Viking>::Value);
static_assert(!HasMember_Courage<Troll>::Value);
```

If you are very unfortnate to work with pre 11 compiler the equivalent solution is
```c++
#define GENERATE_MEMBER_CHECK_PRE_11(MemberName)                        \
template<typename T>                                                    \
class HasMemberPre11_##MemberName                                       \
{                                                                       \
    struct Fallback { int MemberName; };                                \
    struct Derived : T, Fallback {};                                    \
                                                                        \
    template<typename U, U> struct Check;                               \
                                                                        \
    typedef char ArrayOfOne[1];                                         \
    typedef char ArrayOfTwo[2];                                         \
    template<typename U>                                                \
    static ArrayOfOne& Test(Check<int Fallback::*, &U::MemberName>*);   \
                                                                        \
    template<typename U>                                                \
    static ArrayOfTwo& Test(...);                                       \
public:                                                                 \
    enum { Value = (sizeof(Test<Derived>(0)) == sizeof(ArrayOfTwo)) };  \
};
```
[Github full example](https://github.com/Pavaka/Playground/blob/master/cpp/MemberCheck.cpp)











