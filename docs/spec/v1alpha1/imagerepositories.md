<!-- -*- fill-column: 100 -*- -->
# Image Repositories

The `ImageRepository` API specifies how to scan OCI image repositories. A _repository_ is a
collection of images -- e.g., `alpine`, as opposed to a specific image, e.g.,
`alpine:3.1`. `ImagePolicy` objects can then refer to an `ImageRepository` in order to select a
specific image from those scanned.

## Specification

The **ImageRepository** type holds details for scanning a particular image repository.

```go
// ImageRepositorySpec defines the parameters for scanning an image
// repository, e.g., `fluxcd/flux`.
type ImageRepositorySpec struct {
	// Image is the name of the image repository
	// +required
	Image string `json:"image,omitempty"`
	// Interval is the length of time to wait between
	// scans of the image repository.
	// +required
	Interval metav1.Duration `json:"interval,omitempty"`

	// Timeout for image scanning.
	// Defaults to 'Interval' duration.
	// +optional
	Timeout *metav1.Duration `json:"timeout,omitempty"`

	// SecretRef can be given the name of a secret containing
	// credentials to use for the image registry. The secret should be
	// created with `kubectl create secret docker-registry`, or the
	// equivalent.
	SecretRef *corev1.LocalObjectReference `json:"secretRef,omitempty"`

	// This flag tells the controller to suspend subsequent image scans.
	// It does not apply to already started scans. Defaults to false.
	// +optional
	Suspend bool `json:"suspend,omitempty"`
}
```

The `secretRef` names a secret in the same namespace that holds credentials for accessing the image
repository. This secret is expected to be in the same format as for
[`imagePullSecrets`][image-pull-secrets]. The usual way to create such a secret is with

    kubectl create secret docker-registry ...

If you are running on a platform (e.g., AWS) that links service permissions (e.g., access to ECR) to
service accounts, you may need to create the secret using tooling for that platform instead. There
is advice specific to some platforms [in the image automation guide][image-auto-provider-secrets].

For a publicly accessible image repository, you don't need to provide a `secretRef`.

The `Suspend` field can be set to `true` to stop the controller scanning the image repository
specified; remove the field value or set to `false` to resume scanning.

## Status

```go
// ImageRepositoryStatus defines the observed state of ImageRepository
type ImageRepositoryStatus struct {
	// +optional
	Conditions []metav1.Condition `json:"conditions,omitempty"`

	// ObservedGeneration is the last reconciled generation.
	// +optional
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`

	// CannonicalName is the name of the image repository with all the
	// implied bits made explicit; e.g., `docker.io/library/alpine`
	// rather than `alpine`.
	// +optional
	CanonicalImageName string `json:"canonicalImageName,omitempty"`

	// LastScanResult contains the number of fetched tags.
	// +optional
	LastScanResult *ScanResult `json:"lastScanResult,omitempty"`

	meta.ReconcileRequestStatus `json:",inline"`
}
```

The `CanonicalName` field gives the fully expanded image name, filling in any parts left implicit in
the spec. For instance, `alpine` expands to `docker.io/library/alpine`.

The `LastScanResult` field gives a summary of the most recent scan:

```go
type ScanResult struct {
	TagCount int         `json:"tagCount"`
	ScanTime metav1.Time `json:"scanTime,omitempty"`
}
```

### Conditions

There is one condition used: the GitOps toolkit-standard `ReadyCondition`. This will be marked as
true when a scan succeeds, and false when a scan fails.

[image-pull-secrets]: https://kubernetes.io/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod
[image-auto-provider-secrets]: https://toolkit.fluxcd.io/guides/image-update/#imagerepository-cloud-providers-authentication