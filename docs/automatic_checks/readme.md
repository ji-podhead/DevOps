
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




- ***adding the required rules:***
