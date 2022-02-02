Problems in the current situation:
  - Every new feature must go through main monolithic repository - hard to track and maintain
  - Duplicate projects - stand-alone helm chart must be copied to a monolithic monorepo to be deployed to production
  - Coupling of unrelated projects in the same git repo creates many garbage unmeaningful commits
  - No RBAC flexibility - Development of Operators and Creation of the CR's themselves should be seperated. Not all projects should have the same level of authorization

What this solution tries to achieve:
  - Developers work on seperate helm charts on feature branches
  - Developers Reference their service's branch on group's preprod service-set, sync, and check sanity and integration with other projects
  - Developers open a Merge Request with commits squash and a meaningful name in order to deploy to production
  - Authorized team member will merge it to `master` branch and the feature branch will be deleted
  - Group's production service-set in which the service's `master` branch is referenced, will auto-detect changes
  - Developers will sync their changes to master
  - For each project, Prod monorepo will always point to specific tags, to own deployment control
  - For each project, Preprod monorepo will point to specific feature branches or to `master` 

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
  - Create a merge request to master, with commits squash and a meaningful name
  - Assign the merge request to an authorized team member
  - An authorized team member will see the merge request assigned to them, briefly validate it, and merge it to master 
    > Important note - The term "authorized team member" is now much broader and based on the specific project, which enables much more flexible RBAC
  - Create a tag from master following Semantic Versioning (0.0.1) and document the changes.
  - Add the new service to `values.yaml` in `Prod Management ServiceSet`, with `targetRevision` being the new tag
  - Sync `Prod Management ServiceSet` in ArgoCD
  - Your new service is now live on production :)


