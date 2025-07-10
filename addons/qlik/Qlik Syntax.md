# Complete Qlik Syntax Compact Reference

## Core Load Statements
**LOAD**: `LOAD [DISTINCT] fields FROM src [WHERE cond] [GROUP BY grp] [ORDER BY ord];`
**SELECT**: `SELECT fields FROM src [WHERE cond] [GROUP BY grp] [ORDER BY ord];`
**RESIDENT**: `LOAD fields RESIDENT table [WHERE cond];`
**INLINE**: `LOAD * INLINE [data];` or `LOAD * INLINE "data";`

## Data Sources & Connections
**Files**: 
- CSV: `FROM [lib://path/file.csv] (txt, codepage is 1252, embedded labels, delimiter is ',')`
- Excel: `FROM [lib://path/file.xlsx] (ooxml, embedded labels, table is Sheet1)`
- QVD: `FROM [lib://path/file.qvd] (qvd)`
- XML: `FROM [lib://path/file.xml] (XmlSimple, table is [path/table])`
- JSON: `FROM [lib://path/file.json] (jsonl)`

**Database**: 
```sql
LIB CONNECT TO 'connection_name';
SQL SELECT * FROM table_name;
LOAD * FROM connection_name.table_name;
```

**REST/Web**:
```sql
LIB CONNECT TO 'REST_connection';
RestConnectorMasterTable:
SQL SELECT * FROM JSON_TABLE;
```

## Variables & Parameters
**SET**: `SET var = 'literal_string';` (no evaluation)
**LET**: `LET var = expression;` (evaluated)
**Include**: `$(Include=path_to_script)`
**Expansion**: `$(variable_name)` or `$(=expression)`
**Dollar Expansion**: `$1, $2, $3` (parameters)

## Control Flow Structures
**IF Statement**:
```sql
IF condition THEN
    statements
[ELSEIF condition THEN
    statements]
[ELSE
    statements]
END IF
```

**FOR Loop**:
```sql
FOR i = start TO end [STEP increment]
    statements
NEXT [i]
```

**FOR EACH**:
```sql
FOR EACH value IN 'item1', 'item2', 'item3'
    statements
NEXT [value]
```

**DO WHILE/UNTIL**:
```sql
DO [WHILE|UNTIL condition]
    statements
    [EXIT DO [WHEN condition]]
LOOP
```

**SUB/CALL**:
```sql
SUB subroutine_name(param1, param2)
    statements
END SUB

CALL subroutine_name(value1, value2)
```

## Table Operations
**JOIN Types**:
- `JOIN [(table)] LOAD...` (inner)
- `LEFT JOIN [(table)] LOAD...`
- `RIGHT JOIN [(table)] LOAD...`
- `OUTER JOIN [(table)] LOAD...`
- `KEEP [(table)] LOAD...` (keep matching)

**CONCATENATE**: `CONCATENATE [(table)] LOAD...`
**CROSSTABLE**: `CROSSTABLE(attribute_field, data_field [,n]) LOAD...`
**GENERIC**: `GENERIC LOAD key, attribute, value FROM...`
**INTERVALMATCH**: `INTERVALMATCH(field [,key]) LOAD start, end FROM...`
**HIERARCHY**: `HIERARCHY(node, parent, name, [parent_name], [path], [depth]) LOAD...`

**Mapping**:
```sql
mapping_table:
MAPPING LOAD key, value FROM source;
ApplyMap('mapping_table', expression [, default_value])
```

## Field Operations
**ALIAS**: `ALIAS field_name AS alias_name;`
**RENAME**: `RENAME FIELD old_name TO new_name;`
**DROP**: `DROP FIELD field_name [FROM table_name];`
**DROP TABLE**: `DROP TABLE table_name;`
**QUALIFY**: `QUALIFY field_name|*;`
**UNQUALIFY**: `UNQUALIFY field_name|*;`
**TAG**: `TAG FIELD field_name WITH tag_name;`
**COMMENT**: `COMMENT FIELD field_name WITH 'comment';`

