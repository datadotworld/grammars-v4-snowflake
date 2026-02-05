# KOS-Specific Modifications to Snowflake Grammar

## Purpose
This fork contains the ANTLR Snowflake grammar used by the data.world KOS Snowflake collector. This document tracks all modifications made from the upstream grammar.

## Fork Information
- **Upstream**: https://github.com/antlr/grammars-v4
- **Original Grammar**: sql/snowflake/
- **Fork Date**: 2025-10-23
- **Production Branch**: snowflake-kos-modifications

---

## Modification History

### 2025-10-23 - Initial KOS Modifications

**Status**: Grammar enhanced with features needed for KOS lineage extraction

**Changes to SnowflakeParser.g4:**

1. **Added OKTA to non-reserved words** (line 3842)
   - **Reason**: Customer uses OKTA as schema/database name
   - **Change**: Added `| OKTA` to `non_reserved_words` rule

2. **Added REGEXP as alternative to RLIKE** (lines 4065, 4647)
   - **Reason**: Snowflake supports both REGEXP and RLIKE for regex matching
   - **Change**: Changed `expr NOT? RLIKE expr` to `expr NOT? (RLIKE | REGEXP) expr`
   - **Locations**: Both in `expr` rule and `predicate` rule

3. **Added window frame support to over_clause** (lines 4183-4203)
   - **Reason**: Window functions can have ROWS/RANGE frame specifications
   - **Change**: Added optional `window_frame?` to all over_clause alternatives
   - **New Rules Added**:
     - `window_frame` - Defines ROWS/RANGE BETWEEN/UNBOUNDED/PRECEDING/FOLLOWING
     - `rows_range` - ROWS | RANGE
     - `preceding_following` - PRECEDING | FOLLOWING

4. **Added EXTRACT/DATE_PART function support** (line 4220)
   - **Reason**: Common Snowflake date/time extraction functions
   - **Change**: Added `(EXTRACT | DATE_PART) LR_BRACKET id_ FROM expr RR_BRACKET` to `function_call`

5. **Enhanced object_ref flexibility** (lines 4444-4445)
   - **Reason**: Support IDENTIFIER() function and improve table reference handling
   - **Change**: Changed `object_name` to `object_name_or_identifier` in first two alternatives
   - **Impact**: Allows `IDENTIFIER('DB.SCHEMA.TABLE')` syntax

**Changes to SnowflakeLexer.g4:**

1. **Added EXTRACT keyword** (line 336)
   - **Reason**: Support EXTRACT function
   - **Change**: Uncommented `EXTRACT : 'EXTRACT';`

2. **Added FOLLOWING keyword** (line 373)
   - **Reason**: Window frame FOLLOWING clause
   - **Change**: Added `FOLLOWING : 'FOLLOWING';`

3. **Uncommented PRECEDING keyword** (line 700)
   - **Reason**: Window frame PRECEDING clause
   - **Change**: Changed `// PRECEDING: 'PRECEDING';` to `PRECEDING: 'PRECEDING';`

4. **Uncommented RANGE keyword** (line 736)
   - **Reason**: Window frame RANGE clause
   - **Change**: Changed `// RANGE: 'RANGE';` to `RANGE: 'RANGE';`

5. **Added REGEXP keyword** (line 814)
   - **Reason**: Support REGEXP operator (synonym for RLIKE)
   - **Change**: Added `REGEXP : 'REGEXP';`

**Testing**: All changes tested via SnowflakeAntlrParserTest (79/79 tests passing)

**Upstream Compatible**: Mostly yes - these are legitimate Snowflake SQL features. Could potentially be contributed back to upstream.

---

### 2025-11-06 - Bind Variable Support for Stored Procedures

**Status**: Grammar enhanced to support bind variables in stored procedure parameters

**Changes to SnowflakeParser.g4:**

1. **Added bind variable support to primitive_expression** (line 4151)
   - **Issue**: Stored procedures with parameters use bind variables (`:param_name`) which were causing parse errors
   - **Change**: Added `COLON id_` as first alternative in `primitive_expression` rule
   - **Reason**: Enables parsing of INSERT/UPDATE statements with bind variables in stored procedure bodies
   - **Example Syntax**:
     ```sql
     INSERT INTO table (col1, col2) VALUES (:param1, :param2);
     UPDATE table SET col = :newValue WHERE id = :id;
     ```
   - **Important**: Must come BEFORE `id_ ('.' id_)*` to avoid conflicting with JSON field access (`col:field.subfield`)
   - **Testing**: Tested via SnowflakeAntlrInsertUpdateParserTest, all JSON variant tests continue to pass
   - **Upstream Compatible**: Yes - bind variables are standard SQL/Snowflake feature

