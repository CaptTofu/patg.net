Here is a nice one-liner to get the password and username set in a kubernetes secret for the Grafana UI after installing Prometheus:

`kubectl get secret patg-prometheus-stack-grafana -o json|jq '.data."admin-password"'|sed -e 's/"//g'|base64 -D`
