### **Meilisearch deployment in K8s**


## Requirements

- EBS CSI DRIVER `Will be added in CICD` [@OfficialDocumentation](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)

---

### **EBS Volumes GP2/GP3**

| Volume Type                 | gp3                       | gp2                                                   |
|-----------------------------|---------------------------|-------------------------------------------------------|
| Volume Size                 | 1 GiB – 16 TiB            | 1 GiB – 16 TiB                                        |
| Default/Baseline IOPS       | 3000                      | 3 IOPS/GiB (min. 100 IOPS, max. 16,000 IOPS)          |
| Max IOPS/volume             | 16,000                    | 16,000                                                |
| Default/Baseline Throughput | 125 MiB/s                 | 128 MiB/s (min.) to 250 MiB/s (max.)                  |
| Max Throughput/volume       | 1,000 MiB/s               | 250 MiB/s                                             |
| Price                       | $0.08/GiB-month           | $0.10/GiB-month                                       |
|                             | 3,000 IOPS free and       |                                                       |
|                             | $0.005/prov. IOPS-month   |                                                       |
|                             | over 3,000 IOPS           |                                                       |
|                             | 125 MiB/s free and        |                                                       |
|                             | $0.04/prov. MiB/s-month   |                                                       |
|                             | over 125 MiB/s            |                                                       |



### **Calcul / Example**

Consider a 2 TB volume which requires 9000 IOPS. To provision these volumes:

gp2: 3000GB * $0.10/GB-month = $300 per month.

but for gp3 :

| Storage       | 2000GB * $0.08/GB-month                              | $160  |
|---------------|------------------------------------------------------|-------|
| IOPS          | 3000 IOPS free. Provision 6000 IOPS: 6k*0.005        | $30   |
| Throughput    | 125 MB/s free. Provision 125 MB/s:                   | $5    |
|               | 125 MB/s * $0.04/provisioned MB/s-month*             |       |
|               | or Provision up to max throughput, 1000 – 125 = 875: | $35   |
|               | 875 MB/s * $0.04**                                   |       |
| Total         |                                                      | $195* |            
|               |                                                      | $230**|

As you can see, GP3 volumes offer a number of advantages over GP2 volumes, including:

- Better performance: GP3 volumes offer up to 20% better performance than GP2 volumes.
- Lower price: GP3 volumes are priced at $0.02/GB-month less than GP2 volumes.
- Automatic tiering: GP3 volumes automatically tier data to the most cost-effective storage tier, which can save money

Overall, GP3 volumes are a good choice for workloads that require high performance and low cost. If your workload is predictable and you need a high level of burst capacity, then GP2 volumes may be a better choice.



---

First of all, we need to set up a PVC to achieve this, follow the steps below:

### **DYNAMIC**

Dynamic PV Provisioning: When you create a PVC (Persistent Volume Claim), it can automatically create a PV (Persistent Volume) for you if dynamic provisioning is enabled in the Kubernetes cluster. However, this dynamic provisioning process depends on the storage class configuration. If a suitable storage class is available and the PVC requests storage from it, a PV will be dynamically created to fulfill the PVC's requirements.

- PV and PVC Binding: When a PVC is created and dynamic provisioning is used, Kubernetes will attempt to bind the PVC to an available PV that satisfies the PVC's requirements in terms of storage capacity, access mode, and storage class. Once the binding is successful, the PV and PVC are associated with each other.

- Deletion Behavior: If you delete a POD, the PVC and PV will remain intact, and you can attach the same PVC to another POD if needed. However, if you delete both the POD and the PVC, the PV will change its state to Released.

Reclaim Policies (Written in storage class):

- Retain: If the reclaim policy is set to Retain, the PV will change its state to Released when the associated PVC is deleted. The data on the PV will not be deleted automatically, and you can manually reuse the PV by removing the claim reference from it using
 `kubectl patch pv <pvname> -p "{\"spec\":{\"claimRef\": null}}"`
- Delete: If the reclaim policy is set to Delete, the PV will be deleted, along with all its data, when the associated PVC is deleted.
- Recycle (Deprecated): The Recycle reclaim policy is deprecated and is used to empty the PV when the associated PVC is deleted.


