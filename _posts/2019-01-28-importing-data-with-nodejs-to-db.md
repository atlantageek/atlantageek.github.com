---
layout: post
title: "Importing data with node.js into DB"
categories:
- programming
- mysql
tags:
- mysql
comments: true
---
## Frustration
Node's greatest strength is often the source of frustration for a node programmer.  Most recently I was attempting to write a simple node.js script that took a csv file, parsed it and inserted it into a postgresql db.  This is simple in ruby, python and many other scripting related languages but I soon hit a roadblock on node.js. The following script was my first attempt at importing the data.


```js
//date,date_block_num,shop_id,item_id,item_price,item_cnt_day
const company = 'ACME';
const csvFilePath = 'acme.csv';
const { Client } = require('pg');

const csv = require("csvtojson");
const client = new Client({
    host: 'localhost',
    port: 5432,
    user: 'me',
    password: 'secretpassword',
    database: 'cooldb'
})
client.connect();


csv().fromFile(csvFilePath)
    .subscribe((json) => {
        //console.log(json);
        var dt = json.date.split('.');
        client.query("INSERT into series(orgName,  cat1, cat2, cat3, cat4,cat5,dt, val, attr1)" + 
                                " values($1::text, $2,   $3,   $4,   $5,  $6,  $7, $8,   $9)",
            [company, json.shop_id, json.item_id,'','','', dt[2] + '-' + dt[1] + '-' + dt[0], json.item_cnt_day, json.item_price]).then((result) => {
                console.log("-------------------");

            }).catch(e => {
                console.error(e.stack);
            })
    }, onError, onComplete)

function onError(err) { console.log(err) }
function onComplete(err) { console.log(err) }
```

Though its a little long the script is fairly simple.  It takes a csv file and inserts the data into a postgresql db.  Unfortunately it didnt work.  I would get the following error which is a fancy way of running out of memory on the stack for node.js

```
FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed - JavaScript heap out of memory
```
In other words the reading and processing of the csv file quickly ran much faster than the insertion of data into the db and soon the queue of records overwhelmend the memory for node.js

So how can we handle this.  How can we slow down the import of the data while the writing of the db runs ahead. The solution was to use streams.
Streams are streams of data that are divided into chunks.  There are 4 types of streams.  ReaderStream, WriterStream, TransformStream and DuplexStream.  A reader stream takes reads data from the source.  A writer stream takes input and processes the data. The writer stream normally has the ability to tell the reader stream to pause while it catches up.  A TransformStream is a combination of reader/writer streams where it takes input, modifies the data and then sends output. And then there are Duplex streams where the data is bidirectional.  

So the final decision was to use an readerStream and a writerStream. The readerstream was easy enough. csv-stream allowed the creation of a reader stream but as far as I could tell there was no good writerStream for the database. Creating streams are not well documented and seems overly complex.  However there is a node package called [thought2](https://github.com/rvagg/through2)  that simplifies the creation of a writeStream to a simple function.  You pass thought2() a function that has the signature of func(chunk, enc, callback)  Of these parameters the more important ones are chunk which contain the data being streamed and callback, a function that is called after the data in chunk is processed.  So our write stream would look like this.
```js
dbStream = through2.obj(function (data, enc, callback) {
        var dt = data.date.split('.');
        var formatted_dt = dt[2] + '-' + dt[1] + '-' + dt[0];
        client.query("INSERT into series(orgName,  cat1, cat2,dt, val)" +
                " values($1::text, $2,   $3,   $4,   $5)",
                [company, data.store, data.item,data.date, data.sales]).
                then((result) => {
                        callback();

                }).catch(e => {
                        callback(e);
                });
})
```
Here the function passed to through2 takes each row of data and Inserts the record into the db.  When insert is complete, the callback function is executed.

The Final script looks like the following:

```js
const company = 'ACME';
const csvFilePath = 'acme.csv';
const { Client } = require('pg');
const fs = require('fs')
const util = require('util');
const stream = require('stream');
const Writable = stream.Writable || require('readable-stream').Writable;
const through2 = require('through2');


dbStream = through2.obj(function (data, enc, callback) {
        var dt = data.date.split('.');
        var formatted_dt = dt[2] + '-' + dt[1] + '-' + dt[0];
        client.query("INSERT into series(orgName,  cat1, cat2,dt, val)" +
                " values($1::text, $2,   $3,   $4,   $5)",
                [company, data.store, data.item,data.date, data.sales]).
                then((result) => {
                        callback();

                }).catch(e => {
                        callback(e);
                });
})

const csv = require("csvtojson");
const client = new Client({
    host: 'localhost',
    port: 5432,
    user: 'me',
    password: 'secretpassword',
    database: 'cooldb'
})
client.connect();


var csvStream = csv.createStream({});
fs.createReadStream(csvFilePath).pipe(csvStream).pipe(dbStream)
```
