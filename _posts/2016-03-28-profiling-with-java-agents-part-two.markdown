---
layout: post
title:  "Profiling with Java Agents - Part 2"
date:   2016-03-28 12:00:00 +0530
categories: java
description: Profiling with Java Agents
comments: true
---
In-case you haven't read Part 1, you can find it [here][Part1].

### Introduction
Let's look at the Java Pyramid program we will be profiling. We will assume we do not have access to the source code. The program gives us the following output.
{% highlight text %}
   1
  1 1
 1 1 1
1 1 1 1
    1
   1 1
  1 1 1
 1 1 1 1
1 1 1 1 1
     1
    1 1
   1 1 1
  1 1 1 1
 1 1 1 1 1
1 1 1 1 1 1
{% endhighlight %}

On running our profiler with the Java Pyramid program we get the following metrics
{% highlight text %}
========= Method Count Metrics =========
com.oliver.printpyramid.PrintPyramid.printPyramid-->3
com.oliver.printpyramid.PrintPyramid.main-->1
com.oliver.printpyramid.PrintPyramid.getLine-->15
========= End Method Count Metrics =========
{% endhighlight %}
Now we have some insight into which methods are being called by the Pyramid program. 


This basic idea can be extended to record other metrics like method execution time, the results of which can be used to ease out performance bottlenecks.

#### Motivation
Java development tools like [__JRebel__][JRebel] and [__XRebel__][XRebel] as well as Application Monitoring tools like [__New Relic APM__][New-Relic] are built upon similar albeit more complicated concepts of Java agents and byte-code instrumentation.

### Let's Begin
In sections below will look at:

1. A quick overview of [`Javassist`][Javassist]. 
2. Define the Metrics Collector Interface.
3. Implement and Register a custom Class File Transformer.
4. Generate a Fat Jar i.e. a jar with dependencies.


### A quick overview of Javassist
_`Javassist` is a class library for editing bytecodes in Java; it enables Java programs to define a new class at runtime and to modify a class file when the JVM loads it._

![Javassist Components]({{ site.url }}/images/jast.png)
_[Image Source][JAST-source]_

The main components of Javassist as indicated by the image above consist of

- CtClass
- ClassPool
- CtMethod

#### CtClass
A `CtClass` object is an abstraction of a `class` file. In order to manipulate the byte-code of a `class` we will have to obtain its `CtClass` representation from the `ClassPool`.

#### ClassPool
The `ClassPool` object holds multiple `CtClass` objects.

#### CtMethod
A `CtMethod` object is an abstraction of a method in a `class`. Once we obtain the `CtMethod` representation of a method we can manipulate its byte-code using the various methods present. In this example we will use the `insertBefore` method to add our instrumentation code.

_Note: For a more complete and detailed explanation on Javassist please see references 2 and 3._

### The Metrics Collector Interface
Here is the `MetricsCollector` interface which will be instrumented into the Java Pyramid program. The implementation of this interface is pretty straightforward and can be followed through with just the javadoc comments.
{% highlight java %}
package com.oliver.jagent.mectrics;

public interface MetricsCollector {

	/**
	 * This method is invoked at the beginning of every method call, it increments
	 * the counter associated with the the <code>methodName</code> param.
	 * 
	 * @param className
	 * @param methodName
	 */
	public void logMethodCall(String className, String methodName);

	/**
	 * This method registers the names of the methods which will invoke
	 * logMethodCall() on every call.
	 * 
	 * @param className
	 * @param methodName
	 */
	public void registerMethod(String className, String methodName);

	/**
	 * Prints out the recorded Metric values
	 * 
	 * @return
	 */
	public String printMetrics();
}

{% endhighlight %}

- `registerMethod()` will be invoked by our agent code to register all the method names that will be instrumented.
- `logMethodCall()` will be instrumented into the Java Pyramid program code via byte-code modification so that it is invoked every-time a method of the Java Pyramid program is invoked.
-  `printMetrics()` will be invoked by our agent code on shut-down and will print out the metrics.

### Registering a custom Class File Transformer
In the [first post][Part1] we implemented the `premain` method and displayed a "Hello World! Java Agent" message. In order to modify byte-code we need to write and register a custom `ClassFileTransformer`. This is done by using the `addTransformer` method of the `java.lang.instrument.Instrumentation` interface. 

{% highlight java %}
package com.oliver.jagent;

import java.lang.instrument.Instrumentation;

public static void premain(String agentArgs, Instrumentation inst){
		inst.addTransformer(new MyClassTransformer());
}
{% endhighlight %}

