---
description: 版本：10
---

# 9.14. XML函式

本節中描述的函數和類函數表示式對 xml 型別的值進行操作。有關 xml 型別的訊息，請查看[第 8.13 節](../data-types/8.13.-xml-xing-bie.md)。這裡不再重複用於轉換為 xml 型別的函數表示式 xmlparse 和 xmlserialize。使用大多數這些函數需要使用 configure --with-libxml 編譯安裝。

## 9.14.1. 産生 XML 內容

一組函數和類函數的表示式可用於從 SQL 資料産生 XML 內容。因此，它們特別適合將查詢結果格式化為 XML 文件以便在用戶端應用程序中進行處理。

### **9.14.1.1. xmlcomment**

```text
xmlcomment(text)
```

函數 xmlcomment 建立一個 XML 字串，其中包含指定文字作為內容的 XML 註釋。文字不能包含「 -- 」或以「 - 」結尾，以便産生的結構是有效的 XML 註釋。 如果參數為 null，則結果為 null。

例如：

```text
SELECT xmlcomment('hello');

  xmlcomment
--------------
 <!--hello-->
```

### **9.14.1.2. xmlconcat**

```text
xmlconcat(xml[, ...])
```

函數 xmlconcat 連接列表中各個 XML 字串，以建立包含 XML 內容片段的單個字串。空值會被忽略；如果都沒有非空值參數，則結果僅為 null。

例如：

```text
SELECT xmlconcat('<abc/>', '<bar>foo</bar>');

      xmlconcat
----------------------
 <abc/><bar>foo</bar>
```

XML 宣告（如果存在）組合如下。如果所有參數值具有相同的 XML 版本宣告，則在結果中使用該版本，否則不使用任何版本。如果所有參數值都具有獨立宣告值「yes」，則在結果中使用該值。如果所有參數值都具有獨立的宣告值且至少有一個為「no」，則在結果中使用該值。否則結果將沒有獨立宣告。如果確定結果需要獨立宣告但沒有版本聲明，則將使用版本為 1.0 的版本宣告，因為 XML 要求 XML 宣告包含版本宣告。在所有情況下都會忽略編碼宣告並將其刪除。

例如：

```text
SELECT xmlconcat('<?xml version="1.1"?><foo/>', '<?xml version="1.1" standalone="no"?><bar/>');

             xmlconcat
-----------------------------------
 <?xml version="1.1"?><foo/><bar/>
```

### **9.14.1.3. xmlelement**

```text
xmlelement(name name [, xmlattributes(value [AS attname] [, ... ])] [, content, ...])
```

xmlelement 表示式産生具有給定名稱、屬性和內容的 XML 元素。

範例：

```text
SELECT xmlelement(name foo);

 xmlelement
------------
 <foo/>

SELECT xmlelement(name foo, xmlattributes('xyz' as bar));

    xmlelement
------------------
 <foo bar="xyz"/>

SELECT xmlelement(name foo, xmlattributes(current_date as bar), 'cont', 'ent');

             xmlelement
-------------------------------------
 <foo bar="2007-01-26">content</foo>
```

透過用 _xHHHH_ 序列替換有問題的字符來轉譯非有效 XML 名稱的元素和屬性名稱，其中 HHHH 是十六進位表示法中字元的 Unicode 代碼。例如：

```text
SELECT xmlelement(name "foo$bar", xmlattributes('xyz' as "a&b"));

            xmlelement
----------------------------------
 <foo_x0024_bar a_x0026_b="xyz"/>
```

如果屬性值是引用欄位，則無需明確指定屬性名稱，在這種情況下，預設情況下欄位的名稱將用作屬性名稱。在其他情況下，必須為該屬性明確指定名稱。所以這個例子是有效的：

```text
CREATE TABLE test (a xml, b xml);
SELECT xmlelement(name test, xmlattributes(a, b)) FROM test;
```

但這些不行：

```text
SELECT xmlelement(name test, xmlattributes('constant'), a, b) FROM test;
SELECT xmlelement(name test, xmlattributes(func(a, b))) FROM test;
```

元素內容（如果已指定）將根據其資料型別進行格式化。如果內容本身是 xml 型別，則可以建構複雜的 XML 文件。例如：

```text
SELECT xmlelement(name foo, xmlattributes('xyz' as bar),
                            xmlelement(name abc),
                            xmlcomment('test'),
                            xmlelement(name xyz));

                  xmlelement
----------------------------------------------
 <foo bar="xyz"><abc/><!--test--><xyz/></foo>
```