### **For The ClaimName:**

Make sure to add/edit persistentVolumeClaim in `deployment`

File : `deployment.yaml` (Line 91)
```
  persistentVolumeClaim:
    claimName: <nameofPVC>
```

Ensure to add all the files in kustomization.yaml resources:

```
resources:
- ./deployment.yaml
- ./pvc.yaml
- ./ingress.yaml
```

To apply all the resources, run the following command:

`kustomize build . | kubectl apply -f - `

If you want to delete all the resources:

`kustomize build . | kubectl delete -f - `

### **Ingress**

Request your domain in AWS ACM. Once the request is approved and the certificate is issued, copy the Certificate ARN in
 `ingress.yaml`

File : `ingress.yaml` (Line 10)
```
    alb.ingress.kubernetes.io/certificate-arn: <yourcertificate> # Certificate ARN in ACM (for your domain)
```

Next, change the domain to your desired domain:

File : `ingress.yaml` (Line 16)
```
    - host: "meilisearch.domain.com"
```

### **Test Phase**

now let's insert some indexes using the json file `movies.json`


In this example we didn't set any masterKey, which means we can run this command without any issues, you can modify `aSampleMasterKey` to anything you want, but once you set up the masterKey in meilisearch's manifest, this will no longer work and will receive an error :

```
{"message":"The provided API key is invalid.","code":"invalid_api_key","type":"auth","link":"https://docs.meilisearch.com/errors#invalid_api_key"}
```

CMD without setting up MasterKey in meilisearch's manifest :
```
cd .\assets\data\sample\
curl   -X POST 'http://localhost:7700/indexes/movies/documents?primaryKey=id'   -H 'Content-Type: application/json'   -H 'Authorization: Bearer aSampleMasterKey'   --data-binary @movies.json
```

Since we're creating a secret.env for the MasterKEY, configMap for `MEILI_ENV` and `MEILI_NO_ANALYTICS`. It's defined in `statefulset.yaml`

```
          env:
          - name: MEILI_MASTER_KEY
            valueFrom:
              secretKeyRef:
                name: meilisearch-secret
                key: MEILI_MASTER_KEY

          - name: MEILI_ENV
            valueFrom:
              configMapKeyRef:
                name: meilisearch-configmap
                key: MEILI_ENV

          - name: MEILI_NO_ANALYTICS
            valueFrom:
              configMapKeyRef:
                name: meilisearch-configmap
                key: MEILI_NO_ANALYTICS
```

 If you want to change it, you will have to add it to your repository's secret or variable environment :

- Go to your repository's settings.
- Under the settings, navigate to the 'Secrets' section.
- Click on 'New repository secret' or 'Add a secret environment' button.
- For the name, enter `MEILI_MASTER_KEY`.
- Input the new master key value in the 'Value' field.


The secrets & config variables has to be defined in a workflow `Github Actions` : 

```
      - name: Create secrets file
        run: |
          cd ./k8s
          echo "MEILI_MASTER_KEY=${{ secrets.MEILI_MASTER_KEY }}" > secret.env
```

```
      - name: Create ConfigMap file
        run: |
          cd ./k8s
          echo "MEILI_ENV=${{ vars.MEILI_ENV }}" > configmap.env
          echo "MEILI_NO_ANALYTICS=${{ vars.MEILI_NO_ANALYTICS }}" >> configmap.env
```
Once you run the workflow, it will create a `secret.env` , `configmap.env` file, and Meilisearch can directly read from the secret that was created.


```
curl   -X POST 'http://localhost:7700/indexes/movies/documents?primaryKey=id'   -H 'Content-Type: application/json'   -H 'Authorization: Bearer <yournewmasterkey>'   --data-binary @movies.json
```

---

### **For the snapshot Life-cycle**

I have already added how to do it in this repo : [@Snapshot](https://github.com/TrouveTaVoie/lago/tree/Deployment/LAGO-K8S#for-the-snapshot-life-cycle)
