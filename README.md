# NAME

DBD::Avatica - Driver for Apache Avatica compatible servers

# VERSION

version 0.01.0

# SYNOPSIS

    use DBI;

    # for Apache Phoenix
    $dbh = DBI->connect("dbi:Avatica:adapter_name=phoenix;url=http://127.0.0.1:8765", '', '', {AutoCommit => 0}) or die $DBI::errstr;
    # The AutoCommit attribute should always be explicitly set

    $rows = $dbh->do('UPSERT INTO mytable(a) VALUES (1)') or die $dbh->errstr;

    $sth = $dbh->prepare('UPSERT INTO mytable(a) VALUES (?)') or die $dbh->errstr;
    $sth->execute(2) or die $sth->errstr;

# DESCRIPTION

DBD::Avatica is a Perl module that works with the DBI module to provide access to
databases with Apache Avatica compatible protocol.

# MODULE DOCUMENTATION

This documentation describes driver specific behavior and restrictions. It is
not supposed to be used as the only reference for the user. In any case
consult the **DBI** documentation first!

[Latest DBI documentation.](https://metacpan.org/pod/DBI)

# THE DBI CLASS

## DBI Class Methods

### **connect**

This method creates a database handle by connecting to a database, and is the DBI
equivalent of the "new" method. To connect to a database with a minimum of parameters,
use the following syntax:

    $dbh = DBI->connect("dbi:Avatica:adapter_name=phoenix;url=http://127.0.0.1:8765", '', '', {AutoCommit => 0});

This connects to the database accessed by url  `http://127.0.0.1:8765` without any user authentication.

The following connect statement shows almost all possible parameters:

    $dbh = DBI->connect("dbi:Avatica:adapter_name=phoenix;url=$url;",
                        $username,
                        $password,
                        {
                          AutoCommit => 0,
                          RaiseError => 1,
                          PrintError => 0,
                          TransactionIsolation => 2
                          UserAgent => HTTP::Tiny->new(),
                          MaxRetries => 2
                        }
                       );

Currently username/password not supported. To set username/password for authentication need to redefine UserAgent attribute.

Your UserAgent must implement `POST` request with HTTP::Tiny semantics.
So, after some timeouts, network errors UserAgent must return http status 599 and do not retries after failure.

Attrubute MaxRetries specifies the number of attempts to send request (in case HTTP 500 response).

### **connect\_cached**

    $dbh = DBI->connect_cached("dbi:Avatica:adapter_name=phoenix;url=$url", $username, $password, \%options);

Implemented by DBI, no driver-specific impact.

## Methods Common To All Handles

For all of the methods below, **$h** can be either a database handle (**$dbh**)
or a statement handle (**$sth**). Note that _$dbh_ and _$sth_ can be replaced with
any variable name you choose: these are just the names most often used. Another
common variable used in this documentation is $_rv_, which stands for "return value".

### **err**

    $rv = $h->err;

Returns the error code from the last method called. If the error is generated by the perl module
it will always be 1. If an error was received from the server, then the value will be
Avatica::Client::Protocol::ErrorResponse: error\_code, which corresponds to an error in the protobuf protocol.

### **errstr**

    $str = $h->errstr;

Returns the error message from the last method called.

### **state**

    $str = $h->state;

Returns a five-character "SQLSTATE" code. Success is indicated by a `00000` code, which
gets mapped to an empty string by DBI.

### **trace**

    $h->trace(1);
    $h->trace(1, $trace_filename);
    $trace_settings = $h->trace;

Changes the trace settings on a database or statement handle.
The optional second argument specifies a file to write the
trace information to. If no filename is given, the information
is written to `STDERR`. Note that tracing can be set globally as
well by setting `DBI->trace`, or by using the environment
variable _DBI\_TRACE_.

### **trace\_msg**

    $h->trace_msg($message_text);
    $h->trace_msg($message_text, $min_level);

Writes a message to the current trace output (as set by the ["trace"](#trace) method). If a second argument
is given, the message is only written if the current tracing level is equal to or greater than
the `$min_level`.

### **parse\_trace\_flag** and **parse\_trace\_flags**

    $h->trace($h->parse_trace_flags('ALL')); # not recommended
    $h->trace($h->parse_trace_flags(1));

    ## Simpler:
    $h->trace('ALL'); # not recommended
    $h->trace('1');

The parse\_trace\_flags method is used to convert one or more named
flags to a number which can passed to the ["trace"](#trace) method.

Implemented by DBI, no driver-specific impact.
See the [DBI section on TRACING](https://metacpan.org/pod/DBI#TRACING) for more information.

# ATTRIBUTES COMMON TO ALL HANDLES

### **InactiveDestroy** (boolean)

If set to true, then the ["disconnect"](#disconnect) and statement close methods will not be automatically
called when the database handle goes out of scope. This is required if you are forking,
and even then you must tread carefully and ensure that either the parent or the child (but not
both!) handles all database calls from that point forwards, so that messages from the
Postgres backend are only handled by one of the processes. The best solution is to either
have the child process reconnect to the database with a fresh database handle, or to
rewrite your application not to use forking.

### **AutoInactiveDestroy** (boolean)

The InactiveDestroy attribute, described above, needs to be explicitly set in the child
process after a fork. If the code that performs the fork is in a third party module such
as Sys::Syslog, this can present a problem. Use AutoInactiveDestroy to get around this
problem.
DBD::Avatica has additional security, but after the fork you still need to create a new
connection in the child process.

### **RaiseError** (boolean, inherited)

Forces errors to always raise an exception. Although it defaults to off, it is recommended that this
be turned on, as the alternative is to check the return value of every method (prepare, execute, fetch, etc.)
manually, which is easy to forget to do.

### **PrintError** (boolean, inherited)

Forces database errors to also generate warnings, which can then be filtered with methods such as
locally redefining _$SIG{\_\_WARN\_\_}_ or using modules such as `CGI::Carp`. This attribute is on
by default.

### **ShowErrorStatement** (boolean, inherited)

Appends information about the current statement to error messages. If placeholder information
is available, adds that as well. Defaults to false.

Note that this will not work when using ["do"](#do) without any arguments.

### **Warn** (boolean, inherited)

Enables warnings. This is on by default, and should only be turned off in a local block
for a short a time only when absolutely needed.

### **Executed** (boolean, read-only)

Indicates if a handle has been executed. For database handles, this value is true after the ["do"](#do) method has been called, or
when one of the child statement handles has issued an ["execute"](#execute). Issuing a ["commit"](#commit) or ["rollback"](#rollback) always resets the
attribute to false for database handles. For statement handles, any call to ["execute"](#execute) or its variants will flip the value to
true for the lifetime of the statement handle.

### **TraceLevel** (integer, inherited)

Sets the trace level, similar to the ["trace"](#trace) method. See the sections on
["trace"](#trace) and [parse\_trace\_flag](#parse_trace_flag-and-parse_trace_flags) for more details.

### **Active** (boolean, read-only)

Indicates if a handle is active or not. For database handles, this indicates if the database has
been disconnected or not. For statement handles, it indicates if all the data has been fetched yet
or not. Use of this attribute is not encouraged.

### **Kids** (integer, read-only)

Returns the number of child processes created for each handle type. For a driver handle, indicates the number
of database handles created. For a database handle, indicates the number of statement handles created. For
statement handles, it always returns zero, because statement handles do not create kids.

### **ActiveKids** (integer, read-only)

Same as `Kids`, but only returns those that are active.

### **CachedKids** (hash ref)

Returns a hashref of handles. If called on a database handle, returns all statement handles created by use of the
`prepare_cached` method. If called on a driver handle, returns all database handles created by the ["connect\_cached"](#connect_cached)
method.

### **ChildHandles** (array ref)

Implemented by DBI, no driver-specific impact.
See the [DBI ChildHandles](https://metacpan.org/pod/DBI#ChildHandles) for more information.

### **PrintWarn** (boolean, inherited)

Implemented by DBI, no driver-specific impact.
See the [DBI PrintWarn](https://metacpan.org/pod/DBI#PrintWarn) for more information.

### **HandleError** (boolean, inherited)

Implemented by DBI, no driver-specific impact.
See the [DBI HandleError](https://metacpan.org/pod/DBI#HandleError) for more information.

### **HandleSetErr** (code ref, inherited)

Implemented by DBI, no driver-specific impact.
See the [DBI HandleSetErr](https://metacpan.org/pod/DBI#HandleSetErr) for more information.

### **ErrCount** (unsigned integer)

Implemented by DBI, no driver-specific impact.
See the [DBI ErrCount](https://metacpan.org/pod/DBI#ErrCount) for more information.

### **FetchHashKeyName** (string, inherited)

Implemented by DBI, no driver-specific impact.
See the [DBI FetchHashKeyName](https://metacpan.org/pod/DBI#FetchHashKeyName) for more information.

### **Taint** (boolean, inherited)

Implemented by DBI, no driver-specific impact.
See the [DBI Taint](https://metacpan.org/pod/DBI#Taint) for more information.

### **TaintIn** (boolean, inherited)

Implemented by DBI, no driver-specific impact.
See the [DBI TaintIn](https://metacpan.org/pod/DBI#TaintIn) for more information.

### **TaintOut** (boolean, inherited)

Implemented by DBI, no driver-specific impact.
See the [DBI TaintOut](https://metacpan.org/pod/DBI#TaintOut) for more information.

### **Profile** (inherited)

Implemented by DBI, no driver-specific impact.
See the [DBI Profile](https://metacpan.org/pod/DBI#Profile) for more information.

### **Type** (scalar)

Returns `dr` for a driver handle, `db` for a database handle, and `st` for a statement handle.

# DBI DATABASE HANDLE OBJECTS

## Database Handle Methods

### **selectall\_arrayref**

    $ary_ref = $dbh->selectall_arrayref($sql);
    $ary_ref = $dbh->selectall_arrayref($sql, \%attr);
    $ary_ref = $dbh->selectall_arrayref($sql, \%attr, @bind_values);

Returns a reference to an array containing the rows returned by preparing and executing the SQL string.
See the [DBI selectall\_arrayref](https://metacpan.org/pod/DBI#selectall_arrayref) for more information.

### **selectall\_hashref**

    $hash_ref = $dbh->selectall_hashref($sql, $key_field);

Returns a reference to a hash containing the rows returned by preparing and executing the SQL string.
See the [DBI selectall\_hashref](https://metacpan.org/pod/DBI#selectall_hashref) for more information.

### **selectcol\_arrayref**

    $ary_ref = $dbh->selectcol_arrayref($sql, \%attr, @bind_values);

Returns a reference to an array containing the first column
from each rows returned by preparing and executing the SQL string. It is possible to specify exactly
which columns to return.
See the [DBI selectcol\_arrayref](https://metacpan.org/pod/DBI#selectcol_arrayref) for more information.

### **prepare**

    $sth = $dbh->prepare($statement, \%attr);

The prepare method prepares a statement for later execution.

You cannot send more than one command at a time in the same prepare command
(by separating them with semi-colons) when using server-side prepares.

The actual `PREPARE` is usually not performed until the first execute is called, due
to the fact that information on the data types (provided by ["bind\_param"](#bind_param)) may
be provided after the prepare but before the execute.

#### **Placeholders**

Driver support placeholders and bind values. Placeholders, also called parameter markers,
are used to indicate values in a database statement that will be supplied later, before
the prepared statement is executed. For example, an application might use the following to
insert a row of data into the SALES table:

    INSERT INTO sales (product_code, qty, price) VALUES (?, ?, ?)

or the following, to select the description for a product:

    SELECT description FROM products WHERE product_code = ?

The `?` characters are the placeholders. The association of actual values with placeholders
is known as binding, and the values are referred to as bind values. Note that the `?` is not
enclosed in quotation marks, even when the placeholder represents a string.

In the final statement above, DBI thinks there is only one placeholder, so this
statement will replace placeholder:

    $sth->bind_param(1, 2045);
    $sth->execute;

While a simple execute with no `bind_param` calls requires only a single argument as well:

    $sth->execute(2045);

See the  documentation for more details.
See the [DBI Placeholders and Bind Values](https://metacpan.org/pod/DBI#Placeholders-and-Bind-Values) for more information.

### **prepare\_cached**

    $sth = $dbh->prepare_cached($statement, \%attr);

Implemented by DBI, no driver-specific impact.
See the [DBI prepare\_cached](https://metacpan.org/pod/DBI#prepare_cached) for more information.

### **do**

    $rv = $dbh->do($statement);
    $rv = $dbh->do($statement, \%attr);
    $rv = $dbh->do($statement, \%attr, @bind_values);

Prepare and execute a single statement. Returns the number of rows affected if the
query was successful, returns undef if an error occurred, and returns -1 if the
number of rows is unknown or not available. Note that this method will return **0E0** instead
of 0 for 'no rows were affected', in order to always return a true value if no error occurred.

### **last\_insert\_id**

    $rv = $dbh->last_insert_id(undef, $schema, $table, undef);
    $rv = $dbh->last_insert_id(undef, $schema, $table, undef, {sequence => $seqname});

Attempts to return the id of the last value of sequence to be inserted into a table.
You can either provide a sequence name (preferred) or provide a table
name with optional schema, and DBD::Avatica will attempt to find the sequence itself.

If you do not know the name of the sequence, you can provide a table name and
DBD::Avatica will attempt to return the correct value. To do this, there must be at
least one column in the table which uses a sequence, sequence schema name must be similar
schema table name and sequnce prefix name must contain table name.

Example:

    $dbh->do('CREATE SEQUENCE test_seq;');
    $dbh->do('CREATE TABLE test(id bigint primary key, x varchar)');
    $sth = $dbh->prepare('UPSERT INTO test VALUES (NEXT VALUE FOR test_seq, ?)');
    for (qw(foo bar baz)) {
      $sth->execute($_);
      my $newid = $dbh->last_insert_id(undef, undef, undef, undef, {sequence=>'test_seq'});
      print "Last insert id was $newid\n";
    }

At the moment this method/feature is not implemented.

### **commit**

    $rv = $dbh->commit;

Issues a COMMIT to the server, indicating that the current transaction is finished and that
all changes made will be visible to other processes. If AutoCommit is enabled, then
a warning is given and no COMMIT is issued. Returns true on success, false on error.
See also the section on ["Transactions"](#transactions).

### **rollback**

    $rv = $dbh->rollback;

Issues a ROLLBACK to the server, which discards any changes made in the current transaction. If AutoCommit
is enabled, then a warning is given and no ROLLBACK is issued. Returns true on success, and
false on error. See also the the section on ["Transactions"](#transactions).

### **begin\_work**

This method turns on transactions until the next call to ["commit"](#commit) or ["rollback"](#rollback), if [AutoCommit](#autocommit-boolean) is
currently enabled. If it is not enabled, calling `begin_work` will do nothing. Note that the
transaction will not actually begin until the first statement after `begin_work` is called.
Example:

    $dbh->{AutoCommit} = 1;
    $dbh->do('UPSERT INTO foo VALUES (123)'); ## Changes committed immediately
    $dbh->begin_work;
    ## Not in a transaction yet, but AutoCommit is set to 0
    $dbh->do('INSERT INTO foo VALUES (345)');
    $dbh->commit;
    ## AutoCommit is now set to 1 again

### **disconnect**

    $rv = $dbh->disconnect;

Disconnects from the Postgres database.

If the script exits before disconnect is called (or, more precisely, if the database handle is no longer
referenced by anything), then the database handle's DESTROY method will call the disconnect()
methods automatically. It is best to explicitly disconnect rather than rely on this behavior.

### **quote**

    $rv = $dbh->quote($value);
    $rv = $dbh->quote($value, $data_type);

Quote a string literal for use as a literal value in an SQL statement, by escaping any special characters
(such as quotation marks) contained within the string and adding the required type of outer quotation marks.

Implemented by DBI, no driver-specific impact.
See the [DBI quote](https://metacpan.org/pod/DBI#quote) for more information.

### **quote\_identifier**

    $string = $dbh->quote_identifier( $name );
    $string = $dbh->quote_identifier( undef, $schema, $table);

Returns a quoted version of the supplied string, which is commonly a schema,
table, or column name.

Implemented by DBI, no driver-specific impact.
See the [DBI quote\_identifier](https://metacpan.org/pod/DBI#quote_identifier) for more information.

### **get\_info**

    $value = $dbh->get_info($info_type);

Supports a small set of the information types which is reported by the server in the request `database_property`:

- SQL\_DRIVER\_NAME
- SQL\_DRIVER\_VER
- SQL\_SEARCH\_PATTERN\_ESCAPE
- SQL\_DBMS\_NAME
- SQL\_DBMS\_VERSION
- SQL\_CATALOG\_LOCATION
- SQL\_CATALOG\_NAME\_SEPARATOR
- SQL\_IDENTIFIER\_CASE
- SQL\_IDENTIFIER\_QUOTE\_CHAR
- SQL\_KEYWORDS

### **table\_info**

    $sth = $dbh->table_info($catalog, $schema, $table, $type);

Returns all tables and views visible to the current user.

See the [DBI table\_info](https://metacpan.org/pod/DBI#table_info) for more information.

### **column\_info**

    $sth = $dbh->column_info( $catalog, $schema, $table, $column );

Supported by this driver as proposed by DBI.

See the [DBI column\_info](https://metacpan.org/pod/DBI#column_info) for more information.

### **primary\_key\_info**

    $sth = $dbh->primary_key_info( $catalog, $schema, $table, \%attr );

Supported by this driver as proposed by DBI.

See the [DBI primary\_key\_info](https://metacpan.org/pod/DBI#primary_key_info) for more information.

### **primary\_key**

    @key_column_names = $dbh->primary_key($catalog, $schema, $table);

Simple interface to the ["primary\_key\_info"](#primary_key_info) method. Returns a list of the column names that
comprise the primary key of the specified table. The list is in primary key column sequence
order. If there is no primary key then an empty list is returned.

### **tables**

    @names = $dbh->tables( undef, $schema, $table, $type, \%attr );

Supported by this driver as proposed by DBI.

### **type\_info\_all**

    $type_info_all = $dbh->type_info_all;

At the moment not supported by DBD::Avatica.

### **type\_info**

    @type_info = $dbh->type_info($data_type);

At the moment not supported by DBD::Avatica.

### **selectrow\_array**

    @row_ary = $dbh->selectrow_array($sql);
    @row_ary = $dbh->selectrow_array($sql, \%attr);
    @row_ary = $dbh->selectrow_array($sql, \%attr, @bind_values);

Returns an array of row information after preparing and executing the provided SQL string. The rows are returned
by calling ["fetchrow\_array"](#fetchrow_array). The string can also be a statement handle generated by a previous prepare. Note that
only the first row of data is returned. If called in a scalar context, only the first column of the first row is
returned. Because this is not portable, it is not recommended that you use this method in that way.

### **selectrow\_arrayref**

    $ary_ref = $dbh->selectrow_arrayref($statement);
    $ary_ref = $dbh->selectrow_arrayref($statement, \%attr);
    $ary_ref = $dbh->selectrow_arrayref($statement, \%attr, @bind_values);

Exactly the same as ["selectrow\_array"](#selectrow_array), except that it returns a reference to an array, by internal use of
the ["fetchrow\_arrayref"](#fetchrow_arrayref) method.

### **selectrow\_hashref**

    $hash_ref = $dbh->selectrow_hashref($sql);
    $hash_ref = $dbh->selectrow_hashref($sql, \%attr);
    $hash_ref = $dbh->selectrow_hashref($sql, \%attr, @bind_values);

Exactly the same as ["selectrow\_array"](#selectrow_array), except that it returns a reference to an hash, by internal use of
the ["fetchrow\_hashref"](#fetchrow_hashref) method.

### **clone**

    $other_dbh = $dbh->clone();

Creates a copy of the database handle by connecting with the same parameters as the original
handle, then trying to merge the attributes.
See the [DBI clone](https://metacpan.org/pod/DBI#clone) for more information.

## Database Handle Attributes

### **AutoCommit** (boolean)

Supported by DBD::Avatica as proposed by DBI. Without starting a transaction,
every change to the database becomes immediately permanent. The default of
AutoCommit is on, but this may change in the future, so it is highly recommended
that you explicitly set it when calling ["connect"](#connect).
For details see the notes about ["Transactions"](#transactions) elsewhere in this document.

### **ReadOnly** (boolean)

    $dbh->{ReadOnly} = 1;

Specifies if the current database connection should be in read-only mode or not.

### **TransactionIsolation** (integer)

    $dbh->{TransactionIsolation} = 2;

Specifies the transaction isolation level.

- '0'

    Transactions are not supported

- '1'

    READ UNCOMMITED. Dirty reads, non-repeatable reads and phantom reads may occur.

- '2'

    READ COMMITED. Dirty reads are prevented, but non-repeatable reads and phantom reads may occur.

- '4'

    REPEATABLE READ. Dirty reads and non-repeatable reads are prevented, but phantom reads may occur.

- '8'

    SERIALIZABLE. Dirty reads, non-repeatable reads, and phantom reads are all prevented.

### **AVATICA\_DRIVER\_NAME** (string, read-only)

Return avatica driver name.

### **AVATICA\_DRIVER\_VERSION** (string, read-only)

Return avatica driver version.

### **Name** (string, read-only)

Returns the name of the database and the URL to which it is connected.
This is the same as the DSN, without the "dbi:Avatica:" part.

# DBI STATEMENT HANDLE OBJECTS

## Statement Handle Methods

### **bind\_param**

    $rv = $sth->bind_param($param_num, $bind_value);
    $rv = $sth->bind_param($param_num, $bind_value, $bind_type);
    $rv = $sth->bind_param($param_num, $bind_value, \%attr);

Allows the user to bind a value and/or a data type to a placeholder. This is
especially important when using server-side prepares. See the
["prepare"](#prepare) method for more information.

The value of `$param_num` is a number of '?' placeholders (starting from 1).

The `$bind_value` argument is fairly self-explanatory.

The `$bind_type` and `%attr` is not supported currently.

### **bind\_param\_array**

    $rv = $sth->bind_param_array($param_num, $array_ref_or_value)
    $rv = $sth->bind_param_array($param_num, $array_ref_or_value, $bind_type)
    $rv = $sth->bind_param_array($param_num, $array_ref_or_value, \%attr)

Binds an array of values to a placeholder, so that each is used in turn by a call
to the ["execute\_array"](#execute_array) method.

### **execute**

    $rv = $sth->execute(@bind_values);

or

    $sth->bind_param(1, $value1);
    $sth->bind_param(2, $value2);
    $rv = $sth->execute;

Executes a previously prepared statement. In addition to `UPSERT`, `DELETE` for which
it returns always the number of affected rows.

### **execute\_array**

    $tuples = $sth->execute_array() or die $sth->errstr;
    $tuples = $sth->execute_array(\%attr) or die $sth->errstr;
    $tuples = $sth->execute_array(\%attr, @bind_values) or die $sth->errstr;

    ($tuples, $rows) = $sth->execute_array(\%attr) or die $sth->errstr;
    ($tuples, $rows) = $sth->execute_array(\%attr, @bind_values) or die $sth->errstr;

Execute a prepared statement once for each item in a passed-in hashref, or items that
were previously bound via the ["bind\_param\_array"](#bind_param_array) method. See the DBI documentation
for more details.

### **execute\_for\_fetch**

    $tuples = $sth->execute_for_fetch($fetch_tuple_sub);
    $tuples = $sth->execute_for_fetch($fetch_tuple_sub, \@tuple_status);

    ($tuples, $rows) = $sth->execute_for_fetch($fetch_tuple_sub);
    ($tuples, $rows) = $sth->execute_for_fetch($fetch_tuple_sub, \@tuple_status);

Used internally by the ["execute\_array"](#execute_array) method, and rarely used directly. See the
DBI documentation for more details.

### **fetchrow\_arrayref**

    $ary_ref = $sth->fetchrow_arrayref;

Fetches the next row of data from the statement handle, and returns a reference to an array
holding the column values. Any columns that are NULL are returned as undef within the array.

If there are no more rows or if an error occurs, then this method return undef. You should
check `$sth->err` afterwards (or use the [RaiseError](#raiseerror-boolean-inherited) attribute) to discover if the undef returned
was due to an error.

Note that the same array reference is returned for each fetch, so don't store the reference and
then use it after a later fetch. Also, the elements of the array are also reused for each row,
so take care if you want to take a reference to an element. See also ["bind\_columns"](#bind_columns).

### **fetchrow\_array**

    @ary = $sth->fetchrow_array;

Similar to the ["fetchrow\_arrayref"](#fetchrow_arrayref) method, but returns a list of column information rather than
a reference to a list. Do not use this in a scalar context.

### **fetchrow\_hashref**

    $hash_ref = $sth->fetchrow_hashref;
    $hash_ref = $sth->fetchrow_hashref($name);

Fetches the next row of data and returns a hashref containing the name of the columns as the keys
and the data itself as the values. Any NULL value is returned as an undef value.

If there are no more rows or if an error occurs, then this method return undef. You should
check `$sth->err` afterwards (or use the [RaiseError](#raiseerror-boolean-inherited) attribute) to discover if the undef returned
was due to an error.

The optional `$name` argument should be either `NAME`, `NAME_lc` or `NAME_uc`, and indicates
what sort of transformation to make to the keys in the hash.

### **fetchall\_arrayref**

    $tbl_ary_ref = $sth->fetchall_arrayref();
    $tbl_ary_ref = $sth->fetchall_arrayref( $slice );
    $tbl_ary_ref = $sth->fetchall_arrayref( $slice, $max_rows );

Returns a reference to an array of arrays that contains all the remaining rows to be fetched from the
statement handle. If there are no more rows, an empty arrayref will be returned. If an error occurs,
the data read in so far will be returned. Because of this, you should always check `$sth->err` after
calling this method, unless [RaiseError](#raiseerror-boolean-inherited) has been enabled.

If `$slice` is an array reference, fetchall\_arrayref uses the ["fetchrow\_arrayref"](#fetchrow_arrayref) method to fetch each
row as an array ref. If the `$slice` array is not empty then it is used as a slice to select individual
columns by perl array index number (starting at 0, unlike column and parameter numbers which start at 1).

With no parameters, or if $slice is undefined, fetchall\_arrayref acts as if passed an empty array ref.

If `$slice` is a hash reference, fetchall\_arrayref uses ["fetchrow\_hashref"](#fetchrow_hashref) to fetch each row as a hash reference.

See the [DBI fetchall\_arrayref](https://metacpan.org/pod/DBI#fetchall_arrayref) for more information.

### **fetchall\_hashref**

    $hash_ref = $sth->fetchall_hashref( $key_field );

Returns a hashref containing all rows to be fetched from the statement handle.

See the [DBI fetchall\_hashref](https://metacpan.org/pod/DBI#fetchall_hashref) for more information.

### **finish**

    $rv = $sth->finish;

Indicates to DBI that you are finished with the statement handle and are not going to use it again. Only needed
when you have not fetched all the possible rows.

### **rows**

    $rv = $sth->rows;

Returns the number of rows returned by the last query. Note that the ["execute"](#execute) method itself
returns the number of rows itself, which means that this method is rarely needed.

### **bind\_col**

    $rv = $sth->bind_col($column_number, \$var_to_bind);
    $rv = $sth->bind_col($column_number, \$var_to_bind, \%attr );
    $rv = $sth->bind_col($column_number, \$var_to_bind, $bind_type );

Binds a Perl variable and/or some attributes to an output column of a SELECT statement.
Column numbers count up from 1. You do not need to bind output columns in order to fetch data.

See the [DBI bind\_col](https://metacpan.org/pod/DBI#bind_col) for a discussion of the optional parameters `\%attr` and `$bind_type`.

### **bind\_columns**

    $rv = $sth->bind_columns(@list_of_refs_to_vars_to_bind);

Calls the ["bind\_col"](#bind_col) method for each column in the SELECT statement, using the supplied list.

See the [DBI bind\_columns](https://metacpan.org/pod/DBI#bind_columns) for more information.

### **dump\_results**

    $rows = $sth->dump_results($maxlen, $lsep, $fsep, $fh);

Fetches all the rows from the statement handle, calls `DBI::neat_list` for each row, and
prints the results to `$fh` (which defaults to `STDOUT`). Rows are separated by `$lsep` (which defaults
to a newline). Columns are separated by `$fsep` (which defaults to a comma). The `$maxlen` controls
how wide the output can be, and defaults to 35.

This method is designed as a handy utility for prototyping and testing queries. Since it uses
"neat\_list" to format and edit the string for reading by humans, it is not recommended
for data transfer applications.

### **last\_insert\_id**

    $rv = $sth->last_insert_id(undef, $schema, $table, undef);
    $rv = $sth->last_insert_id(undef, $schema, $table, undef, {sequence => $seqname});

This is simply an alternative way to return the same information as
`$dbh->last_insert_id`.

At the moment not supported.

## Statement Handle Attributes

### **NUM\_OF\_FIELDS** (integer, read-only)

Returns the number of columns returned by the current statement. A number will only be returned for
SELECT statements.

### **NUM\_OF\_PARAMS** (integer, read-only)

Returns the number of placeholders in the current statement.

### **NAME** (arrayref, read-only)

Returns an arrayref of column names for the current statement. This
attribute will only work for SELECT statements.

### **NAME\_lc** (arrayref, read-only)

The same as the `NAME` attribute, except that all column names are forced to lower case.

### **NAME\_uc**  (arrayref, read-only)

The same as the `NAME` attribute, except that all column names are forced to upper case.

### **NAME\_hash** (hashref, read-only)

Similar to the `NAME` attribute, but returns a hashref of column names instead of an arrayref. The names of the columns
are the keys of the hash, and the values represent the order in which the columns are returned, starting at 0.

### **NAME\_lc\_hash** (hashref, read-only)

The same as the `NAME_hash` attribute, except that all column names are forced to lower case.

### **NAME\_uc\_hash** (hashref, read-only)

The same as the `NAME_hash` attribute, except that all column names are forced to lower case.

### **TYPE** (arrayref, read-only)

Returns an arrayref indicating the data type for each column in the statement.

### **PRECISION** (arrayref, read-only)

Returns an arrayref of integer values for each column returned by the statement.
The number indicates the precision for `NUMERIC` columns, the expected average size in number
for all other types.

### **SCALE** (arrayref, read-only)

Returns an arrayref of integer values for each column returned by the statement. The number
indicates the scale of the that column. The only type that will return a value is `NUMERIC`.

### **NULLABLE** (arrayref, read-only)

Returns an arrayref of integer values for each column returned by the statement. The number
indicates if the column is nullable or not. 0 = not nullable, 1 = nullable, 2 = unknown.

### **Database** (dbh, read-only)

Returns the database handle this statement handle was created from.

### **ParamValues** (hash ref, read-only)

Returns a reference to a hash containing the values currently bound to placeholders.
Starting at one and increasing for every placeholder.

### **Statement** (string, read-only)

Returns the statement string passed to the most recent "prepare" method called in this database handle, even if that method
failed. This is especially useful where "RaiseError" is enabled and the exception handler checks $@ and sees that a `prepare`
method call failed.

# FURTHER INFORMATION

## Transactions

Transaction behavior is controlled via the ["AutoCommit"](#autocommit) attribute. For a
complete definition of `AutoCommit` please refer to the DBI documentation.

According to the DBI specification the default for `AutoCommit` is a true
value.

# AUTHOR

Alexey Stavrov &lt;logioniz@ya.ru>

# CONTRIBUTOR

Denis Ibaev &lt;dionys@gmail.com>

# COPYRIGHT AND LICENSE

This software is Copyright (c) 2021 by Alexey Stavrov.

This is free software, licensed under:

    The MIT (X11) License
