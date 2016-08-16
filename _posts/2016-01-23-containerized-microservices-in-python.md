---
layout: post
title: "Containerized Microservices in Python"
description: "Create small services using Python, Docker and other cool stuff."
date: 2016-03-01 12:00:00 -0400
category: python
tags: [python, docker]
---

Lets throw around some buzzwords and create a containerized, linearly scalable micro service in Python. 
There will be containers, proxies and full test coverage.

Technologies we’ll use:
- Python — All components are build in Python
- Nginx — HTTP Proxy to sitting in front of the Python Service
- Flask — Web application framework suitable for creating micro services in Python.
- Gunicorn — Pre-fork Python application server to efficiently serve the Flask application in production.
- Docker — Container technology to create simple and consistent deployments.
- Nose — Unit testing framework.

## Why Micro Services?
You and your buddies built a web app over a weekend at a hack-a-thon and it’s getting some traction. You’re having 
trouble scaling and keeping it running. You can’t perform changes without outages. This is the proverbial “good problem 
to have”, unless of course it is on you to fix it. One solution to this problem is to break up your monolithic application 
into smaller pieces. Not just modularize the code to make it easier to maintain, but actually break functionality out 
into its own service that runs, potentially, on separate locations in your infrastructure. These are micro services. The 
benefit for you, suddenly-successful-hack-a-thon-guy, is that now you can work on different pieces of your 
infrastructure separately, on separate timelines, separate teams, different languages, whatever, to ensure 
environmental integrity.


