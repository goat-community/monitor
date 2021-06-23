* Monitoring.
    To configure monitoring in "send metrics from cluster #1 > gather metrics in cluster #2" flow, prometheus federation feature was used. 
    To create monitoring infrasctructure follow the next steps:
    1. Metrics exporter for postgres cluster can be enabled using `--metrics` flag when cluster creating:
    
        pgo create cluster hippo --metrics

    There will be created sidecar container with meterics exporter

    2. Deploy crunchy metrics deployer:

            kubectl apply -f kube-infrastructure/crunchy-operator-monitoring-internal.yml
    
    It will create pod deployer and monitoring stack in pgo namespace( prometheus, alertmanager, grafana) will be created. For prometheus and alertmanager will be created LoadBalancer service type, so we will have external Ip that can be used to configuring monitoring in external cluster.

    3. Get alertmanager and prometheus external IP:
        
            kubectl get svc -n pgo

    ![alt text](http://img.empeek.net/1K4PZOC.png)

    4. Connect to to cluster #2

    5. Modify configurations file for configmaps:

        5.1
        kube-configmaps/prometheus.yml file contains configurations for prometheus instance that will collect metrics from external prometheus and alertmanager ( in cluster #1) using federation feature

        <ip> should be replaced with real external IP from step #3.
            ---
            global:
            scrape_interval: 15s
            scrape_timeout: 15s
            evaluation_interval: 5s

            scrape_configs:
            - job_name: 'external-cluster'
            scrape_interval: 30s
            honor_labels: true
            metrics_path: '/federate'
            params:
                'match[]':
                - '{job="crunchy-postgres-exporter"}'
            static_configs:
                - targets:
                - '<ip>:9090'

            rule_files:
            - /etc/prometheus/alert-rules.d/*.yml
            alerting:
            alertmanagers:
            - scheme: http
                static_configs:
                - targets:
                - "<ip>:9093"

        kubec-configmaps/alertmanager.yml file contains configurations for alertmanager. To enable e-mail notification you need to modify this value and replace values with real data for smtp configuration. Or you can leave as it is and test only monitoring part.

    6. Create custom configmaps for prometheus and alertmanager. Even if alermanagert config files wasn’t modified create configmap from this file too:

            kubectl create configmap crunchy-prometheus -n pgo --from-file=kube-configmaps/prometheus.yml
                
            kubectl create configmap crunchy-alertmanager -n pgo --from-file=kube-configmaps/alertmanager.yml

    6. Create custom configmaps for Grafana. They will contain Loki datasource and custom dashboard configurations to enable Loki's logs visualization:

            kubectl apply -f kube-configmaps/crunchy_grafana_dashboards.yml

            kubectl create configmap crunchy-grafana-datasources -n pgo --from-file=kube-configmaps/crunchy_grafana_datasources.yml
    
    7. Create monitoring stack in cluster #2

        Create namespace if not created:
            
            kubectl create namespace pgo
        
        Create stack:

            kubectl apply -f kube-infrastructure/crunchy-operator-monitoring-external.yml

    That’s all. You can access prometheus or grafana dashboard setting port-forwarding for this services or changing from `ClusterIP` to `LoadBalancer` value in `grafana_service_type` or `prometheus_service_type` in kube-infrastructure/crunchy-operator-monitoring-external.yml 

* Logging
    To collect logs from pods in entire cluster was used Grafana Loki. It allows us to collect logs from each cluster inside of entire k8 cluster. Then Loki can be used as datasource in Grafana or any other supported visualizing tool.
    To install Loki use the following command:

        helm upgrade --install loki --namespace=pgo grafana/loki-stack

    It will deploy Loki stack ( Loki itself and promtail) inside of pgo namespace.
    To access Loki from external cluster and to secure logs there is loki/loki-ingress.yaml file. You can use it to create ingress and establish basic auth.
    First of all you need to create secret with credentials for basic auth:

        htpasswd -c auth <username>
    Then create secret from this file:

        kubectl create secret generic basic-auth -n pgo --from-file=auth 

    loki/loki-ingress.yaml already contains configurations for basic auth, so just create it:

        kubectl apply -f loki/loki-ingress.yaml

    Remember that Loki must be deployed inside of cluster from which you want to collect logs.

* SSL configuration

    Apply letscrypt.yaml file from goat repository into monitoring cluster.

    To enable SSL for grafana ingress first of all wee need to create nginx-ingress configuration:

        kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.47.0/deploy/static/provider/do/deploy.yaml

    Wait for installation.

    DO managed k8 cluster have strange issue when pods inside of cluster cannot communicate with each other through Ingress service, and this cause problem with creating and verifying ssl certificates so we need to modify this service and add annotation:

        kubectl edit svc -n ingress-nginx ingress-nginx-controller

    Then add this annotation:

        service.beta.kubernetes.io/do-loadbalancer-hostname: "workaround.example.com"

    Where `workaround.example.com` is domain name that assigned to nginx-ingress loadbalancer public IP.

    After this you can create ingress for grafana service with https enabled:

        kubectl apply -f kube-infrastructure/grafana-ingress.yaml






    




            
    


