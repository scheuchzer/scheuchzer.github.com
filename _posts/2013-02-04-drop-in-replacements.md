---
layout: post
title: "Dangerous drop-in replacement Jars"
description: "Things to consider when you're replacing Jar files without compiling"
category: 
tags: [compile, java, library]
---
{% include JB/setup %}

Have you ever put an updated jar into your classpath, eg. a new version of
a library for your webapp? In this post I'd like to show you something you should
be aware of, especially if you get strange behavior after a simple jar update. 

## The riddle
Create an interface:

    public interface MyInterface {
		String GREETING = "Hello World";
	}
    
Now we compile the interface and pack it into a jar.

	> javac MyInterface.java
	> jar -cf lib.jar MyInterface.class
	
Next is a simple demo application

	public class App {
		public static void main(String[] args) {
			System.out.println("Greeting: " + MyInterface.GREETING);
		}
	}
	
The app gets compiled using the library-jar we created before and pack it.

	> javac -cp lib.jar App.java
	> jar -cf app.jar App.class
	
Now that the demo app is ready we start it with the command:

	> java -cp app.jar:lib.jar App
	
What a suprise, the output is as expected:

	Greeting: Hello World
	
Now the fun starts. Edit the MyInterface, change the value of GREETING.

	public interface MyInterface {
        String GREETING = "Good Bye";
	}

Recompile and repack our interface

	> javac MyInterface.java
	> jar -cf drop-in.jar MyInterface.class

And now, here comes the million dollar question! What will we get if we start
our app with the new drop-in-jar instead of the old lib.jar in the classpath?

	> java -cp app.jar:drop-in.jar App
	
## Suggestions? 
What do you guess? `Greeting: Good Bye`? Wrong! We still get `Greeting: Hello World` even
though our interface in the classpath says good bye!

## What did just happen?
For optimization reasons the java compiler writes our constant value directly into
the App bytecode. Java doesn't read it from the interface at runtime! Therefore, the
value from the drop-in jar is never read at runtime.

## Conclusion
When using drop-in replacement jars keep in mind that changed constant values of the
library will only make it into the runtime if you recompile your code!

