#codewars-python-1
> 发现了个有趣的网站[http://www.codewars.com/](http://www.codewars.com/)，觉得这个网站的魅力不是在于它有多少的题库，而是在你解出来题后，去观摩其他大佬的题解时的，惊讶，和自愧不如，以及一步步的成长

**Highest and Lowest**
Description:

In this little assignment you are given a string of space separated numbers, and have to return the highest and lowest number.

Example:
```python
high_and_low("1 2 3 4 5")  # return "5 1"
high_and_low("1 2 -3 4 5") # return "5 -3"
high_and_low("1 9 3 4 -5") # return "9 -5"
```
author Solution:
```python
def high_and_low(numbers): #z.
    nn = [int(s) for s in numbers.split(" ")]
    return "%i %i" % (max(nn),min(nn))
```
my Solution:
```python
def high_and_low(numbers):
    list1 = numbers.split(' ')
    list1 = map(int,list1)
    list2 = []
    list2.append(max(list1))
    list2.append(min(list1))
    numbers = ' '.join(str(i) for i in list2)
    return numbers
```
哎，这种想啥写啥的臭毛病必须得改一改了，python的简洁被我搞得一塌糊涂