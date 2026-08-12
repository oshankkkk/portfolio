---
date: 2026-08-06
Title:
tags: []
---
### Floating point numbers 
<iframe title="Floating Point Numbers - Computerphile" src="https://www.youtube.com/embed/PZRI1IfStY0?feature=oembed" height="113" width="200" allowfullscreen="" allow="fullscreen" style="aspect-ratio: 1.76991 / 1; width: 40%; height: 40%;"></iframe>

scientific notation. significant digits.
floating points is scientific notation and it is very fast and efficient for computers (even us to work with)
```
...  10²   10¹   10⁰  .  10⁻¹   10⁻²  ...
     100    10    1   .   0.1    0.01
```

binary version of this is 0.0011001100110011

why does 10⁻¹ = 1/10? 
That's forced by the exponent law bᵐ · bⁿ = bᵐ⁺ⁿ. 

```
10¹ · 10⁻¹ = 10⁰ = 1
    10 · 10⁻¹ = 1
         10⁻¹ = 1/10
```

Binary does not work like that
32 bits only store 23 significant digits. why? they also store where the decimal point is
scienctfic notation in binary is floating point nums

