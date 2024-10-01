# Sqlparse-testrepo

import sqlparse
from sqlparse.sql import Identifier, IdentifierList
from sqlparse.tokens import Keyword, DML

def extract_tables_and_columns(parsed):
    tables = {}  # Store table aliases and real names
    columns = []
    is_select = False
    is_from = False
    current_table_alias = None

    for stmt in parsed:
        for token in stmt.tokens:
            if isinstance(token, IdentifierList):
                # Handle a list of identifiers (e.g., columns)
                for identifier in token.get_identifiers():
                    if is_select:
                        # We're in the SELECT clause, collect column info
                        real_name = identifier.get_real_name()
                        alias = identifier.get_alias() or real_name
                        parent_name = identifier.get_parent_name() or current_table_alias
                        columns.append({'table': parent_name, 'column': real_name, 'alias': alias})
            elif isinstance(token, Identifier):
                # Handle a single identifier (e.g., column or table)
                if is_from:
                    # If we're in the FROM clause, we expect a table
                    table_name = token.get_real_name()
                    alias = token.get_alias() or table_name
                    tables[alias] = table_name  # Store alias to table mapping
                    current_table_alias = alias  # Track current alias for columns
                elif is_select:
                    # If we're in the SELECT clause, we expect a column
                    real_name = token.get_real_name()
                    alias = token.get_alias() or real_name
                    parent_name = token.get_parent_name() or current_table_alias
                    columns.append({'table': parent_name, 'column': real_name, 'alias': alias})
            elif token.ttype is DML and token.value.upper() == 'SELECT':
                is_select = True  # Start of the SELECT clause
            elif token.ttype is Keyword and token.value.upper() in ('FROM', 'JOIN'):
                is_from = True  # Start of the FROM or JOIN clause
                is_select = False  # End SELECT when we hit FROM
            else:
                is_from = False  # Reset after FROM/JOIN is processed

    return tables, columns

# Example SQL query with "AS" for aliasing
query = """
SELECT A.id, A.name, B.department, COUNT(*)
FROM employees AS A
JOIN departments AS B ON A.department_id = B.id
WHERE A.status = 'active'
GROUP BY A.id, A.name, B.department;
"""

# Parse the SQL query
parsed = sqlparse.parse(query)

# Extract tables, aliases, and columns
tables, columns = extract_tables_and_columns(parsed)

# Output the results
print("Tables and Aliases:")
for alias, table in tables.items():
    print(f"Table name: {table}, Alias: {alias}")

print("\nColumns:")
for col in columns:
    table_name = tables.get(col['table'], 'Unknown')
    print(f"Table name: {table_name}, Alias: {col['table']}, Column name: {col['column']}")
