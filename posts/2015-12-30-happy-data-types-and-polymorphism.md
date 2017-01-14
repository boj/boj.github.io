---
title: Happy Data Types & Polymorphism
---

I am ridiculously proud of this little bit of code:

```haskell
data Range a = Range a a

between :: (Num a, Eq a, Ord a) => a -> Range a -> Bool
between a (Range b c) = a > b && a < c

betweenl :: (Num a, Eq a, Ord a) => a -> Range a -> Bool
betweenl a (Range b c) = a >= b && a < c

betweenr :: (Num a, Eq a, Ord a) => a -> Range a -> Bool
betweenr a (Range b c) = a > b && a <= c

betweenBoth :: (Num a, Eq a, Ord a) => a -> Range a -> Bool
betweenBoth a (Range b c) = a >= b && a <= c
```

Results in:

```haskell
27.0 `between`     Range 15.0 40.0 => True
90   `between`     Range 15   40   => False
15   `betweenl`    Range 15   40   => True
40.0 `betweenr`    Range 15.0 40.0 => True
0.0  `betweenBoth` Range  0.0  1.0 => True
2.0  `between`     Range  0.0  1.0 => False
```
