---
title: The Folly of Try-With-Resources in Java 7
summary: I make the case for why the try-with-resources feature in Java 7 is a waste of time and development effort.
categories: journal
tags:
- java
- java 7
legacyId: 620
legacydate: 1/23/2011 16:54:00
layout: post
---

Java 7, which is slated to be released ninety years ago, is supposed to add a feature as part of the [Project Coin initiative](http://openjdk.java.net/projects/coin/) called "try-with-resources". The idea is fairly simple, and is intended to tidy up a common development problem in Java: the safe life-cycle management of volatile resources.

In concept I have no problem with this sort of support. There are innumerable examples on the web of absurd Java try/catch/finally block hierarchies to properly managing input streams and output streams; something that is the norm, not the exception to the rule.

{% highlight java %}
InputStream in;
try {
  in = loadInput(...);
    OutputStream out;
  try {
    out = createOutput(...);
    copy(in, out);
  }
  finally {
    if(out != null) {
      try {
        out.close();
      }
      catch(IOException ioe) {
        // Problem closing the output stream.
      }
    }
  }
}
catch(IOException e) {
  // Problem reading or writing with streams.
  // Or problem opening one of them.
}
finally {
  if(in != null) {
    try {
      in.close();
    }
    catch(IOException ioe) {
      // Problem closing the input stream.
    }
  }
}
{% endhighlight %}

I'm not even showing here how you handle the situation where reading/writing caused an error, **and** closing one or both of the streams caused an error. You want a way to cleanly collect all of these up into one exception bundle that can be properly logged/handled, but short of rolling your own wrapper exception to potentially hold all conditions, there simply is no way.

This abhorrent pile of braces can be cleanly tightened up with the new facilities in Java 7 like so:

{% highlight java %}
try (InputStream in = loadInput(...); 
     OutputStream out = createOutput(...) ){
  copy(in, out);
}
catch (Exception e) {
  // Problem reading and writing streams.
  // Or problem opening one of them.
  // If compound error closing streams occurs, it will be recorded on this exception 
  // as a "suppressedException".
}
{% endhighlight %}

Java will know how to close items in the try parenthesis by those elements implementing the "AutoCloseable" interface. This is much like the "Iterable" interface that was added to support the enhanced-for loops in Java 5.
Clearly this code is much cleaner, and automatic resource management is a facility that Java has needed (for a *lonnnnnngg* time) - so why am I miffed?

This feature, which is consuming a non-trivial amount of draft specification time and development time to get right, is being implemented as a language feature via the use of compiler changes - again, much in the same way as enhanced-for was implemented. For the record, I was miffed about enhanced-for as well (although I do use it regularly).

Adding this as a language feature, rather than a library feature, bloats the compiler specification and implementation, and completely hides the real implementation in generated code. It's functional to a point, but is not extensible by developers. Sadly, none of this would be necessary at all if Java 7 shipped with lambda/closure support, and I think the syntax is more natural. Here is a naive Ruby file copy/transform algorithm.

{% highlight java %}
begin
  File.open("myInput.txt", "r") do |in|
    File.open("myOutput.txt", "w") do |out|
      in.each_line do |line|
        out << line.upcase << "\n"
      end
    end
  end
rescue exc
  # Exception occurred reading/writing!
end
{% endhighlight %}

Now, admittedly the syntax is very different than the Java version - the most notable difference is that the Ruby version is actual three nested lambas, where as the Java version is sort-of two lambdas (one to open the resources, one to work with them). I say sort-of, because the first is this special quasi-code-block that also magically accumulates opened resources for later closure.

I find the lambda version to be much less hand-wavy - the syntax is naturally intuitive and more flexible. I should also note that the inner-most lambda, which iterates over lines in the file, shows that the enhanced for loop in Java is completely unnecessary as well if you have lambdas.

Using one of the proposed syntaxes for Java lambda support (as ugly as they can be), let's take a shot at this same idea with Java:

{% highlight java %}
try {
  File.read("myInput.txt", { in ->
    File.write("myOutput.txt", { out ->
      in.eachLine( { line ->
        out.write(line.toUpperCase() + "\n");
      });
    });
  });
}
catch(IOException e) {
  // handle as appropriate. Can still have suppressed exceptions on it.
}
{% endhighlight %}

This is just one way this could be implemented; because it only relies on functions, how expressive the APIs are is up to the library designer, *not* the language designer. An important distinction. Why build special scenarios into the compiler when one language feature can provide the flexibility you need?

Now - is this as short and tab-friendly as the Java 7 feature? Admittedly no. However, the goal of the try-with-resources is not to save on typing, but rather, to ensure resource management is as accurate as possible, relying as little as possible on the developer to dial-in boilerplate, and error-prone exception management.

Here is a short run-down of the library changes that would be required to support this particular code:

* `java.io.File` needs a new method called `read` which takes a String file name, and also takes a function that accepts a `java.io.FileReader`, and returns nothing. The code block may throw an `IOException`. This method does all of the resource management for the `FileReader` it provides to the function. If the code block throws an exception and closing the file reader also throws an exception, the method would add the close exception to the main exception as a suppressedException (just like the language feature would).
`java.io.File` also needs a new method called `write` which takes a String file name, and also takes a function that accepts a `java.io.FileWriter`, and again, returns nothing. This can also throw and `IOException`, and just like the read method, it handles all of the minutia of exception suppression and resource-closing.
`java.io.FileReader` needs a new method called `eachLine` that simple scans for line-terminators and lets a provided function see the string representation of each line. The function may throw `IOException`, and the method simply lets IOException propagate.

Just to circle back around - Java today could support the "eachLine" method on FileReader via enhanced-for loop, if the "eachLine" method returned an `Iterable<String>` (as opposed to accepting a block of code) - it could look something like this:

{% highlight java %}
for(String line : in.eachLine()) {
  out.write(line.toUpperCase() + "\n");
}
{% endhighlight %}

Sadly, this feature will be here with Java 7, and lambdas will not. As such, it's yet another way to buy time without lambdas, and look a little better next to the C# features list.