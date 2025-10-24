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
