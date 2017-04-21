#Format a string of names like 'Bart, Lisa & Maggie'
>codewars里完成的第二题

**Description:**

Given: an array containing hashes of names

Return: a string formatted as a list of names separated by commas except for the last two names, which should be separated by an ampersand.

**Example:**
```python
namelist([ {'name': 'Bart'}, {'name': 'Lisa'}, {'name': 'Maggie'} ])
# returns 'Bart, Lisa & Maggie'

namelist([ {'name': 'Bart'}, {'name': 'Lisa'} ])
# returns 'Bart & Lisa'

namelist([ {'name': 'Bart'} ])
# returns 'Bart'

namelist([])
# returns ''
```
**mySolutions**
```python
def namelist(names):
    if len(names)==0:
        return ''
    elif len(names)==1:
        return names[0]['name']
    elif len(names)==2:
        return names[0]['name'] + ' & ' + names[1]['name']
    else:
        tmp = names[-2]['name'] + ' & ' + names[-1]['name']
        for i in range(0,len(names)-2)[::-1]:
            tmp = names[i]['name'] + ', ' + tmp
        return tmp
```
**Best practices**
```python
def namelist(names):
    if len(names) > 1:
        return '{} & {}'.format(', '.join(name['name'] for name in names[:-1]), 
                                names[-1]['name'])
    elif names:
        return names[0]['name']
    else:
        return ''
```
**Clever**
```python
def namelist(names):
  return ", ".join([name["name"] for name in names])[::-1].replace(",", "& ",1)[::-1]
```