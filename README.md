# New Relic Integration with Prometheus and Grafana (Docker)
This tutorial goes over the steps to run Prometheus and Grafana locally.  The objective is to:
1. Send data from Prometheus to New Relic
2. Send Prometheus data in New Relic to Grafana for dashboards

This tutorial assumes you have the following installed with environment variables setup:
- `Docker Desktop`
- `kubectl`
- `helm`

## Send Prometheus data to New Relic

1. Edit `prometheus.yml` and paste in the `remote_write` `url` and `bearer_token` from New Relic.

2. Set environment variable for the path containing `.yml` files and run Prometheus locally.
    ```powershell
    $env:repo=$pwd.path
    docker run -d -p 9090:9090 -v $env:repo\prometheus.yml:/etc/prometheus/prometheus.yml --name prometheus-docker prom/prometheus
    ```

3. Check http://127.0.0.1:9090 to make sure Prometheus is running. Type something in the search bar and click `Execute` to see some data.  Now go to New Relic and check for Prometheus data with this NRQL query:
    ```sql
    SELECT cardinality(metricName) FROM Metric WHERE instrumentation.name = 'remote-write' FACET metricName LIMIT MAX
    ```
## Send New Relic Prometheus data to Grafana 

4. Run Grafana
    ```PowerShell
    docker run -d -p 3000:3000 --name grafana-docker grafana/grafana-oss
    ```
5. Go to http://127.0.0.1:3000 and login with default username and password.  It will ask you set a new password once you log in.
    ```
    username: admin
    password admin
    ```
6. In Grafana, go to the left-side menu and click on Configuration > Data Sources and click on `Add data source`

7. Select Prometheus and use the following settings:

    | Field               | Setting                             |
    |---------------------|-------------------------------------|
    | Name                | Prometheus-NRDB                     |
    | URL                 | https://prometheus-api.newrelic.com |
    | Custom HTTP: Header | `API-Key`                           |
    | Custom HTTP: Value  | `NRAK-XXXXXXXXXXXXXXXXXXXXXXXXXXX`  |

8. The `API-Key` here is the User API Key from New Relic.  Click ` Save & Test` to make sure it's working.  It should show `Data source is working`.

9. Scroll up to the top and click on the `Dashboards` tab and import as needed.

# Kubernetes Implementation with Helm

1. Using Docker Desktop, go to `Settings > Kubernetes > Enable Kubernetes > Apply & Restart`.  This will create a single node kubernetes cluster on your local machine without needing `minikube`.

2. Check to make sure your kubernetes cluster is working
    ```
    kubectl get nodes
    ```
3. Once your node is ready, run the Kubernetes dashboard and create an admin user to access the dashboards:
    ```
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended.yaml
    kubectl apply -f dashboard-adminuser.yaml
    ```
4. Open a new new terminal and create a proxy to access the Kubenetes dashbaord
    ```
    kubectl proxy
    ```

5. To access the Kubenetes dashboard, go to:  
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

6. To access the Kubernetes dashboard, you'll need an access token.  Run this command and copy the token to sign in:
    ```
    kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"
    ```

7. Once you sign into the Kubenetes Dashboard, use the drop-down menu to switch from `default` to `All namespaces`.

8. Add the following helm charts and update
    ```
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo update
    ```
9. Before we install the Prometheus helm chart, grab the `values.yaml` from here:  
https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/prometheus/values.yaml

10. We'll need to make a small modification to `values.yml`.  Look for `remoteWrite:` and paste in the `url` and `bearer_token` from New Relic.  Note, for this file, it should be `remoteWrite` instead of `remote_write`.

11. Install the helm chart with the modified `values.yaml` file.  Check your Kubernetes Dashboard to see the pods spin up.
    ```
    helm install -f values.yaml prometheus-community/prometheus --generate-name
    ```

12. Open a new terminal and run the following commands, then check http://127.0.0.1:9090 and New Relic for data
    ```
    $env:POD_NAME=kubectl get pods --namespace default -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}"
    kubectl --namespace default port-forward $env:POD_NAME 9090
    ```
13. Next, install the helm chart for Grafana. Also check your Kubernetes Dashboard to see pods spinning up.
    ```
    helm install grafana/grafana --generate-name
    ```
14. Open a new terminal and run the following commands, then check http://127.0.0.1:3000
    ```
    $env:POD_NAME=kubectl get pods --namespace default -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana-1646412168" -o jsonpath="{.items[0].metadata.name}"
    kubectl --namespace default port-forward $env:POD_NAME 3000
    ```

15. The default username for Grafana is `admin`.  To get the password, run this command:
```powershell
$encoded_password=kubectl get secret --namespace default grafana-1646412168 -o jsonpath="{.data.admin-password}"
[Text.Encoding]::Utf8.GetString([Convert]::FromBase64String($encoded_password))
```

16. In Grafana, go to the left-side menu and click on Configuration > Data Sources and click on `Add data source`

17. Select Prometheus and use the following settings:

    | Field               | Setting                             |
    |---------------------|-------------------------------------|
    | Name                | Prometheus-NRDB                     |
    | URL                 | https://prometheus-api.newrelic.com |
    | Custom HTTP: Header | `API-Key`                           |
    | Custom HTTP: Value  | `NRAK-XXXXXXXXXXXXXXXXXXXXXXXXXXX`  |

18. The `API-Key` here is the User API Key from New Relic.  Click ` Save & Test` to make sure it's working.  It should show `Data source is working`.

19. Scroll up to the top and click on the `Dashboards` tab and import as needed.

20. Note, with this Kubenetes deployment, there is more data in New Relic and Grafana this time due to the other Kubernetes pods running.
    ```sql
    SELECT max(scrape_duration_seconds) FROM Metric WHERE instrumentation.name = 'remote-write' TIMESERIES AUTO FACET job,instance
    ```

21. To stop Grafana and Prometheus from running on your local Kubernetes cluster, stop the port forwarding from the other open terminals, then use `helm list` to get the name of the deployed helm chart. In this case, we have
    ```
    NAME
    grafana-1646412168
    prometheus-1646411938
    ```

22. To uninstall, use the following (be sure to replace the `NAME` with your own)
    ```
    helm uninstall grafana-1646412168
    helm uninstall prometheus-1646411938
    ```

23. To uninstall the Kubernetes Dashboard, stop the `kubectl proxy` from running in your terminal. Then:
    ```
    kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended.yaml
    ```

24. Shut down your kubernetes cluster by going to `Docker Desktop > Settings > Kubernetes > Enable Kubernetes > Apply & Restart`.