---

### 2025-12-01 - Reserved Word Support for DATABASE and JSON

**Status**: Grammar enhanced to allow DATABASE and JSON as column/table names

**Changes to SnowflakeParser.g4:**

1. **Added DATABASE to non-reserved words** (line 3797)
   - **Issue**: Customer uses "database" as a column name, causing parse failures
   - **Change**: Added `| DATABASE` to `non_reserved_words` rule (between DATA and DAYS)
   - **Reason**: Snowflake allows DATABASE as an identifier when not in a reserved context (e.g., CREATE DATABASE)
   - **Example Syntax**: `SELECT foo, bar, database FROM t`
   - **Testing**: Tested via SnowflakeAntlrParserTest.testDatabaseNameIdentifier

2. **Added JSON to non-reserved words** (line 3830)
   - **Issue**: Customer uses "JSON" as a column name with colon accessor, causing parse failures
   - **Change**: Added `| JSON` to `non_reserved_words` rule (between JAVASCRIPT and LAST_NAME)
   - **Reason**: Snowflake allows JSON as an identifier, particularly common with variant data type columns
   - **Example Syntax**: `SELECT JSON:VehicleId::STRING FROM my_table`
   - **Testing**: Tested via SnowflakeAntlrParserTest.testJsonColumnNameWithAccessor

**Upstream Compatible**: Yes - these tokens should be allowed as identifiers in non-reserved contexts

---

### 2025-12-08 - Improved Column and Table.* Parsing

**Status**: Grammar refactored to eliminate ambiguity in column qualification and star selection

**Changes to SnowflakeParser.g4:**

1. **Added qualified_column_name rule** (lines 4386-4391)
   - **Issue**: Parser ambiguity with implicit aliases - `SELECT A.B.C ALIAS FROM ...` was incorrectly parsed as object_name=A.B.C + column_name=ALIAS instead of schema.table.column with alias
   - **Change**: Created new `qualified_column_name` rule with explicit qualification levels:
     ```antlr
     qualified_column_name
         : id_ DOT id_ DOT id_ DOT id_  // db.schema.table.column
         | id_ DOT id_ DOT id_           // schema.table.column
         | id_ DOT id_                   // table.column
         | id_                           // column
         ;
     ```
   - **Reason**: Makes column qualification unambiguous - parser can directly determine qualification level
   - **Example Syntax**:
     - `SELECT MYSCHEMA.MYTABLE.COL ALIAS FROM ...` - correctly parsed as 3-part qualified column with alias
     - `SELECT COL1, TABLE.COL2 AS ALIAS2 FROM ...` - both implicit and explicit aliases work correctly

2. **Updated column_elem to use qualified_column_name** (lines 4381-4384)
   - **Change**: Changed from `object_name_or_alias? column_name` to `qualified_column_name`
   - **Note**: Kept `object_name_or_alias? DOLLAR column_position` for positional column references (e.g., `SELECT $1, TABLE.$2`)
   - **Result**: Eliminates ambiguity between table qualification and column name

3. **Fixed column_elem_star to require explicit DOT before STAR** (lines 4376-4379)
   - **Issue**: `object_name_or_alias? STAR` allowed ambiguous patterns where table.* syntax wasn't clearly distinguished
   - **Change**: Changed to explicit alternatives:
     ```antlr
     column_elem_star
         : (object_name | alias) DOT STAR
         | STAR
         ;
     ```
   - **Reason**: Makes table.* syntax unambiguous - either bare `SELECT *` or qualified `SELECT table.*`
   - **Example Syntax**: `SELECT t1.*, t2.*, * FROM ...` - all star patterns parse correctly

**Testing**: All existing tests continue passing, new tests added for qualification levels and implicit/explicit aliases

**Upstream Compatible**: Yes - these changes clarify legitimate Snowflake SQL patterns without changing supported syntax

---

## Future Modifications Template

When making additional changes, document them here:

```markdown
### [Date] - [Change Description]

**File**: SnowflakeParser.g4 or SnowflakeLexer.g4
**Lines**: [line numbers]
**Issue**: [What problem does this solve?]
**Change**: [Detailed description]
**Reason**: [Why needed for KOS]
**Testing**: [Test coverage]
**Upstream Compatible**: [Yes/No]
```

---

## License

This grammar is licensed under the MIT License.

Copyright (c) 2022, Michał Lorek.

See LICENSE file or grammar file headers for full license text.
