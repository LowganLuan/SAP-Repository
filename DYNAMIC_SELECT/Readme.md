# ZCLUTIL_DYNAMIC_SELECT

## 1. Functional Description

`ZCLUTIL_DYNAMIC_SELECT` is a reusable utility class designed to perform **dynamic data selection based on a configurable access sequence**, allowing different search criteria to be evaluated hierarchically until a combination of parameters that returns data from the target table is found.

From a functional perspective, the component decouples the data selection logic from the specific implementation of the consuming program. Instead of implementing multiple conditional `SELECT` statements for each possible combination of search fields, the consuming program provides:

* the table containing the access sequence;
* the table where the data should be searched;
* a structure containing the values to be used as search criteria;
* an internal table to receive the results.

The class then processes the access sequence, identifies which fields from each sequence level actually exist in the target table, and dynamically builds the corresponding `WHERE` condition.

The search is performed according to the order defined in the access sequence. If the first combination of criteria does not return any records, the class automatically proceeds to the next sequence level and performs another search attempt.

This approach allows the component to implement **priority-based data determination scenarios**, where different combinations of keys can be used as fallback criteria to locate the required information.

### Functional Example

Consider a configuration table containing the following access sequence:

| Priority | Field 1 | Field 2 | Field 3 |
| -------: | ------- | ------- | ------- |
|        1 | BUKRS   | WERKS   | MATNR   |
|        2 | BUKRS   | WERKS   |         |
|        3 | BUKRS   |         |         |

The consuming program provides the following values:

```text
BUKRS = 1000
WERKS = 1010
MATNR = MAT001
```

The class will initially attempt to locate the data using the most specific combination:

```text
BUKRS = '1000'
AND WERKS = '1010'
AND MATNR = 'MAT001'
```

If no records are found, the class can use the next combination:

```text
BUKRS = '1000'
AND WERKS = '1010'
```

And subsequently the third combination:

```text
BUKRS = '1000'
```

This behavior allows a **priority-based fallback strategy** to be implemented without requiring the consuming program to know or individually implement each possible search condition.

---

## 2. Technical Description

Technically, `ZCLUTIL_DYNAMIC_SELECT` uses **RTTI (Run Time Type Identification)**, dynamic data references, and dynamic Open SQL to determine table structures and build selection conditions at runtime.

The `SELECT_DATA` method receives the names of the access sequence and data tables, together with the structure containing the search values, and coordinates the entire selection process.

The processing is divided into the following steps:

1. Dynamically create the internal table corresponding to the access sequence table.
2. Read the access sequence.
3. Evaluate each access sequence entry.
4. Identify valid fields in the target table.
5. Dynamically build the `WHERE` clause.
6. Execute the `SELECT` statement against the target table.
7. Return the data when a valid combination is found.
8. Continue with the next access sequence entry when no records are found.

### 2.1. Access Sequence Reading

The access sequence table is dynamically instantiated using the table name provided in `IV_TABNAME_SEQ`.

The `GET_ACCESS_SEQUENCE` method reads the entries using dynamic Open SQL:

```abap
SELECT *
  INTO TABLE <fs_tdata>
  FROM (iv_tabname).
```

This allows the same class to be used with different access sequence tables without requiring changes to the source code.

### 2.2. Valid Field Identification

For each access sequence entry, the `GET_ONLY_VALID_FIELDS` method compares the components of the sequence structure with the structure of the target data table.

The target table structure is dynamically obtained using:

```abap
cl_abap_structdescr=>describe_by_name( )
```

The structure of the current access sequence entry is obtained using:

```abap
cl_abap_structdescr=>describe_by_data( )
```

Only fields that actually exist in the target table are added to the list of valid fields.

This approach allows the access sequence table to contain fields that may not necessarily exist in every data table used by the component.

### 2.3. Dynamic WHERE Construction

After identifying the valid fields, the `BUILD_WHERE_DATA_W_V_FIELDS` method dynamically builds the `WHERE` condition.

The values are obtained from the structure provided by the consuming program and associated with their respective field names.

Example:

```text
BUKRS = '1000'
AND WERKS = '1010'
AND MATNR = 'MAT001'
```

