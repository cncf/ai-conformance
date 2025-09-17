# How to submit conformance results

## The self assessment

The standard set of conformance requirements are defined in `docs/AiConformance-x.yy.yaml` for each Kubernetes Version

## Uploading

Prepare a PR to
[https://github.com/cncf/ai-conformance](https://github.com/cncf/ai-conformance).
Here are [directions](https://help.github.com/en/articles/creating-a-pull-request-from-a-fork) to
prepare a pull request from a fork.
In the descriptions below, `X.Y` refers to the kubernetes major and minor
version, and `$dir` is a short subdirectory name to hold the results for your
product.  Examples would be `gke` or `openshift`.


### Contents of the PR

You need to submit the completed self assesment yaml for the relevant major/minor Version of Kubernetes

```
vX.Y/$dir/PRODUCT.yaml: See below.
```

#### PRODUCT.yaml

This file describes your product. It is YAML formatted with the following root-level fields. Please fill in as appropriate.

| Field                   | Description                                                                                                                                                                                                                             |
|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `vendorName`            | Name of the legal entity that is certifying. This entity must have a signed participation form on file with the CNCF                                                                                                                    |
| `kubernetesVersion`     | Kubernetes Version to verify                                                                                                                                                                                                            |
| `platformName`          | Name of the product being certified.                                                                                                                                                                                                    |
| `platformVersion`       | The version of the product being certified (not the version of Kubernetes it runs).                                                                                                                                                     |
| `websiteUrl`            | URL to the product information website                                                                                                                                                                                                  |
| `repoUrl`               | If your product is open source, this field is necessary to point to the primary GitHub repo containing the source. It's OK if this is a mirror. OPTIONAL                                                                                |
| `documentationUrl`     | URL to the product documentation                                                                                                                                                                                                        |
| `productLogoUrl`      | URL to the product's logo, (must be in SVG, AI or EPS format -- not a PNG -- and include the product name). OPTIONAL. If not supplied, we'll use your company logo. Please see logo [guidelines](https://github.com/cncf/landscape#logos) |
| `description`           | One sentence description of your offering                                                                                                                                                                                               |
| `contactEmailAddress` | An email address which can be used to contact maintainers regarding the product submitted and updates to the submission process                                                                                                         |

Examples below are for a fictional Kubernetes implementation called _Turbo
Encabulator_ produced by a company named _Yoyodyne_.

```yaml
metadata:
  kubernetesVersion: v1.34
  platformName: Turbo Encabulator
  platformVersion: v1.7.4
  vendorName: Yoyodyne
  website_url: https://yoyo.dyne/turbo-encabulator
  repo_url: https://github.com/yoyo.dyne/turbo-encabulator
  documentation_url: https://yoyo.dyne/turbo-encabulator/docs
  product_logo_url: https://yoyo.dyne/assets/turbo-encabulator.svg
  description: "The Yoyodyne Turbo Encabulator is a superb Kubernetes distribution for all of your Encabulating needs."
  contact_email_address: yoyodyne@turbo-encabulator.org
```

### Requirements

The Self conformance file needs to be submitted without any adjustments or changes in spec to the fields `id, description, level`.
The fields `status, evidence, and notes` need to be filled if the `level` is `MUST`.

To reach conformance all `MUST` spec fields need to be addressed. The evidence needs to be publicly reachable. 


## Amendment for Private Review

If you need a private review for an unreleased product, please email a zip file containing what you would otherwise submit
as a pull request to conformance@cncf.io and the documentation for the evidence. We'll review and confirm that you are ready to be Certified AI Kubernetes Platform
as soon as you open the pull request. We can then often arrange to accept your pull request soon after you make it, at which point you become Certified Kubernetes.

## Review

A reviewer will shortly comment on and/or accept your pull request, following this [process](reviewing.md).
If you don't see a response within 10 business days, please contact conformance@cncf.io.


## Issues

If you have problems certifying that you feel are an issue with the conformance
program itself (and not just your own implementation), you can file an issue in
the [repository](https://github.com/cncf/ai-conformance). Questions and
comments can also be sent to the working group's
[mailing list and slack channel](README-WG.md).
[WG AI Conformance](https://github.com/kubernetes/community/tree/master/wg-ai-conformance)
is the change controller of the conformance definition. 
