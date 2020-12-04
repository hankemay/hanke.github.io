---
layout: post
title: "Markdown Practice"
subtitle: Learn how to use the markdown frequent grammar
header-style: text
author: Hanke
mathjax: true
tags: [Markdown, Editor]
---

It will record the daily markdown grammar.

### Title
```text
# First
## Second
### Third
#### Fourth
##### Fivth
###### Sixth
```

### Fonts
RUNBOOK.COM  
GOOGLE.COM

*italic text*  
**bold text**  
***bold and italic text*** 

### Separator
----

### Delete Line
RUNBOOK.COM  
GOOGLE.COM  
~~BAIDU.COM~~

### Underline
<u>underline</u>

### Footer
Footer [^example]  
Footer [^another]

[^example]: for the footer case
[^another]: for the footer another case

### List
* first
* second
* third

**Another way**
- first
- second
- third

1. first
2. second
3. third

**Nested**
1. first
    * first
    * second
2. second
    - first
    - second

### Block
> Here is block  
> Inside block  
> Put text here  

**Nested Block**
> Here is block
>> Here is the first nested block
>>> Here is the second nested block


**Nested List in Block**
> Here is block
> 1. First
> 2. Second  
> -. First  
> -. Second  

**Nested Block in List**
* First
    > First Block  
    > Second Block
* Second

### Code
```javascript
$(document).ready(function() {
    alert('RUNBOOK');
});
```

### Link
Here is the [Link](https://www.runoob.com/markdown/md-tutorial.html) example.

**Link by Variable**  
- This link use 1 as the google website link variable [Google][1]  
- This link use runbook as the website link variable [Runbook][runbook]   

### Picture
![RUNBOOK Logo](http://static.runoob.com/images/runoob-logo.png)
![RUNBOOK Logo](http://static.runoob.com/images/runoob-logo.png "RUNBOOK")

**Advance picture**  
- This link will use 2 as the website picture link variable [Runbook][2]

[Jump to the formular part](#formular)

### Table
**Must have one empty line before the table**

 header | header
 -      | -          
 content| content   
 content| content   

*left align*  

 header | header
 :-     | :-          
 content| content   
 content| content   

*right align*  

 header | header
 -:     | -:          
 content| content   
 content| content   

*middle align*  

 header | header
 :-:    | :-:          
 content| content   
 content| content   


### HTLM Element
use <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>Del</kbd> Restart the computer  
use <sup> hello </sup>  
use <b> bold </b>   
use <i> italic </i>  
use <em>element</em>  
use <strong>strong</strong>  
use <br>br   
use <sub>sub</sub>  


### Escape
use the escape \\ to escape the special character  
**bold text**  
\*\* display the star \*\*  


### Formular
**in one line**  
  
$\sum_{i=0}^N\int_{a}^{b}g(t,i)\text{d}t$  


**in several lines**  

$$
\begin{bmatrix}
1&0&0 \\
0&1&0 \\
0&0&1
\end{bmatrix}
$$

### Others
`font` CSS

<pre>
wechat
please send a red package
</pre>

### Reference
[learn website](https://www.runoob.com/markdown/md-tutorial.html)

<p id = "build">##Build</p>

[1]: http://www.google.com/
[runbook]: http://www.runbook.com/ 
[2]: http://static.runoob.com/images/runoob-logo.png