其他型別的內容將被格式化為有效的 XML 字元資料。這尤其意味著字符 &lt;、&gt; 和 ＆ 將被轉換為其他形式。二進位資料（資料型別 bytea）將以 base64 或十六進位編碼表示，具體取決於組態參數 xmlbinary 的設定。為了使 SQL 和 PostgreSQL 資料型別與 XML Schema 規範保持一致，預計各種資料型別的特定行為將會各自發展，此時將出現更精確的描述。

### **9.14.1.4. xmlforest**

```text
xmlforest(content [AS name] [, ...])
```

xmlforest 表示式使用給定的名稱和內容産生元素的 XML 序列。

範例：

```text
SELECT xmlforest('abc' AS foo, 123 AS bar);

          xmlforest
------------------------------
 <foo>abc</foo><bar>123</bar>


SELECT xmlforest(table_name, column_name)
FROM information_schema.columns
WHERE table_schema = 'pg_catalog';

                                         xmlforest
-------------------------------------------------------------------------------------------
 <table_name>pg_authid</table_name><column_name>rolname</column_name>
 <table_name>pg_authid</table_name><column_name>rolsuper</column_name>
 ...
```

如第二個範例所示，如果內容值是欄位引用，則可以省略元素名稱，在這種情況下，預設情況下使用欄位名稱。 否則，必須指定名稱。

非有效的 XML 名稱的元素名稱將被轉譯，如上面的 xmlelement 所示。類似地，內容資料會被轉譯以産生有效的 XML 內容，除非它已經是 xml 型別。

請注意，如果 XML 序列由多個元素組成，則它們不是有效的 XML 文件，因此將 xmlforest 表示式包裝在 xmlelement 中可能很有用。

### **9.14.1.5. xmlpi**

```text
xmlpi(name target [, content])
```

xmlpi 表示式建立 XML 處理指令。內容（如果存在）不得包含字元序列 ?&gt;。

例如：

```text
SELECT xmlpi(name php, 'echo "hello world";');

            xmlpi
-----------------------------
 <?php echo "hello world";?>
```

### **9.14.1.6. xmlroot**

```text
xmlroot(xml, version text | no value [, standalone yes|no|no value])
```

xmlroot 表示式改變 XML 值的根節點屬性。如果指定了版本，它將替換根節點的版本宣告中的值；如果指定了獨立設定，則它將替換根節點的獨立宣告中的值。

```text
SELECT xmlroot(xmlparse(document '<?xml version="1.1"?><content>abc</content>'),
               version '1.0', standalone yes);

                xmlroot
----------------------------------------
 <?xml version="1.0" standalone="yes"?>
 <content>abc</content>
```

### **9.14.1.7. xmlagg**

```text
xmlagg(xml)
```

與此處描述的其他函數不同，函數 xmlagg 是一個彙總函數。它將輸入值連接到彙總函數呼叫，就像 xmlconcat 一樣，除了它是跨資料列而不是在單個資料列中的表示式進行連接。有關彙總函數的其他訊息，請參閱[第 9.20 節](9.20.-hui-zong-han-shi.md)。

例如：

```text
CREATE TABLE test (y int, x xml);
INSERT INTO test VALUES (1, '<foo>abc</foo>');
INSERT INTO test VALUES (2, '<bar/>');
SELECT xmlagg(x) FROM test;
        xmlagg
----------------------
 <foo>abc</foo><bar/>
```

要確定連接的順序，可以將 ORDER BY 子句加到彙總呼叫中，如第 4.2.7 節中所述。例如：

```text
SELECT xmlagg(x ORDER BY y DESC) FROM test;
        xmlagg
----------------------
 <bar/><foo>abc</foo>
```

以前的版本中推薦使用以下非標準方法，在特定情況下可能仍然有用：

```text
SELECT xmlagg(x) FROM (SELECT * FROM test ORDER BY y DESC) AS tab;
        xmlagg
----------------------
 <bar/><foo>abc</foo>
```

## 9.14.2. XML Predicates

The expressions described in this section check properties of `xml` values.

### **9.14.2.1. IS DOCUMENT**

```text
xml IS DOCUMENT
```

