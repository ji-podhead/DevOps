# cheatsheets & Unreleased Guides

## Jenkins & Github
***jenkins has a weird way of signing in to git.***
- 1. create a pat via dev settings in git
  2. add ajenkins credential
    - ***you need to select  to select `USERNAME+PASSWORD`***
    - add your git user name  and paste the pat into the password field
    - ***DO NOT USE SSH + PASSWORD***
--- 
## Libvirt
- install
  ```bash
     su root
     dnf install qemu-kvm libvirt virt-install virt-viewer
     for drv in qemu network nodedev nwfilter secret storage interface; do systemctl start virt${drv}d{,-ro,-admin}.socket; done
  ```
- create volume in default storage pool
  ```bash
  sudo qemu-img create -f qcow2 /var/lib/libvirt/images/your_volume.qcow2 2G
  ```
- use your volume
  ```xml
	 <devices>
	    <emulator>/usr/libexec/qemu-kvm</emulator>
	    <disk type='file' device='disk'>
	      <source file='/var/lib/libvirt/images/your_volume.qcow2'/>
	      <target dev='vda' bus='virtio'/>
	    </disk>
	    # ... more devices
	 </devices>
  ```
  
- boot from iso (virtual cd reader)
  ```xml 
     <devices>
	<disk type='file' device='cdrom'>
		<driver name='qemu' type='raw'/>
	        <source file='/var/lib/libvirt/images/ubuntu-24.04-live-server-amd64.iso'/>
	      	<target dev='hdb' bus='ide'/>
	        <readonly/>
	</disk>
        # ... more devices
  </devices>    
  ```

   - note that you have to use `machine='pc'` in the os section of your xml for debian and you also need to specify the boot device, which is our cdrom/hd.
     - if the vm cant find any bootable os in our disk, it will look in cdrom, so you can set `boot dev='hd'`
     - this is how it could look like:
     ```xml
	  <os>
	    <type arch='x86_64' machine='pc'>hvm</type>
	    <boot dev='hd'/>
	  </os>
      ```
		    
## Ansible Variables, Secrets and Vault
this is how you can use variables and secrets in ansible:
### 1. Variables as Constructor argument
You can pass secrets/variables to ansible using the -e flag (eg `-e var=<value>`).
  - example:
    ```bash
       ansible-playbook ansible/proxmox_playbook.yml -i ansible/inventory.yml  -e vault_token=<yout token> -vvv
    ```
### 2. Environment Variables
 
- create env. variable manualy
     ```bash
     	export variable = value
     ```
