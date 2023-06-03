### ref:   https://github.com/ojoimmanuel/ansible-config-1.git === https://github.com/ojoimmanuel/php-todo.git

=================================================================  

#### video file for review: `https://drive.google.com/file/d/1z9YbnoP0pnh3gj0NaJlFBKwAtJTInr2y/view?usp=sharing`  


## SIMULATING A TYPICAL CI/CD PIPELINE FOR A PHP BASED APPLICATION
As part of the ongoing infrastructure development with Ansible started from Project 11, you will be tasked to create a pipeline that simulates continuous integration and delivery. Target end to end CI/CD pipeline is represented by the diagram below. It is important to know that both Tooling and TODO Web Applications are based on an interpreted (scripting) language (PHP). It means, it can be deployed directly onto a server and will work without compiling the code to a machine language.

The problem with that approach is, it would be difficult to package and version the software for different releases. And so, in this project, we will be using a different approach for releases, rather than downloading directly from git, we will be using Ansible uri module.

![bk](./images/00-bk-dgm.PNG)

### Set Up
This project is partly a continuation of your Ansible work, so simply add and subtract based on the new setup in this project. It will require a lot of servers to simulate all the different environments from `dev/ci` all the way to `production`. This will be quite a lot of servers altogether (But you don’t have to create them all at once. Only create servers required for an environment you are working with at the moment. For example, when doing deployments for development, do not create servers for integration, pentest, or production yet).

Try to utilize your AWS free tier as much as you can, you can also register a new account if you have exhausted the current one. Alternatively, you can use Google Cloud (GCP) to rent virtual machines from this cloud service provider – you can get $300 credit here or here `https://www.startups.com/products/benefits/googlecloud` (NOTE: Please read instructions carefully to get your credits)

NOTE: This is still NOT a cloud-focus project yet. AWS cloud end to end project begins from project-15. Therefore, do not worry about advanced AWS or GCP configuration. All we need here is virtual machines that can be accessed over SSH.

To minimize the cost of cloud servers, you don not have to create all the servers at once, simply spin up a minimal server set up as you progress through the project implementation and have reached a need for more.

To get started, we will focus on these environments initially.

- Ci
- Dev
- Pentest  

Both `SIT` – For System Integration Testing and `UAT` – User Acceptance Testing do not require a lot of extra installation or configuration. They are basically the webservers holding our applications. But `Pentest` – For Penetration testing is where we will conduct security related tests, so some other tools and specific configurations will be needed. In some cases, it will also be used for `Performance and Load` testing. Otherwise, that can also be a separate environment on its own. It all depends on decisions made by the company and the team running the show.

What we want to achieve, is having Nginx to serve as a reverse proxy for our sites and tools. Each environment setup is represented in the below table and diagrams.

![table](./images/03-table.PNG)

### CI-Environment
![ci](./images/01-ci.PNG)

- spin up ec2
- install git and clone a repo
- install jenkins (as root `sudo su`)

### Other Environments from Lower To Higher
![other](./images/02-other.PNG)

### DNS requirements
Make DNS entries to create a subdomain for each environment. Assuming your main domain is darey.io

You should have a subdomains list like this:

Server	Domain
Jenkins	https://ci.infradev.darey.io
Sonarqube	https://sonar.infradev.darey.io
Artifactory	https://artifacts.infradev.darey.io
Production Tooling	https://tooling.darey.io
Pre-Prod Tooling	https://tooling.preprod.darey.io
Pentest Tooling	https://tooling.pentest.darey.io
UAT Tooling	https://tooling.uat.darey.io
SIT Tooling	https://tooling.sit.darey.io
Dev Tooling	https://tooling.dev.darey.io
Production TODO-WebApp	https://todo.darey.io
Pre-Prod TODO-WebApp	https://todo.preprod.darey.io
Pentest TODO-WebApp	https://todo.pentest.darey.io
UAT TODO-WebApp	https://todo.uat.darey.io
SIT TODO-WebApp	https://todo.sit.darey.io
Dev TODO-WebApp	https://todo.dev.darey.io  

Ansible Inventory should look like this
├── ci
├── dev
├── pentest
├── pre-prod
├── prod
├── sit
└── uat  

ci inventory file
```
[jenkins]
<Jenkins-Private-IP-Address>

[nginx]
<Nginx-Private-IP-Address>

[sonarqube]
<SonarQube-Private-IP-Address>

[artifact_repository]
<Artifact_repository-Private-IP-Address>
```  

dev Inventory file
```
[tooling]
<Tooling-Web-Server-Private-IP-Address>

[todo]
<Todo-Web-Server-Private-IP-Address>

[nginx]
<Nginx-Private-IP-Address>

[db:vars]
ansible_user=ec2-user
ansible_python_interpreter=/usr/bin/python

[db]
<DB-Server-Private-IP-Address>
```
pentest inventory file
```
[pentest:children]
pentest-todo
pentest-tooling

[pentest-todo]
<Pentest-for-Todo-Private-IP-Address>

[pentest-tooling]
<Pentest-for-Tooling-Private-IP-Address>
```

**Observations:**

1. You will notice that in the pentest inventory file, we have introduced a new concept `pentest:children` This is because, we want to have a group called `pentest` which covers Ansible execution against both `pentest-todo` and `pentest-tooling` simultaneously. But at the same time, we want the flexibility to run specific Ansible tasks against an individual group.  

2. The `db` group has a slightly different configuration. It uses a RedHat/Centos Linux distro. Others are based on Ubuntu (in this case user is `ubuntu`). Therefore, the user required for connectivity and path to python interpreter are different. If all your environment is based on Ubuntu, you may not need this kind of set up. Totally up to you how you want to do this. Whatever works for you is absolutely fine in this scenario.  

