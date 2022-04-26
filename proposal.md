# Proposal for Native CSV parsing

## Background
CSV (Comma-Separated Values)(Separate by `,`) or TSV (Tab-Separated Values)(Separate by `\t`) are both simple text formats for storing data in a tabular structure. Commonly one example is as follows:

```csv
a,c,d,e
1,2,3,4
5,6,7,8
```
The above content corresponds to the following table:

| a | c | d | e |
| -- | -- | -- | -- |
| 1 | 2 | 3 | 4 |
| 5 | 6 | 7 | 8 |

Recently in 2016, there was also an enhanced format developed by W3C Working Group https://www.w3.org/TR/tabular-data-primer/ to express useful metadata about CSV files to store other kinds of tabular data.

CSV files are the most popular file format to store AI training data, so for vw it's essential to natively support it so that user can get more convenience.

## Design of configurable portions and option surface of the parser
1. Use header line for feature names (disable with `-H` option)
2. If no header is present, will number features as 1..k based on column number
3. Multiclass labels will be auto-converted to 1..k if they are
    non-numeric e.g. Species: {setosa, versicolor, virginica} -> {1, 2, 3}
4. Categorical features are auto-converted to vw boolean name=value
    (a.k.a 'one-hot-encoding')
5. Numerical features will use `name:value`
6. -l \<labelcol> and -i \<idcol> command-line options allow specifying the label and id column numbers respectively. **(No matter what [kind of labels](https://github.com/VowpalWabbit/vowpal_wabbit/wiki/Input-format#labels) it will be, it will only be stored in one cell)**
Also: negative numbers support the "from the end of line" convention (e.g., -1 is the last column, -2 is next-to-last, etc.)
7. Allows specifying/overriding the input separator: -s \<sep>. **By default, auto-splits columns on commas and/or tabs. For file end with `.csv`, the default separator will be `,`, for `.tsv` it will be `\t`.**
8. By default, skips empty and zero values (-e to disable skips)
9.  Supports -b (binary label) option for auto converting 0 -> -1
    for vw logistic regression.
10.  **Allow specifying escape characters: -c \<escape> so that for csv files, multilabels which are also separated with `,` can be escaped. Default escape character will be `\`**

## Design of Stretch goals
### Design of the multiline (a more complex example where one atomic input to VW has two dimensions) compatible parser will look like

11.  **Allow specifying additional separator in one cell: -a \<sep>. For file end with `.csv`, the default additional separator will be `\t`, while for `.tsv` it will be `,`.**

### Design how namespaces may be expressed in a CSV datafile
One solution is to just read from the command args repeatedly. e.g., `-n ns1.csv -n ns2.csv -n ns3.csv`, the filename will be considered as the namespace name.

Another way ideally will need to introduce the enhanced format developed by W3C Working Group https://www.w3.org/TR/tabular-data-primer/#metadata for the namespace to be stored.

I will use such `csv-metadata.json` to organize namespaces just compatible with the W3C format. An example of `csv-metadata.json`:

```json
{
  "@context": "http://www.w3.org/ns/csvw",
  "tables": [{
    "url": "table1.csv",
    "schema:name": "ns1"
  }, {
    "url": "table2.csv",
    "schema:name": "ns2"
  }, {
    "url": "table3.csv",
    "schema:name": "ns3"
  }]
}
```

In this example, assume that in `table1.csv`:
```csv
a,c,d,e
1,2,3,4
5,6,7,8
```

In `table2.csv`:
```csv
k,b
10,12
11,15
```

In `table3.csv`:
```csv
f,g
20,22
```

So I will have the following equivalent vw format:
```vw
|ns1 a:1 c:2 d:3 e:4 |ns2 k:10 b:12 |ns3 f:20 g:22
|ns1 a:5 c:6 d:7 e:8 |ns2 k:11 b:15
```

### Additional
If time allows, in addition, I will choose to create a full realization of the CSV enhanced format developed by W3C Working Group. https://www.w3.org/TR/tabular-data-primer/#metadata In that case, most of the configurations in the [first part](#design-of-configurable-portions-and-option-surface-of-the-parser) will be unneeded. All the configs then will be stored permanently just in the json file.

## Coding
The parser will be pretty much like the [csv2vw](https://github.com/VowpalWabbit/vowpal_wabbit/blob/master/utl/csv2vw) However, instead of first converting `csv` files into `vw` files, this parser will act just like [VW text parser](https://github.com/VowpalWabbit/vowpal_wabbit/blob/master/vowpalwabbit/parse_example.cc) to directly store related data structures when parsing, thus being "native".

First the programme will open and load the `csv-metadata.json` for determining the files to parse.

Then the parser will be in a loop.
1. Create a C++ Variant for the whole instance
2. Create a C++ Variant for the specific csv file and push it into the C++ Variant in step 1.
3. First consumes input until it reaches a `\n` then it walks back the `\n` and also `\r` if it exists.
4. For the consumed line, initialize a C++ Variant for a line and push it into the C++ Variant in step 2. 
5. For the line, keep consuming (if it encounters an escape character, just skip the character next to it) until it reaches a separator,  determine the type of the string just consumed, push it into the C++ Variant in step 4.
6. If it's the additional separator that was met, initialize a C++ Variant for a cell, push it into this C++ Variant and push the C++ Variant into C++ Variant in step 4.
7. If the end of line is met, push the string into the C++ Variant in step 5, jump to step 3.
8. If the end of file is met, open the file for another namespace and jump to step 2.
9. When all files are consumed, end the parser.

When all the data is loaded by the parser, we can then assign the vw data structures with the value of this C++ Variant file for it to work.

## Timetable
The project will be test-driven to ensure that everything is included before coding. The timetable for this project is as follows:
- May 9 - 15, 2022: Further learn the test cases for vw custom text format to get a deep understanding.
- May 16 - 22, 2022: Interfaces and options design for the parser.
- May 23 - 29, 2022: Write unit test files for the parser.
- May 30 - Jun 5, 2022: Continue writing unit test files for the parser.
- Jun 6 - 12, 2022: Interface coding.
- Jun 13 - 19, 2022: Initial basic parsing options coding.
- Jun 20 - 26, 2022: Parsing logic coding.
- Jun 27 - July 3, 2022: Continue parsing logic coding.
- July 4 - 10, 2022: Document the achievement.
- July 11 - 17, 2022: Learn the W3C CSV enhancement format.
- July 18 - 24, 2022: Replace basic parsing options with CSV metadata configs.
- July 25 - 31, 2022: Coding on realizing W3C CSV validating functionality.
- Aug 1 - 7, 2022: For documenting as well as time for any lags.
- Aug 8 - 15, 2022: Also time for any lags as well as preparation for presentation.

## Deliverables
Just as required:
- A repository which implements a custom VW binary which supports CSV parsing.
- Documentation on what is supported and how to use it.
- Test suite verifying correctness.