### Implementing MyClassTransformer
The `MyClassTransformer` class implements the `ClassFileTransformer` interface and the `transform` method. We have initialized a `ClassPool` with the default class pool, this is fine when running a simple application like the Java Pyramid program in this example. However for applications running on web application servers like Tomcat and JBoss which use multiple class loaders, creating multiple instances of `ClassPool` might be necessary; an instance of `ClassPool` should be created for each class loader. You may find mode details on this [here][javassist-tutorial].
{% highlight java %}
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import javassist.ClassPool;

public class MyClassTransformer implements ClassFileTransformer {

  private ClassPool pool;

  public MyClassTransformer() {
	this.pool = ClassPool.getDefault();
  }	

  public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,
	ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
	...
	}
}			
{% endhighlight %}

#### Filtering out Classes we do not intent to modify
Our agent code will intercept all the classes to be loaded by the VM including the its own classes, hence we need to filter out these classes in the transformer and return their byte-code without any modification. The following code snippet does just this.
{% highlight java %}
byte[] modifiedByteCode = classfileBuffer;
String clazzName = className.replace("/", ".");
//Skip all agent classes
if (clazzName.startsWith("com.oliver.jagent")) {
	return classfileBuffer;
}
//Skip class if it doesn't belong to our Java Pyramid program
if (!clazzName.startsWith("com.oliver.printpyramid")) {
	return classfileBuffer;
}
{% endhighlight %}

#### Adding the instrumentation code
Now that we have filtered out the unwanted classes it's time to add our instrumentation code.
The following code snippet will obtain a `class` representation from the `ClassPool`, iterate over all the methods presents in the class and use `insertBefore()` to add our instrumentation code at the start of every method.
{% highlight java %}
private MetricsCollector collector = MetricsCollectorImpl.instance();
...
//Records a package name so that the Javassist compiler may resolve a class name.
//This enables the compiled to resolve the class "MetricsCollectorImpl" below
pool.importPackage("com.oliver.jagent.mectrics");
//Retrieve the class representation i.e. CtClass object
CtClass cclass = pool.get(clazzName);
		
for (CtMethod method : cclass.getDeclaredMethods()) {
	//Register the method with our MetricsCollector
	collector.registerMethod(clazzName, method.getName());
	//Insert the instrumentation code at the start of the method.
	method.insertBefore("MetricsCollectorImpl.instance().logMethodCall(\"" + clazzName + "\",\"" + method.getName() + "\");");
}
{% endhighlight %}
_Note: Exception handling has been omitted for brevity._

### Generating a Fat JAR
In this example, we have used `Javassist` which is a third-party library. When running this version of the agent you will need to specify a path to this jar as well. In order to keep things simple, tools like JRebel create a "fat jar" or "jar with dependencies", which uses a single JAR file. This can be achieved using the `maven-assembly-plugin`.

{% highlight java %}
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-assembly-plugin</artifactId>
	<version>2.6</version>
	<configuration>
	  <descriptorRefs>
	    <descriptorRef>jar-with-dependencies</descriptorRef>
	  </descriptorRefs>
	  <archive>
	    <manifestFile>src/main/resources/META-INF/MANIFEST.MF</manifestFile>
	  </archive>
	</configuration>
	<executions>
	  <execution>
	    <id>assemble-all</id>
	    <phase>package</phase>
	    <goals>
		 <goal>single</goal>
	    </goals>
	  </execution>
	</executions>
</plugin>
{% endhighlight %}
Similar to Part 1, `<manifestFile>` contains the path to our custom manifest file, no changes have been made there except a change in the agent class name.
{% highlight java %}
Manifest-Version: 1.0
Premain-Class: com.oliver.jagent.Agent
{% endhighlight %}

The agent is run in exactly the same way as described in [Part 1][Part1].

### Conclusion
We have created a basic java profiler which gives us insights into the inner workings of applications and have run the profiler as a java agent.
 

### References
1. [Java Instrumentation Package][java-instrumentation]
2. [Javassist][Javassist]
3. [Diving into Bytecode Manipulation: Creating an Audit Log with ASM and Javassist][ByteCodeManipulation].
4. [Callspy][callspy] -  A simple trace agent.


[Part1]:{% post_url 2016-03-22-profiling-with-java-agents-part-one %}
[JRebel]:https://zeroturnaround.com/software/jrebel/
[XRebel]:https://zeroturnaround.com/software/xrebel/
[java-instrumentation]:https://docs.oracle.com/javase/7/docs/api/java/lang/instrument/package-summary.html
[Javassist]:http://jboss-javassist.github.io/javassist/
[ByteCodeManipulation]:https://blog.newrelic.com/2014/09/29/diving-bytecode-manipulation-creating-audit-log-asm-javassist/
[JAST-source]:http://blog.newrelic.com/wp-content/uploads/BM4.png
[New-Relic]:http://newrelic.com/java
[javassist-tutorial]:https://jboss-javassist.github.io/javassist/tutorial/tutorial.html
[callspy]:https://github.com/zeroturnaround/callspy