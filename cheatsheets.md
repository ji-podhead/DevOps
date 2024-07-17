# cheatsheets

## Jenkins & Github
***jenkins has a weird way of signing in to git.***
- 1. create a pat via dev settings in git
  2. add ajenkins credential
    - ***you need to select  to select `USERNAME+PASSWORD`***
    - add your git user name  and paste the pat into the password field
    - ***DO NOT USE SSH + PASSWORD***



## tailscale and github action 

---

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
    branches: [ "testjob" ]
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
          oauth-client-id: ${{ secrets.TS_OAUTH_CLIENT_ID }}
          oauth-secret: ${{ secrets.TS_OAUTH_SECRET }}
          tags: tag:ci                                        # <<<--- THIS IS IMPORTANT

      - name: ping                                           # <<<--- LETS PING A SERVER FOR TESTING
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
