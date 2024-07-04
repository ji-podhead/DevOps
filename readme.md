# DevOps
This is my DevOps - Repo. This serves as my Playground, Portfolio, Research-Notes, Cheat-Sheet and Guide. 

## Topics
- IAC with Terraform
- CICD-serving with Jenkins
- CI with Packer,Gitea and Github-Actions
- CD with Argo-CD
- Automated Security Testing
- Virtualisation with Kubevirt
- Monitoring & Logging with Grafana, Kafka & OpenTelemetry
- Authorisation with Vault Github-Secrets and Oauth-Proxy
- Streaming with NGINX RTMP Module and gRPC
- StatefulSets, Monoliths & Microservices
- RateLimiting, realtime threat-detection & Vulnabillity-Scanning 

---

## CI-CD Pipeline
### Basis Infrastructure Deployment
- we initialize our k8s cluster, branches, set up our monoliths and install our tools
- k8s network policies & topology 
- user & administration
- basis iam policies
- ingress, dashboards, host- names&mapping, dns, firewall & persistent storage
- ***IAC-Basis-Configuration Check*** 
### Source Control & Integration
> triggered by commit to Integration Branch
-  static checks before triggering pipeline 
	- Sast, Dast & Linters
### Build
- IAC-Codequality & misconfiguration Checks 
> can also be used while writing terraform code

	- Chekov 
	- trivy
	
- build based on parameters & multiple stages
>	not every change need to trigger every component

- push into Test Branch

### Test 
>  Triggered by pushing into Test-Branch
```mermaid
graph TB

subgraph "backend"[
<div style="text-align: center;margin-left: 15px; line-height: 0.4;">
		<h4>Backend</h4>
		<ul style="text-align: left;margin-left: 15px;list-style: square; line-height: 0.4;">
		<li>microservice</li>
		<li>API-Server</li>
		<ul style="text-align: left;margin-left: 15px;list-style: square; line-height: 0.4;"> 
		<li>jobs</li>
		<li>gRPC</li>
		<li>webservers</li>
		<li>websockets&streams</li>
		</ul>
		</ul>
		</div>
]
end

subgraph "frontend"[
<div style="text-align: center;margin-left: 15px; line-height: 0.4;">
		<h4>Frontend</h4>
		<ul style="text-align: left;margin-left: 15px;list-style: square; line-height: 0.4;">
		<li>Browser</li>
		<li>Smartphone</li>
</ul>
</div>
]
end


subgraph "Configuration"[
		<div style="text-align: center;margin-left: 15px; line-height: 0.4;">
		<h4>Policies & Authorization</h4>
		<ul style="text-align: left;margin-left: 15px;list-style: square; line-height: 0.4;">
		<li>IAM Policies</li>
		<li>Authorization - Containers & Frontend</li>
		<li>Network-Configuration & Policies</li>
		</ul>
		</div>
		]
end
subgraph "Resources"[
		<div style="text-align: center;margin-left: 15px; line-height: 0.4;">
		<h4>Resources</h4>
		<ul style="text-align: left;margin-left: 15px;list-style: square; line-height: 0.4;">
		<li>Node-Resources</li>
		<li>Deployment Resources</li>
		<li>Load Balancers</li>
		</ul>
		</div>
		]
end
subgraph "StaticAnalysis"[
		<div style="text-align: center;margin-left: 15px; line-height: 0.4;">
		<h4>Static Image Tests</h4>
		<ul style="text-align: left;margin-left: 15px;list-style: square; line-height: 0.4;">
		<li>Ivy</li>
		<li>Quay/Clair</li>
		</ul>
		</div>
]
end

subgraph "Integration"[
<div style="text-align: center;margin-left: 15px; line-height: 0.4;">
		<h4>Integration Test</h4>
		<ul style="text-align: left;margin-left: 15px;list-style: square; line-height: 0.4;"> 
		<li>e2e/Cross-Browser- und Cross-Device</li>
		<ul style="text-align: left;margin-left: 15px;list-style: square; line-height: 0.4;"> 
<li>k8s E2E Framework</li>
<li>Cypress</li>
		<li>Playwright</li>
		<li>PreDeployment</li>
		</ul>
		</ul>
		</div>
]
end

subgraph "Web&App-Security"[
<div style="text-align: center;margin-left: 15px; line-height: 0.4;">
		<h4>Web & App Security</h4>
<ul style="text-align: left;margin-left: 15px;list-style: square; line-height: 0.4;"> 
<li>OWASP ZAP</li>
</ul>
</div>
]
end

subgraph "Authorisation&Policy-Tests"[
<div style="text-align: center;margin-left: 15px; line-height: 0.4;">
		<h4>Authorization & Compliance Tests</h4>
		<ul style="text-align: left;margin-left: 15px;list-style: square; line-height: 0.4;"> 
<li>OPA-Policy Testing</li>
<li>Kube-Bench</li>
<li>Prowler</li>
<li>Checkov</li>
</ul>
</div>
]
end

subgraph "performance"[
<div style="text-align: center;margin-left: 15px; line-height: 0.4;">
		<h4>Performance & Penetration</h4>
<ul style="text-align: left;margin-left: 15px;list-style: square; line-height: 0.4;"> 
<li> Load Testing </b><li>
<ul style="text-align: left;margin-left: 15px;list-style: square; line-height: 0.4;"> 
		<li>Grafana K6</li>
		<li>Locust</li>
</ul>
</ul>
</div>
]
end


backend-->StaticAnalysis
StaticAnalysis-->Authorisation&Policy-Tests
frontend--> Web&App-Security

Web&App-Security --> Authorisation&Policy-Tests
Authorisation&Policy-Tests --> Integration

Integration-->performance
Configuration-->Authorisation&Policy-Tests

Resources-->Integration
```

- sending pull Request Testing->Main/Production Branch
- awaiting manual Approval
### Deploy
> based on parameters & Infrastructure

  
### Monitoring & Feedback
- telemtry & logs
	- kafka
	- OpenTelemetry
- devSecOps:
	- Suricata
	- Snort
	- OSSEC
	- Falco


---
