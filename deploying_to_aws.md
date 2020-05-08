## Deploying to Amazon Web Services

### 1. Create and Login to EC2 instance

From AWS EC2 console, launch a new EC2 instance
- choose Ubuntu Server 18.04 LTS (HVM), SSD Volume Type - ami-0f7719e8b7ba25c61 (64-bit x86) / ami-02b6622eae4966dfd (64-bit Arm)
- server type: t2.micro
- Review and Launch
- Choose an existing key pair
- Launch Instance

Check the permissions on the SSH key-pair on your system that was used to launch above EC2 instance. This needs to be 400, i.e. read-only
```bash
$ ls -lah ~/.ssh/ | grep pem
# -r--------@   .....
```

SSH into EC2 instance
```bash
ssh -i ${PATH_TO_PEM_FILE} ubuntu@${AWS_INSTANCE_URL}
# PATH_TO_PEM_FILE should be ~/.ssh/YOUR_PEM_FILENAME.pem
# AWS_INSTANCE_URL should look like ec2-xxx-xxx-xxx-xxx.xxxxxxx.compute.amazonaws.com. To get this URL, right-click on your AWS VM instance, and click on 'Connect'
```

---

### 2. Install Ruby and dependencies

Update system libraries and install RVM
```bash
sudo apt-get update
sudo apt-get install -y curl gnupg build-essential

sudo apt-get install gnupg2 -y
sudo gpg2 --keyserver hkp://pool.sks-keyservers.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB

curl -sSL https://get.rvm.io | sudo bash -s stable
sudo usermod -a -G rvm ubuntu

# log out and back in again
exit
ssh -i ${PATH_TO_PEM_FILE} ubuntu@${AWS_INSTANCE_URL}

rvm -v
# rvm 1.29.10 (latest) by Michal Papis, Piotr Kuczynski, Wayne E. Seguin [https://rvm.io]
```

Install Ruby
```bash
rvm install ruby 2.5.3
# ...
# Install of ruby-2.5.3 - #complete
# Ruby was built without documentation, to build it run: rvm docs generate-ri
rvm use ruby 2.5.3 --default
```

Install Bundler
```bash
gem install bundler -no-rdoc -no-ri
# Fetching bundler-2.1.4.gem
# ...
# 1 gem installed
```

Install NodeJS
```bash
sudo apt-get install -y nodejs && sudo ln -sf /usr/bin/nodejs /usr/local/bin/node
```

---

### 3. Set up Nginx and Passenger

This section follows the steps in https://gorails.com/deploy/ubuntu/18.04#nginx
```bash
sudo apt-get install -y dirmngr gnupg
sudo apt-get install -y apt-transport-https ca-certificates

sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7
sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger bionic main > /etc/apt/sources.list.d/passenger.list'
sudo apt-get update

sudo apt-get install -y nginx-extras libnginx-mod-http-passenger
if [ ! -f /etc/nginx/modules-enabled/50-mod-http-passenger.conf ]; then sudo ln -s /usr/share/nginx/modules-available/mod-http-passenger.load /etc/nginx/modules-enabled/50-mod-http-passenger.conf ; fi
sudo ls /etc/nginx/conf.d/mod-http-passenger.conf

# get the path to the Ruby installation
which ruby
# /usr/local/rvm/rubies/ruby-2.5.3/bin/ruby

sudo vim /etc/nginx/conf.d/mod-http-passenger.conf
# change the passenger_ruby line to match the output of `which ruby`
passenger_ruby /usr/local/rvm/rubies/ruby-2.5.3/bin/ruby;

sudo service nginx start
```

In your AWS EC2 console, under Security Groups, add an inbound firewall rule for port 80 (HTTP)

Check and make sure NGINX is running by visiting your server's public IP address (find this in AWS EC2 console) in your web browser and you should be greeted with the "Welcome to NGINX" message