This makes us to introduce another Ansible concept called `group_vars`. With group vars, we can declare and set variables for each group of servers created in the inventory file.

For example, If there are variables we need to be common between both `pentest-todo` and `pentest-tooling`, rather than setting these variables in many places, we can simply use the `group_vars` for `pentest`. Since in the inventory file it has been created as `pentest:children` Ansible recognizes this and simply applies that variable to both children.  

## ANSIBLE ROLES FOR CI ENVIRONMENT
Now go ahead and Add two more roles to ansible:

1. SonarQube (Scroll down to the Sonarqube section to see instructions on how to set up and configure SonarQube manually)
2. Artifactory

### Why do we need SonarQube?
SonarQube is an open-source platform developed by SonarSource for continuous inspection of code quality, it is used to perform automatic reviews with static analysis of code to detect bugs, code smells, and security vulnerabilities. Watch a short description here. There is a lot more hands on work ahead with SonarQube and Jenkins. So, the purpose of SonarQube will be clearer to you very soon.

### Why do we need Artifactory?
Artifactory is a product by JFrog that serves as a binary repository manager. The binary repository is a natural extension to the source code repository, in that the outcome of your build process is stored. It can be used for certain other automation, but we will it strictly to manage our build artifacts.

Watch a short description here Focus more on the first 10.08 mins

### Configuring Ansible For Jenkins Deployment
In previous projects, you have been launching Ansible commands manually from a CLI. Now, with Jenkins, we will start running Ansible from Jenkins UI.

To do this,

1. Navigate to Jenkins URL

2. Install & Open Blue Ocean Jenkins Plugin (install `strict crumb issuer` if error)

3. Create a new pipeline
![pipeline](./images/04-new-pipeline.PNG)

4. Select GitHub



5. Connect Jenkins with GitHub


6. Login to GitHub & Generate an Access Tokenhttps://www.darey.io/wp-content/uploads/2021/07/Jenkins-Create-Access-Token-To-Github.png



7. Copy Access Token


8. Paste the token and connect


9. Create a new Pipeline


At this point you may not have a Jenkinsfile in the Ansible repository, so Blue Ocean will attempt to give you some guidance to create one. But we do not need that. We will rather create one ourselves. So, click on Administration to exit the Blue Ocean console.



Here is our newly created pipeline. It takes the name of your GitHub repository.

![pipelineb](./images/05-blue-pipieline.PNG)


### Let us create our Jenkinsfile
Inside the Ansible project, create a new directory deploy and start a new file Jenkinsfile inside the directory.

![jenkinsfile](./images/06-jenkins-file.PNG)

Add the code snippet below to start building the `Jenkinsfile` gradually. This pipeline currently has just one stage called `Build` and the only thing we are doing is using the `shell script` module to echo `Building Stage`

```
pipeline {
    agent any

  stages {
    stage('Build') {
      steps {
        script {
          sh 'echo "Building Stage"'
        }
      }
    }
    }
}
```  

Now go back into the Ansible pipeline in Jenkins, and select configure



Scroll down to `Build Configuration` section and specify the location of the Jenkinsfile at `deploy/Jenkinsfile`

![jejnkin](./images/07-deploy-jenkin.PNG)

Back to the pipeline again, this time click "Build now"



This will trigger a build and you will be able to see the effect of our basic Jenkinsfile configuration by going through the console output of the build.

To really appreciate and feel the difference of Cloud Blue UI, it is recommended to try triggering the build again from Blue Ocean interface.

1. Click on Blue Ocean

2. Select your project
3. Click on the play button against the branch

Notice that this pipeline is a multibranch one. This means, if there were more than one branch in GitHub, Jenkins would have scanned the repository to discover them all and we would have been able to trigger a build for each branch.

Let us see this in action.

1. Create a new git branch and name it `feature/jenkinspipeline-stages`  

2. Currently we only have the `Build` stage. Let us add another stage called `Test`. Paste the code snippet below and push the new changes to GitHub.  
```
   pipeline {
    agent any

  stages {
    stage('Build') {
      steps {
        script {
          sh 'echo "Building Stage"'
        }
      }
    }

    stage('Test') {
      steps {
        script {
          sh 'echo "Testing Stage"'
        }
      }
    }
    }
}
```

4. To make your new branch show up in Jenkins, we need to tell Jenkins to scan the repository.

41. Click on the "Administration" button


42. Navigate to the Ansible project and click on "Scan repository now"


43. Refresh the page and both branches will start building automatically. You can go into Blue Ocean and see both branches there too.


44. In Blue Ocean, you can now see how the Jenkinsfile has caused a new step in the pipeline launch build for the new branch.

![feature](./images/08-freature-brnch.PNG)

A QUICK TASK FOR YOU!
1. Create a pull request to merge the latest code into the main branch
2. After merging the PR, go back into your terminal and switch into the main branch.
3. Pull the latest change.
4. Create a new branch, add more stages into the Jenkins file to simulate below phases. (Just add an echo command like we have in build and test stages)
   1. Package 
   2. Deploy 
   3. Clean up
5. Verify in Blue Ocean that all the stages are working, then merge your feature branch to the main branch
6. Eventually, your main branch should have a successful pipeline like this in blue ocean
![simple](./images/09-simple-task.PNG)  

## RUNNING ANSIBLE PLAYBOOK FROM JENKINS
Now that you have a broad overview of a typical Jenkins pipeline. Let us get the actual Ansible deployment to work by:

