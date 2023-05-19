# ArgoCD
- It is  Gitops-based CD with pull model design  
- It is NOT a CI tool.
- Does not require open connectivity between CICD systems and the k8s cluster (more secure). 
- Developer and DevOps engineer will update the Git code only
- Wathches config repo for change and apply them (refreshes every 3 mins by default)
- Easy rollback
- Easy deploy the same apps to any k8s cluster

- With Projects, useful when ArgoCD is used by multiple teams:

    - Allow only specific sources "trusted git repos"
    - Allow apps to be deployed into specific clusters and namespaces
    - Allow specific resources to be deployed "deployments, Statefulsets, ..etc"

- Consists of 3 main components(each runs as a pod)
    - API/Web Server:
        
        - gRPC/REST server which exposes the API consumed by the Web UI, CLI
        - Application management (CRUD)
        - Application operations (ex: Sync, Rollback) 
        - Repos and clusters management
        - Authentication

    - Repo Server
        - Clone git repo
        - Generate k8s manifests

    - Application controller
        - Communicate with Repo server and get the generated manifests
        - Communicate with k8s API to get actual cluster state
        - Deploy apps manifests to destination clusters
        - Detects OutofSync Apps to take corrective actions "If needed"
        - Invoking user-defined hooks for lifecyle events(PreSync, Sync, PostSync)


- Contains 3 more components:
    - Redis: used for caching
    - Dex: identity service to integrate with external identity providers (Ex: integrating with Github)
    - ApplicationSet Controller: It automates the generation of ARGO CD Applications (Ex: deploying the app to multiple clusters)

## Tools Supported
    - Helm Charts
        - If there is Chart.yaml 
        - Uses Git repo or helm repo 
        - Provides below options
            - Release name (defaults to application name)
            - Values files
            - Parameters
            - File parameters
            - Values as block file

    - Kustomize application
        - If there is a kustomization.yaml, kustomization.yml, kustomization
        - Provides below for options
            - Name prefix: appended to resources
            - Name suffix: appended to resources
            - Images: to override images
            - Common labels: set labels on all resources
            - Common annotations: set annotations on all resources

    - Directory of Yaml files
        - Default
        - Provides below options
            - Recursive: include all files in sub directories
            - Jsonnet: 
                External Vars: list of external variables for Jsonnet
    - Jsonnet

## Installation Options
-  Non High availability setup
    - Suitable for evaluation or dev/testing environments

        kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

- High availablity setup
    - Recommended for production
    - You need at least 3 worker nodes
        
        kubectl apply -n argocd -f https://github.com/argoproj/argo-cd/raw/stable/manifests/ha/install.yaml

- Light installation "Core"
    - Suitable if ArgoCD is used by administrators only. UI and API server  is not installed for end users
    - By default its installed as Non-HA
        kubectl apply -n argocd -f https://github.com/argoproj/argo-cd/raw/stable/manifests/core-install.yaml

### Installation commands
    - kubectl create ns argocd
    - kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    - kubectl get secret -n argocd argocd-initial-admin-secret -o yaml (decode base64 password) admin / 2ia5FZiDaKgEHaRn
    - kubectl port-forward svc/argocd-server -n argocd 8080:443

### Privileges options
ArgoCD provide two options in-cluster previleges:

    - Cluster-admin privileges: where ArgoCD has the cluster-admin acccess to deploy into the cluster that runs in
    - Namespace level pivileges: Use this manifest set if you do not need Argo CD to deploy applications in the same cluster that argo CD runs in
    
## Application
- It is a k8s resource object representing a deployed application instance in an environment
- It is defined by two key pieces of information
    - Source: reference to the desired state in Git(Repository, revision, path)
    - Destination: reference to the target cluster and namespace
- Can be created using below options
    - Declaratively "Yaml" (Recommended)
    - Web UI
    - CLI

### CLI
    - argocd login localhost:8080 --insecure
    - argocd app create app-2 --repo https://github.com/mabusaa/argocd-example-apps.git --revision master --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace app-2 --sync-option CreateNamespace=true
    - argocd app sync app-2

## Projects
    Provide a logical grouping of applications
    Useful when ArgoCD is used by multiple teams:
        - Allow only specific sources "trusted git repos"
        - Allow apps to be deployed into specific clusters and namespaces
        - Allow specific resources to be deployed "deployments, Statefulsets, ..etc"

    Commands:
        - kubectl get appproject -n argocd (-o yaml)
        - argocd proj role create-token <project-name> <role-name>
            Ex: argocd proj role create-token project-with-role ci-role
        - argocd cluster list --auth-token <token-value>  or set ARGOCD_AUTH_TOKEN env variable

## Repositories
    Access ways:
        - HTTPs: using username and password or access token
        - SSH: using ssh private key
        - GitHub/GitHub Enterprise: GitHub App credentials

    Repo credentials are stored as secrets (with label argocd.argoproj.io/secret-type: repository)
    Credential Templates can be created as secrets with label argocd.argoproj.io/secret-type: repo-creds
        In order for ArgoCD to use a credential template for any give repository:
            - The URL configured for a credential template must match a prefix for the repository URL
            - The repository must either not be configured at all, ir if configured, must not contain any credential information