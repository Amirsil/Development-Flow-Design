
Problems in the current situation:
  - Every new feature must go through main monolithic repository - hard to track and maintain
  - Duplicate projects - stand-alone helm chart must be copied to a monolithic monorepo to be deployed to production
  - Coupling of unrelated projects in the same git repo creates many garbage unmeaningful commits
  - No RBAC flexibility - Development of Operators and Creation of the CR's themselves should be seperated. Not all projects should have the same level of authorization

Possible solutions:
| Criteria | App-of-apps (Current) | Umbrella Chart | Service Set |
| -------- | ----------- | -------------- | ----------- |
| Group Architecutre | Coupled - All in one repo | Seperated - Each subchart has its own repo | Seperated - Each subchart has its own repo | 
| CI needs | None      | CI in each subchart to package helm and push to harbor, CI in Umbrella chart to pull all helm repos from harbor and update helm tars in git repository  | None |
| Argo Management | Easy management using Argo Application   | All projects are deployed together, hard to manage  | Easy management using Argo Application |
| Seperate Version Control | Very hard, all projects are coupled so seperate versioned cannot be reverted to | Easy version control - helm repos have tags and are saved in harbor/git | Easy version control, each helm chart git repo has its own branches commits and tags that can be referenced directly in Service Set | 
| Dependencies | ArgoCD and Git | No dependencies - all projects are packaged into the umbrella, so it can be deployed anywhere | ArgoCD and Git | 


What this solution (Service Se) tries to achieve:
  - Loose coupling between unrelated projects in the same group
  - Seperate management of unrelated projects with different permissions that are in the same group (e.g. Operator CR's)
  - For each project, Prod ServiceSet will always point to specific tags (Or `master` for seperated CD), to own deployment control
  - For each project, Preprod ServiceSet will point to specific feature branches or to `master` 

For this example let's use Management Services, which is a logical group of services that are deployed on management hubs (Previously known as Management Hub Components).

2 Repos will coexist:
  - Preprod Management Service Set
  - Prod Management Service Set

ServiceSet - A repo template for Argo ApplicationSet which deploys a set of services defined in values.yaml to a specific cluster.

ApplicationSet - An ArgoCD Kubernetes resource, which addresses the appofapps Argo paradigm in a more elegant manner.
[**Read more about ApplicationSet**](https://argocd-applicationset.readthedocs.io/en/stable/)

Example of services in `values.yaml`:
```yml
  services:
    - serviceName: redis
      repoURL: github.com/redis.git
      targetRevision: master
      path: .
    - serviceName: tester
      repoURL: github.com/tester.git
      targetRevision: master
      path: .
```

Let's say you want to add a new service to Management Services or change an existing one, you will:
  - create a new branch for the feature in the stand-alone helm chart and make the desired changes
  - Add your service to `Preprod Management ServiceSet`'s `values.yaml`, with the `targetRevision` being the new branch
  - change the targetRevision of the service's reference in "Preprod Management Service" Set's values.yaml
  - sync `Preprod Management Service Set` in Argo
  - Run sanity and integration tests (Or manually check application is behaving expectedly)
<br>

Now you know the project is working well and is ready to be deployed to production, so next you will:
  - Create a merge request to master, with commits squash and a meaningful name, and merge it, deleting the feature branch
  - Create a tag from master following Semantic Versioning (0.0.1) and document the changes.
  - Add the new service to `values.yaml` in `Prod Management ServiceSet`, with `targetRevision` being the new tag
  - Open a merge request in `Prod Management ServiceSet` to deploy your new service to production
  - Assign the merge request to an authorized team member
  - An authorized team member will see the merge request assigned to them, briefly validate it, and merge it to master (Optional CD in the future)
  - Sync `Prod Management ServiceSet` in ArgoCD
  - Your new service is now live on production :)