## String Functions
**Extraction**: `Left(s,n)`, `Right(s,n)`, `Mid(s,start[,len])`, `SubField(s,delim,n)`
**Case**: `Upper(s)`, `Lower(s)`, `Proper(s)`, `Capitalize(s)`
**Cleaning**: `Trim(s)`, `LTrim(s)`, `RTrim(s)`, `PurgeChar(s,chars)`
**Manipulation**: `Replace(s,from,to)`, `KeepChar(s,chars)`, `Repeat(s,n)`
**Info**: `Len(s)`, `Index(s,substr[,n])`, `FindOneOf(s,chars[,n])`
**Conversion**: `Chr(n)`, `Ord(s)`, `Hash128(s)`, `Hash160(s)`, `Hash256(s)`
**Aggregation**: `Concat([DISTINCT] expr[,delim])`, `FirstSortedValue(expr,sort)`

## Math Functions
**Basic**: `Round(n[,step[,offset]])`, `Floor(n)`, `Ceil(n)`, `Fabs(n)`, `Sign(n)`
**Operations**: `Mod(a,b)`, `Div(a,b)`, `Fmod(a,b)`, `Remainder(a,b)`
**Power/Root**: `Sqrt(n)`, `Pow(base,exp)`, `Sqr(n)`
**Logarithmic**: `Log(n)`, `Log10(n)`, `Exp(n)`, `Exp10(n)`
**Trigonometric**: `Sin(n)`, `Cos(n)`, `Tan(n)`, `Asin(n)`, `Acos(n)`, `Atan(n)`, `Atan2(y,x)`
**Random**: `Rand()`, `RangeRand(min,max)`
**Statistical**: `Combin(n,k)`, `Permut(n,k)`, `Fact(n)`

## Date/Time Functions
**Current**: `Now([timer_mode])`, `Today([timer_mode])`, `LocalTime([timer_mode])`
**Creation**: `MakeDate(year,month,day)`, `MakeTime(hour,min,sec)`, `MakeWeekDate(year,week[,day])`
**Formatting**: `Date(number[,format])`, `Time(number[,format])`, `Timestamp(number[,format])`
**Extraction**: `Year(date)`, `Month(date)`, `Day(date)`, `Hour(time)`, `Minute(time)`, `Second(time)`
**Week**: `Week(date)`, `WeekDay(date)`, `WeekName(date)`, `WeekYear(date)`
**Quarter**: `Quarter(date)`, `QuarterName(date)`, `QuarterStart(date)`, `QuarterEnd(date)`
**Month**: `MonthName(date)`, `MonthStart(date)`, `MonthEnd(date)`, `MonthsName(n)`
**Year**: `YearName(date)`, `YearStart(date)`, `YearEnd(date)`
**Arithmetic**: `AddMonths(date,n)`, `AddYears(date,n)`, `Age(date1,date2)`
**Business**: `NetworkDays(start,end[,holidays])`, `FirstWorkDate(date,n[,holidays])`
**Conversion**: `Dual(text,number)`, `Num(expr[,format[,decimal_sep[,thousands_sep]]])`

## Conditional Functions
**Basic**: `if(condition, then_value [, else_value])`
**Multi**: `pick(index, value1 [, value2, ...])`, `match(expr, value1 [, value2, ...])`
**Null**: `alt(expr1 [, expr2, ...])`, `coalesce(expr1 [, expr2, ...])`
**Classification**: `class(expression, interval [, label [, offset]])`
**Range**: `RangeSum(expr1 [, expr2, ...])`, `RangeAvg(expr1 [, expr2, ...])`

## Aggregation Functions (Script Context)
**Basic**: `Count([DISTINCT] expr)`, `Sum([DISTINCT] expr)`, `Avg([DISTINCT] expr)`
**MinMax**: `Min(expr)`, `Max(expr)`, `MinString(expr)`, `MaxString(expr)`
**Order**: `FirstValue(expr)`, `LastValue(expr)`, `FirstSortedValue(expr, sort_weight)`
**Statistical**: `Stdev(expr)`, `Var(expr)`, `Median(expr)`, `Mode(expr)`
**Table Info**: `NoOfRows([table])`, `NoOfFields([table])`, `FieldValueCount(field)`
**Field Info**: `FieldValue(field, index)`, `FieldIndex(field, value)`

