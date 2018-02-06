# CSC8101 Spark batch coursework



## Introduction
In 2006 Netflix announced "The Netflix Prize", challenging teams of computer science researchers to produce an algorithm which predicted the movie ratings of netflix users with greater accuracy than Netflix's approach at the time.
The prize of \$1,000,000 was eventually awarded in 2009 to a team from AT&T Labs.
In this coursework, you will use the [Spark DataFrames](http://spark.apache.org/docs/latest/sql-programming-guide.html) and [Machine Learning algorithms](http://spark.apache.org/docs/latest/ml-guide.html) provided by [Apache Spark](http://spark.apache.org) to do the same. 
Unfortunately ... there is no prize.

## The Data
The data provided to you is, apart from some minor modifications and the addition of a Neo4j database, the same data as used in the netflix prize. It consists of the following files:

* **mv_all_simple.txt**: a text file containing roughly 100 million lines, corresponding to the same number of movie ratings. Each line is of the format `Movie Id, User Id, Rating, Rating Date`. For example: 
    ```
    1,1488844,3,2005-09-06
    1,822109,5,2005-05-13
    1,885013,4,2005-10-19
    1,30878,4,2005-12-26
    1,823519,3,2004-05-03
    ```
    
* **netflix_movie_titles.txt**: a text file containing roughly 18 thousand lines, corresponding to the same number of movies. Each line is of the format `Id, Year of release, Title`. For example:
    ```
    1,2003,Dinosaur Planet
    2,2004,Isle of Man TT 2004 Review
    3,1997,Character
    4,1994,Paula Abdul's Get Up & Dance
    5,2004,The Rise and Fall of ECW
    ```
    
* **qualifying_simple.txt**: a text file containing roughly 2.8 million lines, corresponding to the same number of ratings for which we would like you to predict a value. Each line is of the format `Movie Id, User Id, Date of rating`, obviously the ratings have been omitted. For example:
    ```
    1,1046323,2005-12-19
    1,1080030,2005-12-23
    1,1830096,2005-03-14
    1,368059,2005-05-26
    1,802003,2005-11-07
    1,513509,2005-07-04
    1,1086137,2005-09-21
    ```
    
* **graph.tgz**: a [sample graph database](https://neo4j.com/developer/movie-database/) provided by Neo4j themselves. This database contains movie data about movies, actors and directors pulled from [TheMovieDB](http://themoviedb.org/). The databases node types include `Person, Actor, Director, User, Movie`, along with the relationship types `ACTED_IN, DIRECTED, RATED`. The database is not a text file but rather an archive file containing a folder named `graph.db`.
 
* **movie_titles_canonical.txt**: a text file containing roughly 13 thousand lines, corresponding to the same number of movies. These movie titles have been pulled from the Neo4j TheMovieDB database. Each line is of the format `Title, Year of release`. For example:
    ```
    Avatar,2009
    Amélie,2001
    Full Metal Jacket,1987
    E.T.: The Extra-Terrestrial,1982
    Independence Day,1996
    The Matrix,1999
    ```

#### Obtaining the data 

All the above datasets are stored in an amazon S3 bucket. When you start this coursework you should perform the following steps on your VM: 

1. Create and move into a data folder in your user's home directory with the following command: 
    ```
    mkdir ~/data && cd ~/data
    ``` 
    
2. Download each of the files above into your data folder with the following command: 
    ```
    wget https://s3-eu-west-1.amazonaws.com/csc8101-spark-assignment-data/name_of_the_file
    ``` 
    
3. Remove the old database files from Neo4j:
    ```
    sudo rm -rf /var/lib/neo4j/data/databases/graph.db
    ```

4. Unzip the `graph.tgz` file and move it to the location that Neo4j (pre-installed on your VMs) expects using the following command: 
    ```
    tar -xvf graph.tgz && sudo mv graph.db /var/lib/neo4j/data/databases/
    ```

**Note**: Throughout this coursework you should not need to modify any of the provided .txt files. Infact you must not, as one of the tasks towards the end of the coursework is to attempt to run your spark job on a Cluster, rather than a VM. On the cluster the data files will be provided for you, and therefore if your code assumes a modified structure it may not work. 

## Links

Throughout this coursework you will need to refer heavily to documentation websites as well as the material (incl. books) mentioned in the lectures.
Below are some helpful links:

* [Spark Programming Guide](https://spark.apache.org/docs/latest/programming-guide.html)
* [Spark MLlib Programming Guide](http://spark.apache.org/docs/latest/ml-guide.html)
* [Neo4j Language Drivers](https://neo4j.com/developer/language-guides/)
* [Neo4j Cypher Refcard](https://neo4j.com/docs/cypher-refcard/current/)

**Important**: You may notice that on the left hand side of the MLlib page there are links to two versions of each page, one under the heading "MLlib: Main Guide" and one under the heading "MLlib: RDD-based API Guide". You should **follow the Main Guide** as the other guide is now deprecated..

## Tasks

Below are a list of the individual tasks you will be expected to complete as part of your spark batch coursework. We try to describe each task in detail, however if you are ever in doubt please ask a demonstrator during a practical.

#### Incremental progress

As you can see, there are several tasks for you to complete in this coursework. Some of them may prove fairly challenging depending upon your prior level of experience. You may well not complete them all.
However, we would like to avoid a situation in which students get stuck on task 1 or 2 and then either give up or struggle on for days, never attempting the later tasks. 
So, we have marked certain tasks with the symbol **(?)** which means  that the task is optional.

**(?)** if a task is optional, this simply means that later tasks do not depend on its output. This means you can move on from such a task if you are stuck.
To be clear, an optional task is still worth marks.


#### Task 0: Environment Setup

If you are planning to develop using Python, you can use a Jupyter notebook (like those used in the lectures) which is hosted on your VM. To start the notebook server log into your VM and run the following command:

`$ pyspark`

You will not be able to enter any other commands in this console while the server is running. If you want to run commands on the VM while the notebook is running then login to your VM in a separate ssh session (on Windows you will need to start another putty session).

Once the server is running you can open the notebook in your web browser at: 

`<VM IP ADDRESS>:8888`

Once on the Jupyter home page you can start a new notebook by going to the "New" menu in the top right and selecting "Python 3" under the notebook section. The code for your solution to the tasks below can be saved in this new notebook and started up again from the Jupyter homepage.

Please note that in order for notebook to function properly with Spark, only one kernel (notebook) can be running at any one time. Use the "Running" tab on the Jupyter home page to check how many kernels are running. You may need to shut all kernels down and restart the notebook you are interested in, in order to get a functioning connection to Spark.
 
When using submitting a spark job or starting a pyspark notebook to run spark jobs on your local VM you cannot configure `spark.driver.memory` using `SparkConf`. 
Instead you must use the `driver-memory` command line parameter for `spark-submit` such as:

`$ pyspark --num-executors 5 --driver-memory 2g --executor-memory 2g`
    
A [python sample program](https://raw.githubusercontent.com/tomncooper/CSC8101-Documentation/master/spark/python-stub/SparkNeo4jSample.py) has been created for you as a starting point. This can be used a base for a Jupyter notebook.


#### Task 1 (\*)

You have been given an canonical list of movies (in `movie_titles_canonical.txt`), along with a list of the movies which appear in the netflix prize dataset.
You must write a simple algorithm to determine where a movie title from the netflix dataset is an alias of a movie title from the canonical dataset.
Some examples of aliases would be _"The Lion King"_ => _"Lion King"_, _"Up!"_ => _"Up"_ and _"The matrix"_ => _"The Matrix"_.

The algorithm should be of your own design.
It should be more complicated than just checking for string equality and probably less complicated than something like [Edit Distance](https://en.wikipedia.org/wiki/Edit_distance).
Marks will be awarded for producing a sensible method which will catch most or all aliases, but also for considering the performance of your approach. 
The final output of your algorithm should be a `Map[Int => String]` where each key is the id of a movie from the netflix dataset, and each string is the original title from the canonical dataset. 
This map should be broadcast to all spark executors.

**Hint**: Don't forget you have dates.

#### Task 2 (\*)

You must now use spark to load all the netflix ratings (in `mv_all_simple.txt`) into an `RDD`.
This process will involve parsing the string form of each line into a more appropriate tuple or datatype.
You must randomly split this `RDD` into two unequal `RDD`s (i.e. an 80/20 ratio). 
Finally, you must import `mllib.recommendation.ALS` and pass one of these `RDD`s to the `ALS.train` method, producing a model object capable of predicting movie ratings by users. 
Initially, use the following values for the other parameters to `ALS.train`: 

* _Rank_ = 10
* _Number of Iterations_ = 5
* _Lambda_ = 0.01 

**Hint**: Be careful not to spend a long time replicating (possibly poorly) which already exists in Spark. In particular, make sure you are aware of all the methods available on `RDD`s.

#### Task 3 (?)

For this task you will use the previously generated model to recommend 10 movies to a specific user based on their predicted rating of said movies. 
The user in question has the id _30878_. Once you have used the model to retrieve the 10 recommended movie ids, you should use the alias map created earlier to retrieve their titles and write these recommendations to a file.
To allow you to informally assess the quality of your predictive model you should also filter over the `RDD` containing actual ratings and write some  real ratings made by user _30878_ to another file.

**Hint**: Again, don't reinvent the wheel. Also remember that most operations over an `RDD` are actually evaluated lazily, however there are some operations which will _force_ the datastructure.
If you are going to _force_ an `RDD` in multiple different places, it is a good idea to `persist` it for performance.

#### Task 4 (?)

This next task involves more formal evaluation of the model which you produced in Task 2, rather than just "eyeballing" the output for a single user.
In spark there exist several evaluation methods for both binary classification and regression models. As we are predicting something which may take any value between 0.0 and 5.0 (i.e. a rating) we producing a regression model.

The evaluation method you are using is called [Root Mean Squared Error](https://en.wikipedia.org/wiki/Mean_squared_error), and is related to calculating the standard deviation between predicted and actual values.
In order to calculate this value, intuitively, we need to predict ratings for `(Movie Id, User Id)` pairs where we already know what the real ratings are. 
This is why you split your `RDD` of ratings back in Task 2 and only trained on 80% of the values; the other 20% is what you use to evaluate the results of this training!

You should import `org.apache.spark.mllib.evaluation.RegressionMetrics` and read the [docs](https://spark.apache.org/docs/latest/ml-tuning.html) closely.
One thing which is important to note is that you cannot pass the model produced by `ALS.train` to spark executors, which means you cannot simply call `model.predict` inside a function which you then `map` over the `RDD` of ratings. 
Instead you must pass an `RDD[(User Id, Movie Id)]` to `predict`, which will return an `RDD` of ratings (i.e. a tuple 3 `(User Id, Movie Id, Rating)`) where the rating values themselves are predictions.
This can make it a little tricky to see how you relate the original (real) rating values to the predicited rating values.

Once you have an RMSE score, perhaps try giving your spark `ALS.train` method different parameters to see how this affects the quality of the model produced.

**Note**: `RDD`s **do not** guarantee ordering of elements.

**Hint**: Remember the special operations on `RDD`s of pairs.

#### Task 5 (\*)

This next task should be nice and quick. Using spark, pull in all the `Movie Id, User Id, Date` lines from `qualifying_simple.txt`
and produce an `RDD` of `(User Id, Movie Id)` (you may ignore the dates). Use the model produced in Task 2 to calculate 
ratings for every element of this RDD.

Once you have done this, you should write these ratings to a file. 

**Note**: If you use spark's built in `saveAsTextFile` method, you will see that many files are produced, with names like
`part-0024.txt`. This is due to the fact that the contents of each `RDD` partition (remember that `RDD`s are partitioned 
and distributed) are written out separately. You do not have to recombine these files.


## Deliverables

At the end of your coursework efforts, you should gather as many of the following as you have managed to produce:

* A `.zip` file containing the src for the spark job
* A `.txt` file containing the recommendations and ratings for user _30878_ (may be two `.txt` files).
* A `.zip` file containing all the predicted ratings from `qualifying_simple.txt`. This is likely many text files of the form `part-0000.txt` etc...
* A `.txt` file containing the Cypher read query asked for in Task 7.

The above files should in turn be placed within a file named `submission.zip` and uploaded to Ness.