1. Installing Ansible on Jenkins
2. Installing Ansible plugin in Jenkins UI
3. Creating Jenkinsfile from scratch. (Delete all you currently have in there and start all over to get Ansible to run successfully)  

You can watch a 10 minutes video here `https://www.youtube.com/watch?v=PRpEbFZi7nI` to guide you through the entire setup

**Note**: Ensure that Ansible runs against the Dev environment successfully.

**Possible errors to watch out for:**

1. Ensure that the git module in Jenkinsfile is checking out SCM to main branch instead of master (GitHub has discontinued the use of Master due to Black Lives Matter. You can read more here)

2. Jenkins needs to export the `ANSIBLE_CONFIG` environment variable. You can put the `.ansible.cfg` file alongside `Jenkinsfile` in the `deploy` directory. This way, anyone can easily identify that everything in there relates to deployment. Then, using the Pipeline Syntax tool in Ansible, generate the syntax to create environment variables to set.

https://wiki.jenkins.io/display/JENKINS/Building+a+software+project



**Possible issues to watch out for when you implement this**

1. Remember that `ansible.cfg` must be exported to environment variable so that Ansible knows where to find `Roles`. But because you will possibly run Jenkins from different git branches, the location of Ansible roles will change. Therefore, you must handle this dynamically. You can use `Linux Stream Editor sed` to update the section `roles_path` each time there is an execution. You may not have this issue if you run only from the **main** branch.

2. If you push new changes to `Git` so that Jenkins failure can be fixed. You might observe that your change may sometimes have no effect. Even though your change is the actual fix required. This can be because Jenkins did not download the latest code from GitHub. Ensure that you start the `Jenkinsfile` with a **clean up** step to always delete the previous workspace before running a new one. Sometimes you might need to login to the Jenkins Linux server to verify the files in the workspace to confirm that what you are actually expecting is there. Otherwise, you can spend hours trying to figure out why Jenkins is still failing, when you have pushed up possible changes to fix the error.

3. Another possible reason for Jenkins failure sometimes, is because you have indicated in the `Jenkinsfile` to check out the `main` git branch, and you are running a pipeline from another branch. So, always verify by logging onto the Jenkins box to check the workspace, and run `git branch` command to confirm that the branch you are expecting is there.  

#### important steps
*************************************
- add ssh key in jenkins
![ssh](./images/10-add-ssh-credential.PNG)  

- update ansible path
![ansible path](./images/11-update-ansible-path.PNG)  

- configure pipeline syntax in order to generate playbook command
![syntax1](./images/12-conf-pipeline-syntaxA.PNG)
![syntax2](./images/13-conf-pipeline-syntaxB.PNG)  
![syntax](./images/14-pipeline-syntax.PNG)

-  successful run
![playbook-run](./images/15-dev-env-result.PNG )

If everything goes well for you, it means, the `Dev` environment has an up-to-date configuration. But what if we need to deploy to other environments?

- Are we going to manually update the `Jenkinsfile` to point inventory to those environments? such as `sit, uat, pentest`, etc.
- Or do we need a dedicated git branch for each environment, and have the `inventory` part hard coded there.  

Think about those for a minute and try to work out which one sounds more like a better solution.

Manually updating the `Jenkinsfile` is definitely not an option. And that should be obvious to you at this point. Because we try to automate things as much as possible.

Well, unfortunately, we will not be doing any of the highlighted options. What we will be doing is to parameterise the deployment. So that at the point of execution, the appropriate values are applied.
  

### Parameterizing Jenkinsfile For Ansible Deployment
To deploy to other environments, we will need to use parameters.

1. Update `sit` inventory with new servers
```
[tooling]
<SIT-Tooling-Web-Server-Private-IP-Address>

[todo]
<SIT-Todo-Web-Server-Private-IP-Address>

[nginx]
<SIT-Nginx-Private-IP-Address>

[db:vars]
ansible_user=ec2-user
ansible_python_interpreter=/usr/bin/python

[db]
<SIT-DB-Server-Private-IP-Address>
```

2. Update `Jenkinsfile` to introduce parameterization. Below is just one parameter. It has a default value in case if no value is specified at execution. It also has a description so that everyone is aware of its purpose.
```
pipeline {
    agent any

    parameters {
      string(name: 'inventory', defaultValue: 'dev',  description: 'This is the inventory file for the environment to deploy configuration')
    }
...
```
3. In the Ansible execution section, remove the hardcoded `inventory/dev` and replace with `${inventory}  
![dev-rm](./images/16-dev-removed.PNG)

From now on, each time you hit on execute, it will expect an input.

![param](./images/17-param.PNG)

Notice that the default value loads up, but we can now specify which environment we want to deploy the configuration to. Simply type `sit` and hit **Run**



4. Add another parameter. This time, introduce `tagging` in Ansible. You can limit the Ansible execution to a specific role or playbook desired. Therefore, add an Ansible tag to run against `webserver` only. Test this locally first to get the experience. Once you understand this, update Jenkinsfile and run it from Jenkins.



### CI/CD PIPELINE FOR TODO APPLICATION
We already have `tooling` website as a part of deployment through Ansible. Here we will introduce another PHP application to add to the list of software products we are managing in our infrastructure. The good thing with this particular application is that it has unit tests, and it is an ideal application to show an end-to-end CI/CD pipeline for a particular application.

Our goal here is to deploy the application onto servers directly from `Artifactory` rather than from `git`. If you have not updated Ansible with an Artifactory role, simply use this guide to create an Ansible role for Artifactory (ignore the Nginx part). Configure Artifactory on Ubuntu 20.04

#### Phase 1 – Prepare Jenkins
1. Fork the repository below into your GitHub account
https://github.com/darey-devops/php-todo.git
2. On you Jenkins server, install PHP, its dependencies and Composer tool (Feel free to do this manually at first, then update your Ansible accordingly later)
```
    sudo apt install -y zip libapache2-mod-php phploc php-{xml,bcmath,bz2,intl,gd,mbstring,mysql,zip}