## Logical Operators
**Boolean**: `AND`, `OR`, `NOT`, `XOR`
**Comparison**: `=`, `<>`, `>`, `<`, `>=`, `<=`
**Pattern**: `LIKE`, `MATCH`
**Set**: `IN`, `EXISTS`
**Null**: `IS NULL`, `IS NOT NULL`

## Mathematical Operators
**Arithmetic**: `+`, `-`, `*`, `/`, `MOD`, `DIV`
**String**: `&` (concatenation)
**Precedence**: `()` > `NOT` > `* / MOD DIV` > `+ -` > `< > <= >= = <>` > `AND` > `OR XOR`

## System Variables
**Script**: `ScriptErrorCount`, `ScriptErrorList`, `ScriptErrorDetails`
**Path**: `QvRoot`, `QvPath`, `QvWorkRoot`, `QvWorkPath`
**User**: `OSUser`, `QlikViewVersion`, `ReloadTime`
**Null**: `NullAsValue`, `NullAsNull`, `NullValue`, `NullDisplay`

## Script Control
**Flow**: `EXIT SCRIPT [WHEN condition]`, `EXIT FOR`, `EXIT DO`
**Debug**: `TRACE message`, `SLEEP seconds`, `PAUSE`
**Error**: `SET ErrorMode = 0|1;` (0=continue, 1=stop)
**Binary**: `BINARY [path]filename;`
**Store**: `STORE table INTO [path]filename [(format)];`
**Buffer**: `BUFFER [(option)] LOAD...`

## Set Analysis (Chart Context)
**Basic**: `Sum({<Year={2023}>} Sales)`
**Multiple**: `Sum({<Year={2023}, Region={'North'}>} Sales)`
**Exclusion**: `Sum({<Year-={2022}>} Sales)`
**Advanced**: `Sum({<Year={"=Year(Today())"}>} Sales)`
**Modifiers**: `=`, `-=`, `+=`, `*=`, `/=`

## Chart Functions (Additional)
**Ranking**: `Rank(expr [, mode [, format]])`, `HRank(expr [, mode [, format]])`
**Above/Below**: `Above(expr [, offset [, count]])`, `Below(expr [, offset [, count]])`
**Before/After**: `Before(expr [, offset [, count]])`, `After(expr [, offset [, count]])`
**Total**: `Total [<dimension>] expr`, `Aggr(expr, dimension)`

## Advanced Functions
**Mapping**: `MapSubString('map', string)`, `LookUp(field, match_field, match_value [, table])`
**Peek**: `Peek(field [, row [, table]])`, `Previous(expr)`
**Exists**: `Exists(field [, value])`, `AutoNumber(expr [, AutoID])`
**Null**: `IsNull(expr)`, `Null()`, `EmptyIsNull(expr)`
**Info**: `GetFieldSelections(field [, tag [, max_values]])`, `GetCurrentSelections([record_sep [, tag_sep]])`

## File Functions
**Info**: `FileSize('filename')`, `FileTime('filename')`, `FilePath('filename')`
**Directory**: `DirList('path')`, `SubStringCount(string, substring)`

## Color Functions
**Basic**: `RGB(red, green, blue)`, `HSL(hue, saturation, lightness)`
**Manipulation**: `LightNess(color)`, `Darkness(color)`, `ColorMix1(color1, color2, mix)`

## Format Functions
**Number**: `Num(expr [, format])`, `Money(expr [, format])`, `Integer(expr)`
**Date**: `Date(expr [, format])`, `Time(expr [, format])`, `Interval(expr [, format])`
**Text**: `Text(expr)`, `Evaluate(expr)`

## Performance Optimizations
**QVD Optimized Load**: Load fields in original order, use simple WHERE EXISTS() only
**Memory**: Use AutoNumber() for keys, DROP tables immediately
**Incremental**: `WHERE NOT EXISTS(key)` pattern
**Partial Reload**: `ADD LOAD`, `REPLACE LOAD`

## Syntax Rules
- Statements end with `;`
- Field names with spaces: `[Field Name]`
- String literals: `'text'` or `"text"`
- Comments: `//` or `/* */`
- Case insensitive keywords, case sensitive field/variable names
- Line continuation: No explicit character needed

