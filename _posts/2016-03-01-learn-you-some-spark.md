---
layout: post
title: "Learn You Some Spark"
description: "Get started running spark, even if you are impatient."
date: 2016-03-01 12:00:00 -0400
category: spark
tags: [spark, python]
---

Let's say you have an aging data aggregation system that is straining under the weight of your data processing. Your system 
takes a bunch of event data in JSON, munges it up in a bunch of Perl and shoves it into SQLServer for further munging and 
delivery to the user. You know your processing needs are going to grow significantly in the coming months. Time to consider 
a new platform for processing this data.
<br/>
![center](/images/2016-03-01-learn-you-some-spark/rube.jpg)
<br/>
Enter Spark. Spark is trying to make Hadoop-style data processing accessible and performant. It’s magic and provided it is
applicable to your domain, it can solve lots of problems of scale.
<br/>
Did I mention you have a Windows laptop on this assignment? Sorry about that. Them’s the breaks… but at least you have Cygwin. 
You need a way to spin this system up so you can do the development and prove out the concept. Leave scaling for production, 
you just need to get off the ground today.
<br/>
For that, you need Vagrant. Vagrant hides much of the complexity of VirtualBox for running Linux-y tools on your Windows laptop.
<br/>
<br/>
Let’s start.
<br/>
<br/>

## Setting up Spark on Windows
- Install VirtualBox. Take the defaults.
- Install Vagrant. Again, take the defaults.
- Create a project directory to hold your code and a subdirectory to hold your data. Let’s call it “rube”. Open up a file called “Vagrantfile”

{% highlight bash %}
> cd
> mkdir -p rube/data
> vi Vagrantfile
{% endhighlight %}
<br/><br/>

Insert the following code into Vagrantfile.

<br/><br/>
{% highlight ruby %}
    Vagrant.configure(2) do |config|
      config.vm.box = "ubuntu/trusty64"
      config.vm.provision :shell, inline: "sudo apt-get -y update"
      config.vm.provision :shell, inline: "sudo apt-get -y install vim"
      config.vm.provision :shell, inline: "sudo apt-get -y install openjdk-7-jdk"
      config.vm.provision :shell, inline: "wget http://www-eu.apache.org/dist/spark/spark-1.6.0/spark-1.6.0/spark-1.6.0-bin-hadoop2.6.tgz"
      config.vm.provision :shell, inline: "tar zxf spark-1.6.0-bin-hadoop2.6.tgz"
      
      config.vm.synched_folder "./data" "/data"
      config.vm.provider "virtualbox" do |vb|
        vb.memory = "8192"
      end
    end
{% endhighlight %}
<br/><br/>

What does all this do? I tells vagrant when you fire it up here in a bit to install a Ubuntu Trusty64 Linux instance on VirtualBox, 
fire it up, install some stuff, tie a local directory under “rube” (rube/data) to /data on your Linux instance, and allocate 8G 
of ram. Change this to taste of course.
<br/><br/>
Continuing on…
<br/><br/>
In your shell, fire up vagrant.
<br/><br/>
{% highlight bash %}
> vagrant up
{% endhighlight %}
<br/><br/>
Lots of output goes spinning by. This might take 20 minutes or more the first time. Go get a sandwich and ponder how amazing 
this is.
<br/><br/>
Once that completes, you’re ready to go. Use ‘vagrant ssh’ to get on to your instance.
<br/><br/>
{% highlight shell %}
> vagrant ssh
{% endhighlight %}
<br/><br/>
More output and a prompt. You’re in. Fire up pyspark and play around.
<br/><br/>
{% highlight shell %}
~ whoami
vagrant
~ cd spark*
~ bin/pyspark
{% endhighlight %}

## Running Spark Code
Now that you have Spark running on a Linux box running on a Windows box, you need to process your data. Pyspark is an excellent 
choice. You can use Scala or Java as well, but I’m very familiar with Python, Java/Spark is too painful and I don’t have time 
to learn Scala today. I’ve got stuff to do. Pyspark it is.
<br/><br/>