## The Code
It’s all up on [github](https://github.com/anonymoose/fibonacci).

## The Service
Assuming you know what a Fibonacci Sequence is. The service exposes a REST api, that when supplied with a positive count 
integer, returns a sequence of the first $count Fibonacci numbers. Simple enough.

## Getting Started with the API
<br/><br/>
Prerequisites

<br/><br/>
Start by creating a new directory called “fibonacci”. We’ll call this $DEV_HOME here after. Then create a directory under 
that for your Python code, “fibonacci_api”. fibonacci_api contains the python code for your Flask application.

<br/><br/>
You will need a working Python install on your local machine. The article is not to explain Python ecosystem, so I will 
assume you are well versed in that. Use virtualenv to create a local, disposable environment. Use pip to install some 
prerequisite libraries to your virtual environment.

<br/><br/>
{% highlight bash %}
$ cd $DEV_HOME       # your top level 'fibonacci' directory.
                     # creates "./python" dir
$ virtualenv --no-site-packages python   
                     # use  your local python.  not system.
$ source python/bin/activate
                     # install your dependencies.
$ pip install Flask gunicorn Sphinx coverage nose
{% endhighlight %}

<br/><br/>
## Some Code
There are lots of articles out there on creating a simple Flask application, but here’s a simple start that actually 
serves up Fibonacci results. Let’s call this “main.py”

<br/><br/>
{% highlight python %}
    from common import notify_error, log_api_call
    from fibonacci import fibonacci_calc
    from flask import Flask
    from flask import jsonify
    from flask import redirect
    from flask import request
    import logging
    from logging.handlers import RotatingFileHandler
    
    
    app = Flask("FibonacciAPI")
    
    HTTP_ERROR_CLIENT = 403
    HTTP_ERROR_SERVER = 500
    
    @app.route('/fibonacci/list', methods=['GET'])
    def fibonacci_list_api():
        if 'count' not in request.args or request.args['count'] in ("", None):
            return notify_error("ERR_NO_ARG:  'count' argument required to /fibonacci/api", HTTP_ERROR_CLIENT)
    
        try:
            count = long(request.args.get('count', ''))
        except:
            return notify_error("ERR_INVALID_TYPE:  'count' parameter must be an integer", HTTP_ERROR_CLIENT)
    
        if count < 0:
            return notify_error("ERR_OUT_OF_BOUNDS:  'count' parameger must be a postitive integer", HTTP_ERROR_CLIENT)
    
        try:
            return jsonify(answer=fibonacci_calc(count))
        except Exception as ex:
            return notify_error(ex, HTTP_ERROR_SERVER)
    
    def fibonacci_calc(count):
        assert isinstance(count, (int, long))
        assert count >= 0
    
        sequence = [0, 1]
        if count > 2:
            for f in range(2, count):
                parent = sequence[-1]
                grandparent = sequence[-2]
                sequence.append(grandparent + parent)
        return sequence[:count]
    
    if __name__ == '__main__':
        app.run()

{% endhighlight %}
<br/><br/>
This code takes a HTTP GET call, ensures the parameters are ok, then calculates the Fibonacci sequence requested in a 
non-recursive manner. If you go to the full Github project, you’ll see a lot of additional stuff, mostly around error 
handling, docs, etc, which I have redacted here for simplicity. Easy stuff.

<br/><br/>
Fire it up so you can debug it:
<br/><br/>
{% highlight bash %}
$ cd $DEV_HOME/fibonacci_api
$ python main.py
{% endhighlight %}

<br/><br/>
In another shell, use Curl to hit it.


<br/><br/>
{% highlight bash %}
$ curl http://localhost:5000/fibonacci/list?count=10
{
  "answer": [
    0,
    1,
    1,
    2,
    3,
    5,
    8,
    13,
    21,
    34
  ]
}
{% endhighlight %}

<br/><br/>
Money.

## Testing

As a responsible adult, you need some tests. Let’s create a file called main_test.py and use Nose to ensure we’re 
getting the right answers and everything is getting tested. Your full blown testing should include tests for positive 
interactions, negative interactions at the business logic layer, where the service is doing its thing, and at the Flask 
layer, where all the integration stuff happens. You know all this. Just nagging.

<br/><br/>
Here is an example, simplified from the full Github project’s testing, which shows how to test positive interactions with 
the business logic layer and the Flask layer. Again, this is shortened from my full tests, which test everything and are 
more modular.

<br/><br/>
{% highlight python %}
    from nose.tools import ok_, eq_
    from flask import json
    import main
    
    test_app = main.app.test_client()
    
    def check_fibonacci_calc_output_ok(expected_array):
        l = len(expected_array)
        out = fibonacci_calc(l)
        eq_(len(out), l, "Should be array of length %s" % l)
        eq_(out, expected_array, "Should be array %s" % l)
    
    
    def test_fibonacci_calc():
        check_fibonacci_calc_output_ok([0])
        check_fibonacci_calc_output_ok([0,1])
        check_fibonacci_calc_output_ok([0,1,1])
        check_fibonacci_calc_output_ok([0,1,1,2])
        check_fibonacci_calc_output_ok([0,1,1,2,3])
        check_fibonacci_calc_output_ok([0,1,1,2,3,5])
    
    
    STATUS_OK = 200
    STATUS_REDIR = 302
    STATUS_CLIENT_ERROR = 403
    
    def check_fibonacci_api_output_ok(test_app, expected_array):
        l = len(expected_array)
        resp = test_app.get('/fibonacci/list?count=%d' % l)
        out = json.loads(resp.data)
        eq_(resp.status_code, STATUS_OK, "Response code != %s" % STATUS_OK)
        eq_(out[u'answer'], expected_array, "/fibonacci/list returning wrong answer. Should be: %s" % expected_array)
    
    
    def test_fibonacci_api():
        check_fibonacci_api_output_ok(test_app, [0])
        check_fibonacci_api_output_ok(test_app, [0,1])
        check_fibonacci_api_output_ok(test_app, [0,1,1])
        check_fibonacci_api_output_ok(test_app, [0,1,1,2])
        check_fibonacci_api_output_ok(test_app, [0,1,1,2,3])
        check_fibonacci_api_output_ok(test_app, [0,1,1,2,3,5])
    

{% endhighlight %}

<br/><br/>
When I run the full version on the Github, using Nose’s code coverage facilities, I get this output:

<br/><br/>
{% highlight bash %}

$ nosetests --with-coverage --cover-html \
       --cover-package=fibonacci_api --cover-erase
.....

Name                         Stmts   Miss  Cover   Missing
----------------------------------------------------------
fibonacci_api.py                 0      0   100%   
fibonacci_api/common.py         16      2    88%   37-38
fibonacci_api/fibonacci.py      13      0   100%   
fibonacci_api/main.py           31      4    87%   23, 53-54, 75
----------------------------------------------------------
TOTAL                           60      6    90%   
----------------------------------------------------------
Ran 5 tests in 0.027s

{% endhighlight %}

<br/><br/>
90% coverage is not bad, given that on a project of this size, the amount of boilerplate is a higher percentage of the 
project than if the project were larger. When you dive into what is missed, it is things like unknown exception catching, 
and other trivial stuff. Your goal should be 100%, but use common sense before you miss a deadline, for example, testing 
something that the Flask guys already tested.

<br/><br/>
Now you have a tested core of your micro service. Moving on to getting ready for deployment.



<br/><br/>
## Containerization
If you have moved from a monolithic application to a micro services based application, you are embracing complexity to achieve greater reliability and maintainability. Containers are great for simplifying deployment management to combat the complexity that comes with a micro services based environment.

<br/><br/>
For our container system, we will use [Docker](http://www.docker.com), specifically [Docker Compose](https://www.docker.com/docker-compose). Docker Compose allows us to use Nginx as a front end to proxy multiple connections to Gunicorn app server which is running our Flask application. All this is tied into a container and suitable for deployment anywhere. It’s less Rube Goldberg than it sounds, I promise.

<br/><br/>
Detailed instructions for getting the full Github project up and running are [here](https://github.com/anonymoose/fibonacci/blob/master/fibonacci_api/docs/deployment.rst). I’ll hit some highlights in this article.

### Nginx
Nginx is a lightweight web server that is a great Swiss Army knife of the infrastructure world. For Python services, write a service, expose a port, and let Nginx worry about dealing with the clients.
<br/><br/>
To tell Docker to include it in our container, we need to configure it. First we tell Docker that we want Nginx installed and to use our config file. This is done through a file called Dockerfile in $DEV_HOME/Dockerfile.

<br/><br/>
{% highlight bash %}
FROM tutum/nginx                            # install nginx
RUN rm /etc/nginx/sites-enabled/default     # cleanup last run
ADD fibonacci.conf /etc/nginx/sites-enabled # use my file.

{% endhighlight %}


<br/><br/>
fibonacci.conf is an Nginx configuration file that knows how to talk to the Python service we already wrote on port 8080.


<br/><br/>
{% highlight json %}
server {
    listen 80;
    server_name fibonacci_api;
    charset utf-8;

    location / {
        proxy_pass http://fibonacci_api:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
{% endhighlight %}


<br/><br/>

This file basically tells Nginx to listen on port 80 and pass everything through to whatever is running inside this container on port 8080, returning the results to the user. Now we configure the Python service.

<br/><br/>
### Gunicorn and Flask Service Containerization
Python is notoriously single threaded. The preferred mechanism to accomplish parallelism in Python is to launch multiple processes so the OS can figure it out. Gunicorn handles this seamlessly, knows how to talk to Flask, and works great in Docker.
<br/><br/>
Once again, we need to create a Dockerfile. Docker folks were nice to us and made this common pattern a one-liner. Here is the full content of our Dockerfile at $DEV_HOME/fibonacci_api/Dockerfile

{% highlight bash %}
FROM python:2.7.11-onbuild
{% endhighlight %}

This calls a reusable module provided by Docker to do the following:
- Install python 2.7
- Pull in your source and put it in a conspicuous and runnable place.
- Install your requirements.txt file.

<br/><br/>
Note that we’ve not discussed requirements.txt. This is created by telling pip, your python package manager, to enumerate all the libraries and versions in your environment.

<br/><br/>
{% highlight bash %}
$ pip freeze > requirements.txt
$ head -3 requirements.txt
alabaster==0.7.7
Babel==2.2.0
coverage==4.0.3
{% endhighlight %}

<br/><br/>
If you installed as directed above, Flask, Gunicorn and the kitchen sink will be in this file.

<br/><br/>
## Container Composition

Docker can do some incredibly intricate things. I on the other hand like to keep things simple. So lets use Docker Compose to tie Nginx to our Gunicorn/Flask app and expose some ports.
<br/><br/>
In $DEV_HOME (your top level), create a file called docker-compose.yml. This file describes the various components you want Docker Compose to stuff into a single container. Provided you have all the ports lined up and everything else is aligned, Docker Compose will pull in Nginx, your source code, and run it, allowing you to hit it in the browser or Curl.

<br/><br/>

{% highlight yaml %}

fibonacci_api:
  restart: always
  build: ./fibonacci_api
  expose:
    - "8080"
  ports:
    - "8080:8080"
  volumes:
    - /usr/src/app/static
  env_file: .env
  command: /usr/local/bin/gunicorn -w 4 --bind :8080 main:app

proxy:
  restart: always
  build: ./proxy
  expose:
    - "80"
  ports:
    - "80:80"
  volumes_from:
    - fibonacci_api
  links:
    - fibonacci_api:fibonacci_api
{% endhighlight %}

<br/><br/>
This file, in English says, roughly: “The fibonacci_api component lives in $DEV_HOME/fibonacci_api, runs on port 8080, and you run it with /usr/local/bin/gnunicorn. Additionally, the proxy component lives in $DEV_HOME/proxy runs on port 80, and links to fibonacci_api to allow all the names to align”.

<br/><br/>
Detailed instructions on what commands to run to get this going are here. This is a slow process first time since Docker Compose is basically creating a fully functional Linux box running virtually on your laptop, out of components it downloads off the internet. I was told there would be flying cars by now, but this is pretty close.

<br/><br/>
## Wrapping Up
Micro services and containers are interesting technologies for combating complexity in huge environments. In my opinion, you 
should start with that monolith and only do all this if you need it. Just in case, you should keep this in your pocket for 
when your hack-a-thon project turns into that huge environment.
