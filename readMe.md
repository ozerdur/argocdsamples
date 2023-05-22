# ArgoCD
- It is  Gitops-based CD with pull model design  
- It is NOT a CI tool.
- Does not require open connectivity between CICD systems and the k8s cluster (more secure). 
- Developer and DevOps engineer will update the Git code only
- Watches config repo for change and apply them (refreshes every 3 mins by default)
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
ArgoCD provide two options in-cluster privileges:

    - Cluster-admin privileges: where ArgoCD has the cluster-admin access to deploy into the cluster that runs in
    - Namespace level privileges: Use this manifest set if you do not need Argo CD to deploy applications in the same cluster that argo CD runs in
    
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


## Sync
    Prune deletes unused resources when syncing
    Self Heal syncs the application after kubectl changes
    argocd.argoproj.io/sync-options: Prune=false prevents deletion of the resource even if deleted in Git
    argocd.argoproj.io/sync-options: Validate=false prevents validation of the resource when applying
    argocd.argoproj.io/sync-options: Prune=Last prunes last if he application is pruned
    argocd.argoproj.io/sync-options: Replace=true recreates the resource during the sync operation instead of apply
    with selective-sync only changed resource are recreated
    Waves can be used to order deployment of resources in asc order
        argocd.argoproj.io/sync-wave: "0"


## Tracking Strategies
    Repo
        - Tag
        - Commit Sha (static)
        - Head
        - Branch

    Helm
        - Specific Version
        - Range Version
        - Latest Version

## ApplicationSet
    Used to generate application manifests.
    master generator multiplies source and target
    pull request generator defaults to 30 mins but this can be changed (requeueAfterSeconds) or hooks can be used. (After merge the application is deleted automatically)
    List generator allows us to target ARGO CD Applications to clusters based on a fixed list of cluster name/URL values
        - any key/value element pair is supported
        - Clusters need to be pre-defined in ARGO CD
    Cluster generator allows us to generate applications based on the list of clusters defined within Argo CD
        - name, nameNormalized, server, metadata.labels.<key>, metadata.annotations.<key>
        - additional key/value pairs can be set manually via values field
        - {} can be used to target all clusters

    Git Generator: generates parameters based o files and folders that are contained within the Git Repository
        - Git Directory Generator: generates parameters using the directory structure
        - Git Files Generator: generates parameters using the contents of Json/Yaml file found within  a specified repo
            Ex: per config for every environment


 ## Simple CI Example
    Web App Repo: https://github.com/mabusaa/argocd-course-webapp
    Config repo: https://github.com/mabusaa/argocd-course-webapp-config
    ArgoCD: https://github.com/mabusaa/argocd-course-apps-definitions/blob/main/applications%20and%20projects/webapp-sample-application/webapp%20application.yaml


## Best Practices

    Separate your code and app manifests with different repos
    Use immutable manifest in production. Avoid using HEAD revision and use tags or commit SHA.
    If you want HPA to control the number of replicas, then don't include replicas in Git
    Don't store plain secrets in git
        - Within Git
            - Use sealed secrets
            - SOPS
        - External secrets store
            - Hashicorp Vault
            - External Secrets Operator
            - Cloud secrets store
                -Aws secret operator

    Use a separated ArgoCD instance for Production
        - Use HA setup

    At least two instances should be used for ArgoCD
        - Non Prod
        - Prod

    Use App off apps to manage ArgoCD application

    Use Application Set and the power of generators to generate applications

    App of Apps: used to deploy multiple app with only one app.  (Root App Directory and Root App Helm)
        example: https://github.com/mabusaa/argocd-course-app-of-apps/tree/main

    Static Environments
        - Single branch
            - Values-test.yaml
            - Values-staging.yaml
            - Values-prod-yaml

        - Branch per static environment
            - Test branch
            - Staging branch
            - Production branch(tags or commit SHA for production environments)

    On-demand Environments
        - Pull Request
            - Created when a PR is opened and deleted when the PR is closed

        - Single branch for on-demand environments and each folder will contain the related manifests( We need to write a script using CI pipeline to create these folders and manifest per Pull request)
