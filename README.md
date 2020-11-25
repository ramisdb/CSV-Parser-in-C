# CSV-Parser-in-C
Handles CSV TSV and Delimiter separated files. Handles unusual cases, odd time formats...

This small program/library attempts to provide an easy and fast way to import CSV data into a program. 
In addition to comma separated files it will also handle tab separated (TSV) and delimiter separated files (DSV).
No attempt to be RFC 4180 compliant was made, but it is fully capable of handling all the cases enumerated on
the Wikipedia CSV page https://en.wikipedia.org/wiki/Comma-separated_values.

The program has no dependancies other than a standard compilation environment, and its source code is in the public domain.

 
## Key Capabilities

* Can generate sql insert statements

* Idenitfies data types (string, integer , floating point, date/time, and NULLs) 

* Can normalize columns into their majority data-type

* Handles multiline strings

* Can parse european style numbers e.g "100 000 000,00" == 1.0e6

* Can parse english numbers with commas e.g. "1,000,000" == 1000000

* Arbitrary date/time format parsing including dates before 1/1/1970

* Respects embedded quotes in strings. e.g. "abc""def" == abc"def

* Respects double quotes within single quote strings e.g. 'abc"def"' == abc"def"

* Parses C backslash escapements i.e. \r \n \t \\' \\"  \\ ... 

* Leading and trailing whitespace removal

## Using the API


```
#include  <stdio.h>
#include  <time.h>
#include "csv_parser.h"

void
myFunction()
{
  CSV * csv=openCSV("myFilename.tsv");
  int   row,col;
  
  if(csv!=NULL)
  {
    csv->delimiter='\t';      // change the delimiter from comma to tab 
  
    if(parseCSV(csv)<0);      // now actually parse the file
    {
      fprintf(stderr,"Error: %s\n",csv->errorMsg);
      closeCSV(csv);
      return;
    }
    
    normalizeCSV(csv);        // examine each column and force data type to the majority 
    
    for(row=0;row<csv->rows;row++)      // iterate through the CSVITEMS contained in each row and column
      for(col=0;col<csv->cols;col++)
        switch(csv->item[row][col].type)
        {
            case integer:        printf("%lld",csv->item[row][col].integer);break;
            case floatingPoint:  printf("%lf",csv->item[row][col].floatingPoint);break;
            case string:         printf("\"%s\"",csv->item[row][col].string);break;
            case dateTime:       {
                                    struct tm *t=&csv->item[row][col].dateTime;
                                    printf("%4d-%02d-%02d %02d:%02d:%02d (%ld)",
                                             t->tm_year+1900,
                                             t->tm_mon+1,
                                             t->tm_mday,
                                             t->tm_hour,
                                             t->tm_min,
                                             t->tm_sec,
                                             mktime(t) // use this if you want unix time
                                          );
                                          
                                 }
                                 break;
            case nil:            printf("%16s","NIL");break; 
        }
        
    closeCSV(csv);         // free up the resources
  }
  
}
```


#### ``CSV * openCSV(char *fileName)``

Allocates the CSV object and initializes the parameters to default values. It also opens the specified file.

On success it will return a pointer to the ``CSV`` object. On failure it will return ``NULL`` The global variable ``csvErrNo`` will be set to the value of ``errno`` at the time of failure. By nature this value is not thread safe.

Once you have a valid ``CSV *`` the parser's parameters may be changed by directly modifying the
contents of the struct:

Member | Type | Default Value | Purpose
-------|------|---------------|--------
``stripLeadingWhite`` | boolean | 1 | remove leading whitespace characters from cells
``stripTrailingWhite``| boolean | 1 | remove trailing whitespace characters from cells
``doubleQuoteEscape`` | boolean | 0 |  '\"\"' within strings is used to embed '\"' characters
``singleQuoteNest``   | boolean | 1 |  strings may be bounded by \' pairs and '\"' characters within are ignored
``backslashEscape``   | boolean | 1 |  characters preceded by '\\' are translated and escaped
``allEscapes``        | boolean | 1 |  all '\\' escape sequences known by the 'C' compiler are translated, if false only backslash, single quote, and double quote 
``europeanDecimal``   | boolean | 0 |  numbers like '123 456,78' will be parsed as 123456.78
``tryParsingStrings`` | boolean | 0 |  look inside quoted strings for dates and numbers to parse, if false anything quoted is a string
``delimiter``         | char    | ',' |  use the character 'c' as a column delimiter e.g \\t 
``timeFormat``        |  string | %Y-%m-%d %H:%M:%S | set the format for parsing a date/time see manpage for strptime()

#### ``int parseCSV(CSV *csv)``

Performs the actual parse of the file. Allocates a two dimensional array of CSVITEMS which will represent the parsed rows and columns of the file.

On success it will return the number of rows it located. On failure it will return -1. The failure's may be found by examining csv->errorMsg;

