# float32

1. https://www.cs.yale.edu/homes/aspnes/pinewiki/C(2f)FloatingPoint.html (the reference unless stated otherwise)
2. https://en.wikipedia.org/wiki/Single-precision_floating-point_format
3. chatgpt
4. https://pytorch.org/docs/stable/generated/torch.isclose.html#torch-isclose

## bits
- floating-point types almost always give only an approximation to the correct value, albeit across a much larger range
- the specific layout here is for 32-bit floats:

```
bit  31 30    23 22                    0
     S  EEEEEEEE MMMMMMMMMMMMMMMMMMMMMMM
```

- The first bit is the sign (0 for positive, 1 for negative).
- The following 8 bits are the exponent in excess-127 [(in other words we subtract 127 from exponent)] binary notation;
  - this means that the binary pattern 01111111 = 127[^1] represents an exponent of 0
  - 1000000 = 128, represents 1
  - 01111110 = 126 represents -1, and so forth
- The mantissa fits in the remaining 24 bits, with its leading 1 stripped off as described above.

> The true significand includes 23 fraction bits to the right of the binary point and an implicit leading bit (to the left of the binary point) with value 1, unless the exponent is stored with all zeros.[2]

- Certain numbers have a special representations:

```
0  -> 0 00000000 00000000000000000000000
-0 -> 1 00000000 00000000000000000000000
```
[3]

```py
>>> float('-0'), float('0')
(-0.0, 0.0)
>>> float('inf'), float('-inf')
(inf, -inf)
>>> float('nan')
nan
```

To represent NaN, the exponent field should have all 1's, and the fraction field should have a non-zero value, such as:[3]

| 31  | 30 - 23  | 22 - 0                  |
| --- | -------- | ----------------------- |
| 1   | 11111111 | 10000000000000000000001 |


To represent positive infinity, the exponent field should have all 1's, and the fraction field should have all 0's, such as:[3]

| 31  | 30 - 23  | 22 - 0                  |
| --- | -------- | ----------------------- |
| 0   | 11111111 | 00000000000000000000000 |


To represent negative infinity, the sign bit should be 1, and the exponent and fraction fields should be the same as for positive infinity, such as:[3]

| 31  | 30 - 23  | 22 - 0                  |
| --- | -------- | ----------------------- |
| 0   | 11111111 | 00000000000000000000000 |


> The reason why NaN requires so many bits to represent is that NaN is a special value that needs to carry additional information beyond just the sign, exponent, and fraction bits. NaN values need to be able to indicate the reason why the value is undefined or unrepresentable, such as whether it resulted from a division by zero, an invalid operation, or an indeterminate operation.
>
> To accomplish this, the IEEE 754 standard reserves a range of bit patterns for NaN values, which can be used to encode different kinds of NaNs that correspond to different reasons for the undefined or unrepresentable value. The fraction field of a NaN value is also used to store additional information, such as a diagnostic message or a code that identifies the type of NaN.
>
> While this may seem like a waste of bits, it allows for more precise and informative reporting of errors and exceptions in floating-point arithmetic, which can be important in scientific and engineering applications where accuracy and correctness are critical.[3]
>
> [...]
>
> Since the fraction field in a 32-bit float has 23 bits, the number of possible non-zero bit patterns is 2^23 - 1 (subtracting 1 because the all-zero bit pattern represents zero, not NaN). Therefore, there are 2^23 - 1 different NaN values that can be represented in a 32-bit float.[3]

Some examples:

```
         0 =                        0 = 0 00000000 00000000000000000000000
        -0 =                       -0 = 1 00000000 00000000000000000000000
     0.125 =                    0.125 = 0 01111100 00000000000000000000000
      0.25 =                     0.25 = 0 01111101 00000000000000000000000
       0.5 =                      0.5 = 0 01111110 00000000000000000000000
         1 =                        1 = 0 01111111 00000000000000000000000
         2 =                        2 = 0 10000000 00000000000000000000000
         4 =                        4 = 0 10000001 00000000000000000000000
         8 =                        8 = 0 10000010 00000000000000000000000
     0.375 =                    0.375 = 0 01111101 10000000000000000000000
      0.75 =                     0.75 = 0 01111110 10000000000000000000000
       1.5 =                      1.5 = 0 01111111 10000000000000000000000
         3 =                        3 = 0 10000000 10000000000000000000000
         6 =                        6 = 0 10000001 10000000000000000000000
       0.1 =      0.10000000149011612 = 0 01111011 10011001100110011001101
       0.2 =      0.20000000298023224 = 0 01111100 10011001100110011001101
       0.4 =      0.40000000596046448 = 0 01111101 10011001100110011001101
       0.8 =      0.80000001192092896 = 0 01111110 10011001100110011001101
     1e+12 =             999999995904 = 0 10100110 11010001101010010100101
     1e+24 =   1.0000000138484279e+24 = 0 11001110 10100111100001000011100
     1e+36 =   9.9999996169031625e+35 = 0 11110110 10000001001011111001110
       inf =                      inf = 0 11111111 00000000000000000000000
      -inf =                     -inf = 1 11111111 00000000000000000000000
       nan =                      nan = 0 11111111 10000000000000000000000
```

