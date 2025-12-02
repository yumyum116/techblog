> This blog is for begginers or someone who studies programming to be a software engineer.
**If you are the one who understand its concept, but are struggle with implementing concept into program,** THIS IS FOR YOU.

## Topic : Shell Sort
In this post, I'll share my idea of shell-sorting interpretation in a different ways:

 `when a gap sequence is...`
 * pattern 1 : Given with a test case
 * pattern 2 : Create within a program

### Implementation
* pattern 1:
```js:shell_sort_ver1.py
def insertion_sort(array, n, gap):
    for i in range(gap, n):
        x = array[i]
        j = i - gap

        while j >= 0 and array[j] > x:
            array[j + gap] = array[j]
            j -= gap

        array[j + gap] = x

def shell_sort(array, gap_sequence):
    for gap in gap_sequence:
        insertion_sort(array, n, gap)
```
, where $n$ refers the number of elements in the given array and it's given with a test case.


* pattern 2:
```js:shell_sort_ver2.py
def insertion_sort(array, n, gap):
    for i in range(gap, n):
        x = array[i]
        j = i - gap

        while j >= 0 and array[j] > x:
            array[j + gap] = array[j]
            j -= gap

        array[j + gap] = x

def shell_sort(array, n):
    gap_sequence = []

    gap = n // 2
    while gap > 0:
        gap_sequence.append(gap)
        gap //= 2

    if not gap_sequence:
        gap_sequence.append(1)

    for gap in gap_sequence:
        insertion_sort(array, n, gap)
```


If you find any mistakes or comment, feel free to post your opinion.

I will appreciate for your contribution!
