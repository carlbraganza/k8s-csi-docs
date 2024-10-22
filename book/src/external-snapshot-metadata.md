# CSI external-snapshot-metadata

## Status and Releases

**Git Repository:** [https://github.com/kubernetes-csi/external-snapshot-metadata](https://github.com/kubernetes-csi/external-snapshot-metadata)

### Supported Versions

Latest stable release | Branch | Min CSI Version | Max CSI Version | Container Image | [Min K8s Version](project-policies.md#minimum-version) | [Max K8s Version](project-policies.md#maximum-version) | [Recommended K8s Version](project-policies.md#recommended-version) |
--|--|--|--|--|--|--|--
v0.1.0 | [v0.1.0](https://github.com/kubernetes-csi/external-snapshot-metadata/releases/tag/v0.1.0) | [v1.10.0](https://github.com/container-storage-interface/spec/releases/tag/v1.10.0) | - | gcr.io/k8s-staging-sig-storage/csi-snapshot-metadata:v0.1.0 | v1.32 | - | v1.32


## Alpha

### Description
This sidecar securely serves snapshot metadata to Kubernetes clients through the
[Kubernetes SnapshotMetadata gRPC Service](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/3314-csi-changed-block-tracking#the-kubernetes-snapshotmetadata-service-api).

The sidecar authenticates and authorizes each Kubernetes backup application request made through the
Kubernetes SnapshotMetadata gRPC Service API.
It then acts as a proxy as it fetches the desired metadata from the CSI driver and
streams it directly to the requesting application with no load on the Kubernetes API server.

See ["The External Snapshot Metadata Sidecar"](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/3314-csi-changed-block-tracking#the-external-snapshot-metadata-sidecar)
section in the CSI Changed Block Tracking KEP for additional details on the sidecar.

### Usage
Backup applications, identified by their authorized ServiceAccount objects,
directly communicate with the sidecar using the
[Kubernetes SnapshotMetadata gRPC Service](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/3314-csi-changed-block-tracking#the-kubernetes-snapshotmetadata-service-api).
The authorization needed is described in the 
["Risks and Mitigations"](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/3314-csi-changed-block-tracking#risks-and-mitigations)
section of the CSI Changed Block Tracking KEP.
In particular, this requires the ability to use the Kubernetes
[TokenRequest](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/)
API and to access the objects required to use the service.

The existence of this optional service is advertised by the presence of a
[Snapshot Metadata Service CR](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/3314-csi-changed-block-tracking#snapshot-metadata-service-custom-resource),
named for the CSI driver that provisions the PersistentVolume and VolumeSnapshot objects involved.
The CR contains the service endpoint and CA certificate, and an audience string for authentication.
The backup application must use the Kubernetes
[TokenRequest](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/)
API with the audience string to obtain a Kubernetes authentication token for use in the
Kubernetes SnapshotMetadata gRPC Service call.
The backup application should establish trust for the CA certificate before making the gRPC call
to the service endpoint.
VolumeSnapshot metadata can be lengthy, and the Kubernetes SnapshotMetadata gRPC Service supports
restarting an interrupted metadata request from an intermediate point in case of failure.

The sidecar repository contains a
[snapshot-metadata-lister](https://github.com/kubernetes-csi/external-snapshot-metadata/tree/master/examples/snapshot-metadata-lister)
example command that illustrates the use of the Kubernetes SnapshotMetadata gRPC Service in a Go application.
It uses the
[pkg/iterator](https://github.com/kubernetes-csi/external-snapshot-metadata/tree/master/pkg/iterator)
utility package, which may be used by backup applications if desired.

### Deployment
The CSI `external-snapshot-metadata` sidecar should be deployed by
CSI drivers that support the
[Changed Block Tracking](./changed-block-tracking.md) feature.
The sidecar must be deployed in the same pod as the CSI driver and
will communicate with its CSI [SnapshotMetadata](https://github.com/container-storage-interface/spec/blob/master/spec.md#snapshot-metadata-service-rpcs)
and [Identity](https://github.com/container-storage-interface/spec/blob/master/spec.md#identity-service-rpc) gRPC services
over a UNIX domain socket.

The sidecar should be configured to run under the authority of its
CSI driver ServiceAccount, which must be authorized as described in the 
["Risks and Mitigations"](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/3314-csi-changed-block-tracking#risks-and-mitigations)
section of the CSI Changed Block Tracking KEP.
In particular, this requires the ability to use the Kubernetes
[TokenReview](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-review-v1/)
and
[SubjectAccessReview](https://kubernetes.io/docs/reference/kubernetes-api/authorization-resources/subject-access-review-v1/)
APIs.

A Service object must be created for the TCP based [Kubernetes SnapshotMetadata](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/3314-csi-changed-block-tracking#the-kubernetes-snapshotmetadata-service-api)
gRPC service implemented by the sidecar.

A [SnapshotMetadataService CR](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/3314-csi-changed-block-tracking#snapshot-metadata-service-custom-resource),
named for the CSI driver, must be created to advertise the
availability of this optional feature.
The CR contains the CA certificate and Service endpoint address
of the sidecar and the audience string needed for the client
authentication token.

See the sample [Hostpath CSI driver](example.md) for an illustration on how to deploy a CSI driver
that supports this feature.
