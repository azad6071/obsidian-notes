Attach people to the outcome, not the product
When thinking of combining two classes/Modules, think
	Can one have multiple instances? If yes then we can't combine.
Example: Selection and Cursor. 
The combined object **didn't simplify the interface**; it made it harder (you had to extract cursor info through a boolean test)

``` python

class SelectionAndCursor:
    def __init__(self, start: int, end: int, cursor: int, isSelection: bool):
        self.start = start
        self.end = end
        self.cursor =  end
        self.isSelection = isSelection
        
```
