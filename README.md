# LFX-Mentorship-2025-Kyverno-Proposal

#### This repository contains the documentation and proposal for the LFX Mentorship 2025 Term 3 [Kyverno](https://github.com/kyverno/kyverno/issues/13709), focusing on converting the Kyverno sample policies to use new CEL based policy types.

- **Applied:** To convert sample policies into CEL based policy type.

## 📚Table of Contents
### 1. Conversion of sample policy to CEL based policy type.
- [Sample Policy Traditional Kyverno Format](#1-sample-policy-traditional-kyverno-format)
- [New CEL Based Policy](#2-new-cel-based-policy)
- [General Pattern for Policy Conversion](#3-general-pattern-for-policy-conversion)

### 2. Testing
- [Testing of the New CEL Based Policy](#testing-of-the-new-cel-based-policy)

### 3. Miscellaneous
- [Mentee Details](#1-mentee-details)
- [References](#2-references)

## 🌐Introduction
The following proposal aims to provide a blueprint for Kyverno Project to convert the traditional Kyvenro format sample policies into the new CEL Based `v1alpha1` api policies. For this demonstration one sample policy will be converted into the CEL based format and then being tested to maintain the functionality equivalence between the both.

## Conversion of sample Policy to CEL based policy type
### 1. Sample Policy Traditional Kyverno Format

This is a traditional Kyverno sample policy format of Validate type 

Add PSA Namespace Reporting in CEL expressions

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-psa-namespace-reporting
  annotations:
    policies.kyverno.io/title: Add PSA Namespace Reporting in CEL expressions
    policies.kyverno.io/category: Pod Security Admission, EKS Best Practices in CEL 
    policies.kyverno.io/severity: medium
    kyverno.io/kyverno-version: 1.11.0
    policies.kyverno.io/minversion: 1.11.0
    kyverno.io/kubernetes-version: "1.26-1.27"
    policies.kyverno.io/subject: Namespace
    policies.kyverno.io/description: >-
      This policy is valuable as it ensures that all namespaces within a Kubernetes 
      cluster are labeled with Pod Security Admission (PSA) labels, which are crucial
      for defining security levels and ensuring that pods within a namespace operate 
      under the defined Pod Security Standard (PSS). By enforcing namespace labeling,
      This policy audits namespaces to verify the presence of PSA labels. 
      If a namespace is found without the required labels, it generates and maintain 
      and ClusterPolicy Report in default namespace. 
      This helps administrators identify namespaces that do not comply with the 
      organization's security practices and take appropriate action to rectify the 
      situation.
spec:
  validationFailureAction: Audit
  background: true
  rules:
  - name: check-namespace-labels
    match:
      any:
      - resources:
          kinds:
            - Namespace
          operations:
          - CREATE
          - UPDATE
    validate:
      cel:
        expressions:
          - expression: "object.metadata.?labels.orValue([]).exists(label, label.startsWith('pod-security.kubernetes.io/') && object.metadata.labels[label] != '')"
            message: This Namespace is missing a PSA label.
```

Now using the `v1alpha1` [Validating Policy](https://github.com/kyverno/kyverno/blob/main/api/policies.kyverno.io/v1alpha1/validating_policy.go) this policy will be converted into the CEL based format in which new feilds will be adopted and removal of deprecated ones.

### 2. New CEL Based Policy

For the sample policy Add PSA Namespace Reporting in CEL expressions, the new CEL Based Policy using is here as follows:

```yaml
apiVersion: kyverno.io/v1alpha1
kind: ValidatingPolicy
metadata:
  name: add-psa-namespace-reporting
  annotations:
    policies.kyverno.io/title: Add PSA Namespace Reporting in CEL expressions
    policies.kyverno.io/category: Pod Security Admission, EKS Best Practices in CEL
    policies.kyverno.io/severity: medium
    kyverno.io/kyverno-version: 1.11.0
    policies.kyverno.io/minversion: 1.11.0
    kyverno.io/kubernetes-version: "1.26-1.27"
    policies.kyverno.io/subject: Namespace
    policies.kyverno.io/description: >-
      This policy is valuable as it ensures that all namespaces within a Kubernetes 
      cluster are labeled with Pod Security Admission (PSA) labels, which are crucial
      for defining security levels and ensuring that pods within a namespace operate 
      under the defined Pod Security Standard (PSS). By enforcing namespace labeling,
      This policy audits namespaces to verify the presence of PSA labels. 
      If a namespace is found without the required labels, it generates and maintain 
      and ClusterPolicy Report in default namespace. 
      This helps administrators identify namespaces that do not comply with the 
      organization's security practices and take appropriate action to rectify the 
      situation.
spec:
  validationActions:
  - Audit
  
  evaluation:
    background:
      enabled: true
  matchConstraints:    
    resourceRules:
     - apiGroups:
      - ""
      apiVersions:
      - v1
      operations:
      - CREATE
      - UPDATE
      resources:
      - namespaces
  - expression: "object.metadata.labels.orValue({}).exists(label, label.startsWith('pod-security.kubernetes.io/') && object.metadata.labels[label] != '')"
    message: This Namespace is missing a PSA label.
```

Key Changes made:
- `kind`: Changed from `ClusterPolicy` to `ValidatingPolicy`
- `apiVersion`: Changed from `kyverno.io/v1` to `kyverno.io/v1alpha`
- `spec.validationActions`: Replaces `spec.validationFailureAction`. The value `["Audit"]` ensures that violations are reoprted without blocking the resource creation or update, maintaining functional equivalence.
- `spec.evaluation`: A new block that controls how the policy is evaluated. Setting `background.enabled: true` is equivalent to the original `background: true`.
- `spec.matchConstraints`: Replaces the `match` block. It uses `resourceRules` to specify the trageted resources(`namespaces`) and operations(`CREATE`, `UPDATE`).
- `spec.validations`: Replaces the `validate` block. The CEL `expression` and `message` are moved directly into the list. The expression uses the Kyverno CEL helper `orValue({})` to safely handle cases where a namespace has no labels defined.

This is a breakdown of how the fields from the old policy were mapped to the new one, based directly on the `ValidatingPolicySpec` struct.

#### 1. Action on Validation: `validateFailureAction`-> `validateActions`
- Old Policy: `spec.validationFailureAction`: Audit
- Analysis: In the traditional format, this field determined whether to `Audit` or `Enforce` a violation.
- New Struct Mapping
```go
ValidationAction []admissionregistrationv1.ValidationAction`json:"validationActions,omitempty"`
```
To maintain the original functionality of creating a report for non-compliant resources, `validationFailureAction: Audit` is converted to `validationActions:["Audit"].

#### 2. Background Scanning: `background` -> `evaluation.background`
- Old Policy: `spec.background: true`
- Analysis: This flag told Kyverno to scan pre-existing resources, not just the new ones.
- New Struct Mapping
```go
func (s ValidatingPolicySpec) BackgroundEnabled() bool {
    // ...
    if s.EvaluationConfiguration == nil || s.EvaluationConfiguration.Background == nil || s.EvaluationConfiguration.Background.Enabled == nil {
        return defaultValue
    }
    return *s.EvaluationConfiguration.Background.Enabled
}
```
This logic shows that the setting has moved under a nested structure. Therefore `background: true` is converted to `evaulation.background.enabled: true`.

#### 3. Resource Matching: `match` -> `matchConstraints`
- Old Policy: `spec.rules[0].match.any[0].resources`
- Analysis: This block defined which resources and operations the rule applied to. 
- New Struct Mapping
```go
MatchConstraints*admissionregistrationv1.MatchResources`json:"matchConstraints,omitempty"`
```
The old `match` block maps directly to the new `matchConstraints` block. The internal structure follows the standard Kubernetes `MatchResources` type, making it more standarized.

#### 4. Validation Logic: `validate` -> `validations`
- Old Policy: `spec.rules[0].validate.cel`
- Analysis: CEL expression to evaulate and the message to show on failure.
- New Struct Mapping
```go
Validations []admissionregistrationv1.Validation `json:"validations,omitempty"`
```
The list of rules under the old `validate` block is moved into the `validations` list in the format. Each item in the list contains the `expression` and `message`, keeping same equivalence.


### 3. General Pattern for Policy Conversion

Under this project, I will follow this general pattern to convert most sample policies from old format onto the new CEL based format.

Step 1: Change the Boilerpate
- Changing the apiversion
- Changing the kind of policy

Step 2: Metadata
- The entire metadata block can be kept as same without any direct changes.

Step 3: Map the Core Spec Fields
- Changing the core spec fields by comapring the new functionality from go strcut and then mapping it with the old format.

Step 4: Convert Rules for Matching Constraints and PolicyType
- Using the CEL helpers to change the rules and then map and compare the functionality with the old format.

By following this pattern strategy, I will systematically and accurately convert the sample policies into new CEL based policies.

## Testing
### Testing of the New CEL Based Policy

For Validating this new CEL based policy to work like the traditional sample policy, a couple of simple tests will be run in Kubernetes cluster.

The goal is to verify 2 key functionalities:
- 1. Audit on Violation: A `Namespace` without a PSA label is succesfully created, but a policy report is genearted showing a failure.
- 2. Pass on Compliance: A `Namespace` with a PSA label is succesfully created and the policy report shows it passed.

Prerequisites:
- A running Kubernetes cluster
- Applied the policy to the cluster 
```bash
kubectl apply -f validating-policy.yaml
```
Tests:
#### Test 1: Verifying the Audit Violation Scenario
This test checks if a non-compliant `Namespace` is correcly flagged in a policy report.

- 1. Create a non-compliant Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test-no-psa
```
Now, apply it to the cluster
```bash
kubectl apply -f ns-no-psa.yaml
```
- 2. Policy Report
Result:
```bash
NAMESPACE   NAME                         PASS   FAIL   WARN   ERROR   SKIP   AGE
            cpol-add-psa-namespace-reporting  0      1      0      0       0      2m
```

#### Test 2: Verifying the Compliance Scenario
This test ensures that a compliant `Namespace` passes the validation check.

- 1. Create a compliant Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test-with-psa
  labels:
    pod-security.kubernetes.io/enforce: baseline
```
Now, apply it to the cluster
```bash
kubectl apply -f ns-with-psa.yaml
```
- 2. Policy Report 
Result:
```bash
NAME                              PASS   FAIL   WARN   ERROR   SKIP   AGE
cpol-add-psa-namespace-reporting  1      1      0      0       0      5m
```

## Miscellaneous

### 1. Mentee Details
I have applied for the LFX Mentorship Project Convert Kyverno Sample Policies to New CEL-Based Policy Types on the portal.


[Cover Letter](https://docs.google.com/document/d/1cfWFSIDHBBbBq5uegK0_8XO0ICAsex56rZLjfNtyGHE/edit?usp=drive_link)

[Resume](https://drive.google.com/file/d/1h51Ld3suqT6zCPeOfnpqGP5qOxVCGKQF/view?usp=drive_link)

### 2. References
- https://kyverno.io/policies/
- https://kubernetes.io/docs/concepts/overview/kubernetes-api/
- https://docs.docker.com/desktop/features/kubernetes/
