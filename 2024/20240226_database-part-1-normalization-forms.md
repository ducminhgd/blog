# Database Part 1: Normalization forms

`1NF` -> `2NF` -> `3NF` -> `BCNF` -> `4NF` -> `5NF` -> `6NF`

## 1NF

A relation is said to be in 1NF only when all the entities of the table contain unique or atomic values.

- Given there is a table

| ID  | Name | Courses |
| --- | ---- | ------- |
| 1   | A    | c1, c2  |
| 2   | E    | c3      |
| 3   | M    | c2, c3  |

- Decompose for 1NF

| ID  | Name | Course |
| --- | ---- | ------ |
| 1   | A    | c1     |
| 1   | A    | c2     |
| 2   | E    | c3     |
| 3   | M    | c2     |
| 3   | M    | c3     |

## 2NF

A relation is said to be in 2NF only if it is in 1NF and all the non-key attribute of the table is fully dependent on the primary key.

- Given there is a table

| Student ID | Course ID | Course fee |
| ---------- | --------- | ---------- |
| 1          | C1        | 1000       |
| 2          | C2        | 1500       |
| 1          | C4        | 2000       |
| 4          | C3        | 1000       |
| 4          | C1        | 1000       |
| 2          | C5        | 2000       |

- Decompose for 2F: 2 tables

| Student ID | Course ID |
| ---------- | --------- |
| 1          | C1        |
| 2          | C2        |
| 1          | C4        |
| 4          | C3        |
| 4          | C1        |
| 2          | C5        |

| Course ID | Course fee |
| --------- | ---------- |
| C1        | 1000       |
| C2        | 1500       |
| C3        | 1000       |
| C4        | 2000       |
| C5        | 2000       |

## 3NF

A relation is said to be in 3NF only if it is in 2NF and every non-key attribute of the table is not transitively dependent on the primary key.

- Given there is a table

| Rollno | Game       | Feestructure |
| ------ | ---------- | ------------ |
| 1      | Basketball | 500          |
| 2      | Basketball | 500          |
| 3      | Basketball | 500          |
| 4      | Cricket    | 600          |
| 5      | Cricket    | 600          |
| 6      | Cricket    | 600          |
| 7      | Tennis     | 400          |

- Decompose with 3NF

| Rollno | Game       |
| ------ | ---------- |
| 1      | Basketball |
| 2      | Basketball |
| 3      | Basketball |
| 4      | Cricket    |
| 5      | Cricket    |
| 6      | Cricket    |
| 7      | tennis     |

| Game       | Feestructure |
| ---------- | ------------ |
| Basketball | 500          |
| Cricket    | 600          |
| Tennis     | 400          |

## Boyce Codd Normal Form (BCNF)

It is the higher version of 3Nf which does not have any multiple overlapping candidate keys.

- Given there is a table

  | Stu_ID | Stu_Branch                              | Stu_Course           | Branch_Number | Stu_Course_No |
  | ------ | --------------------------------------- | -------------------- | ------------- | ------------- |
  | 101    | Computer Science & Engineering          | DBMS                 | B_001         | 201           |
  | 101    | Computer Science & Engineering          | Computer Networks    | B_001         | 202           |
  | 102    | Electronics & Communication Engineering | VLSI Technology      | B_003         | 401           |
  | 102    | Electronics & Communication Engineering | Mobile Communication | B_003         | 402           |

- Decompose with BCNF

  | Stu_ID | Stu_Branch                              |
  | ------ | --------------------------------------- |
  | 101    | Computer Science & Engineering          |
  | 102    | Electronics & Communication Engineering |

  | Stu_Course           | Branch_Number | Stu_Course_No |
  | -------------------- | ------------- | ------------- |
  | DBMS                 | B_001         | 201           |
  | Computer Networks    | B_001         | 202           |
  | VLSI Technology      | B_003         | 401           |
  | Mobile Communication | B_003         | 402           |

  | Stu_ID | Stu_Course_No |
  | ------ | ------------- |
  | 101    | 201           |
  | 101    | 202           |
  | 102    | 401           |
  | 102    | 402           |

## Fourth Normal Form (4NF)