- use [git actions environment variables](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/variables#defining-environment-variables-for-a-single-workflow) and just reference them in ansible
  - this is how you can create them in a workflow:
  ```yaml
	  jobs:
	  greeting_job:
	    runs-on: ubuntu-latest
	    env:
	      Greeting: Hello
	    steps:
	      - name: "Say Hello Mona it's Monday"
	        run: echo "$Greeting $First_Name. Today is $DAY_OF_WEEK!"
	        env:
	          First_Name: Mona
  ```
### 3. [ansible vault](https://docs.ansible.com/ansible/latest/cli/ansible-vault.html#ansible-vault)
all this does is to encrypt your inventory file (where you usualy store your hosts and ssh keys), or other files.
- usage:
   ```bash
   $ ansible-vault encrypt inventory.yml
   ```
   - the output will look like this:
   ```bash
   $ANSIBLE_VAULT;1.1;AES256
   6531633239353231303063613464323531643933613336353130383837623537663537343033633
   ...
   ```
   
### 4. Vault (key/value)	
you can use the community.hashi_vault plugin to fetch secrets from vault using lookups.
   - this is how you use lookup:
     ```yaml
       "{{ lookup('community.hashi_vault.vault_kv2_get', 'dir/subdir' , engine_mount_point='keyvalue', url='http://127.0.0.1:8200',  token=vault_token)['data']['data']['<your-secret-key>'] }}"
     ```
<br> However you need to do this in a playbook. 
   - ***You cant put the lookups diretly into the inventory***
   - Even if you can run your script and you have no errors, the variables will remain unknown and lookups wont fire.


    
#### ***Store your secrets in the inventory dynamicaly***
Before you can use your Hosts and do any ssh connection, you need to fetch your secrets  and store them in the inventory dynamically:

- 1. ***create an inventory*** (the placeholder variables are just for prototyping)
     ```yaml
	   workshop_machines:
	  hosts:
	    dcworkshop1:
	      vars:
	        ansible_ssh_host: "placeholder" 
	        ansible_user: "placeholder"
	        ansible_ssh_pass: "placeholder"
	        ansible_connection: ssh
	        ansible_become_password: "placeholder"
      ```
 - 2. ***Fetch your Vault Secrets and store them using set_fact method***
	     ```yaml
		---
		- hosts: workshop_machines
		  gather_facts: no
		  become: true
		  become_method: sudo
		  become_user: root
		  tasks:
		    - name: gethosts
		      debug:
		        var: inventory_hostname
		    - name: get_secrets_from_vault
		      set_fact:
		          proxmox_user_secret: "{{ lookup('community.hashi_vault.vault_kv2_get', 'ansible/proxmox/' + inventory_hostname + '/' , engine_mount_point='keyvalue', url='http://127.0.0.1:8200',  token=vault_token) }}"
		
		    - name: Get secrets from Vault for each host
		      set_fact:
		        ansible_ssh_host: "{{ lookup('community.hashi_vault.vault_kv2_get', 'ansible/proxmox/'+ inventory_hostname, engine_mount_point='keyvalue', url='http://127.0.0.1:8200', token=vault_token)['data']['data']['ip'] }}"
		        ansible_user: "{{ lookup('community.hashi_vault.vault_kv2_get', 'ansible/proxmox/'+ inventory_hostname, engine_mount_point='keyvalue', url='http://127.0.0.1:8200', token=vault_token)['data']['data']['user'] }}"
		        ansible_ssh_pass: "{{ lookup('community.hashi_vault.vault_kv2_get', 'ansible/proxmox/'+ inventory_hostname, engine_mount_point='keyvalue', url='http://127.0.0.1:8200', token=vault_token)['data']['data']['pass'] }}"
		        ansible_become_password: "{{ lookup('community.hashi_vault.vault_kv2_get', 'ansible/proxmox/'+ inventory_hostname, engine_mount_point='keyvalue', url='http://127.0.0.1:8200', token=vault_token)['data']['data']['pass'] }}"
	     ```
---
## Ansible & libvirt
- ***we need `become: true` here, because libvirt requires root***
- in this step we will just create a config for each host(could also be hostgroups)
  - in the next step will use roles and jinja templates instead 
### 1. install
  ```yaml
  - hosts: workshop_machines
  gather_facts: no
  become: true
  become_method: sudo
  become_user: root
  tasks:
    - name: install software
      block:
      - name: Install wget
        ansible.builtin.dnf:
          name: wget
          state: present
      - name: Install libvirt
        ansible.builtin.dnf:
          name: libvirt
          state: present
   ```
### 2. create a vm preset
   - ***it needs to be a *.xml.j2 file***
   - make sure to use the right emulator path
     - i had libvirt & kvm installed on another system, so i just looked up the xml in of an existing vm, using the qemu/kvm manager   
   - also make sure to copy the images to the libvirt image path, but libvirt can have trouble reading your user files
     - we we will do this in our ansible file in the next step 
   ```xml
	<domain type='kvm'>
	  <memory>131072</memory>
	  <vcpu>1</vcpu>
	  <os>
	    <type arch="i686">hvm</type>
	  </os>
	  <clock sync="localtime"/>
	  <devices>
	    <emulator>/usr/libexec/qemu-kvm</emulator>
	    <disk type='file' device='disk'>
	      <source file='/var/lib/libvirt/images/proxmox-ve_8.2-1.iso'/>
	      <target dev='hda'/>
	    </disk>
	    <interface type='network'>
	    <mac address="52:54:00:62:f8:25"/>
	      <source network='default'/>
	    </interface>
	    <graphics type='vnc' port='-1' keymap='de'/>
	  </devices>
	</domain>
   ```
### 4. get your iso
   - im just downloading proxmox here using wget
     -  but you can of course use docker hub, or your self hosted registry (gitea, or gitlab). not sure if it makes sense to use Argo here though
      
     ```yaml
	      - name: get_proxmox_iso
	        ansible.builtin.shell: |
	          cd /var/lib/libvirt/images/
	        # wget -c https://enterprise.proxmox.com/iso/proxmox-ve_8.2-1.iso 
	        args:
	          chdir: ./
      ```    
### 3. create and update vms
   - if you dont want to update your configuration and infra, you dont need the destroy job
   - notice that you have to define a vm before you can create it.
   - i use a config file that is stored under `./config/libvirt/<hostname>.xml.j2`
     - you can of course just use lookup, but i wanted to have multiple presets for different hosts, or groups
       
     ```yaml
          - name: create vms
	      block:
	        - name: destroy VM
	          ignore_errors: true
	          community.libvirt.virt:
	            name:  "{{ inventory_hostname }}_proxmox" 
	            command: destroy
	            autostart: true
	            xml: "{{ lookup('template', '../config/libvirt/' + inventory_hostname + '.xml.j2') }}"
	        # defining and launching an LXC guest
	        - name: define VM
	          community.libvirt.virt:
	            name: "{{ inventory_hostname }}_proxmox"
	            command: define
	            autostart: true
	            xml: "{{ lookup('template', '../config/libvirt/' + inventory_hostname + '.xml.j2') }}"
	        - name: Create VM
	          community.libvirt.virt:
	            name: "{{ inventory_hostname  }}_proxmox"
	            command: create
	            autostart: true
	            xml: "{{ lookup('template', '../config/libvirt/' + inventory_hostname + '.xml.j2') }}"
	        - name: Get VMs list
	          community.libvirt.virt:
	            command: list_vms
	          register: existing_vms
	          changed_when: no
      ```
---

## Ansible roles

| [Ansible Roles](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)| [build-VM-fast-ansible](https://www.redhat.com/sysadmin/build-VM-fast-ansible) | [ansible-templates-configuration](https://www.redhat.com/sysadmin/ansible-templates-configuration) |
- roles let us use jijna templates **-> the files have a `*.j2` suffix**
- we can use ansible variables in our template files, so we dont need a file for each host, or change the file dynamically according to the host specific values (like hostname)
### Example
- lets say we want to change the name attribute for each host in the vm preset xml
- without roles and jina templates we need to use sed(regex) for each value:
   ```bash
   	$ sed -i "s|<name>[^>]*</name>|<name>$vm_name</name>|" $xml_path
   ```
- when using roles and jinja templates, we can use our ansibles variables directly in the xml preset:
   ```xml
   	<name>{{ vm_name }}</name>
   ```
### Create a Role 
- go to your playbook folder
- initialize a role
  ```bash
     $ ansible-galaxy role init kvm_provision
  ```
- remove the files that are not required
   ```bash
   $ rm -r files handlers vars
   ```
- add path to ansible.cfg (not sure if required)
  - ansible should usualy look for a roles folder in your playbookfolder
  ```yaml
  roles_path = ./roles:./playbooks/roles
  ``` 
- create defaults in `kvm_provision/defaults/main.yml`
  ```yaml
	  ---
	# defaults file for kvm_provision
	base_image_name: Fedora-Cloud-Base-34-1.2.x86_64.qcow2
	base_image_url: https://download.fedoraproject.org/pub/fedora/linux/releases/34/Cloud/x86_64/images/{{ base_image_name }}
	base_image_sha: b9b621b26725ba95442d9a56cbaa054784e0779a9522ec6eafff07c6e6f717ea
	libvirt_pool_dir: "/var/lib/libvirt/images"
	vm_name: f34-dev
	vm_vcpus: 2
	vm_ram_mb: 2048
	vm_net: default
	vm_root_pass: test123
	cleanup_tmp: no
  ```
- create your jinja template under `<your role>/templates`
  - i used the one from the [redhat blog](https://www.redhat.com/sysadmin/build-VM-fast-ansible)
  - i named it `kvm.xml.j2`
  
- create a task in `<your role>/tasks/main.yml`
   ```yaml
	---
	# tasks file for kvm_provision
	- name: gethosts
	  debug:
	    var: inventory_hostname
	- name: Get VMs list
	  become: true
	  become_method: sudo
	  become_user: root
	  community.libvirt.virt:
	    command: list_vms
	  register: existing_vms
	  changed_when: no
   ```
- ***reference and use your role in your playbook***
  - you can reference the role in the play directly
    - *this will fire the taks in the role before the tasks in the play*
    ```yaml
	 - hosts: dcworkshop1
	  gather_facts: no
	  become: true
	  become_method: sudo
	  become_user: root
	  roles:
	    - kvm_provision
     ```
  - you can also reference the role in a single task
    ```yaml
          - name: single task with role
            include_role:
                name: kvm_provision
    ```
---
   
## compare commits in Gitub Actions
```yaml
	- name: Checkout repository
        uses: actions/checkout@v2
      
      - name: Get current commit ID
        run: echo "COMMIT_ID=$(git rev-parse HEAD)" >> $GITHUB_ENV

      - name: Get previous commit ID
        run: |
          PREV_COMMIT_ID=$(git rev-list --max-parents=0 HEAD | head -n 1)
          echo "PREV_COMMIT_ID=$PREV_COMMIT_ID" >> $GITHUB_ENV
```
- the commit ids are now also environment variables:
   ```bash
	  shell: /usr/bin/bash -e {0}
	  env:
	    COMMIT_ID: 350add3930d76dbc4abc0f69d9eb650e7159cbbc
	    PREV_COMMIT_ID: 350add3930d76dbc4abc0f69d9eb650e7159cbbc
   ```
---
## Cache & Containers in Github Actions 

It can take quite a long time to install python and a single 40mb package in a workflow running on git.
In Some situations that can be quite anyoing, for example when you just want to deploy a static website, or a package that is in development stage.
  - in my case, I wanted to create a workflow that automatically publishes to anisble galaxy and that are more than 4 commands to test the results and i was to lazy to set up a debugger
     
- We can spare time by caching pip packages, or build containers from the workflow workspace
### Caching pip packages
- the packages get cached but net to get reinstalled though
   ```yaml
	name: Publish Ansible Galaxy Collection
	on:
	  push:
	    branches:
	      - main 
	jobs:
	  job1:
	    runs-on: ubuntu-latest
	    steps:
	      - name: Cache Python dependencies
		uses: actions/cache@v2
		with:
		  path: ~/.cache/pip
		  key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
		  restore-keys: |
		    ${{ runner.os }}-pip-
	      - name: Install dependencies
		run: |
		  pip install ansible[galaxy]
    ```
	- output:
	   ```bash
	   	✅ Cache Python dependencies
		------------------------------
		Run actions/cache@v2
		Cache not found for input keys: Linux-pip-, Linux-pip-
		20s
	   	✅ Install dependencies
		------------------------------
		Run pip install ansible[galaxy]
		Defaulting to user installation because normal site-packages is not writeable
		Collecting ansible[galaxy]
		  Downloading ansible-10.2.0-py3-none-any.whl (48.2 MB)
		...
		Successfully installed ansible-10.2.0 ansible-core-2.17.2 resolvelib-1.0.1
		2m 14s
	      	✅ Post Cache Python dependencies
		------------------------------
		Post job cleanup.
		/usr/bin/tar --posix -z -cf cache.tgz -P -C /home/runner/work/ilohelper-collection/ilohelper-collection --files-from manifest.txt
		Cache Size: ~44 MB (45745781 B)
		Cache saved successfully
		Cache saved with key: Linux-pip-
	   ```
         - notice that ***gh actions automatically created Post Cache Python dependencies stage for us***
- you can limit the cache job, so it only gets fired when the requierements have really changed 
   ```yaml
	jobs:
	  check-changes:
	    runs-on: ubuntu-latest
	    outputs:
	      changed: ${{ steps.check.outputs.changed }}
	    steps:
	      - name: Checkout repository
	        uses: actions/checkout@v2
	      - name: Check if requirements have changed
	        id: check
	        run: |
	          CHANGED=$(git diff --name-only HEAD^ HEAD | grep -q 'requirements.txt' && echo "true" || echo "false")
	          echo "::set-output name=changed::$CHANGED"
	
	  cache-and-install:
	    needs: check-changes
	    if: needs.check-changes.outputs.changed == 'true'
	    runs-on: ubuntu-latest
	    steps:
	      - name: Checkout repository
	        uses: actions/checkout@v2
	      - name: Cache Python dependencies
	        uses: actions/cache@v2
	        .... # the rest
	      - name: Install dependencies
	        run: |
	          pip install ansible[galaxy]
     ```

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

## Github Organisation Access Tokens (Pats)
this is kinda confusing at first:
  - 1. you have to allow the creation of pats in the organisation settings
  - 2. go to your ***personal*** developer settings where you usually create access keys
    - create a new pat, but choose your organisation as the target organisation/account instead of your profile
  - 3. go back to your organisation settings and allow the pat
    - only if the user that requested the pat is not admin and you have set *Require administrator approval* to true

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
