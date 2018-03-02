# Putting the CI in the Splunks

### Forewarning: I am very old and very new to the CI/CD world so if I've done something horribly wrong here, please feel free to tell me. 

Alright, we build A LOT of Splunk apps and they get punted back for a variety of reasons some AppInspect handles and tells you about and some it does not. I decided to try to minimize/shorten the process of us building a good app, that follows their rules and pretty much the rules of the world in general. I would not call this "fully continuous integration" or anything like that because it doesn't really push anywhere but it is definitely some automation we need. 

The core of this automation is GitLab's built-in continuous integration module (CI/CD) and it's very straight forward. First you create a configuration file (.gitlab-ci.yml) and you configured it, once it's configured A LOT of magic happens in the background and you get a pass/fail on your code. Sounds nifty right? Cool, let's look at our configuration file. 

```
image: python:2.7
stages:
    - lint
    - sectest
    - splunktest

delint:
    stage: lint
    script:
    - pip install pylint
    - pylint ${CI_PROJECT_DIR}/bin/*.py
    tags: 
        - docker

sectests:
    stage: sectest
    script:
    - pip install bandit
    - bandit -r ${CI_PROJECT_DIR}/bin
    
    tags: 
        - docker

splunktest:
    stage: splunktest
    script:
    - wget --output-document splunk-appinspect.tar.gz http://dev.splunk.com/goto/appinspectdownload
    - pip install splunk-appinspect.tar.gz 
    - mkdir /tmp/${CI_PROJECT_NAME}
    - git archive --format tar HEAD | tar -xC /tmp/${CI_PROJECT_NAME}
    - splunk-appinspect inspect /tmp/${CI_PROJECT_NAME} --mode precert
    tags: 
        - docker
```



First line is the image line which tells the CI system what kind of thing we're testing, since Splunk is on the cutting edge we're using python:2.7. The next is the "stages" configuration, this is just telling the CI system in what order you want to execute these tests. In a normal piece of code it would be something like "test, build, deploy" but in ours we're just running three different kinds of tests (which I'll explain shortly) so we use "delint, sectests, splunktest" as our stages. Let's dig a little deeper into each stage, shall we? Okay good: 

Delint: 
    This stage simply calls pylint which makes every python developer everywhere cringe because of it's exacting standards. This is the exact reason I love it and we're using it. This is test one, fail this and your app goes no further. 

SecTests: 
    This test is running a neat security checking tool called "Bandit" which will help you with security issues in your code. Same as above, fail and it fails hard. We're a security company so we like security stuff. 

Splunktest: 
    Finally we get to the actual Splunk appinspect testing. This stages downloads and installs the latest appinspect and runs it in precert mode against your code. It will only fail on failures so manual checks are still on your but I've found that the first two stages really take care of a lot of those manual checks. 
    
The reason we use the .gitattributes files is so we can have the git archive command ignore . files, which Splunk's Appinspect hates. That is all that is going on there so if you have other . files in your repository, please ignore them there. 

One final note, you'll notice tags: - docker at the end of each stage, this just tells the CI system to use a docker container to test the code. You'll need a gitlab-runner capable of that for all of this to work. That's about it. Enjoy. 





This is pretty straight forward. 

Grab the .gitlab-ci.yml.prod and rename it to .gilab-ci.yml in the root
of your repository. 

Grab the .gitattributes files and put it in the root of your respository. 

Check out the CI/CD - jobs page in your project or wait for the pass/fail email

If your app contains no python simply remove the delint and sectests stages. 

Enjoy/Curse me/ whatever. 
