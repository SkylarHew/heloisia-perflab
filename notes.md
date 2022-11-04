## Loop Unrolling

### Inner Loops, *i* and *j*
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

### Outer Loop, *plane*
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

## Multi threading
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