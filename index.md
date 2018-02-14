# CSC8101 Spark batch coursework



## Introduction
In 2006 Netflix announced "The Netflix Prize", challenging teams of computer science researchers to produce an algorithm which predicted the movie ratings of netflix users with greater accuracy than Netflix's approach at the time.
The prize of \$1,000,000 was eventually awarded in 2009 to a team from AT&T Labs.
In this coursework, you will use the [Spark DataFrames](http://spark.apache.org/docs/latest/sql-programming-guide.html) and [Machine Learning algorithms](http://spark.apache.org/docs/latest/ml-guide.html) provided by [Apache Spark](http://spark.apache.org) to train a movies recommendation model on a subset of the original dataset used in the challenge.

## The Data
The data provided to you is, apart from some minor modifications and the addition of a Neo4j database, the same data as used in the netflix prize. It consists of the following files:

* **movie_titles_canonical.txt**: a text file containing roughly 13 thousand lines, one per movie, for example:
    ```
    Avatar,2009
    Amélie,2001
    Full Metal Jacket,1987
    E.T.: The Extra-Terrestrial,1982
    Independence Day,1996
    The Matrix,1999
    ```
you will use this small dataset in Task 1.

* **mv_all_simple.txt**: a text file containing roughly 100 million lines, corresponding to the same number of movie ratings. Each line is of the format `Movie Id, User Id, Rating, Rating Date`. For example: 
    ```
    1,1488844,3,2005-09-06
    1,822109,5,2005-05-13
    1,885013,4,2005-10-19
    1,30878,4,2005-12-26
    1,823519,3,2004-05-03
    ```
 
this dataset is used to train the recommender model in **Task 2**, however it is too large to be processed using a single node Spark deployment. It is there in case you want to attempt training on this full-size dataset (**Task2 2**) after you have completed your coursework.

Instead you will use this smaller dataset:

**mv_sampled.parquet**: this is a subset of the whole ratings file, containing about 1 million ratings. You will use this file in **Task 2** to train your ASL model.
Note that this is encoded using Spark's column-wise binary format for Dataframes (parquet), which makes it very space-efficient and fast to load. 

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
  
  Note: the .parquet dataset is a tarfile consisting of an entire folder. Untar into the ~/data directory. This will create a sub-directory called `mv_sampled.parquet`
    
**Note**: Throughout this coursework you should not need to modify any of the provided .txt files. Infact you must not, as one of the tasks towards the end of the coursework is to attempt to run your spark job on a Cluster, rather than a VM. On the cluster the data files will be provided for you, and therefore if your code assumes a modified structure it may not work. 

## Links

Throughout this coursework you will need to refer heavily to documentation websites as well as the material (incl. books) mentioned in the lectures.
Below are some helpful links:

* [Spark Programming Guide](https://spark.apache.org/docs/latest/programming-guide.html)
* [Spark MLlib Programming Guide](http://spark.apache.org/docs/latest/ml-guide.html)
* [Neo4j Language Drivers](https://neo4j.com/developer/language-guides/)
* [Neo4j Cypher Refcard](https://neo4j.com/docs/cypher-refcard/current/)

**Important**: You may notice that on the left hand side of the MLlib page there are links to two versions of each page, one under the heading "MLlib: Main Guide" and one under the heading "MLlib: RDD-based API Guide". You should **follow the Main Guide** as the other guide is now deprecated..

#### Incremental progress

As you can see below, there are several tasks for you to complete in this coursework. Some of them may prove challenging depending upon your prior level of experience.
If you struggle with Task 1, you can re-start from Task 2 as this requires loading the **mv_sampled.parquet.tar.gz** and therefore does not require you to have completed Task 1.

Additionally we have marked certain tasks with the symbol **(?)** which means  that the task is optional.
if a task is optional, this simply means that later tasks do not depend on its output. This means you can move on from such a task if you are stuck.
To be clear, you do not get marks for an optional task that you have not attempted.

## Tasks

#### Task 0: Environment Setup

You should be using a pyspark Jupyter notebook (like those used in the lectures) which is hosted on your VM. To start the notebook server log into your VM and run the following command:

`$ pyspark`

You will not be able to enter any other commands in this console while the server is running. If you want to run commands on the VM while the notebook is running then login to your VM in a separate ssh session (on Windows you will need to start another putty session).

Once the server is running you can open the notebook in your web browser at: 

`<VM IP ADDRESS>:8888`

Once on the Jupyter home page you can start a new notebook by going to the "New" menu in the top right and selecting "Python 3" under the notebook section. The code for your solution to the tasks below can be saved in this new notebook and started up again from the Jupyter homepage.

Please note that in order for notebook to function properly with Spark, only one kernel (notebook) can be running at any one time. Use the "Running" tab on the Jupyter home page to check how many kernels are running. You may need to shut all kernels down and restart the notebook you are interested in, in order to get a functioning connection to Spark.
 
When using submitting a spark job or starting a pyspark notebook to run spark jobs on your local VM you cannot configure `spark.driver.memory` using `SparkConf`. 
Instead you must use the `driver-memory` command line parameter for `spark-submit` such as:

`$ pyspark --num-executors 5 --driver-memory 2g --executor-memory 2g`

this is useful of course if you find that your runtime struggles with the size of the input dataset.


#### Task 1
Input Files:
- **netflix_movie_titles.txt**
- **movie_titles_canonical.txt**

You have been given an canonical list of movies (in `movie_titles_canonical.txt`), along with a list of the movies which appear in the netflix prize dataset: `netflix_movie_titles.txt`.
You must write a simple algorithm to determine where a movie title from the netflix dataset is an alias of a movie title from the canonical dataset.
Some examples of aliases would be _"The Lion King"_ => _"Lion King"_, _"Up!"_ => _"Up"_ and _"The matrix"_ => _"The Matrix"_.

The algorithm should be of your own design.
It should be more complicated than just checking for string equality and probably less complicated than something like [Edit Distance](https://en.wikipedia.org/wiki/Edit_distance).
Marks will be awarded for producing a sensible method which will catch most or all aliases, but also for considering the performance of your approach. 
The final output of your algorithm should be a dictionsry of the form:

`{ <netflix movie ID>: <original title from canonical dataset}`

This map should be `broadcast` to all spark executors.

**Hint**: Don't forget you have dates.

#### Task 2 (\*)
Input File:
- **mv_sampled.parquet**

You must now use spark to load the sampled netflix ratings (in `mv_sampled.parquet`) into a DataFrame (look up the very simple load commands for Parquet files).

You must randomly split this DataFrame into two unequal DataFrames (i.e. an 80/20 ratio). 

You should then use this DataFrame to train an ALS model that can be used to predict user movie ratings (ie which rating a user will give to a movie that they have not already rated).

Remember Spark offers two ML libraries. You must use the newer one: 
`from pyspark.ml.recommendation import ALS`
`from pyspark.ml.recommendation import ALSModel`

Initially, use the following values for the other parameters to `ALS.train`: 

* _Rank_ = 10
* _Number of Iterations_ = 5
* _Lambda_ = 0.01 

#### Task 3 (?)
Input: Model from Task 2

For this task you will use the model you  generated in Taks 2 to recommend 10 movies to a specific user based on their predicted rating of said movies. 
The user in question has the id _30878_. Once you have used the model to retrieve the 10 recommended movie ids, you should use the alias map created earlier to retrieve their titles and write these recommendations to a file.
To allow you to informally assess the quality of your predictive model you should also filter over the `RDD` containing actual ratings and write some  real ratings made by user _30878_ to another file.

**Hint**: look at the `recommendForAllUsers` method available on the ` pyspark.ml.recommendation.ALSModel` class.

**Hint**: Remember that most operations over an `RDD` or DataFrame are actually evaluated lazily, however there are some operations which will _force_ the datastructure.
If you are going to _force_ an `RDD` in multiple different places, it is a good idea to `persist` it for performance.

#### Task 4 (?)

This next task is a quantitative evaluation of the model which you produced in Task 2, rather than just "eyeballing" the output for a single user.
In spark there exist several evaluation methods for both binary classification and regression models. As we are predicting something which may take any value between 0.0 and 5.0 (i.e. a rating) we producing a regression model.

You will use Root Mean Squared Error (RMSE) as described in the example notebooks from class. This involves predict ratings for `(Movie Id, User Id)` *on the evalation set you set aside in Task 2.*

You should use the `RegressionEvaluator` as illustrated in the class PDA notebooks.

#### Task 5 (\*)

In this task you will tune your model's hyperparameters:  

* _Rank_ = 10
* _Number of Iterations_ = 5
* _Lambda_ = 0.01 

for this, build a `ParamGrid` in combination with the `CrossValidator` as seen in class.

See [here](http://spark.apache.org/docs/latest/ml-tuning.html) for documentation and examples.
You should experiment with different parameter and report on the best performance you can achieve (best RMSE / r^2 correlation).

#### Task 6 (\*)

Input File:
- **qualifying_simple.txt**

Using spark, pull in all the `Movie Id, User Id, Date` lines from `qualifying_simple.txt`
and produce a DataFrame of `(User Id, Movie Id)` (you may ignore the dates). Use the model produced in Task 2 to calculate 
ratings for every element of this DataFrame.

Once you have done this, you should write these ratings to a file. 

**Note**: If you use spark's built in `saveAsTextFile` method, you will see that many files are produced, with names like
`part-0024.txt`. This is due to the fact that the contents of each `RDD` partition (remember that `RDD`s are partitioned 
and distributed) are written out separately. You do not have to recombine these files.

#### Task 7 (\*)

The goal of this last task is to establish users similarity based on how they rate the same movies, using the DataFrame that you created in Task 6. 

In this exercise, the similarity between users (u1, u2) is defined as *the number of movies that both u1 and u2 rate at least 3.* .

Your task is to find the top 10 most similar users to user *30878*.

**hint**: do not use explicit for loops. Instead, consider a Map Reduce pattern on the DataFrame that you created in Task 6.

## Deliverables

At the end of your coursework efforts, you should gather as many of the following as you have managed to produce:

* A `.zip` file containing the src for the spark job
* A `.txt` file containing the recommendations and ratings for user _30878_ (may be two `.txt` files).
* A `.zip` file containing all the predicted ratings from `qualifying_simple.txt`. This is likely many text files of the form `part-0000.txt` etc...
* A `.txt` file containing the Cypher read query asked for in Task 7.

The above files should in turn be placed within a file named `submission.zip` and uploaded to Ness.
