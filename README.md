
Problems in the current situation:
  - Every new feature must go through main monolithic repository - hard to track and maintain
  - Duplicate projects - stand-alone helm chart must be copied to a monolithic monorepo to be deployed to production
  - Coupling of unrelated projects in the same git repo creates many garbage unmeaningful commits
  - No RBAC flexibility - Development of Operators and Creation of the CR's themselves should be seperated. Not all projects should have the same level of authorization

What this solution (Service Set) tries to achieve:
  - Loose coupling between unrelated projects in the same group
  - Seperate management of unrelated projects with different permissions that are in the same group (e.g. Operator CR's)
  - For each project, Prod ServiceSet will always point to `values.yaml`
  - For each project, Preprod ServiceSet will point to `values.prep.yaml`, which contains granular patches (Usually of image versions) and is continously deployed to preprod cluster. 
  - Much better resource utilization - Instead of having n*n (clusters * services) Application CR's, there are only 2 ApplicationSets (One per cluster group and one per resource gruop)

Possible solutions comparison:
| Criteria | App-of-apps (Current) | Umbrella Chart | ApplicationSet |
| -------- | ----------- | -------------- | ----------- |
| Group Architecutre | Coupled - All in one repo | Seperated - Each subchart has its own repo | Seperated - Each subchart has its own repo | 
| CI needs | None | CI in each subchart to package helm and push to harbor, CI in Umbrella chart to pull all helm repos from harbor and update helm tars in git repository  | None |
| Argo Management | Easy management using Argo Application   | All projects are deployed together, hard to manage  | Easy management using Argo Application |
| Versioning | No versioning - All projects are glued together and are deployed directly from git, which makes seperate versioning impossible | Helm repo versioning - helm repos can he pushed to harbor with tags, which makes seperate versioning possible, but not very descriptive and hard to rely on | Git versioning - seperate helm charts can utilize git tags and branches, which makes seperate versioning easy, descriptive, reliable and flexible (tags for strict versioning, branches for easier deployment)  | 
| Dependencies | ArgoCD and Git | No dependencies - all projects are packaged into the umbrella, so it can be deployed anywhere | ArgoCD and Git | 
| Permissions | Centered and very strict - Every project change is monitored in the same repository, which gives the maintainer full control of what is deployed to production - Lots of work for monorepo maintainers | Permissions are strict and easy to manage but aren't so flexible - every helm repo tag change must be approved by a monorepo maintainer | Permissions are more flexible but are also more fragile - if `master` branch is referenced in production for a helm charts, their permissions to merge to `master` must be restricted |

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
    redis:
      repoURL: github.com/redis.git
      targetRevision: master
      path: .
    tester:
      repoURL: github.com/tester.git
      targetRevision: master
      path: .
```

Let's say you want to change an existing service on Management Services:
  - add your changes to `values.prep.yaml`, or create a new branch if the changes are too big to only change `values.yaml`.
  - Add your service to `Preprod Management ServiceSet`'s `values.yaml`, referencing `values.prep.yaml` or the new branch
  - sync `Preprod Management Service Set` in Argo
  - Run sanity and integration tests (Or manually check application is behaving expectedly)
<br>

Now you know the project is working well and is ready to be deployed to production, so next you will:
  - Create a merge request to change `values.yaml` or to merge the feature branch to master on the subchart
  - Open a merge request in `Prod Management ServiceSet` to deploy your new service to production
  - Assign the merge request to an authorized team member
  - An authorized team member will see the merge request assigned to them, briefly validate it, and merge it to master (Optional CD in the future)
  - Sync `Prod Management ServiceSet` in ArgoCD
  - Your new service is now live on production :)


