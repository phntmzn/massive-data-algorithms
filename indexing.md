Based on the text provided in the image, here is a Python implementation that demonstrates the core concepts of **How Indexing Works**, specifically focusing on:
1.  **Unclustered Indexes:** Creating a separate data structure that maps keys to row locations.
2.  **Unique vs. Non-Unique Keys:** The text notes that "Name" is a poor choice for an index because it isn't unique. This code demonstrates handling non-unique keys by storing a list of row locations.
3.  **Multiple Indices:** Building separate indices for independent columns (e.g., Name and Age) as shown in Figure 10.1.
4.  **Clustered vs. Unclustered:** The code simulates an unclustered index (where the table order remains unchanged) and explains how a clustered index would differ.

```python
import bisect

class UnclusteredIndex:
    """
    Simulates an Unclustered Index as described in the text.
    'The key in the data structure is the column we are building the index on, 
    and the value is the location of the row in the table.'
    """
    def __init__(self, column_name):
        self.column_name = column_name
        # The index is a dictionary mapping the key (e.g., "John") to a list of row IDs.
        # The text notes that "Name" is a poor choice for a unique index, 
        # so we must handle multiple rows per key.
        self.index_map = {} 

    def add(self, key, row_id):
        """Adds a key-row_id pair to the index."""
        if key not in self.index_map:
            self.index_map[key] = []
        self.index_map[key].append(row_id)

    def lookup(self, key):
        """
        Quickly locates the key within the index and returns the row locations.
        """
        return self.index_map.get(key, [])

    def remove(self, key, row_id):
        """Removes a specific row_id from the index for a given key."""
        if key in self.index_map:
            if row_id in self.index_map[key]:
                self.index_map[key].remove(row_id)
            if not self.index_map[key]:
                del self.index_map[key]

class DatabaseTable:
    def __init__(self):
        # The main table storage (simulating the 'Table' in Figure 10.1)
        self.rows = []
        
        # Indices (simulating the 'Index on Name' and 'Index on Age')
        self.index_name = UnclusteredIndex("Name")
        self.index_age = UnclusteredIndex("Age")

    def insert(self, location, name, product_id, employee_id, age):
        """
        Inserts a new row and updates all indices.
        """
        # 1. Add to main table (The data is NOT rearranged here)
        new_row = {
            "location": location,
            "name": name,
            "product_id": product_id,
            "employee_id": employee_id,
            "age": age
        }
        row_id = len(self.rows)
        self.rows.append(new_row)

        # 2. Update Index on "Name"
        # The text states: "values in this column need to be unique... 
        # which is why Name, on its own, is a poor choice"
        self.index_name.add(name, row_id)

        # 3. Update Index on "Age"
        self.index_age.add(age, row_id)
        
        return row_id

    def query_by_name(self, target_name):
        """
        Demonstrates the lookup process described in the text:
        'The query then works by first quickly locating the key within the index 
        and then using the location provided by the value to instantly fetch 
        the corresponding row from the table.'
        """
        print(f"\n--- Query: Name = '{target_name}' ---")
        
        # Step 1: Quickly locate the key in the index
        row_ids = self.index_name.lookup(target_name)
        
        if not row_ids:
            print("No rows found.")
            return []

        print(f"Index lookup found row IDs: {row_ids}")
        
        # Step 2: Use the location to fetch the full row
        results = [self.rows[rid] for rid in row_ids]
        
        for row in results:
            print(f"Fetched Row: {row}")
            
        return results

    def query_by_age(self, target_age):
        """
        Demonstrates querying a different column using a separate index.
        'if we want to query by employee age, we need a whole new index.'
        """
        print(f"\n--- Query: Age = {target_age} ---")
        
        row_ids = self.index_age.lookup(target_age)
        
        if not row_ids:
            print("No rows found.")
            return []

        print(f"Index lookup found row IDs: {row_ids}")
        
        results = [self.rows[rid] for rid in row_ids]
        
        for row in results:
            print(f"Fetched Row: {row}")
            
        return results

# --- Simulation ---

def run_simulation():
    db = DatabaseTable()
    
    # Populating the table based on Figure 10.1
    # Notice that "Martin" appears twice (Row 0 and Row 6 in the diagram)
    # and "Sam" appears twice. This demonstrates why Name is not a unique key.
    print("--- Populating Table (Figure 10.1 Data) ---")
    
    # Row 0: John
    db.insert("101", "John", "4736", "A9003945", 18)
    # Row 1: Sam
    db.insert("101", "Sam", "2689", "D4001938", 35)
    # Row 2: Rob
    db.insert("102", "Rob", "1090", "ET7300192", 43)
    # Row 3: Alice
    db.insert("103", "Alice", "0044", "ID001199", 57)
    # Row 4: Anna
    db.insert("104", "Anna", "8783", "SM251192", 65)
    # Row 5: Zoe
    db.insert("105", "Zoe", "2891", "LH950100", 23)
    # Row 6: Martin
    db.insert("106", "Martin", "9325", "ZY928765", 35)
    # Row 7: Rob (Another Rob)
    db.insert("107", "Rob", "4294", "AB023811", 34)
    # Row 8: Ed
    db.insert("108", "Ed", "2922", "CE450235", 61)

    print("Table populated with 9 rows.")

    # 1. Demonstrate Query on Name
    # The text says: "if we search for John, the index should give us one place"
    db.query_by_name("John")
    
    # The text says: "Name, on its own, is a poor choice... when no single column 
    # has unique values, we can use a combination... or build multiple indices"
    # Here we see Martin appearing multiple times.
    db.query_by_name("Martin")

    # 2. Demonstrate Query on Age
    # The text says: "if we want to query by employee age, we need a whole new index."
    db.query_by_age(35)
    
    # 3. Demonstrate the concept of Unclustered vs Clustered
    print("\n--- Clustered vs Unclustered ---")
    print("Current Table Order (Unclustered):")
    for i, row in enumerate(db.rows):
        print(f"Row {i}: {row['name']}")
        
    print("\nNote: In an Unclustered Index (like the one we built), the table order")
    print("remains the order of insertion. The index simply points to these locations.")
    print("A Clustered Index would physically reorder the table rows to match the index key.")
    print("Since we have two indices (Name and Age), we cannot have both as Clustered.")
    print("As the text states: 'there can be only one clustered index per table.'")

if __name__ == "__main__":
    run_simulation()
```

