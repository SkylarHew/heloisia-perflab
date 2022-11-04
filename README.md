## Introduction

Understanding basic matrix convolutions and basic image representation is crucial for the solving of this problem. The original readme covers these within enough detail. We know the following:
 - A convolution takes *n* x *n* pixels, multiplies them each by a value specified by the filter class, and then adds all pixels together giving the resulting (output) vector.
 - Each filter is defined to be size *n* x *n*, where each index corresponds to a scalar of which the (i,j)th pixel vector will be scaled by.
 - Each pixel is a 3 x 1 vector that holds the (R)ed, (G)reen, (B)lue values of each pixel.
```math
\begin{pmatrix}
R\\
G\\
B
\end{pmatrix}
```
 - Our filter matrix is represented by
```math
\begin{pmatrix}
f_{0,0} & f_{0,1} & f_{0,2}\\
f_{1,0} & f_{1,1} & f_{1,2}\\
f_{2,0} & f_{2,1} & f_{2,2}
\end{pmatrix}
```
 - Our resulting computation for the pixel at output location 1,1 (using input data centered around 1,1)
 ```math
\begin{pmatrix}
red_{1,1}\\
green_{1,1}\\
blue_{1,1}
\end{pmatrix}
=
\begin{pmatrix}
f_{0,0}R_{0,0} + f_{0,1}R_{0,1} + f_{0,2}R_{0,2} + f_{1,0}R_{1,0} + ... + f_{2,2}R_{2,2}\\
f_{0,0}G_{0,0} + f_{0,1}G_{0,1} + f_{0,2}G_{0,2} + f_{1,0}G_{1,0} + ... + f_{2,2}G_{2,2}\\
f_{0,0}B_{0,0} + f_{0,1}B_{0,1} + f_{0,2}B_{0,2} + f_{1,0}B_{1,0} + ... + f_{2,2}B_{2,2}
\end{pmatrix}
```
 - Within our project, images are encoded such that each "plane" is a different color channel, representing R, G, or B. Thus, our color matrix stores the 3 color channels for each pixel at position (row, col)
 ```c++
// color[plane][row][column]
red2_4 = color[0][2][4]     // red value of pixel at 2,4
green2_4 = color[1][2][4]   // green value at same pixel, 2,4
blue1_3 = color[2][1][3]    // blue value of pixel at 1,3
 ```
For my solution, I obtained a median CPE of 12-22 which always gives the resulting score of 110. Here are my methods for increasing efficiency:

## Loop Unrolling

### Inner Loops (*i* and *j*)
```c++
for(int j = 0; j < filterSize; j++){
    for(int i = 0; i <filterSize; i++){
        currentPixel += pixel[offseti][offsetj] * filter(offseti, offsetj);
    }
}
```
Beginning with the inner most loops, we can see that they sum up the filter transformations on each pixels in a 3x3 area (*see endnotes for detailed explanation). We can unroll this into:
```c++
currentPixel += color[plane][row-1][col-1] * filter(0, 0)
currentPixel += color[plane][row-1][col] * filter(0, 1)
currentPixel += color[plane][row-1][col+1] * filter(0, 2)
currentPixel += color[plane][row][col-1] * filter(1, 0)
...
currentPixel += color[plane][row+1][col+1] * filter(2,2)
```
thus removing the necessity of the inner loop entirely.

### Outer Loop (*plane*)
```c++
for(int plane = 0; plane < 3; plane++){
    currentPixel += color[plane][row-1][col-1] * filter(0, 0)
    ...
    currentPixel += color[plane][row+1][col+1] * filter(2,2)
}
```
Since we know this loop will always run from 0 to 2, we can just unroll this to 3 total groups of different planes. I also chose to name these by their respective color channel that each of them represents:
```c++
    red += color[0][row-1][col-1] * filter(0, 0)
    ...
    red += color[0][row+1][col+1] * filter(2,2)

    greeen += color[1][row-1][col-1] * filter(0, 0)
    ...
    green += color[1][row+1][col+1] * filter(2,2)

    blue += color[2][row-1][col-1] * filter(0, 0)
    ...
    blue += color[2][row+1][col+1] * filter(2,2)
```
Now we have fully unrolled the loop into $3 * 3 * 3 = 27$ individual statements.

### (Bonus) Row Major Order
```c++
for(int col = 1; col < (input -> width) - 1; col = col + 1) {
    for(int row = 1; row < (input -> height) - 1 ; row = row + 1) {
        ...
    }
}
```
The final two loops, *row* and *col* cannot be unrolled since the image's width and height are variable. That is, however, not to say that there aren't any improvements to be made. We can employ row major ordering upon the principle that local values are easier to jump between. In multidimensional arrays, the rightmost index (column) will be closer per incremement than any other index. Thus, we reverse the ordering of these loops:
```c++
for(row = 1; row < height; row++){
    for(col = 1; col < width; col++){
        ...
    }
}
```

## Multi threading (Required for 10% extra credit)
*multithreading explanation...* In short, adding
```
#pragma omp parallel for
```
before your for loop will allow that for loop to be ran on multithreading increasing performance significantly. We also have to add the following flag in our makefile to run our program using `openmp`:
```
-fopenmp
```

## Additional Makefile modifications

I only chose to change `-O0` to `-O2` for better optimization.

## Variable Definitions

Our goal is to minimize RAM usage. We can do this through changing the variable type size (using short/char instead of int), predefining values so they aren't calculated at runtime, and using constants for non-changing values. I have only employed the latter two methods.

### Predefining/referencing pointer values

Everytime we need to reference a member of a pointer (`->` operator), we are wasting computational power. Instead, we predefine and reference all of these at compile time by defining them as variables. The notable ones I've used are

#### Dimensions
 - int inputWidth = (input -> width);
 - int inputHeight = (input -> height);

#### Loop Bounds
 - int width = inputWidth - 1;
 - int height = inputHeight - 1;

#### Output values
 - int red, green, blue;

#### Filter values
 - int filterDivisor = filter -> getDivisor();
 - int filter1 = filter -> get(0,0);
 - int filter2 = filter -> get(0,1);
 - ...
 - int filter9 = filter -> get(2,2);

### Const

Using constant variables defines variables in the HDD/SDD rather than RAM thus saving computational power. Obviously, this can only be applied to non-changing variables.
```
// Loop values
const int width = inputWidth - 1;
const int height = inputHeight - 1;
```

## Ternary ops

Very small improvement, however, ternary operators are faster than traditional if-statements. Thus when we would typically write:
```
if(red < 0) red = 0;
```
We choose to write instead:
```c++
red = (red < 0) ? 0: red;
```

## Endnotes

### How do we know the inside loop (*i* and *j*) is only 3?

Technically, the filter class is designed to have any filter size. However, note that this loop is only designed to sum up `[index-1][index-1]` through ``[index+filterSize-1][index+filterSize-1]` which only will be centered if *filterSize* = 3. 
