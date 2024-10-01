# Sqlparse-testrepo

import sqlparse
from sqlparse.sql import Identifier, IdentifierList
from sqlparse.tokens import Keyword, DML

# Read the SQL script
with open('your_script.sql', 'r') as file:
    script = file.read()

# Parse the SQL script
parsed = sqlparse.parse(script)

def extract_tables_and_columns(parsed):
    tables = {}
    columns = []

    for stmt in parsed:
        table_aliases = {}
        is_select = False
        is_from = False
        for token in stmt.tokens:
            if isinstance(token, IdentifierList):
                # This could be a list of columns in SELECT or tables in FROM
                for identifier in token.get_identifiers():
                    if is_select:
                        # Collect columns from the SELECT statement
                        real_name = identifier.get_real_name()
                        alias = identifier.get_alias() or real_name
                        columns.append({'name': real_name, 'alias': alias})
            elif isinstance(token, Identifier):
                if is_from:
                    # Collect tables and their aliases from the FROM/JOIN clause
                    table_name = token.get_real_name()
                    alias = token.get_alias() or table_name
                    table_aliases[alias] = table_name
                elif is_select:
                    # Collect single column from the SELECT statement
                    real_name = token.get_real_name()
                    alias = token.get_alias() or real_name
                    columns.append({'name': real_name, 'alias': alias})
            elif token.ttype is DML and token.value.upper() == 'SELECT':
                is_select = True
            elif token.ttype is Keyword and token.value.upper() in ('FROM', 'JOIN'):
                is_from = True
                is_select = False  # Reset after SELECT is done
            else:
                is_from = False  # Reset after FROM is processed
    return table_aliases, columns

# Extract tables and columns
table_aliases, columns = extract_tables_and_columns(parsed)

# Output the results
print("Tables and Aliases:")
for alias, table in table_aliases.items():
    print(f"Table: {table}, Alias: {alias}")

print("\nColumns:")
for col in columns:
    print(f"Column: {col['name']}, Alias: {col['alias']}")
