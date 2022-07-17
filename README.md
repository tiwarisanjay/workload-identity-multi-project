# workload-identity-multi-project

Workload Identity between two different project

This article is for you 
- If you are working production and sharing resources between project and you know what workload identity is.
- If you know workload identity is but don't know how to get it working between two different projects. 

If you would like to learn theory on Workload Identity, best place is Google documents. Believe me no one else can explain it batter than google doc. For any topic google documents is really good. I find it way batter than Azure/Aws or any other documentations. It's a personal view and do not really want to hurt any feeling for Azure and AWS cloud fans :). 
Problem Statement: I would like to create a google secret in my Project A and would like to access the same in GKE which is running in Project B. 
Solution: First if you would like to see it in Action I have recorded a Youtube Video on that watch that and in 10 min your are ready to go. But if you like reading here you go with Step by Step : 
1. Create your first project with very unique name ;) as firstproject (You can name it anything just replace it everywhere where I have first project written)
2. Set your project to your first project firstprojectusing gcloud init Follow THIS document if you do not know how to create new profile at your local
3. Create a cluster with workload identity enabled : 
    ```bash
    gcloud container clusters create test-cluster \
    --region=us-central1 \
    --workload-pool=firstproject.svc.id.goog
    ```
4. Get credentials for your cluster:
```bash
gcloud container clusters get-credentials test-cluster 
```
5. Create new namespace 
kubectl create namespace testns 
6. Create a kubernetes service account. 
kubectl create serviceaccount testksa \
    --namespace testns
7. Now create your second project as secondproject . 
8. Switch profile to second project. using gcloud init again. You can follow google document for that.
9. Now lets create a google service account in second project: 
    gcloud iam service-accounts create testgsa \
        --project=secondproject
    We will give some permission to our google service account in order to view project and read secrets in second project. 
10. Lets assign Viewer and secretmanager.secretAccessor role
gcloud projects add-iam-policy-binding secondproject --member=serviceAccount:testgsa@secondproject.iam.gserviceaccount.com --role=roles/viewer
gcloud projects add-iam-policy-binding secondproject --member=serviceAccount:test-gsa@secondproject.iam.gserviceaccount.com --role=roles/secretmanager.secretAccessor
11. Now lets create workload identity 
gcloud iam service-accounts add-iam-policy-binding testgsa@secondproject.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:firstproject.svc.id.goog[testns/testksa]"
12. Lets annotate your kubernetes service account which is in firstproject with email address of IAM service account which is in second project [You are still under the secondproject profile and you can run kubectl on first project because your have got the credentials for cluster before.]: 
kubectl annotate serviceaccount testksa \
    --namespace testns \
    iam.gke.io/gcp-service-account=testgsa@secondproject.iam.gserviceaccount.com 
13. Now lets create a secret in second project which we are going to access from a pod running in GKE of first project. 
gcloud secrets create test-secret \
    --replication-policy="automatic"
echo -n "this is my super secret data" | \
    gcloud secrets versions add test-secret --data-file=-
14. Now download a pod yaml from my github project POD YAML.