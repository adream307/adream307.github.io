---
title: bash 判断字符串是否包含另一个字符串
author: adream307
date: 2020-11-01 15:20:00 +0800
categories: [linux, bash]
tags: [bash]
---

[how to check if a string contains a substring in bash](https://stackoverflow.com/questions/229551/how-to-check-if-a-string-contains-a-substring-in-bash)


You can use [Marcus's answer (* wildcards)](https://stackoverflow.com/a/229585/3755692) outside a case statement, too, if you use double brackets:

```bash
string='My long string'
if [[ $string == *"My long"* ]]; then
  echo "It's there!"
fi
```

Note that spaces in the needle string need to be placed between double quotes, and the `*` wildcards should be outside. Also note that a simple comparison operator is used (i.e. `==`), not the regex operator `=~`.


If you prefer the regex approach:

```bash
string='My string';
if [[ $string =~ "My" ]]
then
   echo "It's there!"
fi
```
