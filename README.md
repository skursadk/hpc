## How to create CI/CD Stack
You can follow the instructions below to create a minikube cluster, install tekton, prepare tekton CI/CD pipeline and configure Github webhook.
> **Note:**  All of the below mentioned commands are run in Ubuntu 22.04.4 LTS. Additional checks to cover different operating systems are not added
- Install minikube and create a kubernetes cluster
	```
	curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
	sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
	
    minikube start
	```
- Install Tekton
	```
	kubectl apply --filename \
	https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
    
    # Wait until all tekton components are ready with the following commands
    kubectl rollout status deployment tekton-pipelines-webhook
	kubectl rollout status deployment tekton-pipelines-controller
	kubectl rollout status deployment tekton-events-controller
	```
- Install Tekton Triggers
	```
	kubectl apply --filename \
	https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
	kubectl apply --filename \
	https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml

	# Wait until all Tekton Triggers components are ready with the following commands
	kubectl rollout status deployment tekton-triggers-controller
	kubectl rollout status deployment tekton-triggers-webhook   
	kubectl rollout status deployment tekton-triggers-core-interceptors
	```
- Install Tekton CLI  (v0.36.0)
	```
	curl -LO https://github.com/tektoncd/cli/releases/download/v0.36.0/tektoncd-cli-0.36.0_Linux-64bit.deb
	sudo dpkg -i tektoncd-cli-0.36.0_Linux-64bit.deb 
	```
- Create Tekton Resources
	```
	kubectl apply -f tekton/
	```
- Install public Tekton tasks
	```
	tkn hub install task git-clone
	tkn hub install task kaniko
	tkn hub install task helm-upgrade-from-source
	tkn hub install task trivy-scanner
	```
- Create docker secret to push built images to docker hub
	```
	# Interactively login into docker hub
	docker login

	# Get docker config file containing auth information and create a k8s secret with that
	DOCKER_CONFIG=$(cat ~/.docker/config.json | base64 -w0)

	kubectl apply -f - <<EOF  
	apiVersion: v1                                                                  
	kind: Secret
	metadata:
	  name: docker-credentials
	data:
	  config.json: $DOCKER_CONFIG
	EOF
	```
- Expose Tekton Event Listener
	```
	k port-forward svc/el-clone-build-push 8080
	```
- Install ngrok to expose your local environment to be accessible by Github
	```
	curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc \
		| sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null \
		&& echo "deb https://ngrok-agent.s3.amazonaws.com buster main" \
		| sudo tee /etc/apt/sources.list.d/ngrok.list \
		&& sudo apt update \
		&& sudo apt install ngrok
	
	# Sign-in into NGrok and get your Auth Token and authenticate with
	ngrok config add-authtoken <NGROK_TOKEN>
    # Copy the URL it created for you to use at Github Webhook
	```
- Now our pipeline is ready to be called from Github Webhook. If you want to implement this in another repository, you should add webhook to respective repository with the following
	- Go to Settings tab at your repository
	- Click `Webhooks` under the tab `Code and Automation`
	- Click  `Add Webhook`
	- Paste the Ngrok URL you copied at the previous step
	- Chose `Content Type` as `application/json`
	- Chose events you want to create webhook  for. Only `Pull_Request` events are chosen at this repository
- Its done! Whenever a Pull request is created or updated, its going to run Tekton CI/CD pipeline which is responsible to
	- Fetch the code
	- Build and push the docker image via `kaniko`
		- It gets the image tag from commit hash via `$(body.pull_request.head.sha)`
	- Run security scan via `trivy` in a way that it fails the pipeline if a critical vulnerability is found
	- Deploy the app via `helm` chart
		- It gets the image tag from commit hash via `$(body.pull_request.head.sha)`
		- It gets branch name from head ref via `$(body.pull_request.head.ref)
			- It creates namespace per `branch_name/pull_request`
			- It uses namespace in ingress object so that each application can be accessed with a different path
				- In your local environment, you need to update your `/etc/hosts` file to resolve the host name defined at the ingress
		- Creates one application per pull-requests in the respective namespaces

## Challenges
This is a straight forward CI/CD pipeline created with the above mentioned tech stack. It is created just to show a simple pipeline with any technology can be prepared within a short period of time. Its worth to mention that its not production grade ready. There are lots of areas things to be covered. Here are the list of some of them
- There is no authentication between `event listener` and `Github`
- Pipeline runs in any Pull Request actions. Those actions needs to be caught and act accordingly
- It all the time creates/updates an application however it needs to delete the correspondent application and namespace when the Pull Request is closed 
- Branch name is used in `namespace name` and `hostname` at the ingress which comes with some restrictions. It should contain only lower case letters, `-` and `[0-9]`. Moreover it has a maximum length that needs to be calculated and filtered
- Last but not least, no need to mention that there is not any promotion of the artifacts to the prod after merging into master

>**NOTE**: The main challenge was to implement all of those in a limited timeframe due to `Feast of Ramadan`
 
 ## Security Practices:
There is a task for `trivy scan` which scans the built image and fails the build if a critical vulnerability is found. There are couple of things needs to be thought here
- Because the steps are running sequentially, it may slow down the pipeline as the size of the image increases
- It fails the pipeline but does not clean up the registry which may cause a security breach
	- In production environments, clusters needs to be secured just to accept images from a certain registries where those registries are vulnerability-free as mush as  possible
- Apart from those, kubernetes resources can be more secured via readonly file system, limited capabilities etc. But within the scope of this assignment, they are not implemented