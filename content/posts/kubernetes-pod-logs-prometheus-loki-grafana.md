---
title: "Kubernetes Pod Log'larını Prometheus / Loki / Grafana Ortamına Göndermek"
date: 2025-12-25T20:00:00+03:00
tags: [kubernetes, logging, loki, grafana, prometheus]
draft: false
---

Bu yazıda Kubernetes'de çalışan pod'ların log'larının nasıl toplanıp, Loki'ye gönderilip Grafana ile incelendiğini ve Prometheus ile metrik entegrasyonunu adım adım gösteriyorum. Hem mimariyi hem de gerekli komut/YAML parçalarını içerir.

![Mimari](/images/architecture.svg)

Özet akış:

- Pod stdout/stderr → Kubernetes node üzerindeki log dosyaları
- Log toplayıcı (Promtail / Fluentd / Filebeat) → Loki
- Loki → log arama, Grafana ile görselleştirme
- Prometheus → uygulama ve Promtail metriklerini toplar, Grafana metrik + log görselleştirmesi yapar

Neden bu yapı?
- Loki: etiket bazlı, düşük maliyetli log depolama, Grafana ile doğal entegrasyon.
- Promtail: Kubernetes'e özgü etiketleme (pod, namespace, labels) ve Prometheus uyumlu metrikler sağlar.

Gereksinimler
- Kubernetes (1.20+ önerilir)
- Helm (kolay kurulum için)
- Grafana erişimi (ör. Grafana Cloud ya da lokal)

Kurulum adımları

1) Loki ve Grafana (Helm ile)

```bash
# Loki + Grafana (grafana ve loki chart'larını kullanabilirsiniz)
# örnek: Grafana ve Loki'yi Loki stack ile kurmak
# grafana helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Loki stack (grafana/loki-stack veya grafana/loki)
helm install loki grafana/loki-stack --namespace observability --create-namespace
```

2) Promtail (node/pod loglarını toplayan ajan)

Basit bir Promtail DaemonSet örneği (kısaltılmış, production için Promtail konfigurasyonu detaylandırılmalı):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: observability
data:
  promtail.yaml: |
    server:
      http_listen_port: 9080
    clients:
      - url: http://loki:3100/loki/api/v1/push
    positions:
      filename: /tmp/positions.yaml
    scrape_configs:
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod
          - source_labels: [__meta_kubernetes_pod_label_app]
            target_label: app

# DaemonSet (özet)
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
  namespace: observability
spec:
  selector:
    matchLabels:
      name: promtail
  template:
    metadata:
      labels:
        name: promtail
    spec:
      serviceAccountName: promtail
      containers:
        - name: promtail
          image: grafana/promtail:latest
          args:
            - -config.file=/etc/promtail/promtail.yaml
          volumeMounts:
            - name: config
              mountPath: /etc/promtail
            - name: varlog
              mountPath: /var/log
            - name: varlibdocker
              mountPath: /var/lib/docker/containers
      volumes:
        - name: config
          configMap:
            name: promtail-config
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdocker
          hostPath:
            path: /var/lib/docker/containers
```

3) Prometheus entegrasyonu

Promtail, kendi metriklerini `http_listen_port` üzerinden açar (ör. :9080). Prometheus server'ınıza bu endpoint'i scrape etmesi için ekleyin:

```yaml
- job_name: 'promtail'
  static_configs:
    - targets: ['<PROMTAIL_NODE_IP>:9080']
```

Eğer Prometheus Operator kullanıyorsanız `ServiceMonitor` tanımı ile Promtail servislerini izletebilirsiniz.

4) Grafana'da Loki datasource ekleme

Grafana UI → Configuration → Data sources → Add Loki
URL: `http://<loki-service>:3100`

5) Log sorgulama ve paneller

Grafana Explore kısmında Loki'yi seçip sorgularla log araması yapabilirsiniz:

örnek sorgu:

```
{namespace="default", app="myapp"} |= "error"
```

6) Örnek dashboard: metrics + logs

Grafana panellerinde bir panel'i metrik (Prometheus) veya log (Loki) sorgusuyla bağlayabilirsiniz. Grafana 8+ sürümlerde "Logs" ve "Metrics" kombinasyonlarını aynı dashboard üzerinde kullanmak kolaydır.

![Akış](/images/flow.svg)

İpuçları ve production notları
- Retention: Loki için uygun retention ve indexten saklama ayarları yapın (bol disk/logrotate gerekebilir).
- Güvenlik: Loki/Grafana endpointlerini uygun şekilde doğrulayın, TLS ve auth kullanın.
- Etiketleme: Promtail ile doğru label'ları (namespace, pod, app) ekleyin; sorgular bu etiketlerle çok daha hızlı çalışır.

Kaynaklar
- Loki: https://grafana.com/oss/loki
- Promtail: https://grafana.com/oss/promtail
- Grafana Docs: https://grafana.com/docs

---

Bu gönderiyi projenize ekledim; isterseniz YAML'leri Helm chart'larına dönüştürebilir, Promtail için daha gelişmiş relabeling örnekleri veya Fluentd/Fluent Bit alternatifleri ekleyebilirim.
