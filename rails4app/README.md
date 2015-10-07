# Dockerized Rails 4.2 stack for Dev and Prod

**Goals:** 
  * Painless, *Fast* development that doesn't require you to rebuild the image frequently. 
  * Leverage Docker caching and a persistent bundler cache to speed bundle updates/installs.
  * Split up the docker files so builds go super fast, and you only rebuild what you care about. 
  * Same image can be used in dev/test/stage/prod
  * Choice of Rails Application Servers- Unicorn or Thin optionally sitting behind nginx.
  * Ability to run and capture logs from multiple processes such as nginx, unicorn, cron.
  * Be able to easily use your production app servers in development instead of webrick.
  * Make it trivially easy to set up a full technology stack on a laptop including 
    * Redis
    * Elastic Search
    * MySQL (sorry postgres fans)
    * LogSpout
    * Sidekiq Background Worker


Tested on Docker 1.8 with docker-machine and OS X. It should work on boot2docker as well. 

Ubuntu roots- which makes it fatter than Debian, but I'm ok with that for now. 

Supervisord is used over foreman and forego because I had problems reliably piping logs to stdout with foreman. (gotta be 12factor, right?)


#### Non-Rails Files added to the Project for Deployment and Development


**docker-compose.yml** - This Docker Compose file will spin up a full development stack on your laptop including all of the common services you'd expect from an average rails app. 
Just type `docker-compose up` and you should get your stack. If you don't need a service, comment it out. To update your bundle taking advantage of the persistent cache: `docker run --name Rails4BundlerUpdater  rails4app_app bundle `


**Dockerfile** - This is the third/final in a chain of base images. The goal here is absolutely minimal time to build this image so development is painless. We go to special trouble to 
cache the bunlder files both using docker's cache and bundler's cache.  There are also some important environment variables, such as the secret key that should be changed in here. In prod, 
pass the secret key in at run time, don't store it in the source code- not even in secrets.yml. 

**supervisord.conf** - Supervisor facilitates running multiple apps in your docker container. In this case, nginx and our rails app server, but you could add something like cron if you like.  Make sure that your app server is outputting all logs to stdout so that the logspout container can send it off to a remote syslog server in prod, and that you get your debug info in dev.  

**nginx.vhost** - It often makes sense to deploy vhost-specific configurations, such as all the hostnames, 301's, etc. with the application itself. This allows for a clean seperation of core nginx infrastructure configs and app specific settings that the product team may care to change without bugging the infrastructure team (not very DevOpsy- I know).


#### Rails Configuration to make this work: 
**database.yml** - I'm not a proponent of using a different DB in dev than I do in prod, so we use MySQL across the board and pass params in as environment vars in the Dockerfile. 

**Gemfile** - Add MySQL2 Gem. There is a bug with the latest version of rails and the latest version of this gem, so I froze the version to the latest known working configuration. 
Add thin gem. This is the app server. I previously loved Unicorn, but couldn't get it to properly log to stdout. 

**unicorn.rb** - Configuration options including setting logging to stdout and socket locations for each RAILS_ENV if you choose to use the unicorn app server.  


#### Firing Up a Full Development Environment with Docker-Compose 

`docker-compose up app`  should do the trick! 

Then visit the docker host's IP address. If you're on a mac using boot2docker or docker-machine it should usually be: 

`http://192.168.59.100:3000/`

but you can do `docker-machine env default` to find out. 
Set up your shell with:  `eval "$(docker-machine env default)"` 

#### Getting the bare minimum database and app server running manually

If you don't want to use docker compose, here's the minimal manual config needed:

```
docker pull loveitlabs/rails4app
docker run 	--name db \
			-d \
			-p 3306:3306 \
			--env MYSQL_ROOT_PASSWORD=reallyterriblepassword \
			mysql 
docker run --name LoveItRailsDemo  \
			-p 80:80 \
			-p 3000:3000 \
			--link db:db \
           	--env RAILS_ENV=development \
           	--env DEV_DBHOST=db \
    		--env DEV_DBUSR=root  \
    		--env DEV_DBPASS=reallyterriblepassword \
    		loveitlabs/rails4app \
    		bash -c "cd /app && bundle exec rake db:create db:migrate db:seed && bundle exec rails server -b 0.0.0.0"
          
 ```