A table is in 4NF if and only if, for every one of its non-trivial multivalued dependencies X ->> Y, X is a superkey—that is, X is either a candidate key or a superset thereof.

- Given there is a table:

| Restaurant       | Pizza Variety | Delivery Area |
| ---------------- | ------------- | ------------- |
| A1 Pizza         | Thick Crust   | Springfield   |
| A1 Pizza         | Thick Crust   | Shelbyville   |
| A1 Pizza         | Thick Crust   | Capital City  |
| A1 Pizza         | Stuffed Crust | Springfield   |
| A1 Pizza         | Stuffed Crust | Shelbyville   |
| A1 Pizza         | Stuffed Crust | Capital City  |
| Elite Pizza      | Thin Crust    | Capital City  |
| Elite Pizza      | Stuffed Crust | Capital City  |
| Vincenzo's Pizza | Thick Crust   | Springfield   |
| Vincenzo's Pizza | Thick Crust   | Shelbyville   |
| Vincenzo's Pizza | Thin Crust    | Springfield   |
| Vincenzo's Pizza | Thin Crust    | Shelbyville   |

- Decompose with 4NF

| Restaurant       | Pizza Variety |
| ---------------- | ------------- |
| A1 Pizza         | Thick Crust   |
| A1 Pizza         | Stuffed Crust |
| Elite Pizza      | Thin Crust    |
| Elite Pizza      | Stuffed Crust |
| Vincenzo's Pizza | Thick Crust   |
| Vincenzo's Pizza | Thin Crust    |

| Restaurant       | Delivery Area |
| ---------------- | ------------- |
| A1 Pizza         | Springfield   |
| A1 Pizza         | Shelbyville   |
| A1 Pizza         | Capital City  |
| Elite Pizza      | Capital City  |
| Vincenzo's Pizza | Springfield   |
| Vincenzo's Pizza | Shelbyville   |

## Fifth Normal Form (5NF)

A table is said to be in the 5NF if and only if every non-trivial join dependency in that table is implied by the candidate keys. It is the final normal form as far as removing redundancy is concerned.

- Given a table

| Traveling salesman | Brand   | Product type   |
| ------------------ | ------- | -------------- |
| Jack Schneider     | Acme    | Vacuum cleaner |
| Jack Schneider     | Acme    | Breadbox       |
| Mary Jones         | Robusto | Pruning shears |
| Mary Jones         | Robusto | Vacuum cleaner |
| Mary Jones         | Robusto | Breadbox       |
| Mary Jones         | Robusto | Umbrella stand |
| Louis Ferguson     | Robusto | Vacuum cleaner |
| Louis Ferguson     | Robusto | Telescope      |
| Louis Ferguson     | Acme    | Vacuum cleaner |
| Louis Ferguson     | Acme    | Lava lamp      |
| Louis Ferguson     | Nimbus  | Tie rack       |

- Decompose with 5NF

| Traveling salesman | Product type   |
| ------------------ | -------------- |
| Jack Schneider     | Vacuum cleaner |
| Jack Schneider     | Breadbox       |
| Mary Jones         | Pruning shears |
| Mary Jones         | Vacuum cleaner |
| Mary Jones         | Breadbox       |
| Mary Jones         | Umbrella stand |
| Louis Ferguson     | Telescope      |
| Louis Ferguson     | Vacuum cleaner |
| Louis Ferguson     | Lava lamp      |
| Louis Ferguson     | Tie rack       |

| Traveling salesman | Brand   |
| ------------------ | ------- |
| Jack Schneider     | Acme    |
| Mary Jones         | Robusto |
| Louis Ferguson     | Robusto |
| Louis Ferguson     | Acme    |
| Louis Ferguson     | Nimbus  |

| Brand   | Product type   |
| ------- | -------------- |
| Acme    | Vacuum cleaner |
| Acme    | Breadbox       |
| Acme    | Lava lamp      |
| Robusto | Pruning shears |
| Robusto | Vacuum cleaner |
| Robusto | Breadbox       |
| Robusto | Umbrella stand |
| Robusto | Telescope      |
| Nimbus  | Tie rack       |
