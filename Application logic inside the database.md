# Application logic inside the database. Templating. Rendering. Newsletter. Examples.

How did I even come up with this idea - to place application logic inside a database? While many people do not like to put even business logic into the database!
I worked with billing for many years and therefore by business logic inside a database I mean all sorts of calculations, restrictions, foreign keys, stored procedures, functions, triggers, etc.
and under applied logic all sorts of actions: template a document, render it in pdf, send the pdf by email, send an SMS about the need to top up the balance, turn on/off the phone at the telephone exchange or the Internet on Cisco/Mikrotik, etc.
Of course, all these actions can be done outside the database, and that’s what everyone usually does. But then there is a need for an infrastructure for all this, how to run it all, parallelize it, monitor it, maintain it, etc.
And I just have an asynchronous task scheduler [pg_task](https://github.com/RekGRpth/pg_task), in which all this is already implemented, i.e. automatic launch, parallelization, as monitoring, everything is written back to the table in the database, even transactionality is present!
In addition, if you already have a database and scheduler, then you don’t need to install any additional infrastructure and managing just the database is much easier.
That’s how I came up with the idea of placing not only business logic, but also application logic inside the database.
To do this, I had to write several plugins for Postgres. Most of them are simple and operate on the principle: a function from an extension is called, something is passed to it as arguments, and it returns the result.
While this function is running, the request seems to freeze, because the Postgres database is not asynchronous. Therefore, there is also no point in using applied logic implemented this way inside the database from external clients to the database.
It is much easier to implement this logic on the database client. It is for this purpose, in order to avoid pointlessly using database clients, that an asynchronous task scheduler is used, which executes application logic independently of database clients.
This is how templating and rendering work.

So,

## Templating.

When I started, I came across a template engine [ctpp2](https://github.com/Azq2/ctpp2), written in C++ and implemented as a plugin [ngx_ctpp2](https://github.com/RekGRpth/ngx_ctpp2) for nginx, with an interesting feature - to use its templates it was necessary to compile.
The compiler was a console program that converted a template text file into a binary one. My templates, like the documents themselves, were stored inside the database and therefore I didn’t really want to call an external program inside the database, although this can be done using the [plsh](https://github.com/petere/plsh) extension.
I can, of course, store already compiled templates inside the database, but this is also a so-so idea.
So I started looking for another template engine with an implementation in C and found [mustache](https://mustache.github.io), which has implementations in almost all languages! The C implementation is called [mustach](https://gitlab.com/jobol/mustach).
Also, I found [handlebars.js](https://github.com/handlebars-lang/handlebars.js) - a superset of mustache, with implementation in C [handlebars.c](https://github.com/jbboehr/handlebars .c) and many other languages.
I installed both of these template engines as plugins in Postgres [pg_mustach](https://github.com/RekGRpth/pg_mustach), [pg_handlebars](https://github.com/RekGRpth/pg_handlebars)
and nginx [ngx_http_mustach_module](https://github.com/RekGRpth/ngx_http_mustach_module), [ngx_http_handlebars_module](https://github.com/RekGRpth/ngx_http_handlebars_module),
and also in Python [pymustach](https://github.com/RekGRpth/pymustach), [pyhandlebars](https://github.com/RekGRpth/pyhandlebars) to compare performance with native Python implementations (the C version of mustache turned out to be order is faster than Python's).
As mentioned above, the interface for templating is very simple. A function is called, to which a document in the form of json and a template in the form of text are passed as arguments, the function templates and returns the result in the form of text.
For example, the following request
```sql
SELECT mustach('{"a":"b"}', '{{a}}');
```
will return
```sql
b
```
More complex example
```sql
SELECT mustach('{"people":[
    {"firstName":"Yehuda","lastName":"Katz"},
    {"firstName":"Carl","lastName":"Lerche"},
    {"firstName":"Alan","lastName":"Johnson"}
]}', '<ul>{{#people}}<li>{{firstName}} {{lastName}}</li>{{/people}}</ul>');
```
will return
```sql
<ul><li>Yehuda Katz</li><li>Carl Lerche</li><li>Alan Johnson</li></ul>
```
The interface of handlebars is absolutely identical.
Among the interesting features of the mustach implementation, I can note that the execution result is transmitted through a file in RAM, which is previously created by the function
```c
file = open_memstream(&data, &len)
```
Next, this file is transferred to the library [mustach](https://gitlab.com/jobol/mustach), which writes the result of templating into it. Then it’s quite easy to calculate the result using a pointer to the data and its length.
The [mustach](https://gitlab.com/jobol/mustach) library can work with different json parsers; it already implements [cJSON](https://github.com/DaveGamble/cJSON), [jansson]( https://github.com/akheron/jansson) and [json_c](https://github.com/json-c/json-c).
For Postgres, it would be logical to implement a parser based on json(b), but I didn’t bother so much and used ready-made implementations.
Templates by themselves are of little use in billing, so I also needed

## Rendering.

Here by rendering I mean converting an HTML file to a PDF file. To do this, I found several libraries and installed them all as plugins in Postgres and Nginx, as well as a couple in Python.
So, the first I found was [wkhtmltopdf](https://github.com/wkhtmltopdf/wkhtmltopdf) - a rather heavy library written in C++ with binding in C. [pg_wkhtmltopdf](https://github.com/RekGRpth/pg_wkhtmltopdf) and [ngx_http_wkhtmltopdf_module](https://github.com/RekGRpth/ngx_http_wkhtmltopdf_module).
Not only does it take up a lot of space, but it also actively uses threads and is not very willing to release/complete them, which is especially bad when working in nginx. But we must pay tribute, her API is convenient.
So I started searching further and found wthtmltopdf, or rather, there is no such library, but I tore it from the [wt](https://www.webtoolkit.eu/wt) framework, which had PDF rendering in C++, and I only added the binding in C, similar to wkhtmltopdf.
[pg_wthtmltopdf](https://github.com/RekGRpth/pg_wthtmltopdf) and [ngx_http_wthtmltopdf_module](https://github.com/RekGRpth/ngx_http_wthtmltopdf_module). But I also didn’t really like it, because it requires libs for the C++.
And I found a pure C library [mupdf](https://github.com/RekGRpth/mupdf). [pg_mupdf](https://github.com/RekGRpth/pg_mupdf), [ngx_http_mupdf_module](https://github.com/RekGRpth/ngx_http_mupdf_module) and [pymupdf](https://github.com/RekGRpth/pymupdf).
Quite a nice library with a very convenient API, it is possible to set custom allocators (I set Postgres ones), there is a try/catch mechanism, very similar to the Postgres one. Among the shortcomings, it seemed to me that it also took up too much space for the docker image.
Finally, I found a console utility [htmldoc](https://github.com/RekGRpth/htmldoc) written in C with elements in C++, but what I liked most was that it was only a few megabytes in size! Unfortunately, it wasn't ready as a shared library, so I had to finish it myself.
A few edits to the makefile and I had a shared library along with a console utility.
[pg_htmldoc](https://github.com/RekGRpth/pg_htmldoc), [ngx_http_htmldoc_module](https://github.com/RekGRpth/ngx_http_htmldoc_module) and [pyhtmldoc](https://github.com/RekGRpth/pyhtmldoc).
I also made the Postgres interface very simple for it. Several functions that add a file, or text, or URL to the context, and a couple more functions to convert everything that was added to the context to PDF or PS. As a result of the transformation, obviously, the result is not text, but binary.
```sql
select htmldoc_addurl('https://github.com');
copy (
    select convert2pdf()
) to 'htmldoc.pdf' WITH (FORMAT binary, HEADER false)
```
Here also, not only the execution result is transmitted through a file in RAM
```c
out = open_memstream(&output_data, &output_len)
```
but also the HTML text itself in one case is transferred through a similar file in RAM, but using another related function
```c
in = fmemopen((void *)html, len, "rb")
```
When the Postgres memory context is released, the extension context is also cleared.
Rendering PDFs is, of course, good, but to transfer them to clients I need

## Newsletter.

It would seem that what does sending emails and curls have to do with it?! But it turns out that Curl can not only make HTTP requests, but also FTP, SMTP and a bunch of other useful protocols!
Initially, I came across a plugin for Postgres [pgsql-http](https://github.com/pramsey/pgsql-http), I studied it in great detail and I really liked it, except that it uses only a small part of the library [curl](https://github.com/curl/curl), and also the implementation of the request format seemed too complicated to me.
It must be said that this plugin mistakenly believes that [curl](https://github.com/curl/curl) returns text. In fact, this is not the case and it returns binary data, which can then be converted into text, knowing the encoding.
For example, the word "Привет" is text, but to write it to disk, I need to specify the encoding. In UTF-8 encoding this will be one sequence of bytes, in KOI8-R encoding it will be a completely different one, and in Windows-1251 encoding it will be a third. It is a mistake to assume that an HTTP request will always return text in UTF-8 encoding!
I also found [pg_net](https://github.com/supabase/pg_net), which allows to make asynchronous HTTP(S) requests using table queues and a background worker process.
It turned out to have the same shortcomings and therefore I made my own version - [pg_curl](https://github.com/RekGRpth/pg_curl), in which I implemented almost all the capabilities of the curl to the maximum, namely, I made an interface for almost all the functions that accept string or numeric arguments.
And now I can make not only HTTP requests directly from the database, but also send email or copy files via FTP or SCP! All pg_curl functions can be divided into several types. There are simple functions that set all sorts of curl options.
Also, there are simple functions that add headers, as well as files, attachments, etc. There are also simple functions that receive different query results. And finally, there is a rather complex function that triggers the actual execution of the query in Curl.
Until recently, I only used curl_easy_perform, which allowed only one request to be executed at a time. But this summer I significantly improved the extension by adding support for curl_multi_perform, which made it possible to make multiple requests at the same time.
Moreover, I changed the signatures of all functions so gently that if someone used the old version of [pg_curl](https://github.com/RekGRpth/pg_curl), and then compiled a new one, then nothing would break for him even without updating the extension command ALTER EXTENSION UPDATE!
An interesting thing to note is that curl allows to set custom memory management, which I took advantage of by transferring control there through Postgres memory contexts.
Alexey noticed that in this case Curl uses multi-thread resolving, which sometimes led to segfaults, so I added mutexes to all wrappers over Postgress memory contexts passed to Curl to make sure that no thread would call the corresponding functions in parallel.
I also added a huge number of ifdefs to support various versions of curl. If some function is not yet supported by the current old version of the curl, or vice versa, some already outdated function has been removed in the current new version of the curl, then a corresponding error message is displayed.
Well, as is tradition, I currently support all versions of Postgres, starting from 9.4 and including 17, on which the extension is built and tested a little.
So,

## Examples.

### 1) Sending an email with a delivery report

To report on the delivery of letters, I needed my local mail server [opensmtpd](https://www.opensmtpd.org), which through the plugin [pgsql](gawkextlib.sourceforge.net/pgsql/pgsql.html) to [gawk](https://www.gnu.org/software/gawk) just writes delivery information to the database.
I implemented everything else directly in the database using the plugin [pg_curl](https://github.com/RekGRpth/pg_curl) and the asynchronous task scheduler [pg_task](https://github.com/RekGRpth/pg_task).
I attached a trigger to the table with letters, which, after insertion, creates a task to send a letter, which is performed using the function
```sql
CREATE OR REPLACE FUNCTION email(url TEXT, username TEXT, password TEXT, subject TEXT, sender TEXT, recipient TEXT, body TEXT, type TEXT) RETURNS TEXT LANGUAGE SQL AS $BODY$
    WITH s AS (SELECT
        curl_easy_reset(),
        curl_easy_setopt_mail_from(sender),
        curl_easy_setopt_password(password),
        curl_easy_setopt_url(url),
        curl_easy_setopt_username(username),
        curl_header_append('From', sender),
        curl_header_append('Subject', subject),
        curl_header_append('To', recipient),
        curl_mime_data(body, type:=type),
        curl_recipient_append(recipient),
        curl_easy_perform(),
        curl_easy_getinfo_header_in()
    ) SELECT curl_easy_getinfo_header_in FROM s;
$BODY$;
```
Also, I made an interface for mass mailing of letters according to a table from a CSV file, in which I can specify columns with the sender, recipient, as well as other variables that will be inserted into the subject and body of the letter using the template engine [pg_mustach](https:/ /github.com/RekGRpth/pg_mustach).
I can add attachments and, if necessary, template them using the template engine [pg_mustach](https://github.com/RekGRpth/pg_mustach) and/or convert them to PDF using [pg_htmldoc](https://github.com/ RekGRpth/pg_htmldoc).
In addition, I also organized the distribution of letters with invoices for payment according to the conditions from the billing system.

### 2) SMS sending

Similarly, through the REST interface of the telecom operator, I implemented sending SMS directly in the database using the plugin [pg_curl](https://github.com/RekGRpth/pg_curl) and the asynchronous task scheduler [pg_task](https://github.com/ RekGRpth/pg_task).
I attached a trigger to the table with messages, which, after insertion, creates a task for sending SMS, which is performed using the function
```sql
CREATE OR REPLACE FUNCTION post(url TEXT, request JSON) RETURNS TEXT LANGUAGE SQL AS $BODY$
    WITH s AS (SELECT
        curl_easy_reset(),
        ( WITH s AS (
            SELECT (json_each_text(request)).*
        ) SELECT array_agg(curl_postfield_append(key, value)) FROM s),
        curl_easy_setopt_url(url),
        curl_easy_perform(),
        curl_easy_getinfo_data_in()
    ) SELECT convert_from(curl_easy_getinfo_data_in, 'utf-8') FROM s;
$BODY$;
```
After successful sending, a task is created to check the status of the message, which is periodically launched until the final status is reached.
```sql
CREATE OR REPLACE FUNCTION get(url TEXT) RETURNS TEXT LANGUAGE SQL AS $BODY$
    WITH s AS (SELECT
        curl_easy_reset(),
        curl_easy_setopt_url(url),
        curl_easy_perform(),
        curl_easy_getinfo_data_in()
    ) SELECT convert_from(curl_easy_getinfo_data_in, 'utf-8') FROM s;
$BODY$;
```
Also, I made an interface for mass sending SMS to a table from a CSV file, in which I can specify columns with the recipient, as well as other variables that will be inserted into the body of the message using the template engine [pg_mustach](https://github.com /RekGRpth/pg_mustach).
In addition, I also organized the sending of SMS with the amounts for payment according to the conditions from the billing.

### 3) Registration of checks in Atol

Registration of checks in Atol also occurs through the REST interface. I attached a trigger to the table with payments, which, after insertion, creates a task for registering a check, which is performed using the function
```sql
CREATE OR REPLACE FUNCTION post(url TEXT, request JSON) RETURNS TEXT LANGUAGE SQL AS $BODY$
    WITH s AS (SELECT
        curl_easy_reset(),
        curl_easy_setopt_postfields(convert_to(request::TEXT, 'utf-8')),
        curl_easy_setopt_url(url),
        curl_header_append('Content-Type', 'application/json; charset=utf-8'),
        curl_easy_perform(),
        curl_easy_getinfo_data_in()
    ) SELECT convert_from(curl_easy_getinfo_data_in, 'utf-8') FROM s;
$BODY$;
```
After successful registration, a task is created to check the registration status, which is periodically launched until the final status is reached.

### 4) Loading invoices into 1C

It was necessary to somehow load the following information into 1C from billing: counterparties, product range, sales of goods and services, invoices for payment and invoices. Only the OData REST interface was issued as an interface to 1C.
I quickly wrote a loading prototype in Python using a few functions first. Search for an entity by filter, selecting only specified fields, returning a list of found entities.
Read an entity, selecting only the specified fields by entity ID, returning the entity. Creating a new entity, to which the entity body is passed in the form of json. Deleting an entity by its identifier.
And finally, updating/changing the entity by its identifier, to which only those fields of the entity that need to be changed are transferred.
After testing the prototype, I rewrote all these functions in PL/pgSQL, and for HTTP requests I used my plugin [pg_curl](https://github.com/RekGRpth/pg_curl) and of course automated everything using the asynchronous task scheduler [pg_task](https://github.com/RekGRpth/pg_task).

### 5) Enable/disable telephony/Internet/TV

I also turned on/off telephony/Internet/TV right in the database!
Because Telephony worked for us through Postgres, then turning it on/off by balance is just some SQL commands that I automated using the asynchronous task scheduler [pg_task](https://github.com/RekGRpth/pg_task).
The Internet had to be turned on/off on Cisco/Mikrotik via SSH using the plugin [plsh](https://github.com/petere/plsh). Of course, [curl](https://github.com/curl/curl) could do this, but it does not have support for executing SSH commands, although I somehow managed to add it there as an experiment.
Television is turned on/off via the REST interface of the equipment.
