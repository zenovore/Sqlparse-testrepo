# Sqlparse-testrepo

import sqlparse
from sqlparse.sql import Identifier, IdentifierList
from sqlparse.tokens import Keyword, DML


def extract_tables_and_columns(parsed):
    tables = {}
    columns = []
    is_select = False
    is_from = False

    for stmt in parsed:
        table_aliases = {}  # For each SQL statement
        current_table_alias = None  # Track the current table alias for column association
        for token in stmt.tokens:
            if isinstance(token, IdentifierList):
                # Handle a list of identifiers, e.g., multiple columns
                for identifier in token.get_identifiers():
                    if is_select:
                        # We're inside the SELECT clause, collect column info
                        column_name = identifier.get_real_name()
                        alias = identifier.get_alias() or column_name
                        columns.append({'table': current_table_alias, 'column': column_name, 'alias': alias})
            elif isinstance(token, Identifier):
                # Single identifier: either a column or a table
                if is_from:
                    # Collect tables and their aliases in the FROM/JOIN clause
                    table_name = token.get_real_name()
                    alias = token.get_alias() or table_name
                    table_aliases[alias] = table_name
                    current_table_alias = alias  # Set the current table alias for column association
                elif is_select:
                    # Collect column information in the SELECT clause
                    column_name = token.get_real_name()
                    alias = token.get_alias() or column_name
                    columns.append({'table': current_table_alias, 'column': column_name, 'alias': alias})
            elif token.ttype is DML and token.value.upper() == 'SELECT':
                is_select = True  # Start of the SELECT clause
            elif token.ttype is Keyword and token.value.upper() in ('FROM', 'JOIN'):
                is_from = True  # Start of the FROM or JOIN clause
                is_select = False  # End SELECT when we hit FROM
            elif token.ttype is Keyword and token.value.upper() == 'AS':
                is_from = True  # Consider the next identifier as an alias
            else:
                is_from = False  # Reset after FROM/JOIN is processed

        tables.update(table_aliases)
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
    table_alias = col['table'] if col['table'] else 'Unknown'
    print(f"Table name: {tables.get(table_alias, 'Unknown')}, Alias: {table_alias}, Column name: {col['column']}")
