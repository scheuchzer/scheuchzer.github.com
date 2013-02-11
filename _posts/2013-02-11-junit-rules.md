---
layout: post
title: "JUnit Rules!"
description: "Better tests with @Rules"
category: Java
tags: [JUnit, Java, Testing]
---
{% include JB/setup %}

Some weeks ago at Devoxx in Belgium I heard about the JUnit feature called Rules.
The feature got introduced without causing a stir but it's exactly what I was
waiting for!

	package org.junit.rules;
	
	import org.junit.runner.Description;
	import org.junit.runners.model.Statement;
	
	public interface TestRule {
		/**
		 * Modifies the method-running {@link Statement} to implement this
		 * test-running rule.
		 *
		 * @param base The {@link Statement} to be modified
		 * @param description A {@link Description} of the test implemented in
		 *        {@code base}
		 * @return a new statement, which may be the same as {@code base},
		 *         a wrapper around {@code base}, or a completely new Statement.
		 */
    	Statement apply(Statement base, Description description);
	}

Well, I have to admit that when I looked at this interface for the first time I had
clue what this can be useful for. Looking at the implementations shipped with JUnit
made things clear. It's just perfect for solving the test-case inheritance mess 
and `@Before` and `@After` stuff that makes tests less readable. Rules reduce
your tests code, setup and teardown code can be moved into separate classes.

## An example

Itegrationtests so far:

	@Test
	public void find() throws Exception {
	  // Assemble
	  PersonDao dao = new PersonDao();
	  Person expectedPerson = new Person("Dummy");
	  dao.save(expectedPerson);
	  try {
	    // Activate
	    Person actualPerson = dao.find(expectedPerson.getName());
	    // Assert
	    assertThat(actualPerson, equalTo(expectedPerson));
	  } finally {
	    // Cleanup
	    dao.delete(expectedPerson);
	  }
	}
  
and pimped with @Rules:

	@Rule
	public PersonRule personCreator = new PersonRule();
	
	@Test
	public void find() throws Exception {
	  // Assemble
	  Person expectedPerson = personCreator.createPerson("Dummy");
	  PersonDao dao = new PersonDao();
	  // Activate
	  Person actualPerson = dao.find(expectedPerson.getName());
	  // Assert
	  assertThat(actualPerson, equalTo(expectedPerson));
	}

Try-catch has been disappeared! And best is, it didn't even move into the `PersonRule`!

	import java.util.ArrayList;
	import java.util.List;
	import org.junit.rules.ExternalResource;
	
	public class PersonRule extends ExternalResource {
	
	  private List<Person> persons = new ArrayList<>();
	
	  @Override
	  protected void after() {
	    System.out.println("Cleaning up");
	    for (Person p : persons) {
	      delete(p);
	    }
	  }
	
	  private void delete(Person p) {
	    System.out.println(String.format("Delete %s from database", p));
	    new PersonDao().delete(p);
	  }
	
	  public Person createPerson(String name) {
	    System.out.println(String.format("Create %s in database", name));
	    Person p = new Person(name);
	    persons.add(p);
	    new PersonDao().save(p);
	    return p;
	  }
	}
	
As you can see the code implements an `after()` method where the cleanup is done. The
base class is implemented in a way that the `after()` method gets called after
each test exactly as methods annotated with `@After`are. 
 
## There's more
 
JUnit comes with some handy base classes providing template methods for different purposes. One 
of these base classes is `ExternalResource` which provides the template methods 
`before()` and `after()`. Working with the base classes is much easier than 
implementing the `TestRule` on your own. Don't forget to have a look
at the other classes! 
 
- ExternalResource
  > before(), after()
- TestWatcher
  > starting(), succeeded(), finished(), skipped(), failed()
- Verifier
  > verify()
  
Also have a look at the `ExpectedException` Rule which enables you to look
in detail at exceptions thrown by your test. Use it if `@Test(expected=Exception.class)`
is not enough!

	@Rule
	public ExpectedException thrown= ExpectedException.none();
	
	@Test
	public void throwsNullPointerExceptionWithMessage() {
	  thrown.expect(NullPointerException.class);
	  thrown.expectMessage("happened?");
	  thrown.expectMessage(startsWith("What"));
	  throw new NullPointerException("What happened?");
	}
	  
So long, happy testing!
 