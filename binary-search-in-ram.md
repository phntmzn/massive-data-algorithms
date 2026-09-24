Yes. The image is showing **binary search in RAM vs. binary search over external memory/disk blocks**. Here is a direct Python implementation of both.

### 1. Binary search in RAM

```python
def binary_search(arr, x):
    left = 0
    right = len(arr) - 1

    while left <= right:
        mid = left + (right - left) // 2

        if arr[mid] == x:
            return mid
        elif arr[mid] < x:
            left = mid + 1
        else:
            right = mid - 1

    return -1


# Example
arr = [0, 1, 2, 3, 4, 5, 6, 7]

index = binary_search(arr, 4)

print("index:", index)
print("value:", arr[index] if index != -1 else None)
```

Output:

```text
index: 4
value: 4
```

---

# 2. Binary search on disk

The important difference is that we don't want to read the entire file into RAM.

Instead, we divide the file into **blocks**:

```text
Disk file

┌───────────────┐
│ 0 1 2 3       │  Block 0
├───────────────┤
│ 4 5 6 7       │  Block 1
├───────────────┤
│ 8 9 10 11     │  Block 2
├───────────────┤
│ 12 13 14 15   │  Block 3
└───────────────┘
```

Each binary-search probe reads a block from disk.

```python
from pathlib import Path
import struct


INTEGER_SIZE = 8
BLOCK_SIZE = 4


def write_sorted_file(filename, values):
    """
    Write sorted integers as fixed-width 64-bit integers.
    """
    with open(filename, "wb") as f:
        for value in values:
            f.write(struct.pack("<q", value))


def read_block(filename, start_index, block_size):
    """
    Read block_size integers beginning at start_index.
    """
    offset = start_index * INTEGER_SIZE

    with open(filename, "rb") as f:
        f.seek(offset)

        data = f.read(block_size * INTEGER_SIZE)

    count = len(data) // INTEGER_SIZE

    return list(
        struct.unpack(
            f"<{count}q",
            data
        )
    )


def binary_search(arr, x):
    """
    Normal in-memory binary search.
    """
    left = 0
    right = len(arr) - 1

    while left <= right:
        mid = left + (right - left) // 2

        if arr[mid] == x:
            return mid
        elif arr[mid] < x:
            left = mid + 1
        else:
            right = mid - 1

    return -1


def binary_search_external(filename, x):
    """
    Binary search a sorted file without loading
    the entire file into RAM.
    """

    file_size = Path(filename).stat().st_size

    total_items = file_size // INTEGER_SIZE

    left = 0
    right = total_items - 1

    disk_reads = 0

    while left <= right:

        # Find middle item.
        mid = left + (right - left) // 2

        # Align the read to the beginning of a block.
        block_start = (mid // BLOCK_SIZE) * BLOCK_SIZE

        # Read one block from disk.
        block = read_block(
            filename,
            block_start,
            BLOCK_SIZE
        )

        disk_reads += 1

        # Search inside the block.
        local_index = binary_search(block, x)

        if local_index != -1:
            global_index = block_start + local_index

            print("Disk reads:", disk_reads)

            return global_index

        # Determine which side of the block we need.
        if x < block[0]:

            right = block_start - 1

        elif x > block[-1]:

            left = block_start + len(block)

        else:
            # x must be inside the range covered by
            # this block, but wasn't found.
            return -1

    print("Disk reads:", disk_reads)

    return -1


def main():
    filename = "numbers.bin"

    values = list(range(0, 1000))

    write_sorted_file(filename, values)

    x = 742

    index = binary_search_external(filename, x)

    print("search:", x)
    print("index:", index)

    if index != -1:
        print("found:", values[index])
    else:
        print("not found")


if __name__ == "__main__":
    main()
```

### The important part

The RAM version does:

```python
mid = left + (right - left) // 2
```

and directly accesses:

```python
arr[mid]
```

because RAM provides essentially direct access to the array.

The external-memory version has another step:

```python
block_start = (mid // BLOCK_SIZE) * BLOCK_SIZE
```

Then:

```python
block = read_block(
    filename,
    block_start,
    BLOCK_SIZE
)
```

So conceptually:

```text
             Binary search
                  │
                  ▼
             middle index
                  │
                  ▼
          Which disk block?
                  │
                  ▼
        ┌─────────────────┐
        │ Read block      │
        │ from disk       │
        └────────┬────────┘
                 │
                 ▼
       Binary search block
                 │
          ┌──────┴──────┐
          ▼             ▼
        found       next block
                        │
                        ▼
                   repeat
```

## 3. Seeing the I/O explicitly

For learning external-memory algorithms, this version makes the disk accesses especially obvious:

```python
import struct


INTEGER_SIZE = 8
BLOCK_SIZE = 8


def read_block(f, block_number):
    offset = block_number * BLOCK_SIZE * INTEGER_SIZE

    print(
        f"READ BLOCK {block_number} "
        f"(byte offset {offset})"
    )

    f.seek(offset)

    data = f.read(BLOCK_SIZE * INTEGER_SIZE)

    count = len(data) // INTEGER_SIZE

    if count == 0:
        return []

    return list(
        struct.unpack(
            f"<{count}q",
            data
        )
    )


def external_binary_search(filename, target):
    file_size = 0

    with open(filename, "rb") as f:
        f.seek(0, 2)
        file_size = f.tell()

    number_of_items = file_size // INTEGER_SIZE
    number_of_blocks = (
        number_of_items + BLOCK_SIZE - 1
    ) // BLOCK_SIZE

    left = 0
    right = number_of_blocks - 1

    while left <= right:

        middle = (
            left + (right - left) // 2
        )

        with open(filename, "rb") as f:
            block = read_block(f, middle)

        if not block:
            return -1

        if target < block[0]:

            right = middle - 1

        elif target > block[-1]:

            left = middle + 1

        else:

            # Target belongs in this block.
            local_index = binary_search(
                block,
                target
            )

            if local_index == -1:
                return -1

            return (
                middle * BLOCK_SIZE
                + local_index
            )

    return -1


def binary_search(arr, target):
    left = 0
    right = len(arr) - 1

    while left <= right:
        middle = (
            left + (right - left) // 2
        )

        if arr[middle] == target:
            return middle

        if arr[middle] < target:
            left = middle + 1
        else:
            right = middle - 1

    return -1
```

This gets closer to the diagram's key idea:

```text
RAM:

array[mid]
   │
   ▼
┌──────┬──────┬──────┬──────┬──────┐
│      │      │      │      │  X   │
└──────┴──────┴──────┴──────┴──────┘
                         ↑
                    direct access


EXTERNAL MEMORY:

             mid
              │
              ▼
┌─────┬─────┬─────┬─────┬─────┬─────┐
│ B0  │ B1  │ B2  │ B3  │ B4  │ B5  │
└─────┴─────┴─────┴─────┴─────┴─────┘
                    │
                    │ disk I/O
                    ▼
                read B3
                    │
                    ▼
             ┌─────────────┐
             │ x x x X x x │
             └─────────────┘
                  ↑
            search in RAM
```

The key performance distinction is:

**RAM:** `O(log N)` comparisons/accesses.

**External memory:** roughly `O(log_B N)` **block reads**, where `B` is the number of records that fit in one disk block. Once a block is read, searching within that block is done in RAM.

For a massive dataset, that difference is crucial because **disk I/O is much more expensive than an integer comparison in RAM**.
