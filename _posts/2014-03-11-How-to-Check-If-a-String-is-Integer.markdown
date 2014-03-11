---
layout: post
title:  "How to Check If a String is Integer"
date:   2014-03-11 19:46:32
categories:
---

There are a few ways to check if a string is integer or not.Here I will try to implement some of these methods.
First one is using regular expressions and this is the one I will recommend.This matches as <br> 
-?     --> negative sign, could have none or one <br>
\\d+   --> one or more digits
{% highlight java %}
public boolean isInteger(String str){
  return str.matches("-?\\d+");
}
{% endhighlight %}
Using parseInt is possible as well.In this way you should try to parse a String to integer and catch if any exceptions occur.
If your code creats an NumberFormatException then you should return false.We don't care about other exceptions here.If there is no exception that means it went okey and your string is indeed an integer.
Also please be aware that we used parseInt() and not the Integer.valueOf() method.You can check the differences here.
{% highlight java %}
public boolean isInteger(String str) {
    try {
        Integer.parseInt(str);
        return true;
    }
    catch(NumberFormatException e) {
        return false;
    }
}
{% endhighlight %}
Another way is more complex  but much faster than Integer.parseInt() or regex method.It's better to check which works best for you.
Here some of you may ask why not use java.lang.Character.isDigit() and check the char values itself.isDigit returns true for things like Devanagari digits. While Integer.parseInt() handles these as digits, it may not be what's expected for most use cases

{% highlight java %}
//Jonas
public static boolean isInteger(String str) {
	if (str == null) {
		return false;
	}
	int length = str.length();
	if (length == 0) {
		return false;
	}
	int i = 0;
	if (str.charAt(0) == '-') {
		if (length == 1) {
			return false;
		}
		i = 1;
	}
	for (; i < length; i++) {
		char c = str.charAt(i);
		if (c < '0' || c > '9') {
			return false;
		}
	}
	return true;
}
{% endhighlight %}
The last one is using StringUtils or NumberUtils which is not a good idea but it works.You can check the detailed page <a href="http://commons.apache.org/proper/commons-lang/apidocs/org/apache/commons/lang3/StringUtils.html#isNumeric%28java.lang.CharSequence%29">here</a>.
{% highlight java %}
StringUtils.isNumeric(str);
{% endhighlight %}
Which one is the best to use?If you need performance you should use the Jonas one.It may look unnecessary but it is really fast.
Here some test results with these methods.Running many times these methods shows that it may differ a lot.

{% highlight java %}
ByException - integer data: 45
ByException - non-integer data: 465

ByRegex - integer data: 45
ByRegex - non-integer data: 26

ByJonas - integer data: 8
ByJonas - non-integer data: 2
{% endhighlight %}