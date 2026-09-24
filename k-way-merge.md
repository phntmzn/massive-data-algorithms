Here is a complete Python implementation of the **External Memory K-way Merge** algorithm described in the text and Figure 9.7.

This implementation simulates the process of reading one block at a time from multiple sorted files, using a min-heap to merge them efficiently without loading the entire dataset into RAM.

```python
import heapq
import os
import struct

# --- Configuration ---
INTEGER_SIZE = 8  # 64-bit integers
BLOCK_SIZE = 4    # Number of integers per block (simulating disk block size)

# --- Helper Functions for File I/O (Simulating Disk Blocks) ---

def write_sorted_file(filename, values):
    """Writes a list of integers to a binary file."""
    with open(filename, "wb") as f:
        for value in values:
            f.write(struct.pack("<q", value))

def read_block(file_handle, block_index):
    """
    Reads a specific block of integers from an open file handle.
    Returns a list of integers (the block).
    """
    offset = block_index * BLOCK_SIZE * INTEGER_SIZE
    file_handle.seek(offset)
    
    data = file_handle.read(BLOCK_SIZE * INTEGER_SIZE)
    if not data:
        return []
        
    count = len(data) // INTEGER_SIZE
    return list(struct.unpack(f"<{count}q", data))

# --- External K-Way Merge Algorithm ---

def external_k_way_merge(input_filenames, output_filename):
    """
    Merges K sorted files using an external memory approach.
    
    Assumptions:
    - K (number of files) is small enough that we can hold one block 
      from each file in RAM simultaneously.
    - Each file is sorted.
    """
    num_files = len(input_filenames)
    
    # 1. Open all input files
    file_handles = [open(f, "rb") for f in input_filenames]
    
    # 2. Initialize the heap
    # The heap stores tuples: (value, file_index)
    # This allows us to know which file the minimum came from.
    heap = []
    
    # Keep track of the current block for each file and the index within that block
    current_blocks = [None] * num_files
    block_indices = [0] * num_files  # Which block number we are on for each file
    element_indices = [0] * num_files # Which element within the current block
    
    # 3. Read the first block of each file and insert its minimum into the heap
    print("--- Initializing: Reading first block of each file ---")
    for i in range(num_files):
        block = read_block(file_handles[i], block_indices[i])
        if block:
            current_blocks[i] = block
            element_indices[i] = 0
            # Insert the first element of this block into the heap
            heapq.heappush(heap, (block[0], i))
            print(f"File {i}: Loaded block {block_indices[i]}, pushed {block[0]}")
        else:
            print(f"File {i}: Empty file.")

    # 4. Open output file
    with open(output_filename, "wb") as out_f:
        total_written = 0
        
        # 5. Main Merge Loop
        while heap:
            # Extract the global minimum
            min_val, file_idx = heapq.heappop(heap)
            
            # Write to output
            out_f.write(struct.pack("<q", min_val))
            total_written += 1
            
            # Advance the pointer in the specific file
            element_indices[file_idx] += 1
            
            # Check if we need to read the next block from this file
            if element_indices[file_idx] >= len(current_blocks[file_idx]):
                # Current block exhausted, read next block
                block_indices[file_idx] += 1
                next_block = read_block(file_handles[file_idx], block_indices[file_idx])
                
                if next_block:
                    current_blocks[file_idx] = next_block
                    element_indices[file_idx] = 0
                    # Push the first element of the new block
                    heapq.heappush(heap, (next_block[0], file_idx))
                else:
                    # End of file reached
                    pass
            else:
                # Push the next element from the current block
                next_val = current_blocks[file_idx][element_indices[file_idx]]
                heapq.heappush(heap, (next_val, file_idx))

    # Cleanup
    for f in file_handles:
        f.close()
        
    print(f"\n--- Merge Complete ---")
    print(f"Total elements written: {total_written}")

# --- Main Execution ---

def main():
    # Create some dummy sorted files
    # File 1: 1, 5, 9, 13, 17, 21, 25, 29, 33
    # File 2: 2, 6, 10, 14, 18, 22, 26, 30, 34
    # File 3: 3, 7, 11, 15, 19, 23, 27, 31, 35
    # File 4: 4, 8, 12, 16, 20, 24, 28, 32, 36
    
    files_data = [
        list(range(1, 40, 4)),   # 1, 5, 9...
        list(range(2, 41, 4)),   # 2, 6, 10...
        list(range(3, 42, 4)),   # 3, 7, 11...
        list(range(4, 43, 4))    # 4, 8, 12...
    ]
    
    filenames = [f"input_{i}.bin" for i in range(len(files_data))]
    
    # Write data to files
    for fname, data in zip(filenames, files_data):
        write_sorted_file(fname, data)
        print(f"Created {fname} with {len(data)} elements.")

    # Run External K-Way Merge
    output_file = "merged_output.bin"
    external_k_way_merge(filenames, output_file)
    
    # Verify output
    print("\n--- Verifying Output ---")
    with open(output_file, "rb") as f:
        data = f.read()
        count = len(data) // INTEGER_SIZE
        result = list(struct.unpack(f"<{count}q", data))
        print("Merged Output:", result)
        
        # Check if sorted
        is_sorted = all(result[i] <= result[i+1] for i in range(len(result)-1))
        print("Is Sorted?", is_sorted)

    # Cleanup generated files
    for fname in filenames + [output_file]:
        if os.path.exists(fname):
            os.remove(fname)

if __name__ == "__main__":
    main()
```

### Explanation of the Code

1.  **Block Simulation:** The `read_block` function uses `f.seek()` and `f.read()` to simulate reading a specific block from disk. It reads `BLOCK_SIZE * INTEGER_SIZE` bytes at a time.
2.  **The Heap:** We use Python's `heapq` library. The heap stores tuples of `(value, file_index)`. The `file_index` is crucial because when we pop a value, we need to know which file to pull the next element from.
3.  **Initialization:** We loop through all files, read the **first block** of each, and push the **first element** of that block into the heap. This is the "Initial Block Load" described in the text.
4.  **The Merge Loop:**
    *   We pop the smallest element from the heap.
    *   We write it to the output file.
    *   We increment the element pointer for that specific file.
    *   **Block Exhaustion:** If the pointer reaches the end of the current block, we read the **next block** from that file and push its first element.
    *   **Normal Case:** If the block isn't exhausted, we just push the next element from the current block.
5.  **Memory Efficiency:** At any given time, the program only holds one block per file in RAM (plus the heap overhead). It never loads the entire file into memory.

### Output Example

```text
Created input_0.bin with 10 elements.
Created input_1.bin with 10 elements.
Created input_2.bin with 10 elements.
Created input_3.bin with 10 elements.
--- Initializing: Reading first block of each file ---
File 0: Loaded block 0, pushed 1
File 1: Loaded block 0, pushed 2
File 2: Loaded block 0, pushed 3
File 3: Loaded block 0, pushed 4

--- Merge Complete ---
Total elements written: 40

--- Verifying Output ---
Merged Output: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40]
Is Sorted? True
```
