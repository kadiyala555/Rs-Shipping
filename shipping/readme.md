
<p align="center"> 
<img src="https://user-images.githubusercontent.com/100637276/163732513-0201b81d-d6d6-4ab9-9cf3-3f6b6c1e2f44.png" alt="TELUS">
</p>
 
<h1 id="heading" align="center">WireMock Integration with GitHub Actions for Mocking MicroServices</h1>


<br>

<h2 id="table-of-contents"> 🔤 Table of Contents</h2>

<details open="open">
  <summary>Table of Contents</summary>
  <ol>
    <li><a href="#proposed-wiremock-integration-plan"> ➤ Proposed WireMock Integration Plan</a></li>
    <li><a href="#step1"> ➤ Step 1: General Setup Instructions </a></li>
    <li><a href="#step2"> ➤ Step 2: Maven Setup Instructions for Wiremock </a></li>
    <li><a href="#step3"> ➤ Step 3: GitHub Actions for Wiremock Project </a></li>
    <li><a href="#step4"> ➤ Step 4: Screenshots of the test results </a></li>
   </ol>
</details>

![-----------------------------------------------------](https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/rainbow.png)


## Proposed Wiremock Framework Integration Plan

This document details the implementation of Wiremock as a Capability
<br>
<br>
![image](https://user-images.githubusercontent.com/100637276/163222841-6ad7a78b-6937-4718-a5ea-f4a661c9cd67.png)
<br>
<br>

<!-- STEP1 -->
<h3 id="step1"> 🔰 STEP 1: General Setup Instructions</h3>

1. **Generate a GITHUB TOKEN (We need this to get the Pull Request(PR) Information and publish reports back to Github PR)**
 * Go to your GITHUB Account (Not the Repo) 
 * ➡️ Settings ➡️ Then Scroll down to the end, Go to Developer Settings
 * ➡️ Go to Personal access token
 * ➡️ Generate a new token ➡️ Configure SSO 
 * ➡️ Copy the **token**.
 * ➡️ After you obtain GitHub Personal Access Token, Come to your **GitHub Repo**, 
 * ➡️ Click on Settings Tab ➡️ visit Security ➡️ Secrets ➡️ Actions, then 
 * ➡️ Add a secret and name it **GITHUB_TOKEN**  and in the value field paste the **token** you obtained from the above(1st) step

2. **Request SonarQube Team for access to a new project**
 * Access Request Instructions:
 * Identify a unique name of the project to be created in SonarQube [https://sonarqube.cloudapps.telus.com/] ex: cdo-triangulum-ctv for this app
 * This is where all your reports will be published
 * Also request for mapping your GitHub project in SonarQube [You should give the GitHub Repo Name to SonarQube Admins for mapping Repo with SonarQube Project]
 
3. **Once access is obtained to SonarQube ➡️ Go to Your name displayed on the left side Top corner**
 * ➡️ Then Click on My Account
 * ➡️ then go to Security
 * ➡️ Enter a meaningful name for the token ➡️ Click on Generate
 * ➡️ Copy/preserve the token [This is the token that enables authentication of your Github Workflow execution to publish results in SonarQube]

4. **After you obtain SonarQube Token...** 
* ➡️ Come to your GitHub Repo, Click on Settings Tab, visit Security
* ➡️ Secrets ➡️ Actions, then 
* ➡️ Add a secret and name it **SONAR_TOKEN** and in the value field paste the **token** obtained from the above(2nd) step

5. **Request Google Cloud COE Team for creation of Google Artifact Registry (This is needed for storing your Build artifacts)**
* ➡️ Identify a project name, Ex: trianngulum-ctv for this app
* ➡️ Identiy a location or obtain the location from the Google Cloud COE ex: us-central1 (if its across multi region, then you dont need this)
* ➡️ Identify a name for the repository and type (maven, gradle, docker etc). It is maven for this project and name is mockitodemoapp

![-----------------------------------------------------](https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/rainbow.png)

<!-- STEP2 -->
<h3 id="step2"> 🔰 STEP 2: Maven Setup Instructions for Wiremock</h3>

1. **General Info**

WireMock is a library for stubbing and mocking web services. It constructs a HTTP server that we could connect to as we would to an actual web service. When a WireMock server is in action, we can set up expectations, call the service, and then verify its behaviors.
<p>Under Dependencies section in POM.XML, Add the below</p>

```XML
<dependencies>
	<dependency>
	    <groupId>io.rest-assured</groupId>
	    <artifactId>rest-assured</artifactId>
	    <version>3.3.0</version><!--$NO-MVN-MAN-VER$-->
	    <scope>test</scope>
	</dependency>
	<dependency>
	    <groupId>junit</groupId>
	    <artifactId>junit</artifactId>
	    <version>4.4</version><!--$NO-MVN-MAN-VER$-->
	    <scope>test</scope>
	</dependency>
	<dependency>
	    <groupId>com.github.tomakehurst</groupId>
	    <artifacId>wiremock-jre8</artifactId>
	    <version>2.33.1</version>
	    <scope>test</scope>
	</dependency>
</dependencies>
```
2. **RestAssued, Junit and Surefire Reports are used along side Wiremock. Ensure the necessary dependencies and plugins are configured**
 
- [x] JUnit is used for Test Coverange
- [x] REST Assured is a powerful API testing library which makes the tests human readable and makes CI/CD setup a breeze
- [x] SureFire Reports are XMLs of the results of JUnit which can be pulished

3. **Post build, the artifacts has to be pushed to Google Artifact Repository(GAR)**

 * Developers commit and push changes to GitHub
 * The Workflow Engine is triggered when a pull request (PR) to develop a branch is made OR code is pushed to a develop branch PR
 * The Workflow Engine contains all the GitHub Actions automations which are handled via workflows
 * The gcloudbuild.yaml file contains the steps required for the workflow
 * The GitHub Actionsworkflow engine would build and spin up the docker containers
 * The Automated testing will be conducted by running through the test cases that were created
 * Once the workflow is completed, a comprehensive report is generated indicating the status of the build.

<!-- STEP3 -->
<h3 id="step3"> 🔰 STEP 3: GitHub Actions for Wiremock Project</h3>

**Below GitHub Actions will build and push the artifacts to GAR and publish the results and Pull Request(PR) Comments**

```YAML

build-shipping:
    name: Building SHIPPING IMAGE
    needs: changes
    if: ${{ needs.changes.outputs.shipping == 'true' }}
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3
                       
      - name: Setup GCP Service Account
        uses: "google-github-actions/auth@v0"
        with:
          credentials_json: "${{ secrets.GOOGLE_CREDENTIALS }}"
      
      - name: Set up Cloud SDK
        uses: 'google-github-actions/setup-gcloud@v0'
      
      # Configure docker to use the gcloud command-line tool as a credential helper
      - name: Setup Docker
        run: |
          gcloud auth configure-docker us-central1-docker.pkg.dev


      - name: Run Wiremock server
        #uses: docker://wiremock/wiremock:2.33.1 
        #with:
        #  args: >-
        #    --publish: 9999:8080
        #    --name: wiremock
        #    --rm
        #    -it

       
        run: |
          cd shipping
          docker run -d -p 8080:8080 -v $PWD/src/test/resources:/home/wiremock --name wiremock wiremock/wiremock:2.33.1
          docker container inspect wiremock
          echo `docker container port wiremock`
          mvn test
    # curl -LIs http://localhost:8080
      - name: Publish Wiremock Test Results
        uses: EnricoMi/publish-unit-test-result-action@v1
        if: always()
        with:
          files: "shipping/target/surefire-reports/*.xml"
       
      - name: Build-and-push-to-GAR
        run: |
          echo `pwd`
          export TAG=`cat shipping/VERSION.txt`
          echo $TAG
          echo "$GAR_INFO"/"$SHIPPING_APP_NAME":"$TAG"
          docker build -t "$GAR_INFO"/"$SHIPPING_APP_NAME":"$TAG" shipping/
          gcloud info
          docker push "$GAR_INFO"/"$SHIPPING_APP_NAME":"$TAG"  
 ```
 
<!-- STEP4 -->
<h3 id="step4"> 🔰 STEP 4: Screenshots of the test results</h3>

📊 Results of JUnit Tests - Test Coverage

![image](https://raw.githubusercontent.com/koushikgurijala/robotShop/main/wiremock-test-result.png)

<br>

The Wiremock report helps teams in Fast Development and Delivery, Increased Productivity and High App Quality and Performance
