# Php-scan 

The objective of this work is discovering what type of errors do not allow to execute the installation of the application php using helm.

# Solution

## Step #1: Identifying problems

Once we execute the ``helm install`` command, we discover the following problem.

<p align="center">
  <img width="1015" height="116" alt="image" src="https://github.com/user-attachments/assets/224eff9b-c37f-40f1-8977-a1aebaacba85" />
</p>

Next, we present that the problem refer from deployment.yaml. To explain this, we refer back to the previous code.

````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    annotations:
      {{ toYaml .Values.deployment.annotations }}
    name: {{ include "phpsecscan.fullname" . }}
    labels:
        {{- include "phpsecscan.labels" . | nindent 8 }}
spec:
    replicas: {{ .Values.replicaCount }}
    selector:
        matchLabels:
          {{- include "phpsecscan.labels" . | nindent 10 }}
    template:
        metadata:
            labels:
                app: {{ include "phpsecscan.fullname" . }}
        spec:
            containers:
                - name: {{ include "phpsecscan.fullname" . }}
                  image: {{ .Values.container.image }}
                  args: ["-port", "{{ .Values.container.port }}"]
                  imagePullPolicy: {{ .Values.container.imagePullPolicy }}
                  resources:
{{ toYaml .Values.resources | indent 20 }}
                  livenessProbe:
                      httpGet:
                          path: {{ .Values.liveness.path }}
                          port: {{ .Values.container.port }}
                      initialDelaySeconds: {{ .Values.liveness.initialDelay }}
                      periodSeconds: {{ .Values.liveness.period }}
                  ports:
                      - name: "http"
                        containerPort: {{ .Values.container.port }}
````

Among the problems this brings are the following.

- **Inconsistency in labels:** The selector.matchLabels used phpsecscan.labels (generating 5 labels) while template.metadata.labels only defined app: phpsecscan (1 label).
- **Unprotected annotations:** The annotations section did not handle cases where deployment.annotations was not defined in values.yaml, which could cause rendering errors.
- **Excessive indentation in resources:** Indent 20 was used instead of nindent 10, resulting in incorrect spacing and code that was difficult to maintain
- **Inconsistent indentation:** The file mixed 4-space indentation with the Kubernetes standard of 2 spaces.
- **Incorrect use of helpers:** phpsecscan.labels was used in the selector when phpsecscan.selectorLabels should have been used to maintain consistency.
- **Missing hyphens in directives:** Some template directives lacked {{- to control whitespace correctly.

All these problems do not allow to start the application correctly.

## Step #2: Proposing a solution.

To solve the problems present above, we are going to present the solution.

````yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "phpsecscan.fullname" . }}
  labels:
    {{- include "phpsecscan.selectorLabels" . | nindent 4 }}
  {{- with .Values.deployment.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "phpsecscan.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "phpsecscan.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ include "phpsecscan.fullname" . }}
        image: {{ .Values.container.image }}
        args: ["-port", "{{ .Values.container.port }}"]
        imagePullPolicy: {{ .Values.container.imagePullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.container.port }}
        livenessProbe:
          httpGet:
            path: {{ .Values.liveness.path }}
            port: {{ .Values.container.port }}
          initialDelaySeconds: {{ .Values.liveness.initialDelay }}
          periodSeconds: {{ .Values.liveness.period }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
````

In this case the proposed yaml has the following corrections:

* Consistent use of phpsecscan.selectorLabels in selector and template.
* Protection of annotations with {{- with }} directive.
* Standardization of indentation to 2 spaces (instead of 4).
* Correction of resource indentation to nindent 10.
* Appropriate use of {{- and | nindent in all directives.

All these corrections allows the correct instalation using helm.

## Step #3: Provin the application

Once the corrections are done, we execute ``helm install phpsecscan`` in the root folder and we will have the following result.

<p align="center">
  <img width="672" height="684" alt="image" src="https://github.com/user-attachments/assets/4b95d7cb-d25f-4b16-ad21-5138c2b7c5b6" />
</p>

