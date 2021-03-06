---
id: DataImport
title: Data Import
permalink: /docs/manual/DataImport.html
---

We can load data in either distributed or embedded mode. Data import
typically involves the following steps:

* Model our data so that each entity or object is associated with a
  cell type. Make sure that each field has a compatible data type with
  the corresponding field in the data set.

* Implement the code to convert the data from its original format to
  cell objects. We specify the logic to fill the data fields of a cell
  object in this step.

* Dump the memory storage to an persistent storage. We can call
  `SaveStorage` to write the memory storage to harddisk. This step is
  critical because GE is an in-memory system. **Shutting
  down the GE instance before `SaveStorage` finishes will
  drop all the in-memory data**.  In the embedded mode, we can
  directly call `Global.LocalStorage.SaveStorage`. In the distributed
  mode, there are two ways of calling `SaveStorage`. The first one is
  to call `Global.LocalStorage.SaveStorage` on each GE
  instance in the cluster. The second way is to call
  `Global.CloudStorage.SaveStorage` at client side.

### Common Data Import Patterns

There are two major data import patterns for loading data.

#### An Object Per Fetch

Suppose we have a single type of data in a file:

```
#File courses.txt
#======================================================
#course_id  name        total_score
1           Math        100
2           English     100
3           Geography   50
4           Biology     50
5           P.E.        30
...
```

In this file, each row represents one object and the fields are
separated with `\t`. In this case, we can model the data as follows:

```C#
cell Course
{
    /* For brevity, we regard course_id as the cell id. */
    string  Name;
    int     TotalScore;
}
```

We can load this data set to GE by reading the file
line-by-line. For each line, we call the extension method
`SaveCourse(long CellId, string Name, int TotalScore)` generated by
the TSL compiler. The sample code is given below:

```C#
using( var StreamReader reader = new StreamReader("courses.txt") )
{
    string      line;
    string[]    fields;
    while(null != (line = reader.ReadLine()))
    {
        try
        {
            fields = line.Split('\t');
            Global.LocalStorage.SaveCourse(
                long.Parse(fields[0]),
                fields[1],
                int.Parse(fields[2]));
        }catch
        {
            Console.Error.WriteLine("Failed to import the line:");
            Console.Error.WriteLine(line);
        }
    }
}
```

Similarly, if the data file consists of JSON objects one after
another:

```JSON
/* File student.json */
{"CellID" : 10000, "name" : "Alice",    "scores": [ { "cid" : 1, "score": 100 }, { "cid" : 3, "score" : 15  }]}
{"CellID" : 10001, "name" : "Bob",      "scores": [ { "cid" : 1, "score": 72 },  { "cid" : 2, "score" : 97  }]}
...
```

Then we can simply model the data 1:1 to the schema of the JSON objects:

```C#
struct ScoreRecord
{
    long cid; /* The course id */
    int score;
}

cell Student
{
    /* CellID is available for all cells */
    string name;
    /* Both List<T> and Array<T> are compatible with JSON arrays when parsing */
    List<ScoreRecord> scores;
}
```

And the code to import this file is (if not surprisingly) very simple:

```C#
using( var StreamReader reader = new StreamReader("students.json") )
{
    string      line;
    while(null != (line = reader.ReadLine()))
    {
        try
        {
            Global.LocalStorage.SaveStudent(Student.Parse(line));
        }catch
        {
            Console.Error.WriteLine("Failed to import the line:");
            Console.Error.WriteLine(line);
        }
    }
}
```

The `Student.Parse` method is generated automatically by the TSL
compiler. It leverages pre-compiled regex expressions for parsing JSON
strings of _Student_ JSON objects. This usually more efficient than
parsing the string with a generic purpose JSON parser.

No matter where the data is stored, as long as we can obtain one
object per fetch, the pattern holds. For example, importing data from
MongoDB to GE would be as easy as:

```C#
BsonClassMap.RegisterClassMap<ScoreRecord> ( cm => {
    cm.MapField("cid");
    cm.MapField("score");
});
BsonClassMap.RegisterClassMap<Student>( cm => {
    cm.MapIdField("CellID");
    cm.MapField("name");
    cm.MapField("scores");
});
/* Assuming data stored in a collection "students" */
foreach(var student in db.GetCollection<Student>("students"))
{
    Global.LocalStorage.SaveStudent(student);
}
```

#### Partial Object Per Fetch

It is subtler if we cannot fetch an object by one fetch operation.  We
have to join multiple lines to obtain an entry. This is such a data
set:

```
#File: professors.rdf
#Data given in triples <prof_id, property_name, property_value>
1000001    "teaches"    "Computer science"
1000000    "name"       "Jeff"
1000000    "comment"    "A distinguished computer scientist."
1000001    "hobby"      "Taichi"
1000000    "teaches"    "P.E."
1000001    "name"       "Alice"
1000001    "teaches"    "Math"
...
```

It requires some effort in modeling this kind of data. As shown in
the example, each line contains a single property of an object. And
multiple properties with the same name should be grouped into a
`List<T>`. Also, some properties may not be mandatory for all objects,
and we should mark them with the `optional` modifier.  To summarize it
up, we can model this data set as follows:

```C#
cell Professor
{
    /* Let CellID represent prof_id */
    List<string>    teaches;
    string          name;
    optional string hobby;
}
```

The simplest way to import data of this kind is to use accessors to
**upsert** objects. An _upsert_ operation updates an existing object
or insert a new object if no matched existing object found.
`Global.LocalStorage.UseProfessor(long cellId, CellAccessOptions
options)` has `upsert` semantics when
`CellAccessOptions.CreateNewOnCellNotFound` flag is turned on. The
following is a sample code:

```C#
using( var StreamReader reader = new StreamReader("professors.rdf") )
{
    string line;
    string[] cols;
    while(null != (line = reader.ReadLine()))
    {
        try
        {
            cols = line.Split(new[] {' ', '\t'}, StringSplitOptions.RemoveEmpty);
            using (var prof_accessor = Global.LocalStorage.UseProfessor(
                long.Parse(cols[0]),
                CellAccessOptions.CreateNewOnNotFound))
            {
                switch(cols[1])
                {
                    case "teaches":
                        prof_accessor.teaches.Add(cols[2]);
                    break;
                    case "name":
                        prof_accessor.name = cols[2];
                    break;
                    case "hobby":
                        prof_accessor.hobby = cols[2];
                    break;
                }
            }
        }catch
        {
            Console.Error.WriteLine("Failed to import the line:");
            Console.Error.WriteLine(line);
        }
    }
}
```

{% comment %}
As we can see, the import code now involves handling different fields with
different logic. To simplify things a bit, one could declare all fields of such
a cell type as `List<string>`.
{% endcomment %}

This approach, however, usually has performance issues.  Constantly
appending new fields one by one will cause a lot of cell resize
operations, which will pose great pressure on the GE memory
management system.  The best practice in this case is to sort the
data set so that properties of an object are grouped together. Then we
can commit a cell as a whole without fragmenting the memory
storage. After sorting, the data set looks like:

```
#File: entities.rdf
#Data sorted by keys.
1       name        Book
1       type        ObjectClassDefinition
1       property    title
1       property    author
1       property    price
2       title       "Secret Garden: An Inky Treasure Hunt and Coloring Book"
2       author      "Johanna Basford" 
2       price       "$10.24"
2       type        Book
3       type        Author
3       name        "Johanna Basford"
3       write_book  2
4       name        Author
4       property    write_book
4       property    name
4       type        ObjectClassDefinition
...
```

Now we can read a few consecutive lines to reconstruct a data
object. Moreover, we can observe that the data modeling schema is
self-contained: schema triples starting with 1 and 4 specify the data
schemata; while data triples starting with 2 and 3 are ordinary data
objects of the types defined by the schema triples. For each schema
object, we can add a corresponding cell type in the TSL file:

```C#
/* The definition for the data type definition objects */
cell ObjectClassDefinition
{
    string       name;
    List<string> property;
}

/* Constructed from object #1 */
cell Book
{
    string title;
    string author;
    string price;
}

/* Constructed from object #4 */
cell Author
{
    List<CellID>    write_book;
    string          name;
}
...
```

[Generic cells](/docs/manual/DataAccess/GenericCell.html) can be
leveraged to reduce the efforts of handling different types of data,
as long as the type of a cell can be determined directly from the
input data. The following code demonstrates the usage of generic cells
 with sorted data:

```C#
/* Assuming data sorted by cell id, but fields may come in shuffled. */
using( var StreamReader reader = new StreamReader("professors.rdf") )
{
    string          line;
    string[]        cols;
    string          cell_type_str   = "";
    long            cell_id         = long.MaxValue;
    List<string>    field_keys      = new List<string>();
    List<string>    field_values    = new List<string>();
    ICell           generic_cell;
    while(null != (line = reader.ReadLine()))
    {
        try
        {
            cols                = line.Split(new[] {' ', '\t'}, StringSplitOptions.RemoveEmpty);
            long line_cell_id   = long.Parse(cols[0]);

            if( line_cell_id != cell_id ) // We're done with the current one
            {
                if( field_key.Count != 0 ) // Commit the current cell if it
                                           // contains data
                {
                    /* We assume that cell_type_str exactly represents the
                     * type string of the cell. If the cell_type_str is invalid,
                     * an exception will be thrown and the current record will
                     * be dropped.
                     */
                    generic_cell = Global.LocalStorage.NewGenericCell(cell_id, cell_type_str);

                    /* We use AppendToField to save data to a GenericCell. The
                     * value will be converted to the compatible type at the
                     * best effort automatically. The semantics of AppendToField 
                     * is determined by the data type. If the target data is a
                     * list or a string, the supplied value will be appended to
                     * its tail. Otherwise, the target field will be
                     * overwritten. This makes it convenient for using a single
                     * interface for saving to both appendable data fields(lists
                     * and strings) and non-appendable data fields(numbers,
                     * objects, etc).
                     */
                    for( int i = 0, e = field_keys.Count; i != e; ++i )
                    {
                        generic_cell.AppendToField(field_keys[i], field_values[i]);
                    }

                    /* In this example, the local memory storage is accessed
                     * only once for each imported cell. It will have higher
                     * efficiency than the upsert method shown above.
                     */
                    Global.LocalStorage.SaveGenericCell(generic_cell);
                    
                    /* We're done with the current cell. Clear up the
                     * environment and move on to the next cell.
                     */
                    cell_type_str = "";     // Reset the cell type string.
                    field_keys.Clear();     // Reset the data field buffers.
                    field_values.Clear();
                    cell_id = line_cell_id; // Set the target cell id.
                }
            }

            /* Process the line now */
            if(cols[1] == "type") // We're reading the type string
            {
                cell_type_str = cols[2];
            }else                 // Otherwise, a data field.
            {
                field_keys.Add(cols[1]);
                field_values.Add(cols[2]);
            }

        }catch
        {
            Console.Error.WriteLine("Failed to import the line:");
            Console.Error.WriteLine(line);
        }
    }
}
```

### Data Import via RESTful APIs

Sometimes it is more convenient to processing certain data set in a
programming language other than `C#`. With the built-in support of
user-defined RESTful APIs, we can easily post data to GE
instances via http interfaces. Please refer to [Serving a Streaming
Tweet Graph](/docs/manual/DemoApps/TweetGraph.html) for an example.
