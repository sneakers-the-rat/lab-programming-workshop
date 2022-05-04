# Tests

There are lots of "[types of testing](https://www.atlassian.com/continuous-delivery/software-testing/types-of-software-testing)"
and they are probably "important," but I think it's easier to start thinking about
tests as provable statements about what your code does. 

## 

## Writing Testable Code

* always terminate with an else: that raises an error if you're doing a switch case statement.
* If something isn't right, you *should* raise an error rather than just letting it pass silently.