---
layout: post
title:  "Profiling with Java Agents - Part 1: A Hello World Example"
tags: [java]
subtitle: Simplified Profiling with Java Agents
---
The Java Agent technology has been around since JDK 1.5. Here we will build a simple profiler which will instrument our Java program and give us method execution count metrics. The profiler will be run as a Java Agent.

### Java Agent
A Java agent is started by adding the following command line argument.
{% highlight java linenos%}
-javaagent:jarpath[=options]
{% endhighlight %}
Where `jarpath` is the path to the JAR file.

A Java agent JAR file must conform to the following conventions:

1. The manifest of the JAR must contain the `Premain-Class` attribute, its value is the name of the agent class.
2. The agent class must implement one of the two following methods
- `public static void premain(String agentArgs, Instrumentation inst);`
- `public static void premain(String agentArgs);`

The `premain` method serves as an entry point for the Java agent the same way `main` servers as an entry point to regular Java class.

_Note: A Java agent class may also contain an `agentmain` method which is invoked when an agent is attached to an already running VM._

### "Hello World" Java Agent
Let's use the conventions mentioned above to create a simple "Hello World" implementation.

Create a simple `maven` project. Add the  Apache `maven-jar-plugin` plugin to the `pom.xml`.
{% highlight xml linenos%}
<build>
  <plugins>
    <plugin>
	 <groupId>org.apache.maven.plugins</groupId>
	 <artifactId>maven-jar-plugin</artifactId>
	 <version>2.4</version>
	 <configuration>
	   <archive>
	     <manifestFile>src/main/resources/META-INF/MANIFEST.MF</manifestFile>
	   </archive>
	 </configuration>
   </plugin>
 </plugins>
</build>
{% endhighlight %}
`<manifestFile>` contains the path to our custom `MANIFEST.MF` file. This is the only modification needed to the `pom.xml`.

Create the agent class `HelloWorldAgent`.
{% highlight java linenos%}
package com.oliver.javaagent.helloworldagent;

import java.lang.instrument.Instrumentation;

public class HelloWorldAgent {

	public static void premain(String agentArgs, Instrumentation inst) {
		System.out.println("Hello World! Java Agent");
	}
	
}
{% endhighlight %}

Create `MANIFEST.MF` at src/main/resources/META-INF/ with the following content.
{% highlight java linenos %}
Manifest-Version: 1.0
Premain-Class: com.oliver.javaagent.helloworldagent.HelloWorldAgent

{% endhighlight %}
To test our Agent class let's use the default App class generated by maven. I have added the word "App" to the `sysout` statement. 
{% highlight java linenos %}
package com.oliver.javaagent.helloworldagent;
/**
 * Hello world!
 *
 */
public class App 
{
    public static void main( String[] args )
    {
        System.out.println( "Hello World! App" );
    }
}
{% endhighlight %}

_Note: For the purposes of this example we are using a test class in the same package as the Agent class, however this will work with any class._

Now run a `maven install` to create the agent JAR. Here is a screenshot of the disassembled JAR.

![DataFrame Image]({{ site.url }}/img/jar-screenshot.png)

Run the App class with the `-javaagent` command line argument.
{% highlight java linenos %}
java -javaagent:<path-to-jar-file>/helloworldagent-0.0.1-SNAPSHOT.jar com.oliver.javaagent.helloworldagent.App

Output:
Hello World! Java Agent
Hello World! App
{% endhighlight %}

### Conclusion and Next Steps
In this post we have seen how to create a simple Java Agent. Our agent however doesn't really do much, in the next post we will take a look at the `java.lang.instrument` package and `javassist` which will enable us to manipulate byte-code and profile our Java program.


### References
1. [Java Instrumentation Package][java-instrumentation]

[java-instrumentation]:https://docs.oracle.com/javase/7/docs/api/java/lang/instrument/package-summary.html