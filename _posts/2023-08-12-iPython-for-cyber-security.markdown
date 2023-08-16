---
layout: post
title:  "iPython for cyber security data processing and automation"
date:   2023-08-16 18:12:00 +1100
author: Stephen Bradshaw
tags:
- iPython
- Python
- Burp
- extension
---

A lot of my day job in pentesting/offensive security involves processing varied chunks of data, and ad hoc automation of tasks. For the last several years, Ive been using iPython, the interactive Python environment to do this. While iPython has pretty wide use in various other computing fields, to my knowledge it's not used very widely in security. Whenever I have the rare opportunity to demonstrate how I use it to other pentesters however, they seem to be impressed by how useful it is. This post will be an attempt to explain why I think iPython is so useful for security related workflows.

# What is iPython and what are the basics of using it?

iPython is essentially a souped up version of a Python REPL (Read Evaluate Print Loop) - very much like what you get when you run python with no input script, but way more usable. 

Being a REPL, what iPython does at its core is run code snippets, provide any output, and return to a prompt to allow you to repeat the process.

The following shows me starting iPython and using print to output a simple string.
```
stephen@mac:~$ ipython
Python 3.10.12 (main, Jun 10 2023, 16:04:55) [Clang 14.0.3 (clang-1403.0.22.14.1)]
Type 'copyright', 'credits' or 'license' for more information
IPython 8.3.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: print('Hello from iPython!')
Hello from iPython!
```

Each input and output are numbered. Just entering a string in iPython returns that string as output, labelled below as `Out[2]`. I can retrieve that some output again in a subsequent input, by referencing it by name.  
```

In [2]: 'Hello'
Out[2]: 'Hello'

In [3]: Out[2]
Out[3]: 'Hello'
```

This means that you dont lose access to output generated in previous steps if you want to refer to it again later in your processing.

You can print a history of commands, to see what you have run so far.

```
In [4]: history
print('Hello from iPython!')
'Hello'
Out[2]
history
```

You can also assign output to variables explicitly, and then access them by name in later operations.

```
In [5]: myvariable = 5412

In [6]: myvariable
Out[6]: 5412
```

iPython also has a startup file, that you can use to specify code you want available to you in each session you start. On *nix systems, this file is located at `~/.ipython/profile_default/startup/startup.py`.

My startup file on each machine I use regularly contains one line, which I will explain in the following section.


# Using iPython to debug Python scripts or run code from external scripts

One great use of iPython is to act as a debugging and testing environment when creating larger Python programs. If you're writing a Python program which contains a standalone function to perform some particular task, and want to quickly test that function, running this within iPython is a great way to achieve this.

To facilitate this, you may want to put sections of Python code in a file on disk and import them into iPython to run them interactively.

My iPython startup file contains the following to do exactly this:

```
coder = lambda x : compile(open(x).read(), x, 'exec')
```


This is a helper lambda (one liner function) called `coder` that assists with reading Python content from an external file and executing it. This is written to be executed using the inbuilt function `exec` to run the provided script in the context of the current session. 

When you do this you will get access to any local variables in the script in your iPython session. The following shows importing a two liner script `pythoncode.py` into the current session, and demonstrates how the variable `variablex` from the script is then available locally.

```
In [7]: cat pythoncode.py
print('this is python code')
variablex = 'contents'

In [8]: exec(coder('pythoncode.py'))
this is python code

In [9]: variablex
Out[9]: 'contents'

```

You may notice from the above that we are able to run operating system commands such as `cat`, to display the contents of local files.

If the Python code we execute has any errors, this will also provide us with helpful output that identifies where the problem is. Lets look at an example of trying to import code with errors in file `badcode.py`.

```
In [10]: cat badcode.py
print('Good line')
invalid python

In [11]: exec(coder('badcode.py'))
Traceback (most recent call last):

  File /opt/local/Library/Frameworks/Python.framework/Versions/3.10/lib/python3.10/site-packages/IPython/core/interactiveshell.py:3397 in run_code
    exec(code_obj, self.user_global_ns, self.user_ns)

  Input In [11] in <cell line: 1>
    exec(coder('badcode.py'))

  File ~/.ipython/profile_default/startup/startup.py:1 in <lambda>
    coder = lambda x : compile(open(x).read(), x, 'exec')

  File badcode.py:2
    invalid python
            ^
SyntaxError: invalid syntax
```