```
3. Install Jenkins plugins
31. Plot plugin
32. Artifactory plugin
- We will use plot plugin to display tests reports, and code coverage information.
- The Artifactory plugin will be used to easily upload code artifacts into an Artifactory server.

**important step**
===================
- spin up artifactory server (t2 medium), update ip in inventory ci and install artifactory via jenkins playbook against ci
![artifactory](./images/19-artifactory.PNG)

![artifactory-installed](./images/18-artifactory-installed.PNG)  

![gui](./images/20-arti-gui.PNG)

- follow on screen prompt on JFROG(username:admin, password:password) and create a generic repository
![jfrog-repo](./images/21-jfrog-repo.PNG)

4. In Jenkins UI configure Artifactory
![conf](./images/22-jenkins-jfron-conf.PNG)

Configure the server ID, URL and Credentials, run Test Connection.



#### Phase 2 – Integrate Artifactory repository with Jenkins
1. Create a dummy `Jenkinsfile` in the repository (cloned php-todo)
2. Using Blue Ocean, create a multibranch Jenkins pipeline
3. On the database server, create database and user
```
Create database homestead;
CREATE USER 'homestead'@'%' IDENTIFIED BY 'sePret^i';
GRANT ALL PRIVILEGES ON * . * TO 'homestead'@'%';
```
![create-db](./images/23-create-db.PNG)
**note:** Jenkins ip used  

4. Update the database connectivity requirements in the file .env.sample
5. Update Jenkinsfile with proper pipeline configuration (in cloned php-todo)
```
pipeline {
    agent any

  stages {

     stage("Initial cleanup") {
          steps {
            dir("${WORKSPACE}") {
              deleteDir()
            }
          }
        }

    stage('Checkout SCM') {
      steps {
            git branch: 'main', url: 'https://github.com/darey-devops/php-todo.git'
      }
    }

    stage('Prepare Dependencies') {
      steps {
             sh 'mv .env.sample .env'
             sh 'composer install'
             sh 'php artisan migrate'
             sh 'php artisan db:seed'
             sh 'php artisan key:generate'
      }
    }
  }
}
```
Notice the **Prepare Dependencies** section

- The required file by `PHP` is `.env` so we are renaming `.env.sample` to `.env`
- Composer is used by PHP to install all the dependent libraries used by the application
- `php artisan` uses the `.env` file to setup the required database objects – (After successful run of this step, login to the database, run show tables and you will see the tables being created for you)

**important steps for a successful run**
=====================================
- install mysql on jenkins server `sudo yum install mysql`
- edit binding address on db server `sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf` and restart mysql `sudo systemctl restart mysql`
- update `.env.sample` as shown in the image to enable jenkins server connection to db server
![env](./images/24-env.PNG)

![result](./images/25-php-run-sucess.PNG)

1. Update the Jenkinsfile to include Unit tests step
    ```
    stage('Execute Unit Tests') {
      steps {
             sh './vendor/bin/phpunit'
      }
    ``` 
![unit-test](./images/26-unit-test.PNG)  

#### Phase 3 – Code Quality Analysis
This is one of the areas where developers, architects and many stakeholders are mostly interested in as far as product development is concerned. As a DevOps engineer, you also have a role to play. Especially when it comes to setting up the tools.

For PHP the most commonly tool used for code quality analysis is phploc. Read the article here for more `https://matthiasnoback.nl/2019/09/using-phploc-for-quick-code-quality-estimation-part-1/`

The data produced by phploc can be ploted onto graphs in Jenkins.

1. Add the code analysis step in Jenkinsfile. The output of the data will be saved in `build/logs/phploc.csv` file.
```
stage('Code Analysis') {
  steps {
        sh 'phploc app/ --log-csv build/logs/phploc.csv'

  }
}
```
2. Plot the data using `plot` Jenkins plugin.
This plugin provides generic plotting (or graphing) capabilities in Jenkins. It will plot one or more single values variations across builds in one or more plots. Plots for a particular job (or project) are configured in the job configuration screen, where each field has additional help information. Each plot can have one or more lines (called data series). After each build completes the plots’ data series latest values are pulled from the CSV file generated by `phploc`.
```
    stage('Plot Code Coverage Report') {
      steps {

            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Lines of Code (LOC),Comment Lines of Code (CLOC),Non-Comment Lines of Code (NCLOC),Logical Lines of Code (LLOC)                          ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'A - Lines of code', yaxis: 'Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Directories,Files,Namespaces', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'B - Structures Containers', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Average Class Length (LLOC),Average Method Length (LLOC),Average Function Length (LLOC)', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'C - Average Length', yaxis: 'Average Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Cyclomatic Complexity / Lines of Code,Cyclomatic Complexity / Number of Methods ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'D - Relative Cyclomatic Complexity', yaxis: 'Cyclomatic Complexity by Structure'      
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Classes,Abstract Classes,Concrete Classes', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'E - Types of Classes', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Methods,Non-Static Methods,Static Methods,Public Methods,Non-Public Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'F - Types of Methods', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Constants,Global Constants,Class Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'G - Types of Constants', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Test Classes,Test Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'I - Testing', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Logical Lines of Code (LLOC),Classes Length (LLOC),Functions Length (LLOC),LLOC outside functions or classes ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'AB - Code Structure by Logical Lines of Code', yaxis: 'Logical Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Functions,Named Functions,Anonymous Functions', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'H - Types of Functions', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Interfaces,Traits,Classes,Methods,Functions,Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'BB - Structure Objects', yaxis: 'Count'

      }
    }
```  

You should now see a `Plot` menu item on the left menu. Click on it to see the charts. (The analytics may not mean much to you as it is meant to be read by developers. So, you need not worry much about it – this is just to give you an idea of the real-world implementation).

**important**
========================
- install phploc (and others)
```
sudo dnf --enablerepo=remi install php-phpunit-phploc
wget -O phpunit https://phar.phpunit.de/phpunit-7.phar
chmod +x phpunit
sudo yum  install php-xdebug
```
![plots](./images/27-plots.PNG)

3. Bundle the application code for into an artifact (archived package) upload to Artifactory

**hint** might need to install zip `sudo yum install zip -y`
```
stage ('Package Artifact') {
    steps {
            sh 'zip -qr php-todo.zip ${WORKSPACE}/*'
     }
    }
```
4. Publish the resulted artifact into Artifactory
```
stage ('Upload Artifact to Artifactory') {
          steps {
            script { 
                 def server = Artifactory.server 'artifactory-server'                 
                 def uploadSpec = """{
                    "files": [
                      {
                       "pattern": "php-todo.zip",
                       "target": "<name-of-artifact-repository>/php-todo",
                       "props": "type=zip;status=ready"

                       }
                    ]
                 }""" 

                 server.upload spec: uploadSpec
               }
            }

        }
```
![upload](./images/28-art-zip-upload.PNG)

![uploaded](./images/29-uploaded.PNG)

5. Deploy the application to the dev environment by launching Ansible pipeline
```
stage ('Deploy to Dev Environment') {
    steps {
    build job: 'ansible-config-1/main', parameters: [[$class: 'StringParameterValue', name: 'env', value: 'dev']], propagate: false, wait: true
    }
  }
```

**important**
- update download source
![src](./images/30-download-src.PNG)  

The `build job` used in this step tells Jenkins to start another job. In this case it is the `ansible-config-1` job, and we are targeting the `main` branch. Hence, we have `ansible-config-1/main`. Since the Ansible project requires parameters to be passed in, we have included this by specifying the `parameters` section. The name of the parameter is env and its value is `dev`. Meaning, deploy to the Development environment.

But how are we certain that the code being deployed has the quality that meets corporate and customer requirements? Even though we have implemented **Unit Tests** and **Code Coverage** Analysis with `phpunit` and `phploc`, we still need to implement **Quality Gate** to ensure that ONLY code with the required code coverage, and other quality standards make it through to the environments.

To achieve this, we need to configure **SonarQube** – An open-source platform developed by **SonarSource** for continuous inspection of code quality to perform automatic reviews with static analysis of code to detect bugs, code smells, and security vulnerabilities.

### SONARQUBE INSTALLATION
Before we start getting hands on with SonarQube configuration, it is incredibly important to understand a few concepts:

- Software Quality – The degree to which a software component, system or process meets specified requirements based on user needs and expectations.
- Software Quality Gates – Quality gates are basically acceptance criteria which are usually presented as a set of predefined quality criteria that a software development project must meet in order to proceed from one stage of its lifecycle to the next one.
SonarQube is a tool that can be used to create quality gates for software projects, and the ultimate goal is to be able to ship only quality software code.

Despite that DevOps CI/CD pipeline helps with fast software delivery, it is of the same importance to ensure the quality of such delivery. Hence, we will need SonarQube to set up Quality gates. In this project we will use predefined Quality Gates (also known as The Sonar Way). Software testers and developers would normally work with project leads and architects to create custom quality gates.

### Install SonarQube on Ubuntu 20.04 With PostgreSQL as Backend Database
**important**
==============
sonarqube installation was done by manually running the playbook on the terminal (jenkins app gives errors). Important steps for a successful run are as follows:
- install ansible community on jenkins server `ansible-galaxy collection install community.postgresql`   
- update roles-path in ansible.cfg temporarily   

![path](./images/31-roles-path.PNG)  

- export ansible_config `export ANSIBLE_CONFIG=<path of ansible.cfg>`  
- run playbook `ansible-playbook -i inventory/ci playbooks/site.yml`  

![sonar](./images/32-sonar-installed.PNG)

![sonar-gui](./images/33-sonar-gui.PNG)

Here is a manual approach to installation. Ensure that you can to automate the same with Ansible.

Below is a step by step guide how to install **SonarQube 7.9.3** version. It has a strong prerequisite to have Java installed since the tool is Java-based. MySQL support for SonarQube is deprecated, therefore we will be using PostgreSQL.

We will make some Linux Kernel configuration changes to ensure optimal performance of the tool – we will increase `vm.max_map_count, file discriptor` and `ulimit`.

#### Tune Linux Kernel
This can be achieved by making session changes which does not persist beyond the current session terminal.
```
sudo sysctl -w vm.max_map_count=262144
sudo sysctl -w fs.file-max=65536
ulimit -n 65536
ulimit -u 4096
```
To make a permanent change, edit the file `/etc/security/limits.conf` and append the below
```
sonarqube   -   nofile   65536
sonarqube   -   nproc    4096
```
Before installing, let us update and upgrade system packages:
```
sudo apt-get update
sudo apt-get upgrade
```
Install `wget` and `unzip` packages
```
sudo apt-get install wget unzip -y
```
Install `OpenJDK` and `Java Runtime Environment (JRE) 11`
```
 sudo apt-get install openjdk-11-jdk -y
 sudo apt-get install openjdk-11-jre -y
```
Set default JDK – To set default JDK or switch to OpenJDK enter below command:
```
 sudo update-alternatives --config java
```
If you have multiple versions of Java installed, you should see a list like below:
```
Selection    Path                                            Priority   Status

------------------------------------------------------------

  0            /usr/lib/jvm/java-11-openjdk-amd64/bin/java      1111      auto mode

  1            /usr/lib/jvm/java-11-openjdk-amd64/bin/java      1111      manual mode

  2            /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java   1081      manual mode

* 3            /usr/lib/jvm/java-8-oracle/jre/bin/java          1081      manual mode
```
Type "1" to switch **OpenJDK 11**

Verify the set JAVA Version:
```
java -version
```
Output
```
java -version

openjdk version "11.0.7" 2020-04-14

OpenJDK Runtime Environment (build 11.0.7+10-post-Ubuntu-3ubuntu1)

OpenJDK 64-Bit Server VM (build 11.0.7+10-post-Ubuntu-3ubuntu1, mixed mode, sharing)
```

### Install and Setup PostgreSQL 10 Database for SonarQube
The command below will add PostgreSQL repo to the repo list:
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
```
Download PostgreSQL software
```
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
```
Install PostgreSQL Database Server
```
sudo apt-get -y install postgresql postgresql-contrib
```
Start PostgreSQL Database Server
```
sudo systemctl start postgresql
```
Enable it to start automatically at boot time
```
sudo systemctl enable postgresql
```
Change the password for default `postgres` user (Pass in the password you intend to use, and remember to save it somewhere)
```
sudo passwd postgres
```
Switch to the `postgres` user
```
su - postgres
```
Create a new user by typing
```
createuser sonar
```
Switch to the PostgreSQL shell
```
psql
```
Set a password for the newly created user for SonarQube database
```
ALTER USER sonar WITH ENCRYPTED password 'sonar';
```
Create a new database for PostgreSQL database by running:
```
CREATE DATABASE sonarqube OWNER sonar;
```
Grant all privileges to `sonar` user on sonarqube Database.
```
grant all privileges on DATABASE sonarqube to sonar;
```
Exit from the psql shell:
```
\q
```
Switch back to the sudo user by running the exit command.
```
exit
```

### Install SonarQube on Ubuntu 20.04 LTS
Navigate to the `tmp` directory to temporarily download the installation files
```
cd /tmp && sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-7.9.3.zip
```
Unzip the archive setup to `/opt directory`
```
sudo unzip sonarqube-7.9.3.zip -d /opt
```
Move extracted setup to `/opt/sonarqube` directory
```
sudo mv /opt/sonarqube-7.9.3 /opt/sonarqube
```  

### CONFIGURE SONARQUBE
We cannot run SonarQube as a root user, if you run using root user it will stop automatically. The ideal approach will be to create a separate group and a user to run SonarQube

Create a group sonar
```
sudo groupadd sonar
```
Now add a user with control over the /opt/sonarqube directory
```
 sudo useradd -c "user to run SonarQube" -d /opt/sonarqube -g sonar sonar 
 sudo chown sonar:sonar /opt/sonarqube -R
```
Open SonarQube configuration file using your favourite text editor (e.g., nano or vim)
```
sudo vim /opt/sonarqube/conf/sonar.properties
```
Find the following lines:
```
#sonar.jdbc.username=
#sonar.jdbc.password=
```
Uncomment them and provide the values of PostgreSQL Database username and password:
```
#--------------------------------------------------------------------------------------------------

# DATABASE

#

# IMPORTANT:

# - The embedded H2 database is used by default. It is recommended for tests but not for

#   production use. Supported databases are Oracle, PostgreSQL and Microsoft SQLServer.

# - Changes to database connection URL (sonar.jdbc.url) can affect SonarSource licensed products.

# User credentials.

# Permissions to create tables, indices and triggers must be granted to JDBC user.

# The schema must be created first.

sonar.jdbc.username=sonar
sonar.jdbc.password=sonar
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
```
Edit the sonar script file and set RUN_AS_USER
```
sudo nano /opt/sonarqube/bin/linux-x86-64/sonar.sh
```
```
# If specified, the Wrapper will be run as the specified user.

# IMPORTANT - Make sure that the user has the required privileges to write

#  the PID file and wrapper.log files.  Failure to be able to write the log

#  file will cause the Wrapper to exit without any way to write out an error

#  message.

# NOTE - This will set the user which is used to run the Wrapper as well as

#  the JVM and is not useful in situations where a privileged resource or

#  port needs to be allocated prior to the user being changed.

RUN_AS_USER=sonar
```
Now, to start SonarQube we need to do following:

Switch to sonar user
```
sudo su sonar
```
Move to the script directory
```
cd /opt/sonarqube/bin/linux-x86-64/
```
Run the script to start SonarQube
```
./sonar.sh start
```
Expected output shall be as:

```
Starting SonarQube...

Started SonarQube
```
Check SonarQube running status:
```
./sonar.sh status
```
Sample Output below:
```
./sonar.sh status

SonarQube is running (176483).
```
To check SonarQube logs, navigate to `/opt/sonarqube/logs/sonar.log` directory
```
tail /opt/sonarqube/logs/sonar.log
```
Output
```
INFO  app[][o.s.a.ProcessLauncherImpl] Launch process[[key='ce', ipcIndex=3, logFilenamePrefix=ce]] from [/opt/sonarqube]: /usr/lib/jvm/java-11-openjdk-amd64/bin/java -Djava.awt.headless=true -Dfile.encoding=UTF-8 -Djava.io.tmpdir=/opt/sonarqube/temp --add-opens=java.base/java.util=ALL-UNNAMED -Xmx512m -Xms128m -XX:+HeapDumpOnOutOfMemoryError -Dhttp.nonProxyHosts=localhost|127.*|[::1] -cp ./lib/common/*:/opt/sonarqube/lib/jdbc/h2/h2-1.3.176.jar org.sonar.ce.app.CeServer /opt/sonarqube/temp/sq-process15059956114837198848properties

 INFO  app[][o.s.a.SchedulerImpl] Process[ce] is up

 INFO  app[][o.s.a.SchedulerImpl] SonarQube is up
```
You can see that SonarQube is up and running

#### Configure SonarQube to run as a systemd service
Stop the currently running SonarQube service
```
 cd /opt/sonarqube/bin/linux-x86-64/
```
Run the script to start SonarQube
```
./sonar.sh stop
```
Create a systemd service file for SonarQube to run as System Startup.
```
 sudo nano /etc/systemd/system/sonar.service
```
Add the configuration below for `systemd` to determine how to start, stop, check status, or restart the SonarQube service.
```
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking

ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

User=sonar
Group=sonar
Restart=always

LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
```
Save the file and control the service with `systemctl`
```
sudo systemctl start sonar
sudo systemctl enable sonar
sudo systemctl status sonar
```
### Access SonarQube
To access SonarQube using browser, type server’s IP address followed by port 9000
```
http://server_IP:9000 OR http://localhost:9000
```
Login to SonarQube with default administrator username and password – admin

![sonar-gui2](./images/34-sonar-gui2.PNG)

Now, when SonarQube is up and running, it is time to setup our Quality gate in Jenkins.  
  
### CONFIGURE SONARQUBE AND JENKINS FOR QUALITY GATE
- In Jenkins, install `SonarScanner plugin`

- Navigate to configure system in Jenkins. Add SonarQube server as shown below:

  `Manage Jenkins > Configure System`

![sonar-jenkins](./images/35-sonar-jenkins.PNG)


- Generate authentication token in SonarQube

 `User > My Account > Security > Generate Tokens`

![token](./images/36-token.PNG)
f9f2cd1168297778ee331d1cca3822e51e2ed330

- Configure Quality Gate Jenkins Webhook in SonarQube – The URL should point to your Jenkins server http://{JENKINS_HOST}/sonarqube-webhook/

`Administration > Configuration > Webhooks > Create`
![webhook](./images/37-webhook.PNG)


- Setup SonarQube scanner from Jenkins – Global Tool Configuration

`Manage Jenkins > Global Tool Configuration`

![sonar-scanner](./images/38-sonarsca.PNG)

### Update Jenkins Pipeline to include SonarQube scanning and Quality Gate
Below is the snippet for a **Quality Gate** stage in Jenkinsfile.
```
    stage('SonarQube Quality Gate') {
        environment {
            scannerHome = tool 'SonarQubeScanner'
        }
        steps {
            withSonarQubeEnv('sonarqube') {
                sh "${scannerHome}/bin/sonar-scanner"
            }

        }
    }
```
**NOTE:** The above step will fail because we have not updated `sonar-scanner.properties

- Configure `sonar-scanner.properties` – From the step above, Jenkins will install the scanner tool on the Linux server. You will need to go into the `tools` directory on the server to configure the `properties` file in which SonarQube will require to function during pipeline execution.
```
cd /var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/conf/
```
Open `sonar-scanner.properties` file
```
sudo vi sonar-scanner.properties
```
Add configuration related to `php-todo` project
```
sonar.host.url=http://<SonarQube-Server-IP-address>:9000
sonar.projectKey=php-todo
#----- Default source code encoding
sonar.sourceEncoding=UTF-8
sonar.php.exclusions=**/vendor/**
sonar.php.coverage.reportPaths=build/logs/clover.xml
sonar.php.tests.reportPath=build/logs/junit.xml
```

![vi-sonar](./images/39-vi-sonar.PNG)  

**HINT:** To know what exactly to put inside the `sonar-scanner.properties` file, SonarQube has a configurations page where you can get some directions.



A brief explanation of what is going on the the stage – set the environment variable for the `scannerHome` use the same name used when you configured SonarQube Scanner from **Jenkins Global Tool Configuration**. If you remember, the name was `SonarQubeScanner`. Then, within the `steps` use shell to run the scanner from bin directory.

To further examine the configuration of the scanner tool on the Jenkins server – navigate into the `tools` directory
```
cd /var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/bin
```
List the content to see the scanner tool `sonar-scanner`. That is what we are calling in the pipeline script.

Output of `ls -latr`
```
ubuntu@ip-172-31-16-176:/var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarQubeScanner/bin$ ls -latr
total 24
-rwxr-xr-x 1 jenkins jenkins 2550 Oct  2 12:42 sonar-scanner.bat
-rwxr-xr-x 1 jenkins jenkins  586 Oct  2 12:42 sonar-scanner-debug.bat
-rwxr-xr-x 1 jenkins jenkins  662 Oct  2 12:42 sonar-scanner-debug
-rwxr-xr-x 1 jenkins jenkins 1823 Oct  2 12:42 sonar-scanner
drwxr-xr-x 2 jenkins jenkins 4096 Dec 26 18:42 .
```
So far you have been given code snippets on each of the stages within the Jenkinsfile. But, you should also be able to generate Jenkins configuration code yourself.

- To generate Jenkins code, navigate to the dashboard for the php-todo pipeline and click on the Pipeline Syntax menu item
```
Dashboard > php-todo > Pipeline Syntax 
```

- Click on Steps and select withSonarQubeEnv – This appears in the list because of the previous SonarQube configurations you have done in Jenkins. Otherwise, it would not be there.


Within the generated block, you will use the sh command to run shell on the server. For more advanced usage in other projects, you can add to bookmarks this SonarQube documentation page in your browser.

### End-to-End Pipeline Overview
Indeed, this has been one of the longest projects from Project 1, and if everything has worked out for you so far, you should have a view like below:



But we are not completely done yet!

The quality gate we just included has no effect. Why? Well, because if you go to the SonarQube UI, you will realise that we just pushed a poor-quality code onto the development environment.

- Navigate to `php-todo` project in SonarQube
![sonar-bug](./images/40-sonar-bug.PNG)

There are bugs, and there is 0.0% code coverage. (code coverage is a percentage of unit tests added by developers to test functions and objects in the code)

- If you click on `php-todo` project for further analysis, you will see that there is 6 hours’ worth of technical debt, code smells and security issues in the code.
![sonar-bug2](./images/41-sonar-bug2.PNG)

In the development environment, this is acceptable as developers will need to keep iterating over their code towards perfection. But as a DevOps engineer working on the pipeline, we must ensure that the quality gate step causes the pipeline to fail if the conditions for quality are not met.

### Conditionally deploy to higher environments
In the real world, developers will work on feature branch in a repository (e.g., GitHub or GitLab). There are other branches that will be used differently to control how software releases are done. You will see such branches as:

- Develop
- Master or Main
(The * is a place holder for a version number, Jira Ticket name or some description. It can be something like `Release-1.0.0`)
- Feature/*
- Release/*
- Hotfix/*
etc.

There is a very wide discussion around release strategy, and git branching strategies which in recent years are considered under what is known as GitFlow `https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow` (Have a read and keep as a bookmark – it is a possible candidate for an interview discussion, so take it seriously!)

Assuming a basic gitflow implementation restricts only the develop branch to deploy code to Integration environment like sit.

Let us update our Jenkinsfile to implement this:

- First, we will include a When condition to run Quality Gate whenever the running branch is either develop, hotfix, release, main, or master
```
when { branch pattern: "^develop*|^hotfix*|^release*|^main*", comparator: "REGEXP"}
```
- Then we add a timeout step to wait for SonarQube to complete analysis and successfully finish the pipeline only when code quality is acceptable.
  ```
    timeout(time: 1, unit: 'MINUTES') {
        waitForQualityGate abortPipeline: true
    }
  ```
The complete stage will now look like this:
```
    stage('SonarQube Quality Gate') {
      when { branch pattern: "^develop*|^hotfix*|^release*|^main*", comparator: "REGEXP"}
        environment {
            scannerHome = tool 'SonarQubeScanner'
        }
        steps {
            withSonarQubeEnv('sonarqube') {
                sh "${scannerHome}/bin/sonar-scanner -Dproject.settings=sonar-project.properties"
            }
            timeout(time: 1, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
            }
        }
    }
```   
To test, create different branches and push to GitHub. You will realise that only branches other than develop, hotfix, release, main, or master will be able to deploy the code.

![brnch](./images/43-diff-branch.PNG)

If everything goes well, you should be able to see something like this:

![failed](./images/42-fail-result.PNG)

**important note**
===========
- this line solved a problem at the sonarqube quality gate stage `sudo ln -s "$(which node)" /usr/local/bin/node`

Notice that with the current state of the code, it cannot be deployed to Integration environments due to its quality. In the real world, DevOps engineers will push this back to developers to work on the code further, based on SonarQube quality report. Once everything is good with code quality, the pipeline will pass and proceed with sipping the codes further to a higher environment.

### Complete the following tasks to finish Project 14
1. Introduce Jenkins agents/slaves – Add 2 more servers to be used as Jenkins slave. Configure Jenkins to run its pipeline jobs randomly on any available slave nodes.

- spin up an ec2 server
- install java and edit bash_profile (see README.md)
- go to jenkins-managejenkins-nodes and create a node  

![nodes](./images/44-slave.PNG)

2. Configure webhook between Jenkins and GitHub to automatically run the pipeline when there is a code push.

![webhook](./images/45-webhook1.PNG)  

3. Deploy the application to all the environments (on dev and prod implemented)  

- `dev` and `prod` put in a folder `dev-n-prod` and used as inventory value  

![devnprod](./images/46-devnprod.PNG)  

- job build with `dev-n-prod` parameter  

![param](./images/47-devnprodparam.PNG)  

- app deployed to two env  

![deployed](./images/48-two-env.PNG)

4. Optional – Experience pentesting in pentest environment by configuring Wireshark there and just explore for information sake only. Watch Wireshark Tutorial here
- Ansible Role for Wireshark:
- https://github.com/ymajik/ansible-role-wireshark (Ubuntu)
- https://github.com/wtanaka/ansible-role-wireshark (RedHat)
Congratulations! You have just experienced one of the most interesting and complex projects in you Project Based Learning journey so far. The vast experience and knowledge you have acquired here will set the stage for the next 6 projects to come. You should be ready to start applying for DevOps jobs after completing Project 20.



Instructions On How To Submit Your Work For Review And Feedback
To submit your work for review and feedback – follow this instruction.
In addition to your GitHub projects (both, Ansible and PHP application) also prepare and submit the following:

1. Make a short video on how your pipelines have executed
2. In the video, showcase your SonarQube UI

![video](./images/Project14_review.mp4)

`https://drive.google.com/file/d/1z9YbnoP0pnh3gj0NaJlFBKwAtJTInr2y/view?usp=sharing`