- in practice is that a 32-bit floating-point value (e.g. a float) can represent any number between 1.17549435e-38 and 3.40282347e+38, where the e separates the (base 10) exponent.
- Operations that would create a smaller value will underflow to 0 (slowly—IEEE 754 allows "denormalized" floating point numbers with reduced precision for very small values)
- operations that would create a larger value will produce inf or -inf instead.

---

- floating-point numbers are not exact: they are likely to contain round-off error because of the truncation of the mantissa to a fixed number of bits
- particularly noticeable for large values (e.g. 1e+12 in the table above)

---

- a consequence of the internal structure of IEEE 754 floating-point numbers is that small integers and fractions with small numerators and power-of-2 denominators can be represented **exactly**
- it is very difficult to test floating-point numbers for equality, unless you are sure you have an exact value as described above
- It is generally not the case, for example, that `(0.1+0.1+0.1) == 0.3` in C

---

> Absolute and relative errors are two ways to measure the difference between two floating-point values. In the context of float32, these errors help us understand the level of accuracy between two values being compared.
>
> 1. Absolute error: The absolute error is the difference between the true value and the measured/approximated value. It is the simplest way to quantify the difference between the two values.
>
> 2. Relative error: The relative error is the ratio of the absolute error to the true value. It measures the error in terms of the size of the true value, which helps to identify how significant the error is.[3]


`Absolute error = |100.0 - 100.1| = 0.1`
```
True value   Measured value
   |------------|--------------|
100.0          100.1
    <----------->
      0.1 (absolute error)
```

`Relative error = (Absolute error) / |True value| = 0.1 / 100.0 = 0.001`
```
True value   Measured value
   |------------|--------------|
100.0          100.1
    <----------->
      0.001 (relative error)
```

> For small numbers, the absolute error should be set to a smaller value because the difference between two small numbers is also likely to be small. A small absolute tolerance ensures that the comparison is sensitive to the differences in these small values. The relative error can be set to a higher value, as small numbers can have large relative errors while still being "close" in an absolute sense.
>
> For big numbers, the absolute error can be set to a larger value, as larger numbers will generally have larger differences between them. However, you should still be cautious not to set the absolute tolerance too large, as this may cause genuinely different numbers to be considered close. The relative error should be set to a smaller value, as large numbers will typically have small relative errors.[3]

```py
>>> import torch
>>> torch.isclose(torch.tensor((1., 2, 3)), torch.tensor((1 + 1e-10, 3, 4)))
tensor([ True, False, False])
>>> torch.isclose(torch.tensor((float('inf'), 4)), torch.tensor((float('inf'), 6)), rtol=.5)
tensor([True, True])
>>> torch.tensor((1., 2, 3)).dtype, torch.tensor((1 + 1e-10, 3, 4)).dtype
(torch.float32, torch.float32)
>>> # torch.isclose(input, other, rtol=1e-05, atol=1e-08, equal_nan=False) → Tensor
```
[4]

## some c notes
- If you mix two different floating-point types together, the less-precise one will be extended to match the precision of the more-precise one;
- Unlike integer division, floating-point division does not discard the fractional part (although it may produce round-off error: 2.0/3.0 gives 0.66666666666666663, which is not quite exact).
- Some operators that work on integers will not work on floating-point types. These are % (use `modf` from the math library if you really need to get a floating-point remainder) and all of the bitwise operators ~, <<, >>, &, ^, and |.

```c
     // casting
     // i = (int) f;
   1 /* return the average of a list */
   2 double
   3 average(int n, int a[])
   4 {
   5     int sum = 0;
   6     int i;
   7
   8     for(i = 0; i < n; i++) {
   9         sum += a[i];
  10     }
  11
  12     return (double) sum / n;
  13 }
```
- Any numeric constant in a C program that contains a decimal point is treated as a double by default.
- You can also use e or E to add a base-10 exponent

[^1] do you remember your binary? and remember isn't two's complement:

> The range of an 8-bit signed integer in sign-magnitude binary form is:
>
> Smallest (most negative) number: -127, binary representation 11111111.
> Largest (most positive) number: 127, binary representation 01111111.
> Zero can be represented in two ways:
> Positive zero: 00000000.
> Negative zero: 10000000.

```py
>>> list(reversed([(n * (2**(i))) for i, n in enumerate(reversed([0,1,1,1,1,1,1,1]))]))
[0, 64, 32, 16, 8, 4, 2, 1]
>>> sum(list(reversed([(n * (2**(i))) for i, n in enumerate(reversed([0,1,1,1,1,1,1,1]))])))
127
>>> 2**0
1
>>> 2**1
2
>>> 2**2
4
>>> 2**3
8
>>> 2**4
16
>>> 2**5
32
>>> 2**6
64
>>> 2**7
128
```