We can see this output identifies exactly where in the external script the error is.


# Magic commands and iPython conveniences

iPython has a number of useful [magic commands](https://ipython.readthedocs.io/en/stable/interactive/magics.html) that try to make common actions easier.

For example, copy the following text into your clipboard.

```
print('these')
print('are')
print('multiple')
print('lines')
print('of')
print('code')
```

Then enter the command `%paste`

```
In [12]: %paste
print('these')
print('are')
print('multiple')
print('lines')
print('of')
print('code')

## -- End pasted text --
these
are
multiple
lines
of
code
```

The code in the clipboard is automatically entered and executed.

iPython is also helpful in other ways. It can autocomplete filenames and paths, python code (like in some good IDEs), and it also can autocomplete from your command history.


# Some useful Python concepts

Beyond the basics discussed above, making effective use of iPython relies on your knoweldge of Python coding. 

Before I go into specific code examples however, I want to specifically introduce a few concepts that are very useful when operating in a Python REPL.

## Comprehensions

Comprehensives are a hugely useful way of succinctly specifying iterative operations. It creates a collection containing the results of the operation, with the type of the data structure used defined depending on the syntax used.

The simplest example I can think of is this [list](https://docs.python.org/3/tutorial/datastructures.html) comprehension below. It simply creates a list of each element in the python range of 1-4. 

```
In [16]: [a for a in range(1,5)]
Out[16]: [1, 2, 3, 4]
```

The basic format demonstrated here is:

```
[<result> for <item> in <iterable>]
```

In this most basic of examples:
* The comprehension is wrapped in square brackets `[]`, indicating that the output will be a `list`,
* `<result>` is the value that gets returned for each iteration of the loop, 
* `<item>` is the variable name to which each individual item from the iterable is assigned to _in that instance of the loop_, and 
* `<iterable>` is the collection being processed.

In the case above, `<result>` is the same as `<item>`, but this can be changed by additional processing as you will see in future examples.

As well as for lists, comprehensions can also be used in different contexts, such as for [dictionaries](https://docs.python.org/3/tutorial/datastructures.html#dictionaries), as in the example below.

```
In [18]: {a: 'hello' for a in range(1,5)}
Out[18]: {1: 'hello', 2: 'hello', 3: 'hello', 4: 'hello'}
```

The output type of the comprehension is defined by the internal syntax and the brackets wrapping the operation, e.g. `[]` list, `{}` set/dictionary,  or `()` generator.

Comprehensions can also be nested to arbitrary levels, contain conditions and perform additional operations on the elements of the output. They can get ridiculously complex, and can take some getting used to in their more complex iterations. However they are incredibly powerful and useful in iPython interactive computing. More complex comprehensions will be included in the examples below.


## Lambdas

Lambdas are one line functions. You have seen one example of these already from my `coder` startup function. Heres another example where I define a new lambda named `mylambda`, with input variables `x` and `y`, and then execute it.

```
In [19]: mylambda = lambda x, y : print('Hello {}, its {}'.format(x, y))

In [20]: mylambda('you', 'good to see you!')
Hello you, its good to see you!
```

# Common tasks performed in Python

Now lets look at some specific examples of common tasks that will be very helpful to know about in order to make effective use of iPython for security data processing.

## Reading a file

Reading a text file is as easy as the below. Here we read text file `file1.txt`, which contains 5 lines.

```
In [21]: filecontents = open('file1.txt').read()

In [22]: filecontents
Out[22]: 'Line 1\nLine 2\nLine 3\nLine 4\nLine 5\n'
```

For a binary file, we can specify `b` binary mode. We also need to specifically set the operation mode to `r` read.

```
In [24]: binarycontents = open('binary.bin', 'rb').read()

In [25]: binarycontents
Out[25]: b'\xde\xad\x01\x01'
```

## Writing a file 

Writing a text file can be done like so:

```
In [27]: open('file2.txt', 'w').write('Line 1\nLine 2')
Out[27]: 13
```

A binary file can be written to like so. Note that I am setting binary mode with `b` and also specifying a byte sequence (prefacing the string with `b`) as the input to write.

```
In [28]: open('binary2.bin', 'wb').write(b'\x00\x00\x01')
Out[28]: 3
```

## Reading files++

While its important to understand the basic process of opening and reading from a file for specific use cases or in case of problems, we usually want to do some specific things with files we read.

What about reading each line of a file into a list, while ignoring any blank lines in the file? 

Lets use a list comprehension to split on newline `\n` and only process that line if it has a value. We will use the same text file read earlier, with 5 lines.

```
In [29]: filelines = [a for a in open('file1.txt').read().split('\n') if a]

In [30]: filelines[0]
Out[30]: 'Line 1'

In [31]: len(filelines)
Out[31]: 5

In [32]: filelines[-1]
Out[32]: 'Line 5'
```

Theres a few things going on here to explain. We have taken the basic format of the comprehension as explained above and extended it a little by adding a condition.

So the basic format that we explained above is like this:

```
[<variable> for <variable> in <iterable>]
```

This has now been extended to the following form:

```
[<variable> for <variable> in <iterable> if <condition>]
```

When a condition is specified in the comprehension, only items from the iterable that satisfy the condition are processed further and included in the output.

We can do even more however. Assume we want to seperate each line of the input into multiple variables on the space character.

```
In [33]: filelines_space_separated = [a.split(' ') for a in open('file1.txt').read().split('\n') if a]

In [34]: filelines_space_separated[0]
Out[34]: ['Line', '1']
```

Now our list comprehension is performing an operation on each item from the iterable (which iterates through each line of the file):

```
[<operation>(<variable>) for <variable> in <iterable> if <condition>]
```

## Reading and writing JSON

JSON is a great format for exchanging semi complex data structures with other applications, and writing them from within Python to disk. Lets look at a simple example of using it in iPython.

Heres some data in a Python dictionary (`dict`):

```
In [35]: data = {'one' : 1, "two": 2}
```


Import the json library

```
In [36]: import json
```

Write the data to a JSON file on disk, with indenting. Then dumping the data with cat to show its respresentation on disk.

```
In [37]: open('data.json', 'w').write(json.dumps(data, indent=4))
Out[37]: 30

In [38]: cat data.json
{
    "one": 1,
    "two": 2
}
```


We can also read the data back in from JSON format on disk into a Python representation in memory.

```
In [39]: read_data = json.load(open('data.json'))

In [40]: read_data
Out[40]: {'one': 1, 'two': 2}

In [41]: read_data['one']
Out[41]: 1
```

Some more detailed examples of processing complex JSON files are included later in this post.

## Making HTTP requests

One very common activity in security work flows is making a HTTP request. We can do this easily in Python with the `requests` module. You will need to install this on your machine first, which you can do with pip (e.g. `pip install requests`).

Heres importing the module, making a simple request and looking at some response details:

```
In [42]: import requests

In [43]: response = requests.get('https://github.com/')

In [44]: response.status_code
Out[44]: 200

In [45]: response.content[:100]
Out[45]: b'\n\n\n\n\n\n<!DOCTYPE html>\n<html lang="en"   data-a11y-animated-images="system">\n  <head>\n    <meta chars'
```

Make a HTTPS request, but ignore certificate errors.

```
In [46]: requests.packages.urllib3.disable_warnings()

In [47]: response2 = requests.get('https://192.168.10.34:5001/', verify=False)

In [48]: response2.status_code
Out[48]: 200

```


## Run an Operating System command and get the output for processing

Run the Operating System `cat` command to read `data.json`, and put the output into the `output` variable. 

```
In [51]: import subprocess

In [52]: output = subprocess.check_output(['cat', 'data.json']).decode()

In [53]: output
Out[53]: '{\n    "one": 1,\n    "two": 2\n}'
```

Running `decode` on the output converts it from byte format to a string.


## More...

Ive got a bunch of other Python code snippets listed [here](https://thegreycorner.com/pentesting_stuff/writeups/pythonsnippets.html) for performing other common infosec related tasks.



# Exploring data by example: Active Directory 

I wrote a tool for dumping Active Directory data that can be found [here](https://github.com/stephenbradshaw/ad_ldap_dumper). Though I plan to add Bloodhound output support at some point, at the moment I mainly analyse the results from this in iPython.

Examining the output from this tool provides a really good example of how to use iPython to explore a complex dataset and get useful information out of it. The general approaches used here can be adapted to other datasets if they can be converted to native Python datatypes first.

If you want to follow along, and dont already have your own Active Directory environment to query with the tool, you can set one up using something like [DetectionLab](https://github.com/clong/detectionlab) or [GOAD](https://github.com/Orange-Cyberdefense/GOAD).

Now, lets get to it. The output from the AD dumping tool is in JSON format, so to properly process it using iPython we need to convert it into a native Python format, which we can do using the `json` module.

Import the json module if you havent done so already.

```
In [54]: import json
```

Now import data from the output of the tool. This particular file contains a LDAP dump from one of my AD test environments.

```
In [55]: ad_data = json.load(open('20230719062434_DCAC.TBDPW.LOCAL_AD_Dump.json'))
```

Now we have the data in the `ad_data` variable, lets explore it to try and understand what data is represented and how we can get useful information out of it.

```
In [57]: type(ad_data)
Out[57]: dict
```

The data is a dictionary (`dict`) type. This data type is essentially like a lookup table, with values indexed under keys. Lets start by checking what the keys are.

```
In [58]: ad_data.keys()
Out[58]: dict_keys(['schema', 'containers', 'computers', 'domains', 'forests', 'gpos', 'groups', 'ous', 'trusted_domains', 'users', 'info', 'meta'])

```

So we have 12 different keys in this dictionary. Based on the key names, we could infer that a particular category of information is included under each of these keys. Lets explore further.

Now lets run a dictionary comprehension to see the type of each subobject associated with each key in the dictionary, and its length. For list objects this will be the number of items in the list, and for dict objects it will be the number of keys.

```
In [62]: {a: [type(ad_data[a]), len(ad_data[a])] for a in ad_data.keys()}
Out[62]:
{'schema': [list, 1772],
 'containers': [list, 124],
 'computers': [list, 7],
 'domains': [list, 1],
 'forests': [list, 1],
 'gpos': [list, 3],
 'groups': [list, 55],
 'ous': [list, 3],
 'trusted_domains': [list, 0],
 'users': [list, 8],
 'info': [dict, 11],
 'meta': [dict, 7]}
```

The above output shows that, for example, the subobject at `ad_data['users']` is a list, and contains 8 items.

This comprehension is a little more complex than the examples used so far in this post, so it might help to break it down a little. 

We are using the dictionary comprehension syntax here, which is as follows:

```
{key : value for item in iterable}
```

The comprehension is wrapped in curly braces `{}`, indicating that this will either return a `set` or a dictionary (`dict`), depending on the specific syntax _within_ the comprehension. This format has both a key and a value, so its a dictionary.

The iterable is `ad_data.keys()`, and the item is `a`, which means that for each iteration of the loop expressed in this comprehension, `a` will be set to a key from `ad_data`.

The `key` in this example is just `a`, so this creates a new dictionary with the same keys as `ad_data`. 

The `value` is `[type(ad_data[a]), len(ad_data[a])]`. This creates another list, where the first item is the `type` of  the value in `ad_data[a]`, and the second is the length of the value in `ad_data[a]`.

The intent of this example is to demonstrate how you can use comprehensions in iPython to explore complex data sets, to understand them and ultimately understand how to parse the data to extract useful information.

Given the context that this is Active Directory infomation, we can infer certain things about the data represented here, e.g. that users are likely contained in the `users` key, and there are 8 of them.

Lets look at the first object in the list of users to see what type of object it is:

```
In [66]: type(ad_data['users'][0])
Out[66]: dict
```

So its another dictionary. Lets look at the keys:

```
In [67]: ad_data['users'][0].keys()
Out[67]: dict_keys(['objectClass', 'cn', 'sn', 'givenName', 'distinguishedName', 'instanceType', 'whenCreated', 'whenChanged', 'displayName', 'uSNCreated', 'memberOf', 'uSNChanged', 'nTSecurityDescriptor', 'name', 'objectGUID', 'userAccountControl', 'badPwdCount', 'codePage', 'countryCode', 'badPasswordTime', 'lastLogoff', 'lastLogon', 'pwdLastSet', 'primaryGroupID', 'objectSid', 'accountExpires', 'logonCount', 'sAMAccountName', 'sAMAccountType', 'userPrincipalName', 'objectCategory', 'dSCorePropagationData', 'mS-DS-ConsistencyGuid', 'lastLogonTimestamp', 'userAccountControlFlags', 'nTSecurityDescriptor_raw', 'domain', 'domainShort'])
```

These keys are representative of the LDAP fields for this type of object. Lets look at `sAMAccountName`, which is used to identify logon names:

```
In [68]: ad_data['users'][0]['sAMAccountName']
Out[68]: 'tester'
```


Now lets grab the value for this field for each user object in which its defined.

```
In [69]: [a['sAMAccountName'] for a in ad_data['users'] if 'sAMAccountName' in a]
Out[69]:
['tester',
 'unpriv',
 'attacker',
 'MSOL_925bb72bf36b',
 'krbtgt',
 'vagrant',
 'Guest',
 'Administrator']
```

It turns out that the `sAMAccountName` field is defined for every user object in this case, so the `if` statement above is not strictly necessaryhere. Its good to know how to add this qualification to stop these comprehensions from failing in cases where the field is not present, however.


What about another field - `userAccountControlFlags`
```
In [74]: ad_data['users'][0]['userAccountControlFlags']
Out[74]: ['NORMAL_ACCOUNT', 'DONT_EXPIRE_PASSWORD']
```

In the context of the LDAP dumping tool, this field is actually a parsed value, containing an interpreted list of the different flag values that are set in the `userAccountControl` field in the user account.

What about we try and get a list of all the unique values in this field across all the users in our data set? Another way of stating this is, what are all the configured user account control flags that are set for users in this Active Directory environment?

Heres how that comprehension would look:

```
In [77]: set([b for a in ad_data['users'] if 'userAccountControlFlags' in a for b in a['userAccountControlFlags']])
Out[77]: {'ACCOUNTDISABLE', 'DONT_EXPIRE_PASSWORD', 'NORMAL_ACCOUNT', 'PASSWD_NOTREQD'}
```

This features yet another twist on our comprehension syntax, as we are adding another nested loop into the mix.

We first iterate through each user object in `ad_data['users']` and assign this to `a`, then we filter for values of `a` that have the `userAccountControl` field set.

THEN, we add a new iterator, where we assign items from each of the the lists `a['userAccountControlFlags']` to variable `b`. 

We also use the variable `b` as the first value referenced in the comprehension, _because thats the value we want to return_. 

So this `b` is set at the end of the comprehension's code, but referenced at the start because we want to access it.

Finally we wrap the whole thing in `set()` to get only the unique values from the whole comprehension.

Now we can look at these flag values and query for accounts that have particular flags.

Lets look at accounts that are disabled (e.g. have the `ACCOUNTDISABLE` flag).

Here we use conditions in the `if` part of the comprehension to only return users who have the `ACCOUNTDISABLE` value in the `userAccountControlFlags` list, and we return the `sAMAccountName` logon name for those accounts.

```
In [82]: [a['sAMAccountName'] for a in ad_data['users'] if 'userAccountControlFlags' in a and 'ACCOUNTDISABLE' in a['userAccountControlFlags']]
Out[82]: ['krbtgt', 'Guest']
```

Hopefully this provides a good example of how to explore complex data in iPython.



# Other specific use cases

Some other examples of how to use iPython to do security related tasks are included below.

## Nmap

I wrote a blog post back in 2016 about how to use the `libnmap` module to parse NMap scan files and extract useful information from them using list comprehensions [here](https://thegreycorner.com/2016/04/30/list-comprehension-one-liners-to.html)


## Burp

[This Burp extension](https://github.com/stephenbradshaw/BurpPythonGateway) that I wrote makes Burp internals of the running session available to Python. 

The readme has examples of how to look at the site map, the proxy history, and get the contents of requests and responses.