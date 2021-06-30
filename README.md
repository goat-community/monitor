* Monitoring.
    To create monitoring infrasctructure follow the next steps:
    1. Metrics exporter for postgres cluster can be enabled using `--metrics` flag when cluster creating:
    
        pgo create cluster hippo --metrics

    There will be created sidecar container with meterics exporter

    2. Run the follomind command:

            kubectl create namespace monitoring
            helm install prometheus prometheus-community/prometheus --namespace monitoring --values prometheus/values.yaml 

    3. Helm will install prometheus and alertmanger stack. prometheus/values.yaml file contains configurations for monitoring client/api/databases pods. Configuration for db monitoring is described in `extraScrapeConfigs` section and can be modified in future to enable or disable some features, also this file contains other configurations that you can modify to your needs.
    
    3. Create custom configmaps for Grafana. They will contain Loki datasource and custom dashboard configurations to enable Loki's logs visualization:

            kubectl apply -f grafana/crunchy_grafana_dashboards.yml

            kubectl create configmap crunchy-grafana-datasources -n pgo --from-file=grafana/crunchy_grafana_datasources.yml


    4. Create monitoring stack in cluster #2

        Create namespace if not created:
            
            kubectl create namespace pgo
        
        Create stack:

            kubectl apply -f kube-infrastructure/crunchy-operator-monitoring-external.yml


* Logging
    To collect logs from pods in entire cluster Grafana Loki was used. It allows us to collect logs from each cluster inside of entire k8 cluster. Then Loki can be used as datasource in Grafana or any other supported visualizing tool.
    To install Loki use the following command:

        helm upgrade --install loki --namespace=pgo grafana/loki-stack

    It will deploy Loki stack ( Loki itself and promtail) inside of pgo namespace.
    To access Loki from external cluster and to secure logs there is loki/loki-ingress.yaml file. You can use it to create ingress and establish basic auth.
    First of all you need to create secret with credentials for basic auth:

        htpasswd -c auth <username>
    Then create secret from this file:

        kubectl create secret generic basic-auth -n pgo --from-file=auth 

    Ingress creating described in the last section
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

    After this you can create ingress for services with https enabled:

        kubectl apply -f prometheus/prometheus-ingress.yaml
        kubectl apply -f grafana/grafana-ingress.yaml
        kubectl apply -f loki/loki-ingress.yaml






    




            
    