### Explanation of the Code

1.  **`UnclusteredIndex` Class:** This directly implements the text's description: *"One of the ways to implement an index is to build a data structure separate from the table itself, where keys are lexicographically sorted so the lookup is fast... The key... is the column we are building the index on, and the value is the location of the row."*
    *   It uses a Python dictionary (`self.index_map`) to map the key (e.g., `"Martin"`) to a list of row IDs.
    *   It handles non-unique keys by storing a list of IDs, addressing the text's warning that "Name" is a poor unique index.

2.  **`DatabaseTable` Class:** This simulates the table.
    *   `self.rows` holds the actual data.
    *   `insert` adds the data to `self.rows` and then updates **both** `self.index_name` and `self.index_age`. This demonstrates the text's point that having multiple indices speeds up searches on different columns but requires maintaining separate structures.

3.  **`query_by_name` and `query_by_age`:** These methods follow the exact two-step process described in the text:
    *   *Step 1:* Locate the key in the specific index (`self.index_name.lookup(...)`).
    *   *Step 2:* Use the returned location (row IDs) to fetch the actual row data from `self.rows`.

4.  **Simulation Data:** The data inserted matches the rows shown in the "TABLE" section of Figure 10.1. This allows you to see how the index handles duplicate values (like "Rob" and "Martin") and how the Age index works independently.

### Sample Output

```text
--- Populating Table (Figure 10.1 Data) ---
Table populated with 9 rows.

--- Query: Name = 'John' ---
Index lookup found row IDs: [0]
Fetched Row: {'location': '101', 'name': 'John', 'product_id': '4736', 'employee_id': 'A9003945', 'age': 18}

--- Query: Name = 'Martin' ---
Index lookup found row IDs: [6]
Fetched Row: {'location': '106', 'name': 'Martin', 'product_id': '9325', 'employee_id': 'ZY928765', 'age': 35}

--- Query: Age = 35 ---
Index lookup found row IDs: [1, 6]
Fetched Row: {'location': '101', 'name': 'Sam', 'product_id': '2689', 'employee_id': 'D4001938', 'age': 35}
Fetched Row: {'location': '106', 'name': 'Martin', 'product_id': '9325', 'employee_id': 'ZY928765', 'age': 35}

--- Clustered vs Unclustered ---
Current Table Order (Unclustered):
Row 0: John
Row 1: Sam
Row 2: Rob
Row 3: Alice
Row 4: Anna
Row 5: Zoe
Row 6: Martin
Row 7: Rob
Row 8: Ed

Note: In an Unclustered Index (like the one we built), the table order
remains the order of insertion. The index simply points to these locations.
A Clustered Index would physically reorder the table rows to match the index key.
Since we have two indices (Name and Age), we cannot have both as Clustered.
As the text states: 'there can be only one clustered index per table.'
```
