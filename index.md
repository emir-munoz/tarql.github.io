---
layout: default
---

# Tarql: SPARQL for Tables

Tarql is a command-line tool for converting CSV files to RDF using SPARQL 1.1 syntax. It's written in Java and based on Apache ARQ.


## Introduction

In Tarql, the following SPARQL query:

{% highlight bash %}
  CONSTRUCT { ... }
  FROM <file:table.csv>
  WHERE {
  ...
}
{% endhighlight %}

is equivalent to executing the following over an empty graph:

{% highlight bash %}
  CONSTRUCT { ... }
  WHERE {
  VALUES (...) { ... }
  ...
}
{% endhighlight %}

In other words, the CSV file's contents are input into the query as a table of bindings. This allows manipulation of CSV data using the full power of SPARQL 1.1 syntax, and in particular the generation of RDF using `CONSTRUCT` queries. See below for more examples.


## Command line

For Unix, the executable script is `bin/tarql`. For Windows, `bin\tarql.bat`. Example invocations:

* `tarql --help` to show full command line help
* `tarql my_mapping.sparql input1.csv input2.csv` to translate two CSV files using the same mapping
* `tarql --no-header-row my_mapping.sparql input1.csv` if CSV file doesn't have column headers; variables will be called `?a`, `?b`, etc.
* `tarql my_mapping.sparql` to translate a CSV file defined in SPARQL `FROM` clause
* `tarql --test my_mapping.sparql` to show only the CONSTRUCT template, variable names, and a few input rows (useful for CONSTRUCT query development)`
* `tarql --tabs my_mapping.sparql input.tsv` to read a tab-separated (TSV) file

Full options:

{% highlight bash %}
tarql [options] query.sparql [table.csv [...]]
  Main arguments
      query.sparql           File containing a SPARQL query to be applied to a CSV file
      table.csv              CSV file to be processed; can be omitted if specified in FROM clause
  Options
      --test                 Show CONSTRUCT template and first rows only (for query debugging)
      -d   --delimiter       Delimiting character of the CSV file
      -t   --tabs            Specifies that the input is tab-separagted (TSV), overriding -d
      --quotechar            Quote character used in the CSV file
      -p   --escapechar      Character used to escape quotes in the CSV file
      -e   --encoding        Override CSV file encoding (e.g., utf-8 or latin-1)
      -H   --no-header-row   CSV file has no header row; use variable names ?a, ?b, ...
      --header-row           CSV file's first row is a header with variable names (default)
      --ntriples             Write N-Triples instead of Turtle
  General
      -v   --verbose         Verbose
      -q   --quiet           Run with minimal output
      --debug                Output information for debugging
      --help
      --version              Version information
{% endhighlight %}

## Details

**Input CSV file(s)** can be specified using `FROM` or on the command line. Use `FROM <file:filename.csv>` to load a file from the current directory.

**Variables**: By default, Tarql assumes that the CSV file has column names in the first row, and it will use those column names as variable names. If the file has no header row (indicated via `-H` on the command line, or by appending `#header=absent` to the CSV file's URL), then `?a` will contain values from the first column, `?b` from the second, and so on.

All-empty rows are skipped automatically.

Syntactically, a Tarql mapping is one of the following:

* A SPARQL 1.1 SELECT query
* A SPARQL 1.1 ASK query
* One or more consecutive SPARQL 1.1 CONSTRUCT queries, where `PREFIX` and `BASE` apply to *any* subsequent query

In Tarql mappings with multiple `CONSTRUCT` queries, the triples generated by previous `CONSTRUCT` clauses can be queries in subsequent WHERE clauses to retrieve additional data. When using this capability, note that the order of `OPTIONAL` and `BIND` is significant in a SPARQL query.

Tarql supports a magic `?ROWNUM` variable for accessing the number of the row within the input CSV file. Numbering starts at 1, and skips empty rows.


## Header row, delimiters, quotes and character encoding in CSV/TSV files

The following syntax options can be adjusted:

- Presence or absence of a header row with column names to be used as variable names (default: header present)
- Delimiter character. The character used to separate fields in a row; typically comma (default), tab, or semicolon.
- Quote character. The character used to enclose field values where the delimiter or a line break occurs within the value; typically, double (default) or single quote.
- Escape character. The character used to escape quotes occurring within quoted values; typically none (default; use two quotes to escape a quote) or backslash.
- Character encoding (`utf-8`, `latin1`, etc.; will attempt auto-detection if unspecified)

