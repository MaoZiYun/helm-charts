# Service ExternalIPs Support Task Plan

- [x] Task 1: Add service externalIPs default value
    - 1.1: Open `/Users/yaozaiyong/porter/helm-charts/charts/porter/values.yaml`
    - 1.2: Add `externalIPs: []` under the existing `service` block
    - 1.3: Keep the existing `service.type` and `service.port` defaults unchanged

- [x] Task 2: Render externalIPs in Porter Service template
    - 2.1: Open `/Users/yaozaiyong/porter/helm-charts/charts/porter/templates/service.yaml`
    - 2.2: Add a conditional `with .Values.service.externalIPs` block under `spec.type`
    - 2.3: Render the value with `toYaml` and `nindent 4` so list indentation is valid
    - 2.4: Leave existing `ports` and `selector` rendering unchanged

- [x] Task 3: Validate Helm rendering behavior
    - 3.1: Run `helm template` with default values to confirm `externalIPs` is omitted
    - 3.2: Run `helm template` with `service.externalIPs` set to one or more IPs to confirm `spec.externalIPs` is rendered
    - 3.3: Check that no unrelated templates are changed
