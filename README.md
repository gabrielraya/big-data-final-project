# Big Data 2019: Final Project
## Baby Data Analysis on BBC news

> Stay true to yourself, yet always be **open** to learn. Work hard, and never give up on your dreams, even when nobody else believes they can come true but you. These are not cliches but real tools you need no matter what you do in life to stay focused on your path.  [Phillip Sweet](https://www.brainyquote.com/quotes/phillip_sweet_751180?src=t_open)
> 


[Common Crawl](http://commoncrawl.org/) is a nonprofit organization that crawls the web and everyone can freely use them. Common Crawl uses [Web ARChive](https://en.wikipedia.org/wiki/Web_ARChive) (.warc) files [1](https://en.wikipedia.org/wiki/Common_Crawl). In this project, I counted the occurrence for a word for each domain over the full common crawl data of [April 2019](http://index.commoncrawl.org/CC-MAIN-2019-18) corresponding to 29876 HTML pages from BBC news website. In this specific example, I  searched for the word  occurrence for the words `the` and `Trump` in order to compare the presence of the current president of the USA received in several countries at that period of time.  The aim of this blog is to give you a soft introduction of working with the Common Crawl, working with large data files using Spark in our local pseudo cluster. In the following section, I will explain step by step everything you need to know about this journey.




## Setup

The first need we need to do is to set up our working environment, in this case we need to set all the containers we are gonna be working with. Once everything is set up we can proceed with our data analysis. 

### First time setup

#### Docker Container : snb
We are gonna use this container to interact with Spark notebooks, get to know our data and code our almost final code to run on our pseudo cluster. 

    ## Run the container, in the background
	docker run -d rubigdata/hadoop

	## Create the notebooks folder 
	docker exec snb mkdir /opt/docker/notebooks/BigData

	## Start our sbn container
	docker start sbn

	## Stop our sbn container
	docker stop sbn


#### Clusters
At the end we want to deploy a self-contained standalone Spark application than can run in a cluster and distributed the work load on different nodes. For this reason instead of only running from notebooks, what follows sets a local cluster  using the Docker images from the [Big Data Europe project](https://github.com/big-data-europe/docker-spark):

	## Setup a cluster
	docker network create spark-net	
	docker create --name spark-master -h spark-master --network spark-net -p 8080:8080 -p 7077:7077 -e ENABLE_INIT_DAEMON=false bde2020/spark-master:2.4.1-hadoop2.7
	docker create --name spark-worker-1 --network spark-net -p 8081:8081 --link spark-master:spark-master -e ENABLE_INIT_DAEMON=false bde2020/spark-worker:2.4.1-hadoop2.7
	docker create --name spark-worker-2 --network spark-net -p 8082:8081 --link spark-master:spark-master -e ENABLE_INIT_DAEMON=false bde2020/spark-worker:2.4.1-hadoop2.7

	## Start the master and the workers we created above:
	docker start spark-master 
	docker start spark-worker-1 spark-worker-2


## Data

The data used for this project is a WARC file acquired using this link [http://index.commoncrawl.org/CC-MAIN-2019-18-index?url=www.bbc.com&output=json](http://index.commoncrawl.org/CC-MAIN-2019-18-index?url=www.bbc.com&output=json).  What to do to get the data: 


1. Go to [Common Crawl Index Server](http://index.commoncrawl.org/)
2. Choose the month index you like the most, I used [April 2019 Index](http://index.commoncrawl.org/CC-MAIN-2019-18).
3. Search for BBC news, like in the next image: 
![index_april](https://github.com/rubigdata/cc-2019-gabrielraya/blob/master/img/april_index.PNG)
4. Choose the any BBC captures in the crawl, like : `CC-MAIN-20190423114901-20190423140901-00513.warc.gz`.
5. Download the WARC file on Amazon S3 by prefixing https://commoncrawl.s3.amazonaws.com/ to the crawl-data/CC-MAIN-2019-18/, i.e [https://commoncrawl.s3.amazonaws.com/crawl-data/CC-MAIN-2019-18/segments/1555578602767.67/crawldiagnostics/CC-MAIN-20190423114901-20190423140901-00513.warc.gz](https://commoncrawl.s3.amazonaws.com/crawl-data/CC-MAIN-2019-18/segments/1555578602767.67/crawldiagnostics/CC-MAIN-20190423114901-20190423140901-00513.warc.gz)



## Experiments

### Notebooks: WARC files basics

Spark notebooks allows you to interactively work with Spark. First, I started working with [WARC for Spark notebook](https://github.com/rubigdata/course/blob/gh-pages/assignments/BigData-WARC-for-Spark.snb.ipynb), a helpful notebook  that help you get started on developing code to analyze WARC files in Spark. What I did:

1. Clone the repository
2. Start docker container

		docker start snb
3. Open [http://localhost:9000/notebooks/BigData/](http://localhost:9000/notebooks/BigData/)
4. Drag and drop the notebook on the [http://localhost:9000/notebooks/BigData/](http://localhost:9000/notebooks/BigData/) UI and upload the notebook. This also can be done by the following command:

	    ## Copy notebook into container 
    	docker cp BigData-WARC-for-Spark.snb.ipynb snb:/opt/docker/notebooks/BigData


5. Open bash inside docker
	
		docker exec -it snb  /bin/bash

3. Start your project using a small WARC file that you create. To do this we need first to create a folder to store the data and then we need to download the data:

		mkdir rubigdata/data
		cd rubigdata/data
		wget -r -l 3 "http://rubigdata.github.io/course/" --warc-file="course"

5. Open and play around with the [BigData-WARC-for-Spark.snb.ipynb](https://github.com/rubigdata/course/blob/gh-pages/assignments/BigData-WARC-for-Spark.snb.ipynb)


### Working with a large file

At this moment I have worked only with small dataset, from the url http://rubigdata.github.io/course/, which is the page of this course. Once I managed to understand what is happening I decided to work with the WARC file from BBC news. So I did the following:

1. Copying BBC nes WARC file from host to Docker container
	
		docker cp CC-MAIN-20190423114901-20190423140901-00513.warc.gz  snb:opt/docker/rubigdata/data/CC-MAIN-20190423114901-20190423140901-00513.warc.gz 
	
2. Experiment with this until I finally achieved to count the occurrences of the word `the`, `Trump` and `love`.
3. Copy my data to the clusters. I had to create manually the folders in the master and the workers because for some reason it didn't work so after I created the app folder on such containers I did the following to copy the WARC file.


		# copy into local host
		for node in master worker-1 worker-2
		do
		  docker cp CC-MAIN-20190423114901-20190423140901-00513.warc.gz spark-${node}:/app
		done


4. Created a self contained Spark application: this was the most difficult part but thanks to the invaluable comments on the [2019 course’s Forum](https://github.com/rubigdata/forum-2019) I managed to run it using the following code which  counts the word occurrences of  “the”, “Trump” and "love" in a warc file of bbc.com.

The code can also be found [here](./RUBigDataApp.scala)

### Code

	package org.rubigdata
	import org.apache.spark.SparkConf
	import org.apache.spark.SparkContext
	import org.apache.spark.SparkContext._
	import nl.surfsara.warcutils.WarcInputFormat
	import org.jwat.warc.{WarcConstants, WarcRecord}
	import org.apache.hadoop.io.LongWritable
	import java.io.InputStreamReader
	import java.io.IOException
	import org.jsoup.Jsoup
	
	
	
	object RUBigDataApp {
	 
	  def getContent(record: WarcRecord):String = {
	  	val cLen = record.header.contentLength.toInt
	 	val cStream = record.getPayload.getInputStream()
	  	val content = new java.io.ByteArrayOutputStream();
	
	  	val buf = new Array[Byte](cLen)
	  
	  	var nRead = cStream.read(buf)
	  	while (nRead != -1) {
	    	content.write(buf, 0, nRead)
	    	nRead = cStream.read(buf)
	  	}
	  	cStream.close()
	  	content.toString("UTF-8");
	  }
	  
	  def clean_url (url : String) : String = {
		for (i<-0 to url.length()-1){
			if (!(('a' to 'z').contains(url.charAt(i)) || ('A' to 'Z').contains(url.charAt(i)) || ('0' to '9').contains(url.charAt(i)))) {
				return url.substring(0, i)
			}
		}
		return url
	}
	
	
	  def HTML2Txt(content: String) = {
	  	try {
	    	Jsoup.parse(content).text().replaceAll("[\\r\\n]+", " ")
	  	}
	  	catch {
	    		case e: Exception => throw new IOException("Caught exception processing input row ", e)
	  	}
	  }
	  def main(args: Array[String]) {	
		val conf = new SparkConf().setAppName("RU BigDataApp")
		val sc = new SparkContext(conf)
	  val warcfile = "/opt/docker/rubigdata/data/CC-MAIN-20190423114901-20190423140901-00513.warc.gz"
	
	  val warcf = sc.newAPIHadoopFile(
	              warcfile,
	              classOf[WarcInputFormat],               // InputFormat
	              classOf[LongWritable],                  // Key
	              classOf[WarcRecord]                     // Value
	        )
	  val warc = warcf.map{wr => wr}.cache()
	  //val nHTML = warc.count()
	
	
	  val warcc = warcf.
	      filter{ _._2.header.warcTypeIdx == 2 /* response */ }.
	          map{wr => ( wr._2.header.warcTargetUriStr, HTML2Txt(getContent(wr._2)) )}.cache()
	
	  val warcff = warcc
	       .filter( _._2.contains("the")).cache()
	  val word1 = warcff.count()
	
	  val warcff2 = warcff
	       .filter(_._2.contains("Trump"))
	        val word2 = warcff2.count()
	
	  println("##############\n\n\n\n\n")
	  println("Pages with the word 'the': %s, Pages with the word 'Trump': %s".format(word1, word2))
	  println("\n\n\n\n\n##############")
	   }
	}


*Note* : **Always remember to shutdown your container**:

## Results

I made a comparision of April 2019 Google trends for the mentioned words with my results.

![image2](https://github.com/rubigdata/cc-2019-gabrielraya/blob/master/img/google_trends.png)











## References:

[1] [https://en.wikipedia.org/wiki/Common_Crawl](https://en.wikipedia.org/wiki/Common_Crawl)

[2] [https://rubigdata.github.io/course/assignments/P-commoncrawl.html](https://rubigdata.github.io/course/assignments/P-commoncrawl.html)
)



## Appendix


# Running Hello World
This section describe the basics to run a stand alone scala application

1. Test your code on the Jupyter notebook:
2.  fix null pointer exception : https://github.com/rubigdata/forum-2019/issues/10

