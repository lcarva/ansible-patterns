# Checksum Based k8s Deployment

When using the [k8s](https://docs.ansible.com/ansible/latest/modules/k8s_module.html) ansible module to manage a DeploymentConfig object in an OpenShift project, it's desireable to automatically trigger a new deployment whenever there are changes to the DeploymentConfig object itself, as well as any change to any Secret and/or ConfigMap objects referenced by the DeploymentConfig.

OpenShift has the concept of DeploymentConfig triggers which can be used to start automatic deployments when the DeploymentConfig object is updated. However, there's no corresponding mechanism for triggereing changes when relevant Secrets and/or ConfigMaps are updated. The pattern described in this document aims at addressing this limitation.

Although ansible provides a mechanism for triggering handlers based on task state, it has its own limitations in case of failures. If a task notifies a handler, but a subsequent task fails, the handler is not executed. Re-running the task will not cause changes, thus handler won't be notified. This creates an unstable state.

To address the limitations outlined above, use environment variables in the DeploymentConfig to hold the checksum of content provided by Secret and ConfigMap objects, and rely on OpenShift to determine when a new deployment is needed. For example:

```yaml
- name: Manage a ConfigMap
  k8s:
    state: present
    definition:
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: my-config-map
        namespace: my-project
      data:
        spam.conf: "{{ lookup('template', 'templates/spam.conf') }}"
        ham.conf: "{{ lookup('template', 'templates/ham.conf') }}"

- name: Manage a Secret
  k8s:
    state: present
    definition:
      apiVersion: v1
      kind: Secret
      type: Opaque
      metadata:
        name: my-secret
        namespace: my-project
      data:
        cert: "{{ lookup('file', 'cert', rstrip=False) | b64encode }}"
        key: "{{ lookup('file', 'key', rstrip=False) | b64encode }}"

- name: Manage a DeploymentConfig
  k8s:
    state: present
    definition:
      apiVersion: v1
      kind: DeploymentConfig
      metadata:
        name: my-deployment-config
        namespace: my-project
      spec:
        template:
          spec:
            containers:
            - name: my-application

              volumes:
              - name: my-secret-volume
                secret:
                  secretName: my-secret
                  items:
                  - key: cert
                    path: /opt/secrets/certs/cert
                  - key: key
                    path: /opt/secrets/certs/key
              - name: my-config-map-volume
                configMap:
                  name: my-config-map

              volumeMounts:
              - name: my-secret-volume
                  mountPath: /opt/secrets/certs
                  readOnly: true
              - name: my-config-map-volume
                  mountPath: /opt/configs
                  readOnly: true

              env:
                - name: SPAM_CONF_CHECKSUM
                  value: "{{ lookup('template', 'templates/spam.conf') | checksum }}"
                - name: HAM_CONF_CHECKSUM
                  value: "{{ lookup('template', 'templates/ham.conf') | checksum }}"
                - name: CERT_CHECKSUM
                  value: "{{ lookup('file', 'cert', rstrip=False) | checksum }}"
                - name: KEY_CHECKSUM
                  value: "{{ lookup('file', 'key', rstrip=False) | checksum }}"

        triggers:
        - type: ConfigChange
```

_NOTE: The example above is not a complete definition of a DeploymentConfig object. Sections irrelevant to the example have been removed for the sake of brevity._

The `env` section specifies four environment variables. One for each file stored in either a Secret or ConfigMap object. When the contents of `templates/spam.conf` change, for example, the ConfigMap object will be updated. This will also cause the value of `SPAM_CONF_CHECKSUM` to be updated. Since the DeploymentConfig has the ConfigChange trigger set, OpenShift will automatically trigger a new deployment.

## Requirements

* Items stored in each relevant Secret or ConfigMap must not be literals. The value has to be provided either from a file/template, or a variable.
* For each of the items stored in a Secret or ConfigMap, a corresponding `*_CHECKSUM` environment varialbe must be set in the DeploymentConfig object. Its value should be the checksum of the corresponding item.
* The DeploymentConfig object must have the trigger ConfigChange set.
* The task managing the DeploymentConfig must be executed after the tasks managing the relevant Secret and ConfigMap objects.

_NOTE: Not all operations performed in reading the file are required when setting the checksum vlaue. For example, the "key" file is encoded in b64 when it's included in a Secret object. However, this is not required when computing the checksum of its contents. On the other hand, the `rstrip=False` modifier is recommended to be used since that changes the contents of the file._

## Limitations

The main limitation of this pattern is that it requires maintainers to keep the list of `*_CHECKSUM` definitions up to date with all other relevant Secret and ConfigMap objects.