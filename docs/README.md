# Conformance Tests per Release

## Summary

Each `AIConformance-1.xx.yaml` release document contains a list of questions required for that release of Kubernetes.
The goal of these files is to act as a template to be filled by the vendor for the corresponding Kubernetes Release. 

All `MUST` statements must be fulfilled and documented to achieve Conformance. All `SHOULD` components can be filled and documented.

## Release Freeze

The `AIConformance` release documents are bound to the Kubernetes Release Freeze and will not change after the Freeze for the upcoming 
Kubernetes Release. 
Once the Freeze is in place a new `AIConformance-1.xx.yaml` will be added to this repo for the upcoming Kubernetes Release, so vendors can submit conformance results for review and certification by the CNCF. The content of the 
Conformance self assessment is being defined by the Kubernetes Community - currently [WG AI Conformance](https://github.com/kubernetes/community/tree/master/wg-ai-conformance)