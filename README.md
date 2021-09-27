# Veracode manual for GitHub

## Introduction

This page (or manual) incorporate few option which Veracode customers can use to integrate Veracode Scanning solutions within their own workflow processes. 

The following discussed concepts are valid for most CI solutions, however, the examples given here focus on GitHub workflow and unavoidably GitHub Actions implementations.

## __The basic__ - adding Veracode Security Scanning to workflow

As most integration options, here are few ways to introduce our different AST scanning options to a workflow.

### Veracode Static Scanning - Upload and Scan 
#### Script

A basic script using a wrapper is documented __[at Veracode help center](https://help.veracode.com/r/r_uploadandscan)__ and looks as follow:   

`java -jar vosp-api-wrapper-java<version>.jar -action uploadandscan -vid <Veracode API ID> -vkey <Veracode API key> -appname myapp -createprofile true -teams myteam -criticality VeryHigh -sandboxname sandboxA -createsandbox true -version <unique version> -scantimeout 30 -selectedpreviously true -filepath /workspace/myapp.jar`

To use the above script within a GitHub workflow we will need to download the latest version of the API Wrapper and run the above script.
<details>
<summary>See example</summary>
<p>

```yaml
name: Veracode Static Scan

# Controls when the action will run. 
on:
  # Triggers the workflow on push or pull request events but only for the master branch
  push:
    branches: [ master, release/* ]
  pull_request:
    branches: [ master, develop, main, release/* ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      - name: create dir prior
        run: |
          mkdir dls
      - name: Download Veracode Wrapper
        working-directory: ./dls/
        run: |
          javawrapperversion=$(curl https://repo1.maven.org/maven2/com/veracode/vosp/api/wrappers/vosp-api-wrappers-java/maven-metadata.xml | grep latest |  cut -d '>' -f 2 | cut -d '<' -f 1)
          echo "javawrapperversion: $javawrapperversion"
          curl -sS -o VeracodeJavaAPI.jar "https://repo1.maven.org/maven2/com/veracode/vosp/api/wrappers/vosp-api-wrappers-java/$javawrapperversion/vosp-api-wrappers-java-$javawrapperversion.jar"
      - name: Run Upload and Scan
        continue-on-error: true
        run: |
          java -jar VeracodeJavaAPI.jar -action uploadandscan -vid ${{secrets.VERACODE_ID}} -vkey ${{secrets.VERACODE_KEY}} -appname ${{ github.repository }} -createprofile true -teams teamA -criticality VeryHigh -sandboxname sandboxA -createsandbox true -version ${{ github.run_id }} -scantimeout 30 -selectedpreviously true -filepath /workspace/app.zip
```
</p>
</details>
<br/>

> :grey_exclamation: The above is good to know, however, it is better to use one of the other options below

#### Pre-built Docker Image
Veracode released and maintain a set of public Docker Images which are available at __[DockerHub](https://hub.docker.com/u/veracode)__. One of these has the API Wrapper pre-packaged and does not requires the convoluted download script.

An example using the Docker image use can be found here:
- [https://github.com/lerer-veracode/verademo-java/blob/test-policy-scan-using-docker/.github/workflows/upload-and-scan.yml](https://github.com/lerer-veracode/verademo-java/blob/test-policy-scan-using-docker/.github/workflows/upload-and-scan.yml)

<details>
<summary>See example</summary>
<p>

```yaml
name: Veracode Static Scan

# Controls when the action will run. 
on:
  # Triggers the workflow on push or pull request events but only for the master branch
  push:
    branches: [ master, release/* ]
  pull_request:
    branches: [ master, develop, main, release/* ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

jobs:
  # This workflow contains a single job called "SAST-Scan"
  SAST-Scan:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
    # The Build and Sandbox name generation omitted from the inline sample
    needs: [generate-sandbox-name, Build]
    # Steps represent a sequence of tasks that will be executed as part of the job
    container: 
      image: veracode/api-wrapper-java:latest
      options: --user root
    steps:
      - name: Get-scan-target
        uses: actions/download-artifact@v2 # See https://github.com/actions/download-artifact
        with:
          name: built-file
      - name: scan
        run: |
          java -jar /opt/veracode/api-wrapper.jar -vid ${{secrets.VERACODE_ID}} -vkey ${{secrets.VERACODE_KEY}} -action UploadAndScan -createprofile true -appname ${{ github.repository }} -version "${{ github.run_id }}" -sandboxname ${{needs.generate-sandbox-name.outputs.sandbox-name}} -createsandbox true  -scantimeout 30 -filepath ./verademo.war

```
</p>
</details>
<br/>

> :bulb: The above is also example using multiple jobs with shared artifacts between them

#### GitHub Action
An easier way to incorporate the upload and scan into a workflow is using Veracode supported __[upload-and-scan GitHub action](https://github.com/marketplace/actions/veracode-upload-and-scan)__

To write a workflow with the official action we can use the documentation in the action page
<details>
<summary>See example</summary>
<p>

```yaml
name: Veracode Static Scan

# Controls when the action will run. 
on:
  # Triggers the workflow on push or pull request events but only for the master branch
  push:
    branches: [ master, release/* ]
  pull_request:
    branches: [ master, develop, main, release/* ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: [ self-hosted, generic ]
    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      - name: Veracode Upload And Scan
        uses: veracode/veracode-uploadandscan-action@0.2.1
        with:
          # appname
          appname: ${{ github.repository }}
          # createprofile
          createprofile: true
          # filepath
          filepath: result.zip #Path to dlls/src/jars/wars....
          # version
          version: ${{ github.run_id }}
          # vid
          vid: ${{ secrets.VERACODE_ID }}
          # vkey
          vkey: ${{ secrets.VERACODE_KEY }}
          # true or false
          createsandbox: true
          # name of the sandbox
          sandboxname: sandboxA
          # wait X minutes for the scan to complete
          scantimeout: 0
          # business criticality - policy selection
          criticality: "High"
```
</p>
</details>

#### Align sandbox name with branch name (Optional)
If you look into the above example, you'll noticed the sandbox name is a fixed name. A fixed sandbox name will not do well in the long run as we probably want to scan different changes/versions in a different sandboxes.

An alternative to that is to align the sandbox name with the repository branch name. In order to achieve that we can modify the workflow definition to include a step prior to the scan to save the branch name as an attribute and use it when we submit for scan.

Example workflow:
- [https://github.com/lerer-veracode/verademo-java/blob/test-policy-scan-using-docker/.github/workflows/upload-and-scan.yml](https://github.com/lerer-veracode/verademo-java/blob/test-policy-scan-using-docker/.github/workflows/upload-and-scan.yml)

<details>
<summary>Inline example</summary>
<p>

```yaml
name: Veracode Static Scan

# Controls when the action will run. 
on:
  # Triggers the workflow on push or pull request events but only for the master branch
  push:
    branches: [ master, release/* ]
  pull_request:
    branches: [ master, develop, main, release/* ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  generate-sandbox-name:
    runs-on: ubuntu-latest
    outputs:
      sandbox-name: ${{ steps.set-sandbox-name.outputs.sandbox-name }}
    steps:
      # Creates the sandbox(logical release descriptive status of current branch)
      - id: set-sandbox-name
        name: set-sandbox-name
        run: |
          echo ${{ github.head_ref }}
          branchName="${{ github.head_ref }}"
          if [[ -z "$branchName" ]]; then
            branchName="${{ github.ref }}"
          fi
          
          if [[ $branchName == *"master"* ]]; then
          echo "::set-output name=sandbox-name::Master"
          elif [[ $branchName == *"main"* ]]; then
          echo "::set-output name=sandbox-name::Main"
          elif [[ $branchName == *"elease/"* ]]; then
          echo "::set-output name=sandbox-name::$branchName"
          else
          echo "::set-output name=sandbox-name::Development"
          fi        
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
    needs: [ generate-sandbox-name ]
    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      - name: Veracode Upload And Scan
        uses: veracode/veracode-uploadandscan-action@0.2.1
        with:
          # appname
          appname: ${{ github.repository }}
          # createprofile
          createprofile: true
          # filepath
          filepath: result.zip #Path to dlls/src/jars/wars....
          # version
          version: ${{ github.run_id }}
          # vid
          vid: ${{ secrets.VERACODE_ID }}
          # vkey
          vkey: ${{ secrets.VERACODE_KEY }}
          # true or false
          createsandbox: true
          # name of the sandbox
          sandboxname: "${{needs.generate-sandbox-name.outputs.sandbox-name}}"
          # wait X minutes for the scan to complete
          scantimeout: 0
          # business criticality - policy selection
          criticality: "High"
```
</p>
</details>
<br/>

> :bulb: - The above example uses multiple jobs definition which we will explain further at the [Flow Control section](#flow-control) section

### Pipeline Scan
#### Script
The basic script for Pipeline documented __[at the Veracode help center](https://help.veracode.com/r/Run_a_Pipeline_Scan_from_the_Command_Line)__ and in its most basic form it looks as follow.

`java -jar --veracode_api_id "${{secrets.VERACODE_ID}}" --veracode_api_key "${{secrets.VERACODE_KEY}} pipeline-scan.jar --file <file.zip>`

A more advanced and context aware options are documented __[here](https://help.veracode.com/r/r_pipeline_scan_commands)__. 

Within a Github workflow we can add the above scan script right after a step to download the latest Pipeline Scan code.
<details>
<summary>See example</summary>
<p>

```yaml
name: Veracode Pipeline Scan

# Controls when the action will run. 
on:
  # Triggers the workflow on push or pull request events but only for the master branch
  push:
    branches: [ master, release/* ]
  pull_request:
    branches: [ master, develop, main, release/* ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      - name: create dir prior
        run: |
          mkdir dls
      - name: Download pipeline code
        working-directory: ./dls/
        run: |
          curl https://downloads.veracode.com/securityscan/pipeline-scan-LATEST.zip -o veracode.zip
          unzip veracode.zip
      - name: Run Pipeline Scanner
        continue-on-error: true
        run: java -jar ./dls/pipeline-scan.jar --veracode_api_id "${{secrets.VERACODE_ID}}" --veracode_api_key "${{secrets.VERACODE_KEY}}" --file "result.zip" -jo true -so true --project_url https://www.github.com/$GITHUB_REPOSITORY -p $GITHUB_REPOSITORY -r $GITHUB_REF
```
</p>      
</details> 
<br/>

> :point_right: - it is recommended to include the application name and if possible the sandbox or branch name in the pipeline scan attribute to help with the platform reports. [-p \<project name/repository name\>] [-r \<project ref/branch name\>]

### Agent-Based SCA
#### Script
The basic script for Agent-Based SCA CLI (Command line interface) is documented [at the Veracode help center](https://help.veracode.com/r/Using_the_Veracode_SCA_Command_Line_Agent)

The basic form on a PC using cURL is also the simplest way to be use in a CI workflow.

`curl -sSL https://download.sourceclear.com/install | sh`

As all of our integrations, more advanced scan options are documented in our [help center](https://help.veracode.com/r/c_sc_agent_usage)

For Agent-Based SCA (security scanning of 3<sup>rd</sup> party components) solution we have a very simple script which can easily put in a workflow.


<details>
<summary>See example</summary>
<p>

```yaml
name: SCA on change in dependencies definition

# Controls when the workflow will run
on:
  # Triggers the workflow on push where package-lock.json modifies or pull request events
  push:
    paths:
      - 'package-lock.json'
  pull_request:
  
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # The workflow consist of a single job to quickly scan dependencies
  SCA_Scan:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - name: Check out repo 
        uses: actions/checkout@v2

      # run quick scan on the project
      - name: SCA Scan
        env: 
          SRCCLR_API_TOKEN: ${{ secrets.SRCCLR_API_TOKEN }}
        run: curl -sSL https://download.sourceclear.com/ci.sh | sh -s scan --quick
```
</p>      
</details> 
<br/>

> :bulb: The above example script is using a very specific trigger for changes in TypeScript/JavaScript packaging definition file __`package-lock.json`__. Different programming language will require different __`on`__ scan trigger settings.

## Import Findings

With our different Static scanning, we now have the ability to import the findings using different techniques

### Visualize Pipeline Findings as Pull Request comment
The basic form of 'import' is rather a surfacing of the result. When a Pipeline Scan run within a workflow, all messages are kept inside the workflow. However, when we are using the scan as part of a Pull Request, we may want to copy the Scan output as a comment in the Pull Request main `Conversation` tab.

To do that we can utilize build-in GitHub scripting functionality. (This is not a __Veracode__ function)

See __[basic example](https://github.com/Lerer/veracode-pipeline-PR-comment)__


<details>
<summary>Another inline example</summary>
<p>

```yaml
name: Veracode SAST Scan

# Controls when the action will run. 
on:
  # Triggers the workflow on push or pull request events but only for the master branch
  pull_request:
    branches: [ master, develop, main, release/* ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  static_pipeline_scan:
    # The type of runner that the job will run on
    runs-on: [ self-hosted, generic ]
    
    steps:
      - name: create dir prior
        run: |
          mkdir dls
      - name: Download pipeline code
        working-directory: ./dls/
        run: |
          curl https://downloads.veracode.com/securityscan/pipeline-scan-LATEST.zip -o veracode.zip
          unzip veracode.zip
      - name: Run Pipeline Scanner
        continue-on-error: true
        run: java -Dpipeline.debug=true -jar ./dls/pipeline-scan.jar --veracode_api_id "${{secrets.VERACODE_ID}}" --veracode_api_key "${{secrets.VERACODE_KEY}}" --file "result.zip" -jo true -so true
      - id: get-comment-body
        if: ${{ github.head_ref != '' }}
        run: |
          body=$(cat results.txt)
          body="${body//$'===\n---'/'===\n<details><summary>details</summary><p>\n---'}"
          body="${body//$'---\n\n==='/'---\n</p></details>\n==='}"
          body="${body//$'\n'/'<br>'}"
          echo ::set-output name=body::$body
      - name: Add comment to PR
        if: ${{ github.head_ref != '' }}
        uses: peter-evans/create-or-update-comment@v1
        with:
          issue-number: ${{ github.event.pull_request.number }}
          body: ${{ steps.get-comment-body.outputs.body }}
          reactions: rocket
```
<br/>
<p align="center">
  <img src="/media/img/pr-github-comment-pipeline.png" width="700px" alt="Pipeline scan as comment in PR conversation"/>
</p>

</p>
</details>
</br>

> The inline example is from a [an actual pull request](https://github.com/Lerer/verademo-sarif/pull/3)
     
> :bulb: A different output can be done by further/differently manipulating the text output

### Pipeline Scan as GitHub Security issues
For Enterprise GitHug Accounts with a license for 'GitHub Advanced Security', we can import our Pipeline Scan result to the dedicated `Security` issues.

That option is available in GitHub Actions marketplace - __[Veracode Static Analysis Pipeline Scan and SARIF import](https://github.com/marketplace/actions/veracode-static-analysis-pipeline-scan-and-sarif-import)__ 
A (relatively older) use can be found at: [Lerer/verademo-sarif](https://github.com/Lerer/verademo-sarif/security/code-scanning)


<details>
<summary>See example</summary>
<p>

```yaml
name: Pipeline Scan with creation of GitHub Security issues

# Controls when the workflow will run
on:
  # Triggers the workflow on push where package-lock.json modifies or pull request events
  push:
  pull_request:
  
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # The workflow consist of a single job to quickly scan dependencies
  Pipeline_to_Security_Issues:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - name: create dir prior
        run: |
          mkdir dls
      - name: Download pipeline code
        working-directory: ./dls/
        run: |
          curl https://downloads.veracode.com/securityscan/pipeline-scan-LATEST.zip -o veracode.zip
          unzip veracode.zip
      - name: Run Pipeline Scanner
        continue-on-error: true
        run: java -jar ./dls/pipeline-scan.jar --veracode_api_id "${{secrets.VERACODE_ID}}" --veracode_api_key "${{secrets.VERACODE_KEY}}" --file "result.zip" -jo true -so true 
      - name: Convert pipeline scan output to SARIF format
        id: convert Pipeline results to SARIF format 
        uses: Veracode/veracode-pipeline-scan-results-to-sarif@v0.1.2
        with:
          pipeline-results-json: results.json
          output-results-sarif: veracode-results.sarif
          source-base-path-1: "^com/veracode:src/main/java/com/veracode"
          source-base-path-2: "^WEB-INF:src/main/webapp/WEB-INF"
          finding-rule-level: "3:1:0"

      - name: upload SARIF file to repository
        uses: github/codeql-action/upload-sarif@v1
        with: # Path to SARIF file relative to the root of the repository
          sarif_file: veracode-results.sarif
```
</p>
<br/>
<p align="center">
  <img src="/media/img/security-issues-action-pipeline.png" width="700px" alt="Pipeline Scan results as Security issues"/>
</p>

</details> 


### Upload and Scan / Pipeline Scan as Issues
Lastly, for organization who don't have the license for 'GitHub Advanced Security' or simply don't want to use the `Security` issues, they can import the finding as any other issue in the `Issues` section. 

That is achieve by the __[Veracode scan results to GitHub issues](https://github.com/marketplace/actions/veracode-scan-results-to-github-issues)__ action.

Example for such issues can be seen here:
- [https://github.com/julz0815/veracode-flaws-to-issues/issues](https://github.com/julz0815/veracode-flaws-to-issues/issues)

<details>
<summary>Screenshots</summary>
<p>
<p align="center">
  <img src="/media/img/issues-action-pipeline.png" width="600px" alt="Issues created from Pipeline Scan results"/>
</p>
<br/>
<p align="center">
  <img src="/media/img/issues-action-policy.png" width="600px" alt="Issues created from importing Static Scan results"/>
</p>
</p>
</details>

### SCA import findings
Unfortunately, as of now, we don't have an official (or unofficial) __simple__ action to import finding for SCA result - Agent-Based or Upload and Scan.

However here is an __[example](https://github.com/lerer-veracode/verademo-java/blob/test-multi-job/.github/workflows/security_multi_job.yml)__ on how to surface SCA finding from Agent-Based SCA scan within a workflow job.


<details>
<summary>Part of Workflow example</summary>
<p>

```yaml
name: Secure with SCA Agent Based

# Controls when the workflow will run
on:
  # Triggers the workflow on push where package-lock.json modifies or pull request events
  push:
    branches: [test*]
  pull_request:
  
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:

  # The workflow consist of a single job to quickly scan dependencies
  SCA_Scan:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: SCA Scan
        env: 
          SRCCLR_API_TOKEN: ${{ secrets.SRCCLR_API_TOKEN }}
        run: |
          git version
          curl -sSL https://download.sourceclear.com/ci.sh | sh -s -- scan --recursive > veracode-sca-results.txt
      - name: Check for existing Vulnerabilities
        id: check-vulnerabilities
        run: |
          cat veracode-sca-results.txt
          TEMP_VULN_EXISTS=$(grep 'Risk Vulnerabilities' veracode-sca-results.txt | awk '{sums += $4} END { test = (sums>0); print test}')
          TEMP_VULN_SUM=$(grep 'Risk Vulnerabilities' veracode-sca-results.txt | awk '{sums += $4} END { print sums}')
          echo ::set-output name=fail::$TEMP_VULN_EXISTS
          echo ::set-output name=sums::$TEMP_VULN_SUM
      - name: SCA Pass
        if: ${{ steps.check-vulnerabilities.outputs.fail == 1 }}
        uses: actions/github-script@v3
        env:
          VULN_NUM: ${{ steps.check-vulnerabilities.outputs.sums }}
        with:
          script: |
            console.log(process.env.VULN_NUM);
            core.setFailed(`Found ${process.env.VULN_NUM} Risk Vulnerabilities in your open source libraries`);
```
</p>
<p align="center">
  <img src="/media/img/workflow-multi-jobs.png" width="700px" alt="Github workflow summary with multiple jobs"/>
</p>
</details>
<br/>


:exclamation: This is a place-holder for further future implementation.

## Flow Control

This section is mostly around utilizing GitHub build-in functionality to implement a desire process. 

### Branch protection (good practice for Pull Request)
__[Protected Branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/about-protected-branches)__ feature is the answer to a (repeated) question - How can we block a Pull Request for failed scan.

The answer is in a __['Require Status Checks Before Merging' section](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/about-protected-branches#require-status-checks-before-merging)__ in the above manual.

For those with the right GitHub permissions, here are the __[Step-by-Step instructions](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/managing-a-branch-protection-rule)__ for setting up the Branch Protection Rules

If we enable branch protection, any failed `job` within a workflow can prevent pull request from being approved  :arrow_right: We then need to make sure a failed Scan is also failing the job in the workflow:exclamation:
<details>
<summary>Screenshot from Github</summary>
<p>
<p align="center">
  <img src="/media/img/branch-protection-rule.png" width="700px" alt="Github branch protection rule definition example"/>
</p>
<br/>
<p align="center">
  <img src="/media/img/pr-merge-branch-protection.png" width="700px" alt="PR blocked due to failed checks"/>
</p>
</p>
</details>

### Splitting into Jobs - Adding Checks
Another scenario is for example if we want to have parallel different scan processes to achieve:
- Parallel processing - faster overall flaw time
- Or, separate statuses - for example SCA and Static.

> :bulb: Every __job__ in a workflow will report back its own status as a __`check`__.

We can either create separate workflows or we can create a single flow with separate to Jobs.

The following example is taken from __[this implementation](https://github.com/lerer-veracode/verademo-java/blob/test-multi-job/.github/workflows/security_multi_job.yml)__


<details>
<summary>Example for flaw output with multiple jobs</summary>
<p align="center">
  <img src="/media/img/workflow-multi-jobs.png" width="700px" alt="Github workflow summary with multiple jobs"/>
</p>
</details>

<details>
<summary>Workflow example with parallel Agent-Based and Pipeline Scan</summary>
<p>

```yaml
name: Secure with multiple separate jobs

# Controls when the workflow will run
on:
  # Triggers the workflow on push where package-lock.json modifies or pull request events
  push:
    branches: [test*]
  pull_request:
  
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  Build:
    runs-on: [ubuntu-latest]
    steps:
      - name: Check out repo 
        uses: actions/checkout@v2
      - name: Set up JDK 1.8
        uses: actions/setup-java@v1
        with:
          java-version: 1.8
      - name: Build with Maven
        working-directory: ./app/
        run: mvn -B package --file pom.xml
      - name: Zip Files
        uses: vimtor/action-zip@v1
        with:
          files: ./app/target/verademo.war
          dest: result.zip
      - uses: actions/upload-artifact@v2 # Copy files from repository to docker container so the next uploadandscan action can access them.
        with:
          path: result.zip # See: https://github.com/actions/upload-artifact
          name: built-zip
        
  generate-sandbox-name:
    runs-on: [ ubuntu-latest ]
    outputs:
      sandbox-name: ${{ steps.set-sandbox-name.outputs.sandbox-name }}
    steps:
      # Creates the sandbox(logical release descriptive status of current branch)
      - id: set-sandbox-name
        name: set-sandbox-name
        run: |
          echo ${{ github.head_ref }}
          branchName="${{ github.head_ref }}"
          if [[ -z "$branchName" ]]; then
            branchName="${{ github.ref }}"
          fi
          echo "::set-output name=sandbox-name::$branchName"
  
  Pipeline_Scan:
    runs-on: ubuntu-latest
    needs: [ Build, generate-sandbox-name ]
    steps:
      - uses: actions/download-artifact@v2
        with:
          name: built-zip
      - name: create dir prior
        run: |
          mkdir dls
      - name: Download pipeline code
        working-directory: ./dls/
        run: |
          curl https://downloads.veracode.com/securityscan/pipeline-scan-LATEST.zip -o veracode.zip
          unzip veracode.zip
      - name: Run Pipeline Scanner
        run: java -jar ./dls/pipeline-scan.jar --veracode_api_id "${{secrets.VERACODE_ID}}" --veracode_api_key "${{secrets.VERACODE_KEY}}" --file "result.zip" -jo true -so true
       
  # The workflow consist of a single job to quickly scan dependencies
  SCA_Scan:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
    needs: [ Build, generate-sandbox-name ]
    steps:
      - uses: actions/checkout@v2
      - name: SCA Scan
        env: 
          SRCCLR_API_TOKEN: ${{ secrets.SRCCLR_API_TOKEN }}
        run: |
          git version
          curl -sSL https://download.sourceclear.com/ci.sh | sh -s -- scan --recursive > veracode-sca-results.txt
      - name: Check for existing Vulnerabilities
        id: check-vulnerabilities
        run: |
          cat veracode-sca-results.txt
          TEMP_VULN_EXISTS=$(grep 'Risk Vulnerabilities' veracode-sca-results.txt | awk '{sums += $4} END { test = (sums>0); print test}')
          TEMP_VULN_SUM=$(grep 'Risk Vulnerabilities' veracode-sca-results.txt | awk '{sums += $4} END { print sums}')
          echo ::set-output name=fail::$TEMP_VULN_EXISTS
          echo ::set-output name=sums::$TEMP_VULN_SUM
      - name: SCA Pass
        if: ${{ steps.check-vulnerabilities.outputs.fail == 1 }}
        uses: actions/github-script@v3
        env:
          VULN_NUM: ${{ steps.check-vulnerabilities.outputs.sums }}
        with:
          script: |
            console.log(process.env.VULN_NUM);
            core.setFailed(`Found ${process.env.VULN_NUM} Risk Vulnerabilities in your open source libraries`);
```
</p>
</details>

<details>
<summary>Example for how check will show as part of Pull Request</summary>
<p align="center">
  <img src="/media/img/pr-multi-jobs.png" width="600px" alt="Github pull request checks with multiple jobs"/>
</p>
<center><b> The above example is showing a Pull request without Branch Protection enabled</b></center>
</details>

### Auto Pull request for vulnerable dependencies after merge into Main/Master
Right after a pull request is approved, the target branch will issue a `push` event (as new code is merged). This is an opportunity to run a scan to automatically produce a Pull request for known 3<sup>rd</sup> party components vulnerabilities (based on Vulnerable Methods or Severity of Vulnerabilities found in a scan).

__[Veracode SCA Auto pull request](https://help.veracode.com/r/t_configure_auto_pr)__ only apply to [supported languages](https://help.veracode.com/r/Understanding_Automatic_Pull_Request_Support): Java, Python, Ruby, JavaScript, Objective-C, and PHP 

Make sure to read the instructions as it involves [creation of Github token](https://help.veracode.com/r/t_configure_pr_github) 

See pull Request example:
- [https://github.com/lerer-veracode/verademo-java/pull/1](https://github.com/lerer-veracode/verademo-java/pull/1)

<details>
<summary>See inline workflow</summary>
<p>

```yaml
name: Auto Pull Request on SCA findings

# Controls when the workflow will run
on:
  # Triggers the workflow on push where package-lock.json modifies or pull request events
  push:
    branches: [ master,main ]
  
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # The workflow consist of a single job to quickly scan dependencies
  SCA_Scan for PR:
    # The type of runner that the job will run on
    runs-on: [ self-hosted, generic ]

    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - name: Check out repo 
        uses: actions/checkout@v2

      # run quick scan on the project
      - name: SCA Scan
        env: 
          SRCCLR_API_TOKEN: ${{ secrets.SRCCLR_API_TOKEN }}
          SRCCLR_SCM_TOKEN: ${{ secrets.SRCCLR_GITHUB_TOKEN }}
          SRCCLR_SCM_TYPE: GITHUB
          # PR on: methods
          SRCCLR_PR_ON: low
          SRCCLR_NO_BREAKING_UPDATES: true
          SRCCLR_IGNORE_CLOSED_PRS: true
          EXTRA_ARGS: '--update-advisor --pull-request --unmatched'
        run: |
          git config --global user.email "${{ secrets.USER_EMAIL }}"
          git config --global user.name "${{ secrets.USER_NAME }}"
          curl -sSL https://download.sourceclear.com/ci.sh | sh -s -- scan $EXTRA_ARGS
```

<br/>
<p align="center">
  <img src="/media/img/auto-pr-sca.png" width="700px" alt="Policy scan output as GitHub Issue"/>
</p>

</p>      
</details> 


## Scaling in an Organization
For larger organization who seek more help on how to utilize the above suggestions and ease the onboarding process of their different teams - to encouraging faster and easier adoption of DevSecOps.

### Shared workflow templates
Workflow define and save in each repository, however, instead of copy-paste for every repository we can use GitHug shared workflows by creating `.github` repository in the account.

- See detailed information at the __[Shared workflows](https://docs.github.com/en/actions/learn-github-actions/sharing-workflows-with-your-organization)__ documentation

In addition to shared workflows, you should also consider Organization shared Secrets

> :exclamation: Keep in mind - Pipeline Scan throughput limited to  6 scans/min

See following shared workflows example:
- [https://github.com/lerer-veracode/.github](https://github.com/lerer-veracode/.github/tree/main/workflow-templates)
- [https://github.com/tjarrettveracode/.github](https://github.com/tjarrettveracode/.github)

<details>
<summary>How does it look like in workflow creation screen</summary>
<p align="center">
  <img src="/media/img/new-workflow-screen.png" width="800px" alt="New workflow screen"/>
</p>
<center>For the above example, the Organization name is: <b>lerer-veracode</b></center>
</details>

### Use Pipeline scan baseline
Pipeline scan provides the ability to use baseline acting as the "approved mitigations" or accepted risk level which instruct the Pipeline Scan to return finding other than the ones in the provided baseline.

It can be useful to re-base the Baseline file after approving a Pull Request 
> Only if we __OK__ to look at Pull Request approval also as an __`Approval`__ for accepting security risk going into certain (main/master) branches.

An option to handle such a rebase is to commit the baseline as a file in the repository. The advantage in this option is to have the baseline file available to developers working with the code and synchronize the baseline with their local cloned repository.

<details>
<summary>Example - commit baseline file after Approved Pull request to main/master</summary>
<p>

```yaml
name: Push baseline file

# Controls when the workflow will run
on:
  push:
    branches: [master , main]
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:  
  Build:
    runs-on: [ubuntu-latest]
    steps:
      - name: Check out repo 
        uses: actions/checkout@v2
      - name: Set up JDK 1.8
        uses: actions/setup-java@v1
        with:
          java-version: 1.8
      - name: Build with Maven
        working-directory: ./app/
        run: mvn -B package --file pom.xml
      - name: create dir prior
        run: |
          mkdir dls
      - name: Download pipeline code
        working-directory: ./dls/
        run: |
          curl https://downloads.veracode.com/securityscan/pipeline-scan-LATEST.zip -o veracode.zip
          unzip veracode.zip
      - name: Run Pipeline Scanner
        # Baseline should only be created when using filtered results without baseline file as input
        run: |
          java -jar ./dls/pipeline-scan.jar --veracode_api_id "${{secrets.VERACODE_ID}}" --veracode_api_key "${{secrets.VERACODE_KEY}}" --file "app/target/verademo.war" -fs "Very High, High" -jo true -so true --project_url https://www.github.com/$GITHUB_REPOSITORY -p $GITHUB_REPOSITORY -r $GITHUB_REF 
        continue-on-error: true
      - name: list files
        run: |
          yes | cp filtered_results.json baseline.json
          git config user.email "${{secrets.USER_EMAIL}}"
          git config user.name "${{secrets.USER_NAME}}"
          git add "baseline.json"
          git commit -m "Updates baseline"
          # git push origin ${{ github.head_ref }}
          git push origin ${{ github.ref }}          
```
</p>
</details>
<br/>

> :bulb: Do not use a baseline file in the Pipeline scan command in order to generate a new baseline file, as the __`filtered_results.json`__ file will not output what is needed as a new baseline file 


### Shared Downloaded Policies for Pipeline Scan
Pipeline scan today can __[download a policy file](https://help.veracode.com/r/Using_Policies_with_the_Pipeline_Scan)__ based on policies defined in the Veracode Platform. We can subsequently use a downloaded policy as an input to a Pipeline scan as a fail/pass condition - similar to the Upload-and-Scan logic.

However, we don't recommended downloading the policy file every time we run a pipeline scan:
- Policies are rarely changed (compared to the scanned code) 
- Policies are shared across multiple Application profile - hence, shared between multiple repositories 
  
...maybe we can store it in a shared location...and alias their name to protect from policy name change...

We can control and only refresh policy files once a day and store in a shared location to be used by any other repository workflow.

Here is an option:
1) Let's download and store policies using a nightly scheduled workflow to a  `shared` repository
2) We will download the policy from the shared repository prior to running pipeline scan in a workflow
3) We will template the above for the easy import from any other repository

<details>
<summary>Storing policies in a shared Repository</summary>
<p>

```yaml
name: Update Latest Policies for Veracode Pipeline Scanner 
on:
  workflow_dispatch:
  
  schedule:
    - cron: 30 14 * * *

jobs:
  download_policies_global:
    runs-on: ubuntu-latest
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v2
      - name: create dir prior
        run: |
          git pull
          mkdir dls
      - name: Download pipeline code
        working-directory: ./dls/
        run: |
          curl https://downloads.veracode.com/securityscan/pipeline-scan-LATEST.zip -o veracode.zip
          unzip veracode.zip
      - name: Download Policy High Risk
        continue-on-error: true
        run: java -jar ./dls/pipeline-scan.jar --veracode_api_id "${{secrets.VERACODE_ID}}" --veracode_api_key "${{secrets.VERACODE_KEY}}" --request_policy "Fix High Severity - non scored"
      - name: Download Policy Highly Sensitive
        continue-on-error: true
        run: java -jar ./dls/pipeline-scan.jar --veracode_api_id "${{secrets.VERACODE_ID}}" --veracode_api_key "${{secrets.VERACODE_KEY}}" --request_policy "Acceptable quality"
      - name: Clean-ups and alias
        run: |
          echo "Remove veracode files"
          rm -rf dls/*
          mkdir -p sec
          mv -f Fix_High_Severity_-_non_scored.json sec/High.json
          mv -f Acceptable_quality.json sec/HighlySensitive.json
      - name: Push the policies
        continue-on-error: true
        run: |
          git config user.email "${{secrets.USER_EMAIL}}"
          git config user.name "${{secrets.USER_NAME}}"
          git add "sec/High.json"
          git add "sec/HighlySensitive.json"
          git commit -m "Updates Policies"
          git push origin ${{ github.ref }}       
```
</p>
</details>

<details>
<summary>Downloading the policy from another workflow</summary>
<p>

```yaml
name: Pipeline Scan with remote policy

# Controls when the workflow will run
on:
  push:
  pull_request:
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:  
  Build:
    runs-on: [ubuntu-latest]
    steps:
      - name: Check out repo 
        uses: actions/checkout@v2
      - name: Set up JDK 1.8
        uses: actions/setup-java@v1
        with:
          java-version: 1.8
      - name: Build with Maven
        working-directory: ./app/
        run: mvn -B package --file pom.xml
      - name: create dir prior
        run: |
          mkdir dls
      - name: Download pipeline code
        working-directory: ./dls/
        run: |
          curl https://downloads.veracode.com/securityscan/pipeline-scan-LATEST.zip -o veracode.zip
          unzip veracode.zip
      - name: Get policy file
        run: | 
          curl https://raw.githubusercontent.com/lerer-veracode/policies/main/sec/High.json -o policy.json
      - name: Run Pipeline Scanner
        # Baseline should only be created when using filtered results without baseline file as input
        run: |
          java -jar ./dls/pipeline-scan.jar --veracode_api_id "${{secrets.VERACODE_ID}}" --veracode_api_key "${{secrets.VERACODE_KEY}}" --file "app/target/verademo.war" --policy_file "./policy.json" -jo true -so true --project_url https://www.github.com/$GITHUB_REPOSITORY -p $GITHUB_REPOSITORY -r $GITHUB_REF 
      
```
</p>


</details>

<details>
<summary>Downloading the policy from another workflow - as template (Java)</summary>
<p>

```yaml
name: Pipeline Scan with remote policy

# Controls when the workflow will run
on:
  push:
  pull_request:
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:  
  Build:
    runs-on: [ubuntu-latest]
    steps:
      - name: Check out repo 
        uses: actions/checkout@v2
        # Replace compilation for different languages
      - name: Set up JDK 1.8
        uses: actions/setup-java@v1
        with:
          java-version: 1.8
      - name: Build with Maven
        working-directory: ./app/
        run: mvn -B package --file pom.xml
      - name: create dir prior
        run: |
          mkdir dls
      - name: Download pipeline code
        working-directory: ./dls/
        run: |
          curl https://downloads.veracode.com/securityscan/pipeline-scan-LATEST.zip -o veracode.zip
          unzip veracode.zip
      - name: Get policy file
        # Remove all import policies which are not relevant to your code and
        # keep only the line with the policy relevant to your code in the repository
        run: | 
          curl https://raw.githubusercontent.com/lerer-veracode/policies/main/sec/HighlySensitive.json -o policy.json
          curl https://raw.githubusercontent.com/lerer-veracode/policies/main/sec/High.json -o policy.json
      - name: Run Pipeline Scanner
        # Modify target scan file (here "app/target/verademo.war") for different code base or languages 
        run: |
          java -jar ./dls/pipeline-scan.jar --veracode_api_id "${{secrets.VERACODE_ID}}" --veracode_api_key "${{secrets.VERACODE_KEY}}" --file "app/target/verademo.war" --policy_file "./policy.json" -jo true -so true --project_url https://www.github.com/$GITHUB_REPOSITORY -p $GITHUB_REPOSITORY -r $GITHUB_REF 
      
```
</p>

</details>
<br/>

> :bulb: The last example is what you would put in the workflows shared via your organization __`.github`__ repository.


## Places to contribute

- SCA Agent Based Action :arrow_right: Create Issues from Vulnerabilities, Licenses, Outdated libraries
- Generation of Pipeline Scan Baseline file from Static Profile mitigations
  - Some work was done by Tim Jarrett [here](https://github.com/tjarrettveracode/veracode-pipeline-mitigation) using Python script which can be leveraged
- Policy/Sandbox scan :arrow_right: Summary report action
