---
layout: post
title: "File Synchorinaztion with MongoDB"
author: "Ozgur Budak"
categories: journal
tags: [documentation,sample]
image: cards.jpg
---

Hi! Today I'm going to explain file synchorinaztion with MongoDB, in such a fashion that anybody with an intention of connecting a remote Mongodb server, updating, inserting and deleting data can benefit. I am also going to talk about how someone can do the things above periodically.

> Note: Language used in the next parts is Java. With that being said, I don't think this is going to be an issue for the general audience as the Drivers for MongoDb are fairly similar between languages.

## Connecting Your Application With Remote MongoDb Server

You have a few options here. You could run an EC2 instance in AWS and then ssh into it. Then from the command line you could install MongoDB and then try to connect it from your application. What I did was easier: run a cluster in MongoDb Atlas.

It comes with a cluster of MongoDb already installed. But you still need to do few things:
On the security tab, you need to grant access to IP’s for connection.I entered “0.0.0.0/0” as this means All IP’s can connect to your database. But beware that this is not good practice. I only did this since this DB will not have any sensitive data and will be only used for practice.
You’ll also have to create a database user and grant it access to the databases you want. You can customize the user and its privileges in the “Database Access” tab.

After these configurations you are ready to connect your application to your server.

 ```
 MongoClientURI uri = new MongoClientURI(
   "mongodb+srv://"+mongoUser+":"+mongoPass+"@cluster0-kobje.mongodb.net/test?retryWrites=true&w=majority");
 ```


 With the previously created database user you can get a link similar to above. After you create a cluster, there will be a “connect” tab where you can find a code snippet with your language of choice. In this tab you can find your URI. I suggest that you store your username and password in a config file or as environmental variables rather than hard coding them.

 With the URI of your cluster combined with your credentials you can create a new instance of MongoClient and get your desired database like the code example below.

 I also like to reach a collection after making sure it exists in the first place. You can check that by iterating over the list of collection names.

Now you are all set and ready to do some db operations.

 ```
boolean collectionExists = database.listCollectionNames().into(new ArrayList<String>()).contains("GameStates");
       if(collectionExists){
           this.collection = database.getCollection("GameStates");
       }else{
           this.database.createCollection("GameStates");
           this.collection = database.getCollection("GameStates");
       }
 ```

 ## Inserting and Deleting Documents from MongoDB

 MongoDB makes it ultra simple to insert or delete a Document. You can filter all the collection items with Filter.eq which creates a filter that matches all documents where the field is equal to provided value.

 After getting the desired Documents you just iterate over them and delete them like below:
 (Note that since _id is unique no iteration is required.)

```
collection.deleteOne(Filters.eq("_id", idMap.get(entry.getKey())));

```

Inserting is fairly straightforward as well. Instead of deleting you just call “insertOne”.


## Doing Database Operations Periodically

Java has a nice interface called **ScheduledExecutorService** which enables us to run our methods periodically. What we do here is create a Runnable type object that has the run method which includes what we want to do periodically.
After that we give this Runnable object to  scheduler.scheduleAtFixedRate method which returns us a scheduledFuture object. scheduleAtFixedRate takes in a task you want to do, delay and rate. (The same method was also used for deleting files and regularly asking files from the master by follower).

```
public synchronized void   scheduleOperation() {
       final Runnable updater = new Runnable() {
           public void run() { checkAndUpdateGameStates();}
       };
       final ScheduledFuture<?> updaterHandle =
               scheduler.scheduleAtFixedRate(updater, 2, 10, SECONDS);
   }

```

## Detecting Changes in A File and updating The Changed Ones


We thought of some ways about detecting changes like checking the last modified date, giving an integer to indicate the file is changed, hashcode of the file etc. But in the end what we went with was the hashing. We used the **SHA-256** algorithm that generates almost unique hashes of the SHA family. Java provides access to this algorithm though **MessageDigest** class.


```
MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] encodedhash = digest.digest(
                originalString.getBytes(StandardCharsets.UTF_8));
```

All we had to do is create an instance of MessageDigest class and digest it with our string.
Moreover we had to convert this byte array into a hexadecimal string to get the hashed value.

In our case the string used was the string parsed version of our game state file which was in JSON format. We used JSON.parser which is included in the  “Simple JSON” library and fed it with our filereader. This resulted in a JSONObject and we simply called its toString method to get a string with the contents of our game state.

## File Synchronization with MongoDB

We now combine all the things we have covered so far. The goal is to periodically check the files in a directory and synchronize it with our remote database.

In our case we have a MongoWriter object which connects to a remote database when its initialized. During the initialization part we also create a Hashmap of file name, hash pairs.
We insert all the files in the database and keep the ids in a separate id table.

When we call the update method we create a new Hashmap of the current files in the directory with file name, hash pairs. Comparing the entries of the updated map and the previously created map will be used to detect changes.

If a file is in the old map but not in the new map, this means that the file is not in the directory any more. In this case we don't have to store it in the db any longer. We delete it from db with the id thanks to our id table.

If a file is in the new map but not in the old map, this means that this is a new game file.We insert it in db. We also add the newly added object into the id table as well

If a file is in both maps but their hashes are not equal, this means that the contents of that file is modified. In this case, the first thing we do is delete that file based on its id with the help of the id table. Then we insert the file into db as mentioned above. Afterwards we update the id table with the id of the newly inserted document.

Finally we assign our updated map to be our old map so that in the next call we can compare the changes again.

Our last step is to call a scheduler with our method so that it happens at a fixed rate, say 30 seconds.
