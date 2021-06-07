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
        monitoring/prometheus file contains configurations for prometheus instance that will collect metrics from external prometheus and alertmanager ( in cluster #1) using federation feature

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

        monitoring/alertmanager.yml file contains configurations for alertmanager. To enable e-mail notification you need to modify this value and replace values with real data for smtp configuration. Or you can leave as it is and test only monitoring part.

    6. Create custom configmaps. Even if alermanagert config files wasn’t modified create configmap from this file too:

            kubectl create configmap crunchy-prometheus -n pgo --from-file=kube-configmaps/prometheus.yml
                
            kubectl create configmap alertmanager-config -n pgo --from-file=kube-configmaps/alertmanager.yml
    
    7. Create monitoring stack in cluster #2

        Create namespace if not created:
            
            kubectl create namespace pgo
        
        Create stack:

            kubectl apply -f kube-infrastructure/crunchy-operator-monitoring-external.yml

    That’s all. You can access prometheus or grafana dashboard setting port-forwarding for this services or changing from `ClusterIP` to `LoadBalancer` value in `grafana_service_type` or `prometheus_service_type` in monitoring/crunchy-operator-monitoring-external.yml 




    




            
    


