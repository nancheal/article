#codewars-isograms
>Fuck，今天网站好像瘫痪了，跑不出结果

**Describe:**

An isogram is a word that has no repeating letters, consecutive or non-consecutive. Implement a function that determines whether a string that contains only letters is an isogram. Assume the empty string is an isogram. Ignore letter case.

**example:**
```python
is_isogram("Dermatoglyphics" ) == true
is_isogram("aba" ) == false
is_isogram("moOse" ) == false # -- ignore letter case
```

**Best Solution:**
```python
def is_isogram(string):
    return len(string) == len(set(string.lower()))
```

**My Solution:**
```python
def is_isogram(string):
	return string == "".join(list(set(string.lower())))
```