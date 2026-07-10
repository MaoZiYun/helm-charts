# Service ExternalIPs Support

## 需求场景与处理逻辑

用户需要在 `charts/porter/templates/service.yaml` 渲染的 Kubernetes `Service` 中支持 `spec.externalIPs`，以便通过 Helm values 为 Porter 主服务配置一个或多个外部 IP。

处理逻辑：

- 默认情况下不渲染 `externalIPs` 字段，保持当前 chart 输出不变。
- 当用户在 `values.yaml` 或安装参数中设置 `service.externalIPs` 为非空列表时，在 Service `spec` 下渲染：

```yaml
spec:
  type: ClusterIP
  externalIPs:
    - 192.168.1.10
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
```

- 该字段直接传递给 Kubernetes Service API，不在模板层做 IP 格式校验；Kubernetes API 会负责最终校验。

## 架构与技术方案

这是 Helm chart 模板增强，不涉及应用代码、Deployment、Ingress 或 ServiceAccount 的行为变化。

技术方案：

- 在 Porter chart 的 `service` values 分组下新增 `externalIPs: []` 默认值。
- 在主 Service 模板中使用 Helm `with` 语句只在非空时渲染 `externalIPs`。
- 使用 `toYaml` 和 `nindent 4` 保持 YAML 缩进正确，并允许用户传入标准 YAML 列表。
- 保持 `service.type`、`service.port`、selector、port name 等现有逻辑不变。

## 受影响文件

- 修改：`/Users/yaozaiyong/porter/helm-charts/charts/porter/values.yaml`
  - 受影响键：`.Values.service`
  - 新增键：`.Values.service.externalIPs`
  - 当前相关位置：`service.type` 与 `service.port` 位于文件第 187-189 行。

- 修改：`/Users/yaozaiyong/porter/helm-charts/charts/porter/templates/service.yaml`
  - 受影响模板：Porter 主 Kubernetes `Service`
  - 当前相关位置：`spec.type`、`ports`、`selector` 位于文件第 7-15 行。
  - 新增渲染字段：`spec.externalIPs`

- 不修改：`/Users/yaozaiyong/porter/helm-charts/charts/porter/templates/ingress.yaml`
  - 该模板只读取 `.Values.service.port` 作为 Ingress backend service 端口，`externalIPs` 不影响 Ingress 渲染。

- 不修改：`/Users/yaozaiyong/porter/helm-charts/charts/porter/templates/tests/test-connection.yaml`
  - Helm test 仍通过集群内 Service DNS 和 `.Values.service.port` 访问服务，不需要依赖外部 IP。

- 不修改：`/Users/yaozaiyong/porter/helm-charts/charts/porter/templates/NOTES.txt`
  - 当前 NOTES 根据 ingress、NodePort、LoadBalancer、ClusterIP 输出访问方式。`externalIPs` 是 Service spec 的可选暴露字段，不改变现有 service type 分支；本次保持说明输出不变，避免引入未请求的访问文案变更。

## 实现细节

`values.yaml` 中计划将当前：

```yaml
service:
  type: ClusterIP
  port: 80
```

调整为：

```yaml
service:
  type: ClusterIP
  port: 80
  externalIPs: []
```

`service.yaml` 中计划将当前：

```yaml
spec:
  type: {{ .Values.service.type }}
  ports:
```

调整为：

```yaml
spec:
  type: {{ .Values.service.type }}
  {{- with .Values.service.externalIPs }}
  externalIPs:
    {{- toYaml . | nindent 4 }}
  {{- end }}
  ports:
```

预期渲染行为：

- `service.externalIPs: []` 或未设置时：不输出 `externalIPs`。
- `service.externalIPs: ["192.168.1.10", "192.168.1.11"]` 时：输出两个 external IP 条目。

## 边界条件与异常处理

- 空列表：`with` 判断为空，不渲染字段。
- 未设置键：`with` 判断为空，不渲染字段。
- 非列表类型：模板会按 `toYaml` 输出，但 Kubernetes Service API 可能拒绝无效类型；本次不在模板中额外做 `kindIs` 校验，保持 chart 简单。
- IP 地址格式非法：Helm 模板不校验，交由 Kubernetes API 校验。
- 与 `service.type` 的组合：Kubernetes 允许 Service 配置 `externalIPs`，本次不限制 `ClusterIP`、`NodePort` 或 `LoadBalancer` 的组合。

## 数据流路径

1. 用户在 `values.yaml`、自定义 values 文件或 `helm install --set` 中设置 `service.externalIPs`。
2. Helm 渲染 `charts/porter/templates/service.yaml`。
3. 模板读取 `.Values.service.externalIPs`。
4. 当该值非空时，渲染到 Kubernetes Service 的 `spec.externalIPs`。
5. Kubernetes API 接收 Service manifest 并按自身规则校验、保存。

## 预期结果

- 默认 chart 渲染结果与当前 Service manifest 保持一致。
- 用户配置 `service.externalIPs` 后，Service manifest 包含 `spec.externalIPs`。
- 现有 Ingress、Helm test、Deployment、selector、Service port 行为不变。
- 可通过 `helm template` 验证默认场景和带 external IP 场景的渲染结果。