``parseCSV()`` will examine each cell of the delimited file and attempt to identify its data type. If the data type is determined to be a `string` it will process the escapments as specified by the parameter settings The results are stored in a 2 dimensional array of `CSVITEMS`:

```
CSVITEM
{
   union
   {
      int64_t     integer;
      double      floatingPoint;
      struct tm   dateTime;   // times are stored this way so they can exceed time_t limitations (call mktime())
   };
   byte *         string;
   enum CSVitemType  type;
};
```

Each `CSVITEM`'s type variable will be set to the type which the parser has recognized:

Type name     | Storage type | Details
--------------|--------------|---------
integer       | int64_t   | an integer is identified when a quantity contained neither a decimal point or and exponent
floatingPoint | double    | flointing point is identified when there is a decimal point present (or , for euro) or an exponent
dateTime      | struct tm | times are stored in this format to allow dates before 1/1/1970 or after time_t overflow to be parsed 
string        | byte *    | strings are recognized by being non-numeric or by being quoted (if not overriden by ``tryParsingStrings`` 
nil           | no storage| nil is created when the column was empty or when ``normalizeCSV()`` could not cast the value to the majority type

**It is important to note** that regardless of the parsed type, you may always examine ``item[row][column]->string`` to access the contents of that cell of the original file.

If your parsing of a date/time value fails and produces a string, check ``csv->timeFormat`` and the man page for ``strptime()`` to ensure you have the correct parsing string.

#### ``void normalizeCSV(char *fileName)``

``normalizeCSV()`` examines each column in the parsed `CSV` object to find the majority type of that column. It then casts all the members of that column to the majority type, or sets ``item[row][col->type`` to ``nil`` if it is unable to do so. **NOTE** If a column's majority type is ``integer`` but contains any floating point values, the column will be converted to ``floatingPoint``. This is done to prevent unintentional precision loss.

This function will not alter row 0's contents if it believes row 0 contains column names.

#### ``void normalizeCSVColumn(CSV *csv,int columnN,int startRow)``

This is the worker function for ``normalizeCSV()``. It can be called when you wish to force a single column to its majority type without altering any other columns.

#### ``CSV * closeCSV(CSV *csv)``

Free up the memory from the parser. It returns NULL


## Usage notes

This program reads in the entire file at once and would be inapproriate for streaming or pipe applications.

To use as a library, comment out the ``#define TESTCSV`` in ``parse_csv.h`` 

To use it to generate SQL insert statements do the following

```
gcc csv_parser.c -o csv
```

Given a csv that looks like this:
```
"Airport","Rank","Total Operations","GA Operations","JetA Price","100LL Price"
ATL,1,898356,7892,7.84,8.53
ORD,2,867635,6525,7.7,8.59
LAX,3,696890,24290,5.94,6.35
DEN,5,572520,4376,6.86,7
LAS,7,535740,42617,5.89,8.09
IAH,8,470780,11091,5.47,5.09
SFO,10,450391,12691,7.6,7.99
EWR,12,431214,11058,7.72,8.19
MIA,13,414234,17688,7.75,8.15
MSP,14,412898,11489,7.78,8.07
SEA,15,412170,2802,6.82,8.46
BOS,16,395811,14582,8.06,7.99
APA,21,332111,145807,3.75,5.6
MCO,22,323914,15214,6.74,6.55
```

```
./csv -sql myTable myfile.csv
```

 will produce this:
 
 ```
INSERT INTO myTable VALUES('ATL',1,898356,7892,7.840000000000000,8.529999999999999);
INSERT INTO myTable VALUES('ORD',2,867635,6525,7.700000000000000,8.590000000000000);
INSERT INTO myTable VALUES('LAX',3,696890,24290,5.940000000000000,6.350000000000000);
INSERT INTO myTable VALUES('DEN',5,572520,4376,6.860000000000000,7.000000000000000);
INSERT INTO myTable VALUES('LAS',7,535740,42617,5.890000000000000,8.090000000000000);
INSERT INTO myTable VALUES('IAH',8,470780,11091,5.470000000000000,5.090000000000000);
INSERT INTO myTable VALUES('SFO',10,450391,12691,7.600000000000000,7.990000000000000);
INSERT INTO myTable VALUES('EWR',12,431214,11058,7.720000000000000,8.190000000000000);
INSERT INTO myTable VALUES('MIA',13,414234,17688,7.750000000000000,8.150000000000000);
INSERT INTO myTable VALUES('MSP',14,412898,11489,7.780000000000000,8.070000000000000);
INSERT INTO myTable VALUES('SEA',15,412170,2802,6.820000000000000,8.460000000000001);
INSERT INTO myTable VALUES('BOS',16,395811,14582,8.060000000000000,7.990000000000000);
INSERT INTO myTable VALUES('APA',21,332111,145807,3.750000000000000,5.600000000000000);
INSERT INTO myTable VALUES('MCO',22,323914,15214,6.740000000000000,6.550000000000000);

 ```
 



 







