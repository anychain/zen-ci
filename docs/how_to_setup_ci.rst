install docker on ubuntu
========================

sudo apt-get purge docker.io    
curl -s https://get.docker.io/ubuntu/ | sudo sh     
sudo apt-get update     
sudo apt-get install lxc-docker     

ldap setup
==========
docker command::

  * docker run --name='ldap' -p 389:389 -v /home/hanzf/slapd/ldap:/var/lib/ldap \
      -v /home/hanzf/slapd/config:/etc/ldap/slapd.d \
      -e LDAP_DOMAIN="esse.io" -e LDAP_ORGANISATION="esse.io" -e LDAP_ROOTPASS=3ss3.10 \
      -e SERVER_NAME="ldap" -d nickstenning/slapd

ldap command::
 
  yum install openldap-clients
  ldapadd -h LDAP_HOST  -D "cn=admin,dc=esse,dc=io"  -f LDIF_FILE  -w
  ldapsearch -x -b 'ou=people,dc=esse,dc=io'

gerrit setup
============

docker command::

    docker run -t -d -p 8080:8080 -p 29418:29418 -e AUTH_TYPE=LDAP \
    -e GERRIT_URL=http://119.254.102.76:10001 -e LDAP_SERVER=ldap://172.16.100.2 \
    -e LDAP_ACCOUNT_BASE="ou=people,dc=esse,dc=io" -e REPLICATE_USER=hanzf \
    -e REPLICATE_KEY=/home/gerrit2/.ssh/gerrit_key \
    -v /home/hanzf/gerrit:/home/gerrit2/gerrit \
    -v /home/hanzf/ssh_key:/home/gerrit2/.ssh gerrit

procedures::
   
   1) open 29418 in firewall
   2) make port mapping for 29418 in router
   3) create project with name like hanzf/testgerrit.git, must be the same as github
   4) generate key pair, one upload to hanzf/settings/ssh keys in github, one store in /home/gerrit2/.ssh/gerrit_key
   5) git review add remote git clone ssh://frank@119.254.102.76:29418/hanzf/testgerrit
   6) change to project folder, gitdir=$(git rev-parse --git-dir); scp -p -P 29418 frank@119.254.102.76:hooks/commit-msg ${gitdir}/hooks/
   7) git review, check code has been merge to both gerrit and github 
   8) Deny any direct push to refs/heads/ 
     Reference: Edit reference pattern refs/heads/*
     Push 
     Deny Administrators
     Deny Project Owners

jenkins setup
=============

1) make sure /home/hanzf/jenkins is writable by jenkins user
2) docker rm myjenkins; docker run -d -t --name myjenkins -p 9080:8080 -v /home/hanzf/jenkins/:/var/jenkins_home jenkins
3) port mapping router, from 10002 to 9080 
4) open 10002 in firewall
5) install git, gerrit plugin in jenkins
6) make sure jenkins user is registered on gerrit server
7) go to /home/hanzf/jenkins/.ssh/, ssh-keygen to generate key pair and keep private key in this folder, public key upload to gerrit for user jenkins
8) check connection from with ssh -i id_rsa jenkins@172.16.100.2 -p 29418, make sure the connection works
9) create gerrit server in jenkins, setup ssh key
10) test connection with gerrit, make sure it works with success result
11) add verified label to gerrit

  - git init
  - git remote add origin ssh://frank@ci:29418/All-Projects
  - git fetch origin refs/meta/config:refs/remotes/origin/meta/config
  - git checkout meta/config
  - make following changes::

      [label "Verified"]
             function = MaxWithBlock
             value = -1 Fails
             value =  0 No score
             value = +1 Verified
    
  - git commit -a
  - git push origin meta/config:meta/conf  

12) add jenkins to non-interactive users, and grant 'label verified permission to non-interactive group'
13) create jenkins job, and setup git repository. 
    
    Do forget to setup   Dynamic Trigger Configuration, else jobs will not be triggered automatically.
14) git commit -a -m 'add verified label'
15) git review

zuul configuration
==================

1) Install gearman-plugin on jenkins
2) start docker::

       docker run -d --name='zuul' -e GERRIT_SERVER=172.16.100.2 \
       -e GERRIT_URL=http://119.254.102.76:10001 \
       -e JENKINS_SSH_KEY=/home/jenkins/.ssh/id_rsa \
       -v /home/hanzf/jenkins/.ssh/:/home/jenkins/.ssh \
       -v /home/hanzf/zuul/layout/:/etc/zuul-layout \
       -p 4730:4730 -p 8880:8880 -p 8001:8001 -p 8443:8443 zuul

3) configure gearman-server::

       Manage Jenkins -> configure system -> Gearman Plugin Config -> 
       Input Gearman server host and port -> Test connection -> save

4) Test connection to gear man server
   
   Default gear man server is on zuul server with port 4730
   
5) modify layout.yaml, make sure the job can be trigged successfully

    Modify layout.yaml like below::

        projects:
        - name: hanzf/testgerrit
          template:
            - name: ci-jobs
    
    If layout.yaml is changed, you must reconnect gearman-plugin with zuul gearman server

5) Must add a jenkins slave node for zuul

6) Setup the corresponding Jenkins job on jenkins

   Later should change this use jenkins-job-builer to automatically generate jenkins job from configurations.

6) How to test gearman server: 
   
   telnet 172.16.100.2 4730, then run command 'workers' and 'status' to check gear man status

setup gitlab
============

0)　link for docker gitlab

   https://github.com/sameersbn/docker-gitlab

1) install postgresql::
   
   docker run --name=gitlabpg -d --env='DB_NAME=gitlab' --env='DB_USER=gitlab' \
   --env='DB_PASS=3ss3.10' --volume=/home/hanzf/postgresql:/var/lib/postgresql postgres:9.4.4

2) install redis::

   docker run --name=gitlabredis -d \
   --volume=/home/hanzf/redis:/var/lib/redis \
   sameersbn/redis:latest

3) install gitlab::

   docker run --name='gitlab' -d \
     --link=gitlabpg:postgresql --link=gitlabredis:redisio \
     --publish=10005:22 --publish=10004:80 \
     --env='GITLAB_PORT=10004' --env='GITLAB_SSH_PORT=10005' \
     --volume=/home/hanzf/gitlab:/home/git/data \
   sameersbn/gitlab:7.12.1

how to manually change gerrit db
================================

* stop gerrit server

  /home/gerrit2/gerrit# ./bin/gerrit.sh stop

* connect to h2 db of gerrit

  /home/gerrit2/gerrit# java -jar bin/gerrit.war gsql

* use h2 command to edit h2 db

  like \d, \q, and sql 