# Mongodb indexer

author: HILLION Marc

## Usage

```bash
# you may need to start the mongo service if not already running

sudo systemctl start mongod.service

node splitter.js
node index.js
```

while performing you will see many logs. You can stop it by typing "pause" and then resume it with "resume".

## Concept

This program performs a fast insertion of csv data to a mongdb collection.  

How is it done?
First we split a big csv file into smaller subfiles. Then with PM2 we are parsong those files with several processes
 
## Implementation


### CSV split

I divide the big csv file into files of maximum 1MB in the csv subfolder. First we get the header line with field names and save it to header.csv. Then we read the file by chunks into a buffer and write to a new csv file. After reading 1MB of data, we truncate the buffer to the last endline character, shift the data in the buffer to keep the last bytes already read and start over with a new csv file until reaching the end of the main file.


### Parsing and insertion in mongdb

First step here is to parse the header.csv file and look for the fields we want to save in mongodb. We save in an array the indexes of these fields. On my computer I use 4 threads corresponding to my number of CPU which seems to be the most efficient. Hence each process will be in charge of one forth of the files (or one more, we make sure that every entry is parsed and inserted in the database). Each process is started with pm2 with the following arguments :
* array of paths to parse
* array of number corresponding to the indexes of the fields we have to keep in a csv line (sorted)
* array of keys for each of these fields

To parse each csv file, we read the data by chunks and check the characters one by one. If the current field must be saved, its value is stored in a js object with the corresponding key. When we reach the end of a line, we insert the document in the collection thanks to a mongodb unordered bulk. Operations are performed when reaching the end of the file.


### Communication between process


We use communication with pm2 for two purposes :
1. From process to master

Each process sends informations about its performances to the main process including the number of documents inserted and the time, for each file and for the whole process. This way these informations can be logged and aggreagated by the main process.

2. From master to process

The main process can send commands to the workers. We use it to pause and resume the indexation. We also stop the timer when paused so that the metrics stay relevant.
