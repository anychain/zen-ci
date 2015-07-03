* Gerrit and Jenkins url

  - Gerrit:  http://119.254.102.76:10001
  - Jenkins: http://119.254.102.76:10002

* Login into gerrit 

  - http://119.254.102.76:10001
  - default username is your company email

* Get the project you want to clone from gerrit ui

  - project -> list -> select your project and select the ssh uil for the git repo, our project list is:
    
    + zen-ci
    + zen-common
    + zen-api
    + zen-web
    + zen-ws
    + zen-worker
    + zen-deployer

  - git clone ssh://frank@119.254.102.76:29418/esse-io/zen-common


* How to commit change? 

  - apt-get install git-review
  - git clone ssh://frank@119.254.102.76:29418/esse-io/1future-temp-repo
  - gitdir=$(git rev-parse --git-dir); scp -p -P 29418 frank@119.254.102.76:hooks/commit-msg ${gitdir}/hooks/
  - cd 1future-temp-repo
  - git remote add gerrit ssh://frank@119.254.102.76:29418/esse-io/1future-temp-repo
  - #Make you code change here
  - git add -A; git commit -a -m 'new commit';git review

* Review and approve your change 

  - Go to gerrit page, All -> open, you will see the commit you just submitted
  - check whether it has passed the jenkins user's automation test
  - If yes, the verified tag will be marked as +1. 
  - Then go to the Patch set and Click Review, +2. 
  - Then click Submit Patch Set, the patch set will be merged into github automatically. 
  - Double check whether this change set has been merged successfully.