These options can be specified in two ways:

1. On the command line (`--no-header-row`, `--delimiter`, `--tabs`, `--quotechar`, `--escapechar`, `--encoding`)
2. By appending a pseudo-fragment to the CSV file's URL in the `FROM` clause or on the command line (e.g., `file.csv#header=absent`, `file.csv#delimiter=tab`, `file.csv#delimiter=%3B`, `file.csv#quotechar=singlequote;escapechar=backslash`, `file.csv#encoding=latin1`)   


## Design patterns and examples

### List the content of the file, projecting the first 100 rows and only the ?id and ?name bindings

{% highlight bash %}
  SELECT DISTINCT ?id ?name
  FROM <file:filename.csv>
  WHERE {}
  LIMIT 100
{% endhighlight %}

### Skip bad rows

{% highlight bash %}
  SELECT ...
  WHERE { FILTER (BOUND(?d)) }
{% endhighlight %}

### Compute additional columns

{% highlight bash %}
  SELECT ...
  WHERE {
    BIND (URI(CONCAT('http://example.com/ns#', ?b)) AS ?uri)
    BIND (STRLANG(?a, 'en') AS ?with_language_tag)
  }
{% endhighlight %}

### CONSTRUCT an RDF graph

{% highlight bash %}
  CONSTRUCT {
    ?URI a ex:Organization;
	ex:name ?NameWithLang;
	ex:CIK ?CIK;
	ex:LEI ?LEI;
	ex:ticker ?Stock_ticker;
  }
  FROM <file:companies.csv>
  WHERE {
    BIND (URI(CONCAT('companies/', ?Stock_ticker)) AS ?URI)
    BIND (STRLANG(?Name, "en") AS ?NameWithLang)
  }
{% endhighlight %}

### Generate URIs based on the row number

{% highlight bash %}
  ...
  WHERE {
    BIND (URI(CONCAT('companies/', STR(?ROWNUM))) AS ?URI)
  }
  ...
{% endhighlight %}

### Count the number of triples from a csv file

{% highlight bash %}
  SELECT (COUNT(*) AS ?count)
  FROM <file.csv>
  WHERE {}
{% endhighlight %}

### Provide CSV file encoding and header information in the URL 

{% highlight bash %}
  CONSTRUCT { ... }
  FROM <file.csv#encoding=utf-8;header=absent>
{% endhighlight %}

This is equivalent to using `<file.csv>` in the `FROM` clause and specifying `--no-header-row` and `--encoding` on the command line.

### Legacy convention for header row

Earlier versions of Tarql had `--no-header-row`/`#header=absent` as the default, and required the use of a convention to enable the header row:

{% highlight bash %}
  SELECT ?First_name ?Last_name ?Phone_number
  WHERE { ... }
  OFFSET 1
{% endhighlight %}

Here, the `OFFSET 1` is a convention that indicates that the first row is to be used to provide variable names, and not as data. This convention is still supported, but will only be recognized if none of the header-specifying command line options or URL fragment arguments are used.

### Injected functions

Tarql has extra funcionalities that can be used to filter or transform column values into standard literals with datatype.

{% highlight bash %}
  ...
  WHERE {
    BIND (tarql:date(?Release_Date, 'MM/dd/yyyy') AS ?ReleaseDate)
    BIND (tarql:date-time(?Release_Date, 'MM/dd/yyyy') AS ?ReleaseDateTime)
    BIND (tarql:currency(?Production_Budget, 'USD') AS ?ProductionBudget)
  }
  ...
{% endhighlight %}

Where:

- ?Release_Date ("12/18/2009") is mapped to "2009-12-18"^^<http://www.w3.org/2001/XMLSchema#date>
- ?Release_Date ("12/18/2009") is mapped to "2009-12-18T00:00:00.000Z"^^<http://www.w3.org/2001/XMLSchema#dateTime>
- ?Production_Budget ("$425,000,000") is mapped to "425000000"^^<http://dbpedia.org/datatype/usDollar>

So far, there are only three filters implemented. If you need other filter, please check the [documentation page](http://tarql.github.io/docs/) for reporting issues.

- `tarql:date` and `tarql:date-time` use the Date and Time Patterns defined in the Java 7 specifications for [SimpleDateFormat class](http://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html).
- `tarql:currency` use the three letters currency code standarized in [ISO 4217](http://www.currency-iso.org/dam/downloads/table_a1.xml).

Last Update: {{ site.time | date: '%B %d, %Y' }}
