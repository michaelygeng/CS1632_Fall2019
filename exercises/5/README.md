# Exercise 5 - Static Analysis

In this exercise, you will use three static analysis tools to test a program: a
bug finder named SpotBugs, a linter named CheckStyle, and a model checker named
Java Pathfinder (JPF).  SpotBugs and CheckStyle work in similar ways in that
both look for patterns that are either symptomatic of a bug (former) or are bad
coding style (latter).  So we will look at them together first.  Later we will
look at JPF which is much more rigorous and can prove a program correct.

## SpotBugs and CheckStyle

SpotBugs: https://spotbugs.github.io/  
CheckStyle: https://checkstyle.sourceforge.io/  

Try running both tools on a Sieve of Eratosthenes program, and then fix any
issues found.  This will allow you to see what kinds of bugs a static analysis
program can find (and which ones it cannot).

The Sieve of Eratosthenes is an ancient way of finding all prime numbers below
a specific value.  For details on the algorithm itself, please see
https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes.

This program accepts one integer value and will tell you all prime numbers up
to and including the passed-in value.  However, there are some problems hidden
in the code.  You are going to use SpotBugs and CheckStyle to find and fix
them.  Some problems are actual defects and some are just bad or confusing
code.

I have prepared scripts to run or test the program.  First cd into the Sieve
directory before executing the scripts.

To run the program (for Windows users):
```
$ run.bat [Integer]
```
To run SpotBugs:
```
$ runSpotbugs.bat
```
To run CheckStyle:
```
$ runCheckstyle.bat
```

For Mac or Linux users, please run the corresponding .sh scripts.

* There is a GUI for SpotBugs if that is what you prefer.  You can launch the GUI by using the following command:
```
$ java -jar spotbugs-4.0.0-beta4/lib/spotbugs.jar
```
The following link contains a short tutorial on how to use the GUI:
https://spotbugs.readthedocs.io/en/latest/gui.html

If all goes well you should see the following output:

```
$ java Sieve 100
Sieve of Eratosthenes
> 2 3 5 7 11 13 17 19 23 29 31 37 41 43 47 53 59 61 67 71 73 79 83 89 97

$ java Sieve 14
Sieve of Eratosthenes
> 2 3 5 7 11 13
```

Note that there is a bug in the logic of the code that is not caught by either
SpotBugs or CheckStyle that will prevent you from getting the above output.
Since both work by pattern matching they are bad at finding logic problems.
That is part of the lesson you should learn from this exercise.

## Java Pathfinder (JPF)

Java Pathfinder (JPF): 
https://github.com/javapathfinder/jpf-core/wiki, http://javapathfinder.sourceforge.net/

Java Pathfinder is a tool developed by NASA to model check Java programs.  It
works in exactly the same way we learned in class: it does an exhaustive and
systematic exploration of program state space to check for correctness.

### JPF on Rand

Let's first try out JPF on the example Rand program we saw on the Formal Verification lecture
slides:  

<img src="jpf.png" width="50%" height="50%">

First cd into the DrunkCarnivalShooter directory before executing the scripts.

To run the Rand program (for Windows users):
```
$ runRand.bat
```
To run JPF with Rand:
```
$ runJPFRand.bat
```

For Mac or Linux users, please run the corresponding .sh scripts.

When you run Rand with JPF, you can see from the output that it goes through all possible states, thereby finding the two states with division-by-0 exception errors (I configured JPF to find all possible errors).  Also, when you run JPF, you will get a file named jpf-state-space.dot that is a graph representation of the states that you have traversed.  The file is in DOT format that is viewable from the Graphviz viewer.  There is an online version here: http://graphviz.it/.

All you have to do is open the jpf-state-space.dot on a text editor and copy-paste it to the website.  Then, you should see a diagram very similar to the one shown above.  Don't pay attention to the source code line numbers.  There seems to be a bug in the JPF source code line calculation code.

### JPF on DrunkCarnivalShooter

Now let's try using JPF to debug and verify a real program.  DrunkCarnivalShooter is a simple text-based game where the player goes to a carnival shooting range and tries to win the prize by shooting all 4 provied targets.  The player can designate what target to shoot for pressing 0-3.  But since the player is drunk, there is an equal chance of the player shooting left or right as shooting straight.  Refer to the file [sample_run.txt](sample_run.txt) for an example game play session.

To run the DrunkCarnivalShooter program (for Windows users):
```
$ run.bat
```
To run JPF with DrunkCarnivalShooter:
```
$ runJPF.bat
```

For Mac or Linux users, please run the corresponding .sh scripts.

You may not notice the bug after playing the game once or twice but it is definitely in there.  The JPF tool also doesn't show any errors but that is because DrunkCarnivalShooter takes user input and JPF does not know how to handle it.  Just like random numbers, we would like to have JPF to go over every possibility.  We will do that by using the Verify API.  But before we do, we first have to import a library in order to be able to use that feature at the top of DrunkCarnivalShooter.java:
```
import gov.nasa.jpf.vm.Verify;
```
Now instead of scanning user input using the following statement:
```
int aimedTargetNum = scanner.nextInt();
```
Exhaustively generate all possible inputs using the Verify API:
```
int aimedTargetNum = Verify.getInt(0, 3);
```
The above will direct JPF to generate 4 states each where aimedTargetNum is set to 0, 1, 2, or 3 respectively.  Then it will systematically explore each state.  If you wish, you can test a larger set of numbers beyond 0-3.  It is just going to generate more states and take longer.

Now let's try running runJPF.bat one more time.  This will trigger state space exploration and you will quickly be able to find the error states as well as the exception stack traces leading up to the bug.  Fix the bugs.

Once you fix the bugs, you will get state explosion.  That is, when you run runJPF.bat, JPF is going to fall into an infinite loop and generate an infinite number of states (observed by the ever increasing Round number).  There is no limit to the number of rounds a player can play, hence the explosion.  How can I deal with this explosion and still verify our program?

We have to somehow narrow down the amount of state we test, or we will be forced to but JPF off after testing only a limited set of rounds.  Let's say the state that we are really interested in relation to the specifications is the state of the 4 targets.  Now if you think about it, the 4 targets can only be in a handful of states: 2 * 2 * 2 * 2 = 16 states (standing or down for each).  And this is true no matter how many rounds you do.  The only thing that constantly changes every round is the round number --- and that is the culprit leading to the state explosion.  So, let's filter that state out!

Import the appropriate JPF library at the top of DrunkCarnivalShooter.java again:
```
import gov.nasa.jpf.annotation.FilterField;
```
And now, let's annotate roundNum such that it is filtered out:
```
@FilterField private static int roundNum;
```

Now if we run runJPF.bat again, JPF will only go up to Round #4 and stop and declare no errors detected.  How is it able to do this?  This is because all 16 states can be covered in the space of 4 rounds.  Any further number of rounds will result in a match with an already visited state and therefore will not need to be explored.  You can see the backtracked count in the final summary statistics and observe that a lot of backtracking happened due to this.  You can also try viewing the jpf-state-space.dot file and see for yourself that a lot of transitions ended up on the same state (match).

## Submission

Please do a Text Submission to Courseweb with a link to the GitHub repository where you stored it, along with names of all group members.

Example:

John Doe  
Jane Doe  
https://github.com/wonsunahn/CS1632_Fall2019/tree/master/exercises/5

Please submit by Monday (11/13) 11:59 PM to get timely feedback.

IMPORTANT: Please keep the github private and add the following users as collaborators: nikunjgoel95, wonsunahn