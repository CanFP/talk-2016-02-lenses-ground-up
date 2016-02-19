footer: Introduction to Lens | David Peterson | CanFP 2016.
slidenumbers: true

# Lenses from the ground up


### David Peterson

---
# What is a Lens?

Good question!

* A pure functional approach to manipulating both the content and structure of (often) deeply-nested data structures.

---
# Why use a Lens?

Another good question!

* Provides a powerful mechanism for manipulating data structures, and composing these manipulations to perform higher-order operations.

---
# Let's take a step back!

* The focus of this talk will be on exploring some _very basic_ lens(-like) operations.

* Our goal is to gain an intuitive understanding of what lenses _are_, and how they work.

* The lenses we define here are not quite the same as _real_ lenses. They are much simpler, and much less powerful.

---
# Tuples

Let's start with something simple: *tuples*.

A _tuple_ is just two associated values wrapped up, so that you can treat it as one _thing_.

```haskell
> (1,2)
(1,2)

> (,) 42 42
(42,42)

> :t (,)
(,) :: a -> b -> (a, b)
```

---
# Tuples

Let's define `get` and `set` functions for the _first_ element of the tuple, i.e. here: (**1**,2).

```haskell

-- retrieve the value x
get1 :: (x, y) -> x
get1 (x, _) = x

 -- replace the value x with x'
set1 :: x' -> (x, y) -> (x', y)
set1 x' (_, y) = (x', y)
```

---
# Tuples

```haskell
> get1 (1,2)
1

> set1 10 (1,2)
(10,2)
```

---
# Tuples

Similarly we can define `get` and `set` functions for the _second_ element of the tuple, i.e. (1,**2**)

```haskell
get2 :: (x, y) -> y
get2 = snd

set2 :: y -> (x, y) -> (x, y)
set2 y' (x, _) = (x, y')
```
---
# Tuples

```haskell
> get2 (1,2)
2

> set2 0 (1,2)
(1,0)
```

None of this has anything to do with _Lens_ ... it's just vanilla _pattern matching_.

---
# Defining a Lens
Using standard record syntax, define a type constructor called `Lens`

```haskell
data Lens a b =
  Lens { get :: a -> b
       , set :: b -> a -> a
       }
```

To create a _Lens_, you need to pass two functions, one of type `a -> b`, and the other of type `b -> a -> a`.

---
# Defining a Lens

Hey, we already have some!

```haskell
get1 :: (x, y) -> x
set1 :: x -> (x, y) -> (x, y)
```

Here the tuple type `(x,y)` corresponds to `a`, and `x` corresponds to `b` (in `Lens a b`).

In Lens terminology, we call `a` the _object_, and we call `b` the _focus_.

---
# Defining a Lens

So, let's make a _lens_ ...

```haskell
_1 = Lens get1 set1
```


---
# Defining a Lens

Recall from Haskell record syntax, we automatically get _helper methods_ for each named record field.

```haskell
> :t get
Lens a b -> a -> b
--          ^^^^^^
--             |-- this is the signature of get1

> :t set
Lens a b -> b -> a -> a
--          ^^^^^^^^^^^
--               |--  this is the signature of set1
```

---
# Using a lens
Before, we had

```haskell
> get1 (1,2)
1
```

Now with our lens, we have

```haskell
> get _1 (1,2)
1
```

---
# Using a lens
Similarly, before we had

```haskell
> set1 5 (1,2)
(5,2)
```

Now with our lens, we have

```haskell
> set _1 5 (1,2)
(5,2)
```

---
# Using a lens
We have decoupled the `set` and `get` operations from the specific location on which they operate.

Intuitively, this sounds like a good thing, right?

I think so.

---
# Using a lens

Let's make another lens ...

```haskell
_2 = Lens get2 set2
```

### Alternatively (and perhaps more typically), we can use _anonymous_ functions rather than named ones.

```haskell
_2 = Lens (\(_, y) -> y) (\y (x, _) -> (x, y))
```

---
# Using a lens

```haskell
> get _2 (1,2)
2

> set _2 5 (1,2)
(1,5)
```


---
# Is that all?

NO!

---
# Lens composition

Lenses _compose_!

```haskell
-- Define a composition operator (>-) ...

(>-) :: Lens a b -> Lens b c -> Lens a c

(>-) l1 l2 = Lens (get l2 . get l1) $
                  (\part whole -> set l1 (
                      set l2 part (get l1 whole)) whole)
```

Admittedly, composing `set` is a little funky!

---
# Lens composition

Let's do it!

```haskell
_1_1 = _1 >- _1

_1_2 = _1 >- _2


-- Looking at the types can be helpful!

> :t _1_1
_1_1 :: Lens ((c, b1), b) c
--            ^^^        ^^^

> :t _1_2
_1_2 :: Lens ((a, c), b) c
--               ^^^    ^^^
```



---
# Lens composition

```haskell
> get _1_1 ((1,2),(3,4))
1

> get _1_2 ((1,2),(3,4))
2

> set _1_1 5 ((1,2),(3,4))
((5,2),(3,4))

> set _1_2 5 ((1,2),(3,4))
((1,5),(3,4))
```


---
# Is that all?

NO!


---
# Shortcut operators

```haskell
(.~) :: Lens a b -> b -> a -> a
(.~) = set
infixr 4 .~

(^.) :: a -> Lens a b -> b
(^.) = flip get
infixl 8 ^.
```


---
# Shortcut operators

```haskell
-- Get

> (1,2) ^. _1
1

> (1,2) ^. _2
2
```

```haskell
-- Set

> _1 .~ 4 $ (1,2)
(4,2)

> _2 .~ 5 $ (1,3)
> (1,5)
```

---
# Over

`over` is like `fmap` for a lens

```haskell
over :: Lens a b -> (b -> b) -> a -> a
over l f a = set l (f (get l a)) a
(%~) = over
infixr 4 %~


> _1 %~ (*2) $ (3,2)
(6,2)

> _2 %~ (*2) $ (3,2)
(3,4)
```


---
# Lens Laws

---
> execrabilis ista turba, quae non novit legem
-- Francis Bacon

---
# Lens Laws

Three of them.

1. Get-Set Law

1. Set-Get Law

1. Set-Set Law

---
## Get-Set Law

```haskell
get_set_law :: Eq a => Lens a b -> a -> Bool
get_set_law l =
  \a ->
    set l (get l a) a == a
```

## Doing a `set` using a value obtained from a `get` is equivalent to doing _nothing at all_.

---
## Get-Set Law

```haskell
> get_set_law _1 (3,2)
True
> get_set_law _1 (99,23)
True
> get_set_law _1 (1,2)
True
```

## Basically equivalent to

```haskell
> let v = (1,2) ^. _1 -- get
> _1 .~ v $ (1,2) -- set
(1,2)
```

---
## Set-Get Law

```haskell
set_get_law :: Eq b => Lens a b -> b -> a -> Bool
set_get_law l =
  \s a ->
    get l (set l s a) == s
```

## Doing a `set` operation followed by a `get` operation returns the value that was set.

---
## Set-Get Law

```haskell
> set_get_law _1 5 (1,2)
True
> set_get_law _1 5 (3,2)
True
> set_get_law _1 5 (2,1)
True
```

---
## Set-Set Law

```haskell
set_set_law :: Eq a => Lens a b -> b -> b -> a -> Bool
set_set_law l =
  \s1 s2 a ->
    set l s2 (set l s1 a) == set l s2 a
--  ^^^^^^^^ ^^^^^^^^^^^^
--     |          |- first (inner) set operation (s1)
--     |- second (outer) set operation (s2)
```

## If perform one set operation (s1) followed by a second set operation (s2), only the result of the second operation is preserved.

---
## Set-Set Law

```haskell
> set_set_law _1 12 24 (1,2)
True
```

### Basically equivalent to

```haskell
-- First set operation (s1)
> _1 .~ 12 $ (1,2)
(12,2)

-- Second set operation (s2)
> _1 .~ 24 $ (12,2)
(24,2)

-- (12,2) is gone!

```


---
# Is that all?

_Cue laughter!_

We're just getting started.


---
# References / Next Steps

1. David Peterson, [Lets.TupleLens] (https://github.com/peterson/lets-haskell/blob/master/src/Lets/TupleLens.hs) (code for this talk!)
1. Tony Morris, [Let's Lens] (http://github.com/nicta/lets-lens) (a whole lens course!)
1. Gabriel Gonzales, [Control.Lens.Tutorial] (https://hackage.haskell.org/package/lens-tutorial-1.0.1/docs/Control-Lens-Tutorial.html) (on hackage)
1. Joseph Abrahamson, [A Little Lens Starter Tutorial] (https://www.schoolofhaskell.com/school/to-infinity-and-beyond/pick-of-the-week/a-little-lens-starter-tutorial)
