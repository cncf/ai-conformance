# BETA: Get Involved In Defining AI Conformance

*Join Kubernetes AI Conformance WG to get involved with defining kubernetes AI conformance efforts:*

https://github.com/kubernetes/community/tree/master/wg-ai-conformance

Also see this public planning document:

https://docs.google.com/document/d/1hXoSdh9FEs13Yde8DivCYjjXyxa7j4J8erjZPEGWuzc/edit?tab=t.0#heading=h.9j85ih1tpsk

---

# Certified AI Platform Conformance Program

All vendors are invited to submit conformance testing results for review and certification by the CNCF.

If your company provides a platform for Artificial Intelligence and Machine Learning based on CNCF Projects, we encourage you to get certified today.

<!-- For more information please checkout [cncf.io/cai](https://cncf.io/cai). -->

## Prepare

Learn about the [certification requirements](https://github.com/cncf/ai-conformance/blob/master/terms-conditions/Certified_AI_Platform.md) and technical instructions to prepare your product for certification.

## PR Submit

Please check the [instructions](https://github.com/cncf/ai-conformance/blob/master/instructions.md#contents-of-the-pr) for details about how to prepare your PR.
Also, note that any submission made to this repo will need to first pass a number of checks that are verified by the [verify conformance bot](#verify-conformance-bot-coming-soon) before its reviewed by the CNCF.

## Helpful Resources

### [WG AI Conformance Kubernetes](https://github.com/kubernetes/community/tree/master/wg-ai-conformance)]

The responsible body for the Content of the AI Conformance. The goal of this group is to define a standardized set of capabilities, APIs, and configurations that a Kubernetes cluster must offer to reliably and efficiently run AI/ML workloads.

### [Kubernetes Conformance](https://github.com/cncf/k8s-conformance)

Your Platform must be Kubernetes Conformant before achieving Kubernetes AI Conformance. Find more Information about Kubernetes Conformance here.

Reviewing, approving, and driving changes to the conformance test suite; reviewing, guiding, and creating new conformance profiles.

- [Conformance Tests](https://github.com/kubernetes/kubernetes/blob/master/test/conformance/testdata/conformance.yaml)
- [Conformance Testing in Kubernetes](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)

### [Verify conformance bot](#) (Coming Soon!)

This automation provides timely feedback and reduces the time required by the CNCF to confirm that the PR meets all policy requirements. This will be based on the [Kubernetes Conformance Bot](https://github.com/kubernetes-sigs/verify-conformance)
