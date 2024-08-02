# cheatsheets & Unreleased Guides

## Jenkins & Github
***jenkins has a weird way of signing in to git.***
- 1. create a pat via dev settings in git
  2. add ajenkins credential
    - ***you need to select  to select `USERNAME+PASSWORD`***
    - add your git user name  and paste the pat into the password field
    - ***DO NOT USE SSH + PASSWORD***

---
## Debug Github Actions in VS Code
### commit and trigger the workflow with one command in vs code terminal
```bash
git add . && git commit -m "Your Commit-Message" && git push 
```
### use the extension to open your workflow with 2 clicks
- refresh the window
- open the created workflow in your browser
  
![extensions](https://github.com/ji-podhead/DevOps/blob/main/docs/debug_actions.png?raw=true)


- additioinaly you can use multiple plugins that seem to wrap some api like around the workflow (like [vs-code-server-action](https://github.com/marketplace/actions/vs-code-server-action), or run your workflow locally using something like [GAL ](https://marketplace.visualstudio.com/items?itemName=alphaxek.github-actions-locally).
  - but those plugins seem somewhat overkill/to hard to install when you just want to have some quick edits
  - GAL even requires docker and the other one runs in codespace
- so i figured it's just easier to use the basic actions extension
  - it kinda sucks that the link to your workflow uses the browser instead of vscode webview pane



---

## Github Organisation Pats
this is kinda confusing at first:
  - you have to allow the creation of pats in the organisation settings
  - go to your ***personal*** developer settings where you usually create access keys
  - create a net pat, but choose the target organisation
  - go back to your organisation and allow the pat, if a user is not admin and you set *Require administrator approval* to true

--- 
## clone another (private) repo in gh actions using checkout
when you clone another repo into your current branch in gh actions and your token is used  somewhere in your workflow for the current scope: 
  - ***you have to use a second token for the private repo***
    
this can be confusing, so make sure to [check out the docs](https://github.com/actions/checkout?tab=readme-ov-file#checkout-multiple-repos-private)

```yaml
jobs:
  merge_from_private:
    runs-on: ubuntu-latest
    permissions: write-all 		# not recommended, just for testing!!!!
    steps:
      - name: Checkout					# <-- the-pod-shop/Pod-Shop-App-Configs 
        uses: actions/checkout@v4
        with:
          path: main
      
      - name: Checkout another private repository      
        uses: actions/checkout@v4
        with:
          repository:  the-pod-shop/App-Configs-private # <-- the-pod-shop/App-Configs-private
          token: ${{ secrets.ACTIONS_KEY }}             # <---other scope---.		    .
          path: main							   #|
#									   #|
#							                   #|
      - name: Push to target repo					   #|
        uses: ad-m/github-push-action@master				   #|
        with:								   #|
          github_token: ${{ secrets.API_TOKEN_GITHUB }} # <--current scope--*
          repository: the-pod-shop/Pod-Shop-App-Configs # <-- the-pod-shop/Pod-Shop-App-Configs
          branch: merge_private
          directory: ./
```

---

## how can sensible data get leaked on github trough terraform?
I noticed that under some circumstances you can leak your secrets and it allmost happen to me. There are some protection measures like git warning you about exposing secrets when trying to push. However you can still expose data that might not get registered as secret (vulnerable ips, ports, or code that was never meant to be public), or secrets will simply not getting registered as such.<br>
****possible outcome:****
  - phishing
  - ddos
  - enumeration
  - social engineering
  - many many more
    
**Heres how i can proove it:**
> ***tests fail in a public repo, but the commit is still publicly available***

- write a bash script, call it with local_exec resource and `echo <secret> > file.txt`
  - if you forget to encrypt this file, or did'nt put it on .gitignore:
    - ***your created file will get exposed on github***
- use data instead of variable to get vault secrets
  - if you accidentaly pull w and then push back with no `.gitignore` included in the branch:
    - ***your entire tfstate file and all of your files you have in .gitignore will get exposed on github***
- you renamed your tfstate file, or gave it another filetype

those are just 3 examples, but the tragic reality is that whoever contributes to your public repo can fuck things up if you dont take care about this
>  1. someone exposes and iprange of your acl to a public exposed server
>     - eg. tailscale funnel + web hook-, dns-, or whatever server
>  2. someone created an environment variable for a password instead of sensible variable, or he exposes data that you dont scan detect, but it's a risk for him
### zero trust approach for ica configs and public git repos
Luckily we have a few options that gives us a 99% chance, that no sensible data will never get exposed (if we do t right) <br>
Here are my ideas:
1. ***encrypt your files before push*** `not very safe`
   - a script that you call before even commiting (or with prereceive hooks) can scan for vulnerable data/code and encrypt your code/data
   - **Downside:** you dont know if its ever get called *(maybe somebody just ignores to call it, or he simply just forgot about it)*  
2. ***use a private repo for iac configs*** `safe`
   - you can mirror to a public repo using a  bot (eg. github actions) once you approved and your static tests are all passed
   - ~you cant have branch protection rules for a private repo on gh free plan however~
     - alternatives:
       - *gitea/foregejo:* free but selfhosted
       - *gitlabs:* free but limited to 2000 compute minutes
3. ***use prereceive hooks and github enterprise server*** `safe`
   - github prereceive hooks let us block an incomming push and create a pr in a public repo (code is sealed)
     - we delete the workspace folder but push into our private repo before. 
   - you can review after testing and merge back into the public repo
     - this way we can seal the code in the public repo until approved and our bot will do the rest 
4. ***github enterprise secret scanning*** `not very safe`
   - its cool that we have that feature, but lets be honest:
     - vulnerable ips, hostnames, emails, or usernames are not getting scanned here, so this requires additional review or manual scanning 
---

## tailscale and github action 


***related tailscale docs*** <br>
| [tailscale & github actions](https://tailscale.com/kb/1276/tailscale-github-action#how-it-works) | [oauth clients](https://tailscale.com/kb/1215/oauth-clients) | [ACL-tags](https://tailscale.com/kb/1068/acl-tags) |

---

- ***1. add a oauth client in tailscale***

---

- ***2. create at least one ACL tag in tailscale*** 
    ```yaml
  ...
  // 	],
	// },
	"tagOwners": {
	"tag:ci": ["ji-podhead@github"],
	},
	// Define users and devices that can use Tailscale SSH.
	"ssh": [
  ...
  ```    
  - without `tags: tag:ci` we get this error:
      - ```err 
        "::error title=⛔ error hint::OAuth identity empty, Maybe you need to populate it in the Secrets for your workflow, see more in https://docs.github.com/en/actions/security-guides/encrypted-secrets and https://tailscale.com/s/oauth-clients"
        ```
---

- ***3. add a little server on your local machine***
```python
import http.server
import socketserver

# Pfad zum Verzeichnis, das als Root für den Server dient
PORT = <port>
DIRECTORY_PATH = "/home/ji/Dokumente/python scripts"

class MyHttpRequestHandler(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        if self.path == '/':
            self.send_response(200)
            self.send_header('Content-type', 'text/html')
            self.end_headers()
            message = f"Hello, this is my custom server running on port {PORT}!"
            self.wfile.write(bytes(message, "utf8"))
        else:
            # Für andere Pfade wird der Inhalt des angegebenen Verzeichnisses zurückgegeben
            super().do_GET()

with socketserver.TCPServer(("", PORT), MyHttpRequestHandler) as httpd:
    print(f"Serving at port {PORT}")
    httpd.serve_forever()
```
  - keep in mind to use iptable/firewalld/etc to enable the port, or just stop the firewall service(*not the best practice*)

---

- ***4. add your job***
```
name: CI
on:
  push:
    branches: [ "testjob" ]					# <<<--- add your branches
  pull_request:
    branches: [ "testjob" ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: greetings
        run: |
          echo
          echo "----------------------------------------------------"
          echo "    .------------------------------------.  "
          echo "   |         > running testjob <          |  "
          echo "    °------------------------------------°   "
          echo "----------------------------------------------------"

      - name: Setup Tailscale
        uses: tailscale/github-action@v2
        with:
          oauth-client-id: ${{ secrets.TS_OAUTH_CLIENT_ID }}	# <<<--- add you tailscale creds to your gh secrets
          oauth-secret: ${{ secrets.TS_OAUTH_SECRET }}
          tags: tag:ci                                        	# <<<--- THIS IS IMPORTANT

      - name: ping                                           	# <<<--- LETS PING A SERVER FOR TESTING
        if: always()
        run: |
          curl http://${{ secrets.TAILSCALE_IP }}:${{ secrets.TAILSCALE_SERVER_PORT_1 }}

      #- name: SSH into remote server
      #  run: |
      #    ssh <user>@${{ secrets.TAILSCALE_IP}}
```

---

- ***5 run your job***
```bash
#--------------------------------------------------------------------------------
# ✅ greetings
#0s
#Run echo
----------------------------------------------------
    .------------------------------------.  
   |         > running testjob <          |  
    °------------------------------------°   
----------------------------------------------------
#--------------------------------------------------------------------------------
# ✅ Setup Tailscale
#8s
Run tailscale/github-action@v2
Run if [ X64 = "ARM64" ]; then
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100    64  100    64    0     0    607      0 --:--:-- --:--:-- --:--:--   609
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 22.5M  100 22.5M    0     0  68.4M      0 --:--:-- --:--:-- --:--:-- 68.5M
tailscale.tgz: OK
Run sudo -E tailscaled --state=mem: ${ADDITIONAL_DAEMON_ARGS} 2>~/tailscaled.log &
Run if [ -z "${HOSTNAME}" ]; then
#----------------------------------------------------------------------------------
# ✅ ping 
#0s
Run curl http://***:***
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100    53    0    53    0     0     57      0 --:--:-- --:--:-- --:--:--    57
Hello, this is my custom server running on port ***!
#--------------------------------------------------------------------------------
# ✅ Complete Job
0s
Cleaning up orphan processes
#--------------------------------------------------------------------------------
```
---

## Tailscale API

- get the devices

```bash
$ curl -H "Authorization: Bearer ${var.api_key}" https://api.tailscale.com/api/v2/tailnet/ji-podhead.github/devices > OUTPUTFILE
```
- you have to use  `Bearer` **instead of** `-u`, or you will get prompet to enter the root password of the machine that runs vault
>```bash
> -u <your api key>
>```
- those oauth flagss do not work (neither in terraform, nor in api, even with tags and acl)  
>```bash
>  --data-urlencode 'client_id=<client id> \
>  --data-urlencode 'client_secret=t<client secret>
>```  
