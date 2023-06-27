# Writing Testable Code

I AM JUST WRITING DOWN SOME SCRAPS RIGHT NOW I GOTTA GO TO SLEEP

- No global state! no implicit state! self contained everything!
- Be explicit about requirements: it should be possible to get to a testable state in your code with little setup. If you need to do a lot of state manipulation in order to get to a unit test, you probably need to break up your objects/functions. Rather than having one class create all the necessary things, have it ask for them. It's easier to write compositional code that handles creating that for you than it is taking it out of your class
- Don't mix types of operations: rather than passing a path to a configuration file, pass in the configuration object itself, and then have an additional factory/constructor method to read from a file if needed.
- That said, do very little work in the __init__ - just try and get the object working. Check args, sure, but say you are a webscraping library - don't put the 'connect' call in the __init__ method. 
