
For this example ill be using Management Services, which is a logical group of services that are deployed on management hubs (Previously known as Management Hub Components).

2 Repos will coexist:
- Preprod Management Service Set
- Prod Management Service Set

ServiceSet - A repo template for Argo ApplicationSet which deploys a set of services defined in values.yaml to a specific cluster.

A service is a helm chart reference defined by:
- serviceName
- repoURL
- targetRevision
- path

Lets say I want to add a feature to a project that is a part of Management Service Set (After application CI has passed successfully).
I'll create a new branch for the feature in the helm chart and make the desired changes.

Then, to check the project in a preprod environment, I will change the targetRevision of the desired service reference in "Preprod Management Service" Set's values.yaml, and sync `Preprod Management Service Set` in Argo. 

Lets say everything went well and the application is behaving expectedly and integrating well with its environment.
Now we want to deploy it to production.

All we have to do now is to create a merge request in the stand-alone helm chart from the branch we worked on to `master` (Preferebly squash commits to avoid `Updated values.yaml` garbage commits and give the single commit a meaningful name).

Soon after, An authorized team members will see the merge request assigned to them, briefly validate it (Because it's already tested in preprod environment), and merge it to `master` branch.

After the merge request was accepted, create a tag from `master` following semantic versioning practice (1.0.0) and document your features and changes
The tag is important for documentation of the project's history, and for emergency cases when a rollback is needed

Then, `Prod Management Service Set`, which is pointing to 

`targetRevision: master` for the same project references as `Preprod Management Service Set` will automatically detect the changes in that git repository.

The only thing left to do is to sync `Prod Management Service Set` in Argo, and you are done! 
The changes will now be live on production



