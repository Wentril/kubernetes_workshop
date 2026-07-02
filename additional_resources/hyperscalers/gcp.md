# GKE  
## Google native access  
**gcloud** cli - [Google Cloud CLI docs](https://docs.cloud.google.com/sdk/docs/install-sdk)  
**gcloud components install gke-gcloud-auth-plugin** - auth plugin to provide middleware in between kubectl and GKE  
*`gcloud auth login` - optional, authenticates the cli with google cloud - for general interactions via cli  
`gcloud auth application-default login` - acquires user credentials for Application Default Credentials  
  
`gcloud container clusters get-credentials <cluster-name> --region <region> --project <project-id>` - provisions kubectl config for specified cluster and switches to it's context  

## Storage Classes
```yaml title="storage_balanced.yaml"
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: balanced
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: pd.csi.storage.gke.io
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: pd-balanced
  disk-encryption-kms-key: <cmek-location> # projects/<cmek-project-id>/locations/europe-west3/keyRings/<cmek-keyring-name>/cryptoKeys/<cmek-key-name>
```

## IP Masqarade
```yaml title="ip_masq_agent.yaml"
apiVersion: v1
kind: ConfigMap
metadata:
  name: ip-masq-agent
  namespace: kube-system
data:
  config: |
    nonMasqueradeCIDRs:
    # Default non-masquarade destinations
    #  - 10.0.0.0/8
    #  - 172.16.0.0/12
    #  - 192.168.0.0/16
    #  - 100.64.0.0/10
      - 100.96.0.0/12 # Contains Pods and Services IP ranges that should never be masquaraded. It also contains master nodes and clusterIPs
      # https://cloud.google.com/kubernetes-engine/docs/how-to/ip-masquerade-agent#edit-ip-masq-agent-configmap
    # Subnets used on the SharedVPC IPs in GCP shall be placed here as well!!!!!
    masqLinkLocal: false
    resyncInterval: 60s
```
## "Default" namespace pod security standards
### Default
```yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    kubernetes.io/metadata.name: default
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/enforce: baseline
  name: default
```
### Cos-auditd
```yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    kubernetes.io/metadata.name: cos-auditd
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/enforce: privileged
  name: cos-auditd
```

## Cos-auditd
```yaml title="logging-configmap.yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: cos-auditd-fluent-bit-config
  namespace: cos-auditd
  annotations:
    kubernetes.io/description: 'ConfigMap for Linux auditd logging daemonset on COS nodes.'
data:
  fluent-bit.conf: |-
    [SERVICE]
        Flush         5
        Grace         120
        Log_Level     info
        Daemon        off
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_PORT     2024

    [INPUT]
        # https://docs.fluentbit.io/manual/input/systemd
        Name            systemd
        Alias           audit
        Tag             audit
        Systemd_Filter  SYSLOG_IDENTIFIER=audit
        Path            /var/log/journal
        DB              /var/lib/cos-auditd-fluent-bit/pos-files/audit.db
        Buffer_Max_Size 20MB
        Mem_Buf_Limit   20MB

    [FILTER]
        # https://docs.fluentbit.io/manual/pipeline/filters/modify
        Name                modify
        Match               audit
        Add                 logging.googleapis.com/local_resource_id k8s_node.${NODE_NAME}

    [FILTER]
        Name           modify
        Match          audit
        Add            logging.googleapis.com/logName linux-auditd

    [OUTPUT]
        # https://docs.fluentbit.io/manual/pipeline/outputs/stackdriver
        Name                      stackdriver
        Match                     audit
        Severity_key              severity
        log_name_key              logging.googleapis.com/logName
        Resource                  k8s_node
        # The plugin will read the project ID from the metadata server, but not the cluster name and location for some reason, so they have to be injected.
        k8s_cluster_name          <cluster-name>
        k8s_cluster_location      <cluster-location>
        net.connect_timeout       60
        Retry_Limit               14
        Workers                   2
```
```yaml title="logging-daemonset.yaml"
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cos-auditd-logging
  namespace: cos-auditd
  annotations:
    kubernetes.io/description: 'DaemonSet that enables Linux auditd logging on non-Autopilot COS nodes.'
spec:
  selector:
    matchLabels:
      name: cos-auditd-logging
  template:
    metadata:
      labels:
        name: cos-auditd-logging
    spec:
      # Necessary for ensuring access to Google Cloud credentials from the node's metadata server.
      hostNetwork: true
      hostPID: true
      dnsPolicy: Default
      initContainers:
      - name: cos-auditd-setup
        image: ubuntu
        command: ["chroot", "/host", "systemctl", "start", "cloud-audit-setup"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: host
          mountPath: /host
        resources:
          requests:
            memory: "10Mi"
            cpu: "10m"
      containers:
      - name: cos-auditd-fluent-bit
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - all
            add:
            - DAC_OVERRIDE
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        # SUBSTITUTE
        - name: CLUSTER_NAME
          value: "<cluster-name>"
        - name: CLUSTER_LOCATION
          value: <cluster-location>"
        image: gke.gcr.io/fluent-bit@sha256:c03635e4c828b9c6847df9780d6684b45ff0a70b1ae8c7e7271283cce472085e # v1.8.12-gke.19
        imagePullPolicy: IfNotPresent
        livenessProbe:
          httpGet:
            path: /
            port: 2024
          initialDelaySeconds: 120
          periodSeconds: 60
          timeoutSeconds: 5
        ports:
        - name: metrics
          containerPort: 2024
        resources:
          limits:
            cpu: "1"
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/log
          name: varlog
        - mountPath: /var/lib/cos-auditd-fluent-bit/pos-files
          name: varlib-cos-auditd-fluent-bit-pos-files
        - mountPath: /fluent-bit/etc
          name: config-volume
      nodeSelector:
        cloud.google.com/gke-os-distribution: cos
      restartPolicy: Always
      terminationGracePeriodSeconds: 120
      tolerations:
      - operator: "Exists"
        effect: "NoExecute"
      - operator: "Exists"
        effect: "NoSchedule"
      volumes:
      - name: host
        hostPath:
          path: /
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibcos-auditd-fluent-bit
        hostPath:
          path: /var/lib/cos-auditd-fluent-bit
          type: DirectoryOrCreate
      - name: varlib-cos-auditd-fluent-bit-pos-files
        hostPath:
          path: /var/lib/cos-auditd-fluent-bit/pos-files
          type: DirectoryOrCreate
      - name: config-volume
        configMap:
          name: cos-auditd-fluent-bit-config
  updateStrategy:
    type: RollingUpdate
```
