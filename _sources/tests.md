# Tests

There are lots of "[types of testing](https://www.atlassian.com/continuous-delivery/software-testing/types-of-software-testing)"
and they are probably "important," but I think it's easier to start thinking about
tests as provable statements about what your code does. 

We make a function that promises that it adds numbers together, we can write tests that assert that it does so!

```python
import numpy as np
def add_weirdly(x, y, constant=2):
    return x + y + 2

repeats = 100
def test_add_weirdly():
    for i in range(repeats):
        x = np.random.randint(0,100000)
        y = np.random.randint(0,100000)
        
        # test the default case of + 2
        assert add_weirdly(x, y) == x + y + 2
        
        # test with some other constant
        constant = np.random.randint(0, 1000000)
        assert add_weirdly(x, y, constant=constant) == x + y + constant

```

## Relentless Positivity

For example, we have a function in the prey capture repository that's suppoosed to
find places where some column of a dataframe is higher than a certain value for a certain window

To test that, we could write something like this:

```python
import numpy as np
import pandas as pd
from prey_capture_python.analysis.prey_cap_metrics import relentless_positivity
n_randomizations = 100

def test_relentless_positivity():
    # make fake data with a period that should match
    thresh = 0.9
    
    # try this with a few different durations
    for i in range(n_randomizations):
        # pick a random window
		window = np.random.randint(10,50)
        # and some random start and end point where the value should go above the threshold
        start = np.random.randint(window*2, window*5)
        end = start + np.random.randint(window*2, window*5)
        
        # make a fake dataset with zeros until `start`, 
        # then `threshold` for an amount that is longer than the window
        # then some more zeros
        data = [*[0]*start, *[thresh]*((end-start)+1), *[0]*window*100]
        df = pd.DataFrame({'data':data})
        
        # run the function
        periods = relentless_positivity(df, 'data', window, thresh)
        
        # assert that we got one period, and that the period markers match our start and end
        assert len(periods) == 1
        assert periods[0][0] == start
        assert periods[0][1] == end

```

Robust tests would have a lot more variations and edge cases, but you get the point!


## Using pytest

example from autopilot!



## Writing Testable Code

* always terminate with an else: that raises an error if you're doing a switch case statement.
* If something isn't right, you *should* raise an error rather than just letting it pass silently.