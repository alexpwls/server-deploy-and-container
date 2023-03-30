# Deploying a Flask API

This is the project starter repo for the course Server Deployment, Containerization, and Testing.

In this project you will containerize and deploy a Flask API to a Kubernetes cluster using Docker, AWS EKS, CodePipeline, and CodeBuild.

The Flask app that will be used for this project consists of a simple API with three endpoints:

- `GET '/'`: This is a simple health check, which returns the response 'Healthy'. 
- `POST '/auth'`: This takes a email and password as json arguments and returns a JWT based on a custom secret.
- `GET '/contents'`: This requires a valid JWT, and returns the un-encrpyted contents of that token. 

The app relies on a secret set as the environment variable `JWT_SECRET` to produce a JWT. The built-in Flask server is adequate for local development, but not production, so you will be using the production-ready [Gunicorn](https://gunicorn.org/) server when deploying the app.



## Prerequisites

* Docker Desktop - Installation instructions for all OSes can be found <a href="https://docs.docker.com/install/" target="_blank">here</a>.
* Git: <a href="https://git-scm.com/downloads" target="_blank">Download and install Git</a> for your system. 
* Code editor: You can <a href="https://code.visualstudio.com/download" target="_blank">download and install VS code</a> here.
* AWS Account
* Python version between 3.7 and 3.9. Check the current version using:
```bash
#  Mac/Linux/Windows 
python --version
```
You can download a specific release version from <a href="https://www.python.org/downloads/" target="_blank">here</a>.

* Python package manager - PIP 19.x or higher. PIP is already installed in Python 3 >=3.4 downloaded from python.org . However, you can upgrade to a specific version, say 20.2.3, using the command:
```bash
#  Mac/Linux/Windows Check the current version
pip --version
# Mac/Linux
pip install --upgrade pip==20.2.3
# Windows
python -m pip install --upgrade pip==20.2.3
```
* Terminal
   * Mac/Linux users can use the default terminal.
   * Windows users can use either the GitBash terminal or WSL. 
* Command line utilities:
  * AWS CLI installed and configured using the `aws configure` command. Another important configuration is the region. Do not use the us-east-1 because the cluster creation may fails mostly in us-east-1. Let's change the default region to:
  ```bash
  aws configure set region us-east-2  
  ```
  Ensure to create all your resources in a single region. 
  * EKSCTL installed in your system. Follow the instructions [available here](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html#installing-eksctl) or <a href="https://eksctl.io/introduction/#installation" target="_blank">here</a> to download and install `eksctl` utility. 
  * The KUBECTL installed in your system. Installation instructions for kubectl can be found <a href="https://kubernetes.io/docs/tasks/tools/install-kubectl/" target="_blank">here</a>. 


## Initial setup

1. Fork the <a href="https://github.com/udacity/cd0157-Server-Deployment-and-Containerization" target="_blank">Server and Deployment Containerization Github repo</a> to your Github account.
1. Locally clone your forked version to begin working on the project.
```bash
git clone https://github.com/SudKul/cd0157-Server-Deployment-and-Containerization.git
cd cd0157-Server-Deployment-and-Containerization/
```
1. These are the files relevant for the current project:
```bash
.
├── Dockerfile 
├── README.md
├── aws-auth-patch.yml #ToDo
├── buildspec.yml      #ToDo
├── ci-cd-codepipeline.cfn.yml #ToDo
├── iam-role-policy.json  #ToDo
├── main.py
├── requirements.txt
├── simple_jwt_api.yml
├── test_main.py  #ToDo
└── trust.json     #ToDo 
```
     
## Project Steps

Completing the project involves several steps:

1. Write a Dockerfile for a simple Flask API
<!--- Creating docker image --->
docker build -t myimage .

2. Build and test the container locally
<!--- Running docker image --->
docker run --name myContainer --env-file=.env_file -p 80:8080 myimage

3. Create an EKS cluster
<!--- Creating cluster with name simple-jwt-api --->
eksctl create cluster --name simple-jwt-api --nodes=2 --version=1.22 --instance-types=t3.micro --region=us-east-2
<!--- Check node status after completion --->
aws eks --region us-east-2 update-kubeconfig --name simple-jwt-api
kubectl get nodes
<!--- getting account ID --->
aws sts get-caller-identity --query Account --output text 
<!--- Update trust.json with new account ID --->
<!--- Create the IAM role to deploy on the Kubernetes --->
aws iam create-role --role-name UdacityFlaskDeployCBKubectlRole --assume-role-policy-document file://trust.json --output text --query 'Role.Arn'
<!--- Got this return upon succesful creation: --->
arn:aws:iam::090212595461:role/UdacityFlaskDeployCBKubectlRole

<!--- Attach policy to newly created IAM role --->
aws iam put-role-policy --role-name UdacityFlaskDeployCBKubectlRole --policy-name eks-describe --policy-document file://iam-role-policy.json
kubectl get -n kube-system configmap/aws-auth -o yaml > /tmp/aws-auth-patch.yml
kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"

4. Store a secret using AWS Parameter Store
aws ssm put-parameter --name JWT_SECRET --overwrite --value "myjwtsecret" --type SecureString
aws ssm get-parameter --name JWT_SECRET

5. Create a CodePipeline pipeline triggered by GitHub checkins
<!--- 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 --->

6. Create a CodeBuild stage which will build, test, and deploy your code
kubectl get services simple-jwt-api -o wide

For more detail about each of these steps, see the project lesson.

## API testing

- Auth API: 

CURL -X POST http://127.0.0.1:5000/auth -H 'Content-Type: application/json' -d '{"email": "test@test.com", "password": "test"}'

Returns:

{
  "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2ODE0MDIxNDAsIm5iZiI6MTY4MDE5MjU0MCwiZW1haWwiOiJ0ZXN0QHRlc3QuY29tIn0.LcTiqk_fWT4_7VDivTrPqh9cuzkEiXF9ITk3yJ7xYCY"
}

- Contents API:

CURL -X GET http://127.0.0.1:5000/contents -H 'Content-Type: application/json' -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2ODE0MDIxNDAsIm5iZiI6MTY4MDE5MjU0MCwiZW1haWwiOiJ0ZXN0QHRlc3QuY29tIn0.LcTiqk_fWT4_7VDivTrPqh9cuzkEiXF9ITk3yJ7xYCY"

Returns:

{
  "email": "test@test.com", 
  "exp": 1681402140, 
  "nbf": 1680192540
}