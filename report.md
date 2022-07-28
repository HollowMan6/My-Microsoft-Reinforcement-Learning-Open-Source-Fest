# RLOS Fest 2022 Final report

https://github.com/HollowMan6/vowpal_wabbit/pull/1

## Introduction
Hi everyone! My name is Songlin. I got my bachelor of Engineering in Computer Science at Lanzhou University this year, and I'm also an incoming student of the Erasmus Mundus joint master for security and cloud computing, aka. SECCLO. My project here at the [Reinforcement Learning Open Source Fest 2022](https://www.microsoft.com/en-us/research/academic-program/rl-open-source-fest/) is to add the [native CSV parsing](https://vowpalwabbit.org/rlos/2022/projects#native-csv-parsing) feature for the [Vowpal Wabbit](https://vowpalwabbit.org/).

So why do I choose to implement native CSV parsing? Comma-Separated Values, aka. CSV, is one of the most popular file format used in the machine learning dataset, and are often delivered as the default format in various competitions such as Kaggle. Data is commonly more formatted in CSV or TSV files. VW supports several input formats, the main one being a custom text format. Converters in Python and Perl have been written which convert these files to VW text format. However, it would be convenient if VW were to be able to natively understand CSV files. I also want to challenge myself, as there is a surprising amount of complexity in the design of implementing a generalized parser for CSV, so this project is as much about considering all of the design pieces as implementing a working parser.

### About CSV
CSV files are often separated by commas (`,`) or tabs. however, alternative delimiter-separated files are often given a `.csv` extension despite the use of a non-comma field separator. This loose terminology can cause problems in data exchange. Many applications that accept CSV files have options to select the delimiter character and the quotation character. Semicolons (`;`) are often used instead of commas in many European locales in order to use the comma (`,`) as the decimal separator and, possibly, the period (`.`) as a decimal grouping character.

Separating fields with the field separator is CSV's foundation, but commas in the data have to be handled specially.

### How we handle CSV files
The short answer is that we follow the RFC 4180 and MIME standards.

The 2005 technical standard RFC 4180 formalizes the CSV file format and defines the MIME type "text/csv" for the handling of text-based fields. However, the interpretation of the text of each field is still application-specific. Files that follow the RFC 4180 standard can simplify CSV exchange and should be widely portable. Among its requirements:

1. Lines that end with (CR/LF) characters (optional for the last line).
2. An optional header record (there is no sure way to detect whether it is present, so care is required when importing).
3. Each record should contain the same number of comma-separated fields.
4. Any field may be quoted (with double quotes).
5. Fields containing a double-quote or commas should be quoted. (If they are not, the file will likely be impossible to process correctly. In addition, here we choose not to support line-breaks within quoted fields for the simplicity.)
6. If double-quotes are used to enclose fields, then a double-quote in a field must be represented by two double-quote characters.

## Dig into details
1. Allows specifying the CSV field separator by --csv_separator, default is `,`, but `"` `|` or `:` are reserved and not allowed to use, since the double quote (`"`) is for escape, vertical bar(`|`) for separating the namespace and feature names, `:` can be used in labels.
2. For each separated field, auto remove the outer double-quotes of a cell when it pairs. `--csv_separator` symbols that appeared inside the double-quoted cells are not considered as a separator but a normal string character.
3. Double-quotes that appear at the start and end of the cell will be considered to enclose fields. Other quotes that appear elsewhere and out of the enclose fields will have no special meaning. (This is also how Microsoft Excel parses.)
4. If double-quotes are used to enclose fields, then a double-quote appearing inside a field must be escaped by preceding it with another double quote, and will remove that escape symbol during parsing.
5. Use header line for feature names (and possibly namespaces) / specify label and tag using `_label` and `_tag` by default. For each separated field in header except for tag and label, it may contain namespace and feature name separated by namespace separator, vertical bar(`|`).
6. `--csv_header` to override the CSV header by providing (namespace, `|` and)  feature name separated with `,`. By default, CSV files are assumed to  have a header with feature and/or namespaces names in the CSV first line.  You can override it by specifying `--csv_header`. Combined with `--csv_no_file_header`, we assume that there is no header in the CSV file and under such condition specifying `--csv_header` for the header is a must.
7. If the number of the separated fields for current parsing line is greater than the header, an error will be thrown.
8. Trim the field for ASCII "white space"(`\r\n\f\v`) as well as some UTF-8 BOM characters(`\xef\xbb\xbf`) before separation.
9.  If no namespace is separated, will use empty namespace.
10. Separator supports using `\t` to represent tabs. Otherwise, if assigning more than one character, an error will be thrown.
11. Directly read the label as string, interpret it using the VW text label parser.
12. Will try to judge if the feature values are float or string, if NaN, will consider it as a string. quoted numbers are always considered as strings.
13. If the feature value is empty, will skip that feature.
14. Reset the parser when EOF of a file is met (for possible multiple input file support).
15. Support using `--csv_ns_value` to scale the namespace values by specifying the float ratio.
e.g. --csv_ns_value=a:0.5,b:0.3,:8, which the namespace a has a ratio of 0.5, b of 0.3, empty namespace of 8, other namespaces of 1.
16. If all the cells in a line is empty, then consider it as an empty line. CSV is not a good fit for the multiline format, as evidenced by the large number of empty fields. Multi-line format often means different lines have different schemas. However, I still leave the empty line support to make sure that it’s flexible and extendable enough. We still throw an error if the number of fields separated by the line doesn’t match previous, even all the fields are empty, as this usually means typos that users may not intend.

## Some statistics to share

The project reaches 100% Test / Code Coverage, and on my computer, it takes only 570 millisecond to handle 20000 examples, which is comparable to the parsing speed of the VW custom format.

## Demo

Finally, the demo time! I will show you how to use the csv parsing with VW with the help of the iris dataset tutorial.

...

## Thanks
That's all, thanks for listening, and also special thanks to my mentors Jack Gerrits and Peter Chang, who really helped me a lot during the project.

## Reference
1. [Comma-separated values - Wikipedia](https://en.wikipedia.org/wiki/Comma-separated_values)
