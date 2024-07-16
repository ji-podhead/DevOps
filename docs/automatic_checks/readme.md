
# DevOps



| [Guides](https://ji-podhead.github.io/DevOps/)  |  [Projects](https://github.com/ji-podhead/DevOps/blob/main/readme.md#projects)  |

<div align="center">
      <img src="https://github.com/ji-podhead/ji-podhead/blob/main/logo.jpg?raw=true" align="right" width="100" />
</div>

This is my DevOps Repo.  <br> This serves as my Playground, Portfolio, Research-Notebook, Cheat-Sheet and Guide. 

## Guides

| [TerrForm & Vault](https://ji-podhead.github.io/DevOps/guides/terraform&vault/) |  [Terraform & Proxmox](https://ji-podhead.github.io/DevOps/guides/terraform%26proxmox)   |

---

### Automated Infrastructure as Code (IaC) Status Checks, Rules, and Test Results

- Modern DevOps practices often adopt a "shift left" approach to security, which involves:
  - Utilizing static configuration scanners every time someone attempts to push changes to our development or production branches.
  - This proactive measure allows us to identify and address issues early on, preventing the accumulation of complex spaghetti code in later stages.
  - For further reading on this topic, refer to this Red Hat blog post.
- Our goal is to scan IaC configurations prior to merging them and then share the test results through the GitHub Actions plugin. We will block any merges if the tests fail.

***Preview***

 ![check](https://github.com/ji-podhead/DevOps/blob/main/docs/automatic_checks/statuscheck.png?raw=true)
> [this is the pull request](https://github.com/ji-podhead/Pod-Shop-App-Configs/pull/26), but i closed it

---

***Action Settings***
   - in order to use the [publish test results plugin](https://github.com/marketplace/actions/publish-test-results) in our workflow we need to set the required `Action Permissions`
   - to comment in the pull request and post testresults via [API-call](https://docs.github.com/de/rest/pulls/comments?apiVersion=2022-11-28), we need to set the required `Workflow Permission`

![action-settings](https://github.com/ji-podhead/DevOps/blob/main/docs/automatic_checks/action%20settings.png?raw=true)


---

***add our checkov workflow***
> - checkov is currently failing on its own job  due to permission
> - this will be fixed so check out my  ![review](https://github.com/ji-podhead/Pod-Shop-App-Configs/pull/25#discussion_r1679061282)  and [my related issue](https://github.com/ji-podhead/Pod-Shop-App-Configs/issues/27) 

 - we use `junitxml` as our testresult format
   - [the publish test result plugin cant use the jsonformat checkov creates
     - see [generating-test-result-files](https://github.com/marketplace/actions/publish-test-results#generating-test-result-files) and [checkov cli output flag](https://www.checkov.io/2.Basics/CLI%20Command%20Reference.html)
   - we need to clculate the duration manualy  
- dont forget to use `if: always()` for publishing test results, even if the tests fail
     
```yaml
name: checkov

on:
  push:
    branches: [ "main", "development" ]
  pull_request:
    branches: [ "main", "development" ]
  workflow_dispatch:
#permissions:
#  contents: read
#  actions: read
#  checks: write
#  packages: write
permissions: write-all                                          # required permissions for commenting via API-call
jobs:
  checkov_scan_and_upload_results:                              # this is the name of status check that we will use in our branch rule
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
        
      - name: Get Start time
        id: starttime
        if: always()
        run: |
            start=$(date +%s)
            echo "$start" > start_time.txt                         # i couldnt safe start time as env variable so i used cat
      - name: Setup Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.x'

      - name: Install dependencies                                # install the dependencies (you could also use a preinstalled docker container)
        run: |
          pip install checkov
          pip install gitpython

      - name: Run Checkov scan                                    # we run our scan and output to xml 
        run: |
          checkov --directory .  -o junitxml > checkov_results.xml                   

      - name: Calculate Execution Time and Update JUnit XML      # we calculate the duration and update xml using regex
        if: always()
        run: |
            start=$(date +%s)
            echo "starttime is" $start
            
            END_TIME=$(date +%s)
            echo "endtime is: " $END_TIME
            read -r START_TIME < start_time.txt
            START_TIMER=$((START_TIME)) # Konvertiere START_TIME in Millisekunden
            EXECUTION_TIME=$(( (END_TIME - START_TIMER) /2 ))
            echo "runtime is: " $EXECUTION_TIME
            sed -i "s/time=\"0\"/time=\"$EXECUTION_TIME\"/" checkov_results.xml

      - name: Publish Test Results                              # publish the results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always() 
        with:
          files: checkov_results.xml
      - name: Display Directory Contents and File Content        # this is just for testing the job
        if: always()
        run: |
            echo "Listing directory contents:"
            ls -la          
            echo "Displaying"
            cat ./checkov_results.xml
```
---

***add a Branch-Rule for required Status Checks***
- add a target branch
- search and add our job called `checkov_scan_and_upload_results` in *Status Checks that are required*
![rule](https://github.com/ji-podhead/DevOps/blob/main/docs/automatic_checks/branch-rule.png?raw=true)

---

thats it, you made it trough the tutorial! 
happy devopsing <3