The expression `IS DOCUMENT` returns true if the argument XML value is a proper XML document, false if it is not \(that is, it is a content fragment\), or null if the argument is null. See [Section 8.13](https://www.postgresql.org/docs/10/static/datatype-xml.html) about the difference between documents and content fragments.

### **9.14.2.2. XMLEXISTS**

```text
XMLEXISTS(text PASSING [BY REF] xml [BY REF])
```

The function `xmlexists` returns true if the XPath expression in the first argument returns any nodes, and false otherwise. \(If either argument is null, the result is null.\)

Example:

```text
SELECT xmlexists('//town[text() = ''Toronto'']' PASSING BY REF '<towns><town>Toronto</town><town>Ottawa</town></towns>');

 xmlexists
------------
 t
(1 row)
```

The `BY REF` clauses have no effect in PostgreSQL, but are allowed for SQL conformance and compatibility with other implementations. Per SQL standard, the first `BY REF` is required, the second is optional. Also note that the SQL standard specifies the `xmlexists` construct to take an XQuery expression as first argument, but PostgreSQL currently only supports XPath, which is a subset of XQuery.

### **9.14.2.3. xml\_is\_well\_formed**

```text
xml_is_well_formed(text)
xml_is_well_formed_document(text)
xml_is_well_formed_content(text)
```

These functions check whether a `text` string is well-formed XML, returning a Boolean result. `xml_is_well_formed_document` checks for a well-formed document, while `xml_is_well_formed_content` checks for well-formed content. `xml_is_well_formed` does the former if the [xmloption](https://www.postgresql.org/docs/10/static/runtime-config-client.html#GUC-XMLOPTION) configuration parameter is set to `DOCUMENT`, or the latter if it is set to `CONTENT`. This means that `xml_is_well_formed` is useful for seeing whether a simple cast to type `xml` will succeed, whereas the other two functions are useful for seeing whether the corresponding variants of `XMLPARSE` will succeed.

Examples:

```text
SET xmloption TO DOCUMENT;
SELECT xml_is_well_formed('<>');
 xml_is_well_formed 
--------------------
 f
(1 row)

SELECT xml_is_well_formed('<abc/>');
 xml_is_well_formed 
--------------------
 t
(1 row)

SET xmloption TO CONTENT;
SELECT xml_is_well_formed('abc');
 xml_is_well_formed 
--------------------
 t
(1 row)

SELECT xml_is_well_formed_document('<pg:foo xmlns:pg="http://postgresql.org/stuff">bar</pg:foo>');
 xml_is_well_formed_document 
-----------------------------
 t
(1 row)

SELECT xml_is_well_formed_document('<pg:foo xmlns:pg="http://postgresql.org/stuff">bar</my:foo>');
 xml_is_well_formed_document 
-----------------------------
 f
(1 row)
```

The last example shows that the checks include whether namespaces are correctly matched.

## 9.14.3. Processing XML

To process values of data type `xml`, PostgreSQL offers the functions `xpath` and `xpath_exists`, which evaluate XPath 1.0 expressions, and the `XMLTABLE` table function.

### **9.14.3.1. xpath**

```text
xpath(xpath, xml [, nsarray])
```

The function `xpath` evaluates the XPath expression _`xpath`_ \(a `text` value\) against the XML value _`xml`_. It returns an array of XML values corresponding to the node set produced by the XPath expression. If the XPath expression returns a scalar value rather than a node set, a single-element array is returned.

The second argument must be a well formed XML document. In particular, it must have a single root node element.

The optional third argument of the function is an array of namespace mappings. This array should be a two-dimensional `text` array with the length of the second axis being equal to 2 \(i.e., it should be an array of arrays, each of which consists of exactly 2 elements\). The first element of each array entry is the namespace name \(alias\), the second the namespace URI. It is not required that aliases provided in this array be the same as those being used in the XML document itself \(in other words, both in the XML document and in the `xpath`function context, aliases are _local_\).

Example:

```text
SELECT xpath('/my:a/text()', '<my:a xmlns:my="http://example.com">test</my:a>',
             ARRAY[ARRAY['my', 'http://example.com']]);

 xpath  
--------
 {test}
(1 row)
```

To deal with default \(anonymous\) namespaces, do something like this:

```text
SELECT xpath('//mydefns:b/text()', '<a xmlns="http://example.com"><b>test</b></a>',
             ARRAY[ARRAY['mydefns', 'http://example.com']]);

 xpath
--------
 {test}
(1 row)
```

### **9.14.3.2. xpath\_exists**

```text
xpath_exists(xpath, xml [, nsarray])
```

The function `xpath_exists` is a specialized form of the `xpath` function. Instead of returning the individual XML values that satisfy the XPath, this function returns a Boolean indicating whether the query was satisfied or not. This function is equivalent to the standard `XMLEXISTS` predicate, except that it also offers support for a namespace mapping argument.

Example:

```text
SELECT xpath_exists('/my:a/text()', '<my:a xmlns:my="http://example.com">test</my:a>',
                     ARRAY[ARRAY['my', 'http://example.com']]);

 xpath_exists  
--------------
 t
(1 row)
```

### **9.14.3.3. xmltable**

```text
xmltable( [XMLNAMESPACES(namespace uri AS namespace name[, ...]), ]
          row_expression PASSING [BY REF] document_expression [BY REF]
          COLUMNS name { type [PATH column_expression] [DEFAULT default_expression] [NOT NULL | NULL]
                        | FOR ORDINALITY }
                   [, ...]
)
```

The `xmltable` function produces a table based on the given XML value, an XPath filter to extract rows, and an optional set of column definitions.

The optional `XMLNAMESPACES` clause is a comma-separated list of namespaces. It specifies the XML namespaces used in the document and their aliases. A default namespace specification is not currently supported.

The required _`row_expression`_ argument is an XPath expression that is evaluated against the supplied XML document to obtain an ordered sequence of XML nodes. This sequence is what `xmltable` transforms into output rows.

_`document_expression`_ provides the XML document to operate on. The `BY REF` clauses have no effect in PostgreSQL, but are allowed for SQL conformance and compatibility with other implementations. The argument must be a well-formed XML document; fragments/forests are not accepted.

The mandatory `COLUMNS` clause specifies the list of columns in the output table. If the `COLUMNS` clause is omitted, the rows in the result set contain a single column of type `xml` containing the data matched by _`row_expression`_. If `COLUMNS` is specified, each entry describes a single column. See the syntax summary above for the format. The column name and type are required; the path, default and nullability clauses are optional.

A column marked `FOR ORDINALITY` will be populated with row numbers matching the order in which the output rows appeared in the original input XML document. At most one column may be marked `FOR ORDINALITY`.

The `column_expression` for a column is an XPath expression that is evaluated for each row, relative to the result of the _`row_expression`_, to find the value of the column. If no `column_expression` is given, then the column name is used as an implicit path.

If a column's XPath expression returns multiple elements, an error is raised. If the expression matches an empty tag, the result is an empty string \(not `NULL`\). Any `xsi:nil` attributes are ignored.

The text body of the XML matched by the _`column_expression`_ is used as the column value. Multiple `text()` nodes within an element are concatenated in order. Any child elements, processing instructions, and comments are ignored, but the text contents of child elements are concatenated to the result. Note that the whitespace-only `text()` node between two non-text elements is preserved, and that leading whitespace on a `text()` node is not flattened.

If the path expression does not match for a given row but _`default_expression`_ is specified, the value resulting from evaluating that expression is used. If no `DEFAULT` clause is given for the column, the field will be set to `NULL`. It is possible for a _`default_expression`_ to reference the value of output columns that appear prior to it in the column list, so the default of one column may be based on the value of another column.

Columns may be marked `NOT NULL`. If the _`column_expression`_ for a `NOT NULL` column does not match anything and there is no `DEFAULT` or the _`default_expression`_ also evaluates to null, an error is reported.

Unlike regular PostgreSQL functions, _`column_expression`_ and _`default_expression`_ are not evaluated to a simple value before calling the function. _`column_expression`_ is normally evaluated exactly once per input row, and _`default_expression`_ is evaluated each time a default is needed for a field. If the expression qualifies as stable or immutable the repeat evaluation may be skipped. Effectively `xmltable` behaves more like a subquery than a function call. This means that you can usefully use volatile functions like `nextval` in _`default_expression`_, and _`column_expression`_ may depend on other parts of the XML document.

Examples:

```text
CREATE TABLE xmldata AS SELECT
xml $$
<ROWS>
  <ROW id="1">
    <COUNTRY_ID>AU</COUNTRY_ID>
    <COUNTRY_NAME>Australia</COUNTRY_NAME>
  </ROW>
  <ROW id="5">
    <COUNTRY_ID>JP</COUNTRY_ID>
    <COUNTRY_NAME>Japan</COUNTRY_NAME>
    <PREMIER_NAME>Shinzo Abe</PREMIER_NAME>
    <SIZE unit="sq_mi">145935</SIZE>
  </ROW>
  <ROW id="6">
    <COUNTRY_ID>SG</COUNTRY_ID>
    <COUNTRY_NAME>Singapore</COUNTRY_NAME>
    <SIZE unit="sq_km">697</SIZE>
  </ROW>
</ROWS>
$$ AS data;

SELECT xmltable.*
  FROM xmldata,
       XMLTABLE('//ROWS/ROW'
                PASSING data
                COLUMNS id int PATH '@id',
                        ordinality FOR ORDINALITY,
                        "COUNTRY_NAME" text,
                        country_id text PATH 'COUNTRY_ID',
                        size_sq_km float PATH 'SIZE[@unit = "sq_km"]',
                        size_other text PATH
                             'concat(SIZE[@unit!="sq_km"], " ", SIZE[@unit!="sq_km"]/@unit)',
                        premier_name text PATH 'PREMIER_NAME' DEFAULT 'not specified') ;

 id | ordinality | COUNTRY_NAME | country_id | size_sq_km |  size_other  | premier_name  
----+------------+--------------+------------+------------+--------------+---------------
  1 |          1 | Australia    | AU         |            |              | not specified
  5 |          2 | Japan        | JP         |            | 145935 sq_mi | Shinzo Abe
  6 |          3 | Singapore    | SG         |        697 |              | not specified
```

The following example shows concatenation of multiple text\(\) nodes, usage of the column name as XPath filter, and the treatment of whitespace, XML comments and processing instructions:

```text
CREATE TABLE xmlelements AS SELECT
xml $$
  <root>
   <element>  Hello<!-- xyxxz -->2a2<?aaaaa?> <!--x-->  bbb<x>xxx</x>CC  </element>
  </root>
$$ AS data;

SELECT xmltable.*
  FROM xmlelements, XMLTABLE('/root' PASSING data COLUMNS element text);
       element        
----------------------
   Hello2a2   bbbCC  
```

The following example illustrates how the `XMLNAMESPACES` clause can be used to specify the default namespace, and a list of additional namespaces used in the XML document as well as in the XPath expressions:

```text
WITH xmldata(data) AS (VALUES ('
<example xmlns="http://example.com/myns" xmlns:B="http://example.com/b">
 <item foo="1" B:bar="2"/>
 <item foo="3" B:bar="4"/>
 <item foo="4" B:bar="5"/>
</example>'::xml)
)
SELECT xmltable.*
  FROM XMLTABLE(XMLNAMESPACES('http://example.com/myns' AS x,
                              'http://example.com/b' AS "B"),
             '/x:example/x:item'
                PASSING (SELECT data FROM xmldata)
                COLUMNS foo int PATH '@foo',
                  bar int PATH '@B:bar');
 foo | bar
-----+-----
   1 |   2
   3 |   4
   4 |   5
(3 rows)
```

## 9.14.4. Mapping Tables to XML

The following functions map the contents of relational tables to XML values. They can be thought of as XML export functionality:

```text
table_to_xml(tbl regclass, nulls boolean, tableforest boolean, targetns text)
query_to_xml(query text, nulls boolean, tableforest boolean, targetns text)
cursor_to_xml(cursor refcursor, count int, nulls boolean,
              tableforest boolean, targetns text)
```

The return type of each function is `xml`.

`table_to_xml` maps the content of the named table, passed as parameter _`tbl`_. The `regclass` type accepts strings identifying tables using the usual notation, including optional schema qualifications and double quotes. `query_to_xml` executes the query whose text is passed as parameter _`query`_ and maps the result set. `cursor_to_xml` fetches the indicated number of rows from the cursor specified by the parameter _`cursor`_. This variant is recommended if large tables have to be mapped, because the result value is built up in memory by each function.

If _`tableforest`_ is false, then the resulting XML document looks like this:

```text
<tablename>
  <row>
    <columnname1>data</columnname1>
    <columnname2>data</columnname2>
  </row>

  <row>
    ...
  </row>

  ...
</tablename>
```

If _`tableforest`_ is true, the result is an XML content fragment that looks like this:

```text
<tablename>
  <columnname1>data</columnname1>
  <columnname2>data</columnname2>
</tablename>

<tablename>
  ...
</tablename>

...
```

If no table name is available, that is, when mapping a query or a cursor, the string `table` is used in the first format, `row` in the second format.

The choice between these formats is up to the user. The first format is a proper XML document, which will be important in many applications. The second format tends to be more useful in the `cursor_to_xml` function if the result values are to be reassembled into one document later on. The functions for producing XML content discussed above, in particular `xmlelement`, can be used to alter the results to taste.

The data values are mapped in the same way as described for the function `xmlelement` above.

The parameter _`nulls`_ determines whether null values should be included in the output. If true, null values in columns are represented as:

```text
<columnname xsi:nil="true"/>
```

where `xsi` is the XML namespace prefix for XML Schema Instance. An appropriate namespace declaration will be added to the result value. If false, columns containing null values are simply omitted from the output.

The parameter _`targetns`_ specifies the desired XML namespace of the result. If no particular namespace is wanted, an empty string should be passed.

The following functions return XML Schema documents describing the mappings performed by the corresponding functions above:

```text
table_to_xmlschema(tbl regclass, nulls boolean, tableforest boolean, targetns text)
query_to_xmlschema(query text, nulls boolean, tableforest boolean, targetns text)
cursor_to_xmlschema(cursor refcursor, nulls boolean, tableforest boolean, targetns text)
```

It is essential that the same parameters are passed in order to obtain matching XML data mappings and XML Schema documents.

The following functions produce XML data mappings and the corresponding XML Schema in one document \(or forest\), linked together. They can be useful where self-contained and self-describing results are wanted:

```text
table_to_xml_and_xmlschema(tbl regclass, nulls boolean, tableforest boolean, targetns text)
query_to_xml_and_xmlschema(query text, nulls boolean, tableforest boolean, targetns text)
```

In addition, the following functions are available to produce analogous mappings of entire schemas or the entire current database:

```text
schema_to_xml(schema name, nulls boolean, tableforest boolean, targetns text)
schema_to_xmlschema(schema name, nulls boolean, tableforest boolean, targetns text)
schema_to_xml_and_xmlschema(schema name, nulls boolean, tableforest boolean, targetns text)

database_to_xml(nulls boolean, tableforest boolean, targetns text)
database_to_xmlschema(nulls boolean, tableforest boolean, targetns text)
database_to_xml_and_xmlschema(nulls boolean, tableforest boolean, targetns text)
```

Note that these potentially produce a lot of data, which needs to be built up in memory. When requesting content mappings of large schemas or databases, it might be worthwhile to consider mapping the tables separately instead, possibly even through a cursor.

The result of a schema content mapping looks like this:

```text
<schemaname>

table1-mapping

table2-mapping

...

</schemaname>
```

where the format of a table mapping depends on the _`tableforest`_ parameter as explained above.

The result of a database content mapping looks like this:

```text
<dbname>

<schema1name>
  ...
</schema1name>

<schema2name>
  ...
</schema2name>

...

</dbname>
```

where the schema mapping is as above.

As an example of using the output produced by these functions, [Figure 9.1](https://www.postgresql.org/docs/10/static/functions-xml.html#XSLT-XML-HTML) shows an XSLT stylesheet that converts the output of `table_to_xml_and_xmlschema` to an HTML document containing a tabular rendition of the table data. In a similar manner, the results from these functions can be converted into other XML-based formats.

#### **Figure 9.1. XSLT Stylesheet for Converting SQL/XML Output to HTML**

```text
<?xml version="1.0"?>
<xsl:stylesheet version="1.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:xsd="http://www.w3.org/2001/XMLSchema"
    xmlns="http://www.w3.org/1999/xhtml"
>

  <xsl:output method="xml"
      doctype-system="http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd"
      doctype-public="-//W3C/DTD XHTML 1.0 Strict//EN"
      indent="yes"/>

  <xsl:template match="/*">
    <xsl:variable name="schema" select="//xsd:schema"/>
    <xsl:variable name="tabletypename"
                  select="$schema/xsd:element[@name=name(current())]/@type"/>
    <xsl:variable name="rowtypename"
                  select="$schema/xsd:complexType[@name=$tabletypename]/xsd:sequence/xsd:element[@name='row']/@type"/>

    <html>
      <head>
        <title><xsl:value-of select="name(current())"/></title>
      </head>
      <body>
        <table>
          <tr>
            <xsl:for-each select="$schema/xsd:complexType[@name=$rowtypename]/xsd:sequence/xsd:element/@name">
              <th><xsl:value-of select="."/></th>
            </xsl:for-each>
          </tr>

          <xsl:for-each select="row">
            <tr>
              <xsl:for-each select="*">
                <td><xsl:value-of select="."/></td>
              </xsl:for-each>
            </tr>
          </xsl:for-each>
        </table>
      </body>
    </html>
  </xsl:template>

</xsl:stylesheet>
```
