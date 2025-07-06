 ## Welcome! 

Interests and focus areas:
- 🖥️ Tech: workflow orchestration (prefect!), data visualization, IaC, image processing/geo-spatial
- 🧬 Biology: human <> pathogen interaction, sequencing technologies, primer/guide/probe design, fermentation
- 🎗️ Missions: climate, medicine, civics 
- 🏠 Personal: home-brewing, cycling, gaming, climbing, home automation (zigbee/z-wave)


💕 Projects that I love:
- https://github.com/hgrecco/pint
- https://github.com/amoffat/sh
- https://github.com/baronbrew
- https://github.com/peak/s5cmd
- https://github.com/piskvorky/smart_open


🛠️ Projects that I have contributed to:
- https://github.com/mammothbio-os/palamedes
- https://github.com/onecodex/onecodex
- https://github.com/pinellolab/CRISPResso2
- https://github.com/jroes/twitterforsanders


👨‍💻 Favorite Bits of Code


🥞 `merge-reduce`
A common task in the bioinformatics space is operating on ranges. These might represent gene annotations, sequence alignments or variant calls. The topic of merging these ranges comes up frequently. In the simplest case, we might want to merge ranges that overlap. A more complicated example might look at the distance between ranges and the "identity" of the values within them. Bounds checking and edge cases can often complicated the code to solve these problems. I've found the following framing helps to simplify the logic and code here.
- Sort your list of ranges in the manner applicable to the current problem at hand
- Define a function, which has a signature like `can_merge(left: Range, right: Range) -> bool`. This should return True/False based on whether the 2 input `Range`s can be merged together. In the simple case, this is just an overlap check. More complicated logic is still easy to define in this framing.
- Define a function, which has signature like `merge(left: Range, right: Range) -> Range`. This should implement the actual merging, taking in 2 `Range`s and returning the single merged `Range`. 

With this functions in hand, we can follow a pattern I've named "merge-reduce". It leverages the common `reduce` pattern (available in python's `functools` and other languages) with a stack and some extra logic. At a high level it looks roughly like:
- Given a list of input `Range` items, and implementations for the 2 functions above.
- Initialize the stack with the first item from the list or a provided default/starting value, as is common in the `reduce` interface.
- While the input list of `Range` objects still has items left in it, iteratively perform the following operations:
	- Peak the stack and the next item in the list, calling `can_merge` on them.
	- If the result is `True`, pop the item from the stack, call `merge` and push the merged item back onto the stack.
	- If the result is `False`, push the next item from the list onto the stack.
- Return the stack

A class based python implementation might look something like:
```python
from typing import Callable, Generic, TypeVar

Range = TypeVar("Range")


class MergeReducer(Generic[Range]):
    def __init__(
        self,
        can_merge: Callable[[Range, Range], bool],
        merge: Callable[[Range, Range], Range],
    ):
        self._can_merge = can_merge
        self._merge = merge

    def reduce(
        self, items: list[Range], initalizer: Range | None = None
    ) -> list[Range]:
        """
        Implement the merge-reduce with the provided functions. Assumes the input items are sorted.
        """
        if len(items) <= 1:
            return items

        stack = [items[0]] if initalizer is None else [initalizer]
        offset = 1 if initalizer is None else 0

        items_to_process = iter(items[offset:])
        while (next_item := next(items_to_process, None)) is not None:
            head_item = stack[-1]

            if self._can_merge(head_item, next_item):
                merged = self._merge(head_item, next_item)
                stack.pop()
                stack.append(merged)

            else:
                stack.append(next_item)

        return stack
```

And the following example call
```python
from dataclasses import dataclass

def example():
    @dataclass
    class MyRange:
        start: int
        end: int

    def can_merge(left: MyRange, right: MyRange) -> bool:
        return left.start <= right.end and right.start <= left.end

    def merge(left: MyRange, right: MyRange) -> MyRange:
        return MyRange(
            start=left.start,
            end=right.end,
        )

    my_items = [
        # 0 - 6 all get merged
        MyRange(start=0, end=1),
        MyRange(start=0, end=2),
        MyRange(start=0, end=5),
        MyRange(start=4, end=6),
        # 10 - 30 all get merged
        MyRange(start=10, end=20),
        MyRange(start=15, end=30),
        # not merged
        MyRange(start=150, end=151),
    ]

    merged_items = MergeReducer(can_merge, merge).reduce(my_items)
    assert merged_items == [
        MyRange(start=0, end=6),
        MyRange(start=10, end=30),
        MyRange(start=150, end=151),
    ]


if __name__ == "__main__":
    example()
```
