# Securing Your Software Supply Chain Workshop

Welcome to the Software Supply Chain Workshop, focused on enhancing your skills in securing the software supply chain using open source tools. In this hands-on workshop, participants will engage in practical exercises designed to simulate real-world scenarios involving breached systems, vulnerabilities, and their corresponding solutions. Through these exercises, you'll learn how to identify, mitigate, and defend against security threats within the software supply chain.

## Prerequisite
 - Fork the repository into your Organization
 - On your Forked Repository Go to "Settings" Tab
 - Go to Pages and in the "Source" field choose "Github actions"
 - Go to "Actions" Tab and click "I understand my workflows, go ahead and enable them"
 - Go to "Deploy Page", click on "Run Workflow" and choose from "main" branch
 - Go to your Organization settings, click Action->General->Workflow permissions,  make sure `Allow GitHub Actions to create and approve pull requests` is selected
 - Go to your Repository settings, click Action->General->Workflow permissions, make sure `Allow GitHub Actions to create and approve pull requests` is selected
 - Wait 2 min for the Action Run to be completed
 - Go to the link under the deploy step, or directly to this address `https://<USER>.github.io/<REPOSITORY>`
 - Make sure "you have been pwned"


## Phase 1: Secrets Exposed

### Scenario
In this phase, we'll explore a scenario where a secret of a stale collaborator has been exposed within the repository, similar to the Toyota data breach incident mentioned in [this article](https://www.spiceworks.com/it-security/data-security/news/toyota-data-breach). Your task is to identify and address this security breach.

### Tasks
1. **Find and Revoke Secrets**
   - Create a new pipeline named `01.01 - Exposed Credential.yml` under `.github\workflows` that scan the repository to locate exposed secrets with Trivy Action
```
name: 01.01 - Detect Exposed Credential
on: 
  workflow_dispatch

jobs:
  build:
    name: Secret Scanner
    runs-on: ubuntu-latest
    env:
      IMAGE_NAME: ${{ github.repository }}
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Run secret scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: "fs"
        format: 'table'
        exit-code: '1'
        scanners: 'secret'
```
   - Run the new workflow and find the credential within the workflow logs

2. **Implement Secret Scanner**
   - Configure this workflow to run on every PR Creation based `main` branch by changing the `on` field to:
```
on: 
  pull_request: 
         branches: ["main"]
  workflow_dispatch:
```

3. **Strengthen Main Branch Protection**
   - Configure branch protection to enforce and block secrets on every merge request to `main` branch
     
4. **Remove Compromised Secret**
   - Make necessary code changes in order to remove the exposed secret from the `main` branch.
  

## Phase 2: Compromised CI/CD

### Scenario
In this phase, we'll delve into a scenario where an attacker compromised and manipulate one of the pipeline dependencies we are using. Your objective is to detect and remediate this compromise.

### Tasks
1. **Implement Tracee in CI/CD**
   - Integrate Tracee into the [deploy-gh-pages.yml](.github/workflows/deploy-gh-pages.yml) workflow to detect malicious activities.
     - create a new branch
     - add start tracee step after the `Checkout` step name
   ```
      - name: Start Tracee profiling in background
        uses: aquasecurity/tracee-action@v0.4.0-start
   ```
     - add stop after the `Upload artifact` step name, and allow pr creation when there is profile updates
   ```
      - name: Stop and Check Tracee results and create a PR
        uses: aquasecurity/tracee-action@v0.4.0-stop
        with:
          create-pr: "true"
          fail-on-diff: "false"
   ```
     - Add permission to PR creation by adding `pull-requests: write`  under permission:
   ```
   permissions:
     contents: write
     pages: write
     id-token: write
     pull-requests: write
   ```
    - Push changes to a branch and create the PR
   

2. **Identify and Fix Malicious Payload**
   - Review Tracee Action Action [Documentation](https://github.com/aquasecurity/tracee-action)
   - In order to identify the malicious payload introduced, analyze the PR comments (which include the signatures alerted)
   - a new PR has been created by Tracee named `Updates to tracee profile` which include the CI Runtime Profile, Analyze the different profile in order to find the malicious behaviour(e.g malicious dns)
   - look clously on [deploy-gh-pages.yml](.github/workflows/deploy-gh-pages.yml) steps and dependencies and find the malicious step
   - Fix the malicious step by pin it to the latest sha instead of tag
   - Analyze profiles files again to verify the malicious payloads removed
   - Merge both PRs

4. **Analyze The Malicous Source**
   - Find the malicious commit in the compromised dependency source code you found earlier
     
5. **Enforce Signed Commits**
   - Restrict the acceptance of commits to the 'main' branch to only signed commits, by configure it under the branch protection rule

6. **Bonus - Poison Dependency**
   - Create a new Github Action with legit content
   - Publish it
   - Add it into your [deploy-gh-pages.yml](.github/workflows/deploy-gh-pages.yml) with the Action Tag
   - Review the PR comments and the profile updates & merge
   - In your brand new github action Add a new command that creating new files\override exsiting ones
   - Push it into the exsiting version
   - In your workshop repository create any new change and Open PR
   - Make sure you see the both PR comments with signiture and Profile deviation updates

## Phase 3: Critical Vulnerability Emerged
### Scenario
In this phase, we'll delve into a scenario where an emerging critical vulnrability published and we are in a rush to identify where it used and mitigate the risk.

### Tasks
1. **Scan Dependency Vulnerability**
   - Add a new workflow with vulnerablity scanner for you code dependencies:
```
name: 03.01 - Run Vulnrabilities Scanning
on:
  push:
    branches:
      - main
permissions:
  security-events: write
  
jobs:
  build:
    name: Vulnerabilities-Scanner
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Run vulnerabilities scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: "fs"
        format: 'sarif'
        ignore-unfixed: true
        scanners: 'vuln'
        exit-code: '0'
        severity: 'CRITICAL,HIGH,MEDIUM'
        output: 'trivy-results.sarif'
```
2. **Upload results to Security Tab**
   - At the bottom & after the scanning part add a new step to upload Trivy results to Security Tab
```
    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
        category: Trivy
```
  - create PR and merge into main branch
3. **Check Security Tab**
   - Make sure you see new record in Security Tab, review the Vulnerability
4. **Vulnerability Checks for PRs**
   - Duplicate the previus workflow
   - Change trigger to be on Pr creation 
```
on:
  pull_request:
    branches:
      - main
```
   - Change the build->name field so you can specify it later in the branch protection rules
   - Remove permissions
   - Change format field to `table`
   - Remove output field
   - Change `exit-code: '1'`
   - Remove Upload action step
   - Add to the branch protection enforcment for vulnerability scanner with the new name

5. **Fix Vulnrable Dependency**
   - Fix vulnerable dependencies in `package.json` and don't forget to update `package-lock.json` by `npm i --package-lock` inside `my-app` directory 
7. **Bonus - Add a cron job trigger that will run the vulnerability scanner every night - Bonus**
   - Add a new schedule based trigger into your workflow 
9. **Enfore Code Reviews**
   - Add to the branch protection rule on the main branch, request to review and approve by other contributors before changes are merged.