Create an Nginx configuration to point to your application folder
```bash
sudo vim /etc/nginx/sites-enabled/snorlaxapp

# enter vim
server {
  listen 80;
  listen [::]:80;

  server_name _;
  # modify to your target application directory
  root /home/ubuntu/snorlaxapp/public;

  passenger_enabled on;
  passenger_app_env production;

  location /cable {
    passenger_app_group_name myapp_websocket;
    passenger_force_max_concurrent_requests_per_process 0;
  }

  # Allow uploads up to 100MB in size
  client_max_body_size 100m;

  location ~ ^/(assets|packs) {
    expires max;
    gzip_static on;
  }
}
# exit vim
```

---

### 4. Download and deploy your application code

Add GitHub Deploy keys
- https://developer.github.com/v3/guides/managing-deploy-keys/#deploy-keys
- https://help.github.com/en/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent#generating-a-new-ssh-key

Download your code to the AWS VM instance in your working directory, e.g. `/home/ubuntu`
```bash
pwd
# /home/ubuntu

git clone git@github.com:hanmd82/snorlaxapp.git
bundle install
```

Generate a secret_key_base and store it in ENVVARs
```bash
RAILS_ENV=production rake secret
# GENERATED_SECRET_KEY

cat << EOF >> ~/.bash_profile
export SECRET_KEY_BASE=${GENERATED_SECRET_KEY}
EOF

source ~/.bash_profile
```

Create and Configure PostgreSQL Database
```bash
sudo apt-get install postgresql postgresql-contrib libpq-dev
sudo su - postgres
createuser --pwprompt ubuntu --createdb
# remember your PASSWORD for DB user ubuntu
exit
```

```bash
vi config/database.yml
# production:
#   <<: *default
#   url: <%= ENV['DATABASE_URL'] %>

cat << EOF >> ~/.bash_profile
export DATABASE_URL="postgres://ubuntu:PASSWORD@localhost/snorlaxapp"
EOF

RAILS_ENV=production rake db:create db:migrate
```

Verify that you can connect to your application DB
```bash
psql -h 127.0.0.1 -U ubuntu -d snorlaxapp
# password:...
psql> \l
psql> \q
```

Start Passenger server in standalone mode
```bash
RAILS_ENV=production rails assets:precompile
RAILS_ENV=production bundle exec passenger start
# ===============================================================================
# [ N 2020-05-07 06:45:34.2609 16819/T5 age/Cor/SecurityUpdateChecker.h:519 ]: Security update check: no update found (next check in 24 hours)
# I, [2020-05-07T06:45:35.400521 #16891]  INFO -- : [a2bd5dd0-aaa8-4213-ad4b-aa3171d6f04a] Started HEAD "/" for 127.0.0.1 at 2020-05-07 06:45:35 +0000
# I, [2020-05-07T06:45:35.406247 #16891]  INFO -- : [a2bd5dd0-aaa8-4213-ad4b-aa3171d6f04a] Processing by WelcomeController#index as HTML
# I, [2020-05-07T06:45:35.413478 #16891]  INFO -- : [a2bd5dd0-aaa8-4213-ad4b-aa3171d6f04a]   Rendering welcome/index.html.erb within layouts/application
# I, [2020-05-07T06:45:35.414569 #16891]  INFO -- : [a2bd5dd0-aaa8-4213-ad4b-aa3171d6f04a]   Rendered welcome/index.html.erb within layouts/application (0.9ms)
# I, [2020-05-07T06:45:35.416009 #16891]  INFO -- : [a2bd5dd0-aaa8-4213-ad4b-aa3171d6f04a] Completed 200 OK in 9ms (Views: 4.1ms)
# ...
```

Access your application at your server's public IP address (find this in AWS EC2 console)

---

Where to find logs
- Nginx (web server): `/var/log/nginx/access.log` - handle HTTP requests
- Passenger (application server): `${APP_DIR}/log/passenger.${PORT}.log` - allow Ruby apps (and frameworks like Rails) to speak HTTP
- Rails (application): `${APP_DIR}/log/production.log` - application logic (MVC), routing, DB access etc

Further Reading:
- https://www.phusionpassenger.com/library/walkthroughs/basics/ruby/fundamental_concepts.html#passenger-and-rails-server
- https://www.phusionpassenger.com/library/walkthroughs/basics/ruby/passenger_command.html