Fields that are not considered valid for the target table are handled accordingly so that they are not used as search criteria.

### 2.4. SQL Literal Conversion

The `GET_SQL_LITERAL_FOR_COMPONENT` method is responsible for converting ABAP values into the appropriate SQL literal format used by the dynamic SQL statement.

The implementation specifically handles:

* `CHAR`;
* `STRING`;
* `CLIKE` types;
* `DATS`;
* `TIMS`;
* numeric and decimal values.

For character-based values, single quotes are escaped before the SQL condition is constructed. Date and time values are normalized, while numeric values are used without quotation marks.

### 2.5. Data Selection Execution

After the `WHERE` clause has been constructed, the class dynamically executes the `SELECT` statement against the table provided in `IV_TABNAME_DATA`:

```abap
SELECT *
  INTO TABLE <fs_data>
  FROM (iv_tabname_data)
  WHERE (lv_where)
  ORDER BY PRIMARY KEY.
```

If the query returns data, processing is terminated and the result remains available in the output internal table.

If no records are found, the class clears the criteria used and proceeds to the next access sequence entry.

---

## 3. Benefits

The use of this class provides the following benefits:

* **Reusability:** the same implementation can be used across different programs and scenarios.
* **Decoupling:** data determination logic is centralized within the utility class.
* **Flexibility:** access sequence and data tables are provided dynamically.
* **Automatic fallback:** different priority levels can be evaluated without duplicating code.
* **Code reduction:** eliminates the need for multiple conditional `SELECT` implementations in consuming programs.
* **Maintainability:** changes to the access sequence can be handled through table configuration, reducing the need to modify consuming programs.
* **Dynamic validation:** only fields that exist in the target table structure are used when constructing the selection criteria.

## 4. Limitations and Future Improvements

The current implementation was developed with a focus on reusing the access sequence-based data determination logic and reducing duplicated code across consuming programs.

Additional validations may be incorporated in future versions, particularly regarding input parameter validation, specific exception handling, and usage scenarios involving large data volumes.

> **Note:** The points mentioned above represent potential areas for future improvement and should be evaluated according to the intended usage scenario and project requirements.

## 5. Processing Flow

```text
Consuming Program
        │
        │ SELECT_DATA
        ▼
┌──────────────────────────────┐
│ ZCLUTIL_DYNAMIC_SELECT       │
└──────────────┬───────────────┘
               │
               ▼
      Read access sequence
               │
               ▼
      Evaluate sequence entry
               │
               ▼
      Identify valid fields
               │
               ▼
      Build WHERE dynamically
               │
               ▼
        Execute SELECT
               │
        ┌──────┴──────┐
        │             │
      Found        Not Found
        │             │
        ▼             ▼
    Return data   Next sequence
                      │
                      └──► Repeat
```

## 6. Main Methods

| Method                          | Responsibility                                                    |
| ------------------------------- | ----------------------------------------------------------------- |
| `SELECT_DATA`                   | Orchestrates the entire dynamic selection process                 |
| `GET_ACCESS_SEQUENCE`           | Dynamically reads the access sequence table                       |
| `GET_DATA_FROM_SEQUENCE`        | Iterates through the sequence and performs the selection attempts |
| `GET_ONLY_VALID_FIELDS`         | Identifies sequence fields that exist in the target data table    |
| `BUILD_WHERE_DATA_W_V_FIELDS`   | Dynamically builds the `WHERE` clause                             |
| `GET_SQL_LITERAL_FOR_COMPONENT` | Converts ABAP values into SQL literals                            |
| `CONSTRUCTOR`                   | Class initialization                                              |

## 7. Reusability Concept

The class is intended to be used as an infrastructure/utility component, preventing dynamic selection rules from being duplicated across different developments.

The consuming program should focus only on providing the required parameters and consuming the result returned by the class.

The following responsibilities are encapsulated within `ZCLUTIL_DYNAMIC_SELECT`:

* reading the access sequence;
* identifying valid fields;
* dynamically building the `WHERE` clause;
* converting values to SQL literals;
* executing the selection attempts;
* handling the fallback between priority levels.

This design allows the same determination mechanism to be reused across different ABAP developments while keeping the consuming programs simpler and focused on their respective business processes.
