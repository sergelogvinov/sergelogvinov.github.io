# Kubernetes on Hybrid Cloud: Persistent storages

Persistent storage is a key component of any production-grade Kubernetes deployment.

It is very challenging to rely on just one storage solution for all Kubernetes nodes, especially in hybrid cloud environments. In such cases, we can group the nodes based on their cloud or hypervisor type and use the storage solution that works best for each group. For example, one group of nodes might use a cloud provider's managed storage, while another might rely on local or on-premises storage.

However, many Kubernetes resources or operators are not designed to work seamlessly with multiple types of storage classes. For example, the StatefulSet resource requires a single type of PersistentVolumeClaim (PVC) template to create PVCs during the scale-up process. This means all PVCs created by the StatefulSet will use the same storage class as defined in the template. Once the PVC is created, changing the storage class is not straightforward (technically, it is possible, but it involves a complex process and risks downtime).

## Statefulset and Persistent Volume Claim

How to use different storage classes for one statefulset?

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: test
spec:
  ...
  volumeClaimTemplates:
    - metadata:
        name: storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
        storageClassName: "storage-class-1"
```

Let's assume we have two storage classes: `storage-class-1` and `storage-class-2`. We want to have the first pod to use `storage-class-1` and the second pod to use `storage-class-2` in one statefulSet deployment.

By default, all PersistentVolumeClaims (PVCs) created by the StatefulSet will use `storage-class-1` if it is specified in the PVC template. To assign `storage-class-2` to the second pod, we need to manually create a PVC with `storage-class-2` before scaling up the StatefulSet. After creating this PVC, we can scale up the StatefulSet, and the second pod will automatically use the pre-created PVC with `storage-class-2`.

Additionally, it is possible to permanently change the storage class of PVCs after the StatefulSet has been created. However, this process is not straightforward and typically involves detaching the workload, editing the PVCs, and carefully managing the transition to avoid downtime.

How we can change the storage class of the statefulSet without downtime:

```shell
# Delete the statefulSet resources without deleting the pods
kubectl delete statefulset test --cascade=orphan
# Change the storage class in statefulset yaml, apply it and scale up the statefulSet
kubectl apply -f statefulset.yaml
```

The flag `--cascade=orphan` is very important because it will not delete the pods, and workloads will not be interrupted. After applying the new statefulSet, the new pods will be created with the new storage class.

## Operator and Persistent Volume Claim

Using an operator, we can also configure different storage classes, similar to what we did with the StatefulSet. However, there is an additional step involved when working with operators. Before making any changes to the storage classes, we need to scale down the operator. This is because the operator continuously manages and reconciles the resources it controls, and it may revert any manual changes we make to align with its predefined configurations.

Once the operator is scaled down, we can manually create or modify the PVCs to use different storage classes. After completing the changes, we can safely scale the operator back up to resume its management tasks.

## Dynamic Persistent Volume Claim

All of the above solutions have a major drawback: they require us to know the exact name of the PVC that will be created by the StatefulSet or the operator. This can make the process hard and error-prone, especially in dynamic or large-scale environments.

A better way to solve this problem is to use a universal, dynamic approach for PVC creation. One solution is to leverage a special StorageClass resource. With this method, Kubernetes can automatically create PVCs and PVs with the appropriate storage class based on the cloud environment or node type. This approach removes the need for manual intervention or pre-defining PVC names and simplifies storage management in hybrid or multi-cloud environments. It ensures that storage provisioning is both seamless and adaptable to the underlying infrastructure.

This solution is already available with the [Hybrid CSI](https://github.com/sergelogvinov/hybrid-csi-plugin). The Hybrid CSI plugin acts as a proxy between Kubernetes and existing CSI plugins. It does not have its own storage implementation or any specific knowledge of cloud providers. Instead, it forwards PVC creation requests to the appropriate CSI plugin during the PVC creation process.

With the Hybrid CSI plugin, you can configure priorities for different types of CSI plugins. During PVC provisioning, the Hybrid CSI plugin will select and use the first available plugin based on the defined priorities. This makes it an efficient and flexible solution for managing storage across diverse environments without needing to hard-code storage configurations or manage complex PVC naming schemes.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hybrid
parameters:
  storageClasses: proxmox,hcloud-volumes
provisioner: csi.hybrid.sinextra.dev
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

In this example, we have a StorageClass named hybrid, which combines two underlying storage classes: proxmox and hcloud-volumes.
* The `proxmox` storage class is used for the Proxmox hypervisor.
* The `hcloud-volumes` storage class is designed for the Hetzner Cloud.

When a pod is deployed, the Hybrid CSI plugin determines the underlying environment where the pod is running. If the pod is running on a Proxmox hypervisor, the plugin forwards the PVC creation request to the proxmox storage plugin. Conversely, if the pod is running on Hetzner Cloud, the plugin uses the Hcloud CSI plugin to handle the PVC creation.

## Example

So we've deployed the statefulSet with the `hybrid` storage class with two replicas.

```shell
$ kubectl -n default get pods,pvc
NAME         READY   STATUS    RESTARTS   AGE
pod/test-0   1/1     Running   0          31s
pod/test-1   1/1     Running   0          31s

NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/storage-test-0   Bound    pvc-64440564-75e9-4926-82ef-280f412b11ee   1Gi        RWO            hybrid         <unset>                 32s
persistentvolumeclaim/storage-test-1   Bound    pvc-811cc51e-9c9f-4476-92e1-37382b175e7f   10Gi       RWO            hybrid         <unset>                 32s
```

After deploying the StatefulSet, we can see that the PersistentVolumes (PVs) were created using the `proxmox` and `hcloud-volumes` storage classes. They have different sizes, because the Hetzner Cloud has a minimum size of 10Gi, proxmox has a less than 1Gi.

```shell
$ kubectl -n default get pv pvc-64440564-75e9-4926-82ef-280f412b11ee pvc-811cc51e-9c9f-4476-92e1-37382b175e7f
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS     VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-64440564-75e9-4926-82ef-280f412b11ee   1Gi        RWO            Delete           Bound    default/storage-test-0   proxmox          <unset>                          84s
pvc-811cc51e-9c9f-4476-92e1-37382b175e7f   10Gi       RWO            Delete           Bound    default/storage-test-1   hcloud-volumes   <unset>                          81s
```

## Conclusion

The Hybrid CSI plugin is an excellent solution for hybrid cloud environments. It enables you to use different storage classes for different nodes without needing to modify StatefulSet or operator resources. The plugin simplifies storage management by dynamically forwarding PVC creation requests to the appropriate storage class based on the underlying infrastructure.

This plugin is easy to use and requires no additional configuration once set up. You can even configure it as the default storage class for all your deployments, allowing you to manage storage seamlessly without worrying about specifying or switching between storage classes. This makes it a powerful and flexible tool for modern hybrid cloud Kubernetes deployments.

## References

* [Hybrid CSI](https://github.com/sergelogvinov/hybrid-csi-plugin)
* [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
* [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/)
* [Persistent Volume Claim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