The gist of the code is as follows:
- Connect to spark
- Tell it which files you care about. Perhaps date bracketed like I did. Note, these can be on the filesystem on your Vagrant/laptop, or on an hdfs:// file system when you go to production.
- Transform your data in a series of steps. See generate_report()
- Dump to a single CSV file, presumably for some other team to process.
<br/><br/>
Here’s the code:
<br/><br/>
{% highlight python %}
    # export STDT=20150801
    # export ENDDT=20150831
    # rm -rf /data/results/$ENDDT*
    # ./bin/spark-submit --master local[4] /data/report1.py $STDT $ENDDT
    # cat /data/results/$ENDDT/report1/part-00000
    
    import sys, os
    import json
    from pyspark import SparkContext
    import argparse
    
    #
    # Parse command line arguments
    #
    def parse_args():
        parser = argparse.ArgumentParser(description='Create sample report')
        parser.add_argument('start', type=int, help='INTEGER representation of start date (20160208)')
        parser.add_argument('end', type=int, help='INTEGER representation of end date (20160208)')
        return parser.parse_args()
    
    #
    # Initialize spark
    #
    def init_spark(name):
        # Load up the file and split it into lines.
        return SparkContext(appName=name)
    
    #
    # This function is run on every line in the file to return an array
    #
    #    {
    #        "payload": {
    #            "instance_type": "GP-Large",
    #            "tenant_id": "a336b8bc3f694b91b4cadx2g879ceb3dbs0",
    #            "instance_id": "cbcfd438-c4a5-498f-9exxz67-be220f550024",
    #        },
    #        "timestamp": "2015-08-06 01:00:24.706597",
    #        "event_type": "compute.instance.exists",
    #    }
    #
    # Event types
    #        "compute.instance.create.start",   # being built
    #        "compute.instance.create.end",     # construction is over.  start billing.
    #        "compute.instance.update",         # still alive.
    #        "compute.instance.exists",         # lots of these (every day)
    #        "compute.instance.delete.start",   # started the delete process.  stop billing.
    #        "compute.instance.delete.end"      # deletion has stopped.
    #
    # Returns:
    #    tenant_id:instance_id                                                     created                        alive   deleted
    #   [u'4cff1d9a5a10484ff81e10471214a71d9d:c121kf16f-2efa-4a03-885b-b6f597eae8ec', None,                          True,   None]
    #   [u'4cff1d9a5a10484ff81e10471214a71d9d:c121kf16f-2efa-4a03-885b-b6f597eae8ec', u'2015-09-01 00:01:10.981178', None,   None]
    #
    
    def parse_event(line):
        e = json.loads(line)
        payload = e['payload']
        if 'tenant_id' not in payload:
            return None
        tenant_id = payload['tenant_id']
        instance_id = payload['instance_id']
        instance_type = payload['instance_type']
    
        # fill in the correct column for what is going on for this event.  Only keep it
        # if it is one of the events we care about.
        timestamp = e['timestamp']
        event_type = e['event_type']
        created = deleted = alive = "None"
    
        if event_type == "compute.instance.create.end":
            created = timestamp
        elif event_type == "compute.instance.delete.start":
            deleted = timestamp
        elif event_type in ["compute.instance.update", "compute.instance.exists"]:
            alive = "True"
    
        # dump out a line formatted for next round of processing, otherwise, drop it ("None")
        if created != "None" or deleted != "None" or alive != "None":
            return ",".join(["%s:%s" % (tenant_id, instance_id), created, alive, deleted])
        else:
            return "None"
    
    #
    # you should have three rows for each instance.  Combine them into one interesting row.
    # input:
    # db735e5cae4e4ae2a98a5d4baeaf40a12,625cd73gf-3e89-4060-8456-e57af759e93c,None,True,None
    # db735e5cae4e4ae2a98a5d4baeaf40a12,625cd73gf-3e89-4060-8456-e57af759e93c,None,None,2015-09-09 03:32:30.514042
    # db735e5cae4e4ae2a98a5d4baeaf40a12,625cd73gf-3e89-4060-8456-e57af759e93c,2015-09-09 03:29:48.882906,None,None
    #
    # output
    # db735e5cae4e4ae2a98a5d4beaaf40a12,625cd73gf-3e89-4060-8456-e57af759e93c,2015-09-09 03:29:48.882906,True,2015-09-09 03:32:30.514042
    #
    def combine_events(x, y):
        #    if x is not None and y is not None:
        x = x.split(",")
        y = y.split(",")
        return ",".join(
            [x[0],
             x[1] if x[1] != "None" else y[1],
             x[2] if x[2] != "None" else y[2],
             x[3] if x[3] != "None" else y[3]])
    
    #
    # We are given a string for each row from distinct().
    #    Pull the key out of the row so we can keep mapping
    #
    def extract_key(x):
        if x is not None:
            arr = x.split(",")
            k = arr[0]
            return (k, x)
    
    #
    # Generate the report.
    #
    def generate_report(sc, start, end):
    
        # Starting with a date stamp, iterate over all the files we have and create a list
        base_dir = "/data/raw"
        files = []
        for i in range(0, end-start):
            dt = start + i
            files.append("%s/%s_data.txt" % (base_dir, dt))
    
        # run the list of files into one RDD
        lines = sc.textFile(",".join(files))
    
        # map(parse_event):  parse everything into columns from JSON
        # filter(...):       throw out empty rows
        # distinct:          leave only one row for identical update/exists -> True
        #                    now have a row for start, stop, and alive
        # map(extract_key):  distinct leaves you with list of strings.  get the keys back out.
        # filter(...):       filter out empty rows again
        # reduceByKey():     roll all 3 rows into one row
        # values():          drop the keys, since the keys are in the value to
        # collect():         pull all partitions together into a regular python list
        events = lines.map(parse_event)\
                      .filter(lambda vals: vals != "None")\
                      .distinct()\
                      .map(extract_key)\
                      .filter(lambda vals: vals is not None)\
                      .reduceByKey(combine_events)\
                      .values()\
                      .collect()
    
        # dump out csv file for delivery to consumer.
        with open("/data/results/%s.report.csv" % end, "w") as f:
            for i in events:
                f.write(i)
                f.write("\n")
    
    
    if __name__ == "__main__":
        args = parse_args()
    
        sc = init_spark("test_report")
    
        generate_report(sc, args.start, args.end)
    
        sc.stop()
{% endhighlight %}
<br/><br/>
Ahhh!!!! Thats a lot of code. Relax. Most of it is boilerplate and comments.
<br/><br/>
This is the part that matters:
<br/><br/>
{% highlight python %}
    # run the list of files into one RDD
        lines = sc.textFile(",".join(files))
    
        # map(parse_event):  parse everything into columns from JSON
        # filter(...):       throw out empty rows
        # distinct:          leave only one row for identical update/exists -> True
        #                    now have a row for start, stop, and alive
        # map(extract_key):  distinct leaves you with list of strings.  get the keys back out.
        # filter(...):       filter out empty rows again
        # reduceByKey():     roll all 3 rows into one row
        # values():          drop the keys, since the keys are in the value to
        # collect():         pull all partitions together into a regular python list
        events = lines.map(parse_event)\
                      .filter(lambda vals: vals != "None")\
                      .distinct()\
                      .map(extract_key)\
                      .filter(lambda vals: vals is not None)\
                      .reduceByKey(combine_events)\
                      .values()\
                      .collect()
{% endhighlight %}
<br/><br/>
All this does is twist a bunch of JSON around into a flattened model that I can dump to CSV. Each step spins up parallel 
tasks, ostensibly on a cluster if you were in production, to process the intermediate parts of your data. This takes the 
place of a bunch of loop and intermediate file handling logic that would clutter up your code.
<br/><br/>
## Run It
From your vagrant instance:
<br/><br/>
{% highlight shell %}
~ ./bin/spark-submit \
           --master local[4] \ 
           /data/report1.py 20160303 20160331
{% endhighlight %}
<br/><br/>
When it’s done, go pick up your CSV file to send wherever. For my needs it processed hundreds of MB of JSON file down to a 
CSV file in about 3 minutes on my laptop. When it’s an order of magnitude bigger, it won’t be on my laptop anymore, but the 
code will still work identically. Write once and run at any scale.
<br/><br/>
## The Net of It
Pros:

- Scalable. If you have a true big data problem, this is a great tool. If you don’t, just stuff it in Postgres.
- Functional. Functional programming is a good thing, easy to test, easy to reason about, relatively uncluttered.
- Mixed mode. Though pure functional programming is a good goal, data processing is messy. Tools like this allow you to move freely in and out of standard python to process data in the most applicable tool for the job.

Cons:
- Conceptually difficult. If you’ve not wrapped your head around functional programming, the concepts will be a challenge. You should do it anyway. Makes you a better pro.
- Overkill for small stuff and most stuff is small stuff. If you don’t have a TB of data or more, you should not go down this path. Life is too short.
- Esoteric skill set. This is currently difficult to hire for. This will improve but if you making hiring decisions, be prepared for a higher bill rate for now. On the other hand, if you’re a developer…

