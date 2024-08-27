# mlops-e2e

### Project Overview

This project is the final project of [mlops-zoomcamp](https://github.com/DataTalksClub/mlops-zoomcamp/tree/main).

The **MLOps E2E** project is designed to demonstrate a comprehensive end-to-end machine learning operations (MLOps) pipeline, focusing on the deployment, monitoring, and automation of machine learning models. The project incorporates various tools and frameworks, including Kubernetes, Kubeflow, and Helm, to manage and orchestrate machine learning workflows.

#### Air Quality Pipeline

One of the primary pipelines included in this project is the **Air Quality Pipeline** (`pipeline/dags/air_quality.py`). This pipeline is responsible for ingesting air quality data, performing preprocessing, and subsequently training and evaluating machine learning models to predict air quality levels. The pipeline is orchestrated using Apache Airflow, which ensures that each step of the workflow is executed in the correct sequence, with proper logging and monitoring.

The air quality data is processed to handle missing values, normalize features, and engineer relevant features before feeding it into a machine learning model. This model is then deployed and monitored for performance, with alerts and logging integrated into the workflow to ensure robust operation in a production environment.


### Prerequisites

Followings are Prerequisites to run the pipeline, most probably it should also work with latest versions 

- **Docker**: 24.0.5
  **minikube**: v1.33.1
  **kubectl**: v1.26.0 (use v1.30.0+ per warning from minikube)
  **helm**: v3.15.3

## Getting Started

### Step 1: Start Minikube

Initialize Minikube with the desired CPU and memory resources:

```bash
minikube start --cpus 12 --memory 6144
```

This command starts a Minikube cluster with 12 CPUs and 6144 MB of memory, providing enough resources to run your Kubernetes workloads.

### Step 2: Enable Minikube Tunnel

To expose your Kubernetes services to the host machine, run:

```bash
minikube tunnel
```

This command is essential for accessing services deployed in the cluster from your local machine.

### Step 3: Apply Airflow Requirements ConfigMap

Apply the ConfigMap for Airflow requirements:

```bash
kubectl apply -f helm/config_maps/airflow-requirements.yaml
```

This step configures the necessary Python dependencies for Airflow.

### Step 4: Install Prometheus

Install Prometheus using Helm to monitor your Kubernetes cluster:

```bash
helm install prometheus-release oci://registry-1.docker.io/bitnamicharts/kube-prometheus
```

Prometheus will be used to collect metrics from your cluster and applications.

### Step 5: Install Grafana

Install Grafana using Helm for visualizing the metrics collected by Prometheus:

```bash
helm install grafana-release oci://registry-1.docker.io/bitnamicharts/grafana
```

Grafana provides a powerful dashboarding and visualization capability for your metrics data.

### Step 6: Install Apache Airflow

Deploy Apache Airflow using Helm with the custom configuration:

```bash
helm install airflow-release oci://registry-1.docker.io/bitnamicharts/airflow -f helm/airflow_config.yaml
```

Airflow orchestrates your machine learning workflows, managing tasks and dependencies in your pipeline.

### Step 7: Install MLflow

Deploy MLflow for managing the machine learning lifecycle, including experimentation, reproducibility, and deployment:

```bash
helm install mlflow-release oci://registry-1.docker.io/bitnamicharts/mlflow -f helm/mlflow_config.yaml
```

MLflow tracks experiments, manages models, and facilitates deployment in a scalable environment.


## Accessing Airflow and Mlflow


To connect to Airflow from outside the cluster, perform the following steps:

1. Obtain the LoadBalancer IP:

   **NOTE:** It may take a few minutes for the LoadBalancer IP to be available.
   Watch the status with: 'kubectl get svc --namespace default -w airflow-release'

   ```bash
   export SERVICE_IP=$(kubectl get svc --namespace default airflow-release -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   echo "Airflow URL: http://${SERVICE_IP}:8080"
   ```

2. Open a browser and access Airflow using the obtained URL.

3. Get your Airflow login credentials by running:

   ```bash
   export AIRFLOW_PASSWORD=$(kubectl get secret --namespace "default" airflow-release -o jsonpath="{.data.airflow-password}" | base64 -d)
   echo User:     user
   echo Password: $AIRFLOW_PASSWORD
   ```

To access your MLflow site from outside the cluster, follow the steps below:

1. Get the MLflow URL by running these commands:

   **NOTE:** It may take a few minutes for the LoadBalancer IP to be available.
   Watch the status with: 'kubectl get svc --namespace default -w mlflow-release-tracking'

   ```bash
   export SERVICE_IP=$(kubectl get svc --namespace default mlflow-release-tracking -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   echo "MLflow URL: http://$SERVICE_IP/"
   ```

2. Open a browser and access MLflow using the obtained URL.

3. Login with the following credentials below to see your blog:

   ```bash
   echo Username: $(kubectl get secret --namespace default mlflow-release-tracking -o jsonpath="{ .data.admin-user }" | base64 -d)
   echo Password: $(kubectl get secret --namespace default mlflow-release-tracking -o jsonpath="{.data.admin-password }" | base64 -d)
   ```

## Adding a PostgreSQL Connection in Airflow

To add a connection in Airflow for PostgreSQL, follow these steps:

1. **Log in to the Airflow Web UI**:

   Access the Airflow web UI using the IP and port obtained earlier. 

2. **Navigate to the Airflow Connections**:

   Once logged in, go to the "Admin" tab on the top menu and select "Connections" from the dropdown.

3. **Create a New Connection**:

   - Click the **"+"** button to add a new connection.

4. **Enter the Connection Details**:

   In the "Create a new record" form, fill out the following fields:

   - **Conn Id**: `postgres_air_quality`
   - **Conn Type**: `Postgres`
   - **Databse**: `bitnami_airflow`
   - **Host**: `airflow-release-postgresql.default.svc.cluster.local`
   - **Schema**: *(leave this blank unless you have a specific schema to connect to)*
   - **Login**: `bn_airflow`
   - **Password**: 
     ```bash
     $(kubectl get secret --namespace "default" airflow-release-postgresql -o jsonpath="{.data.password}" | base64 -d)
     ```
   - **Port**: `5432`

5. **Save the Connection**:

   After filling in the details, click the **"Save"** button at the bottom of the form.

## Running the ETL and Training Pipeline in Airflow

After setting up the PostgreSQL connection in Airflow, you can run the `etl_air_quality_postgres_training_pipeline` DAG to perform the ETL process, train your machine learning model, and log the results in MLflow.

### Steps to Run the Pipeline

1. **Access the Airflow Web UI**:

   Log in to the Airflow web UI using the IP and port obtained earlier.

2. **Locate the DAG**:

   In the Airflow dashboard, find the DAG named `etl_air_quality_postgres_training_pipeline`.

3. **Trigger the DAG**:

   - Click the **"Play"** button next to the DAG name to manually trigger the pipeline.
   - Confirm the trigger in the dialog that appears.

4. **Monitor the DAG Execution**:

   - You can monitor the progress of the DAG by clicking on the DAG name and viewing the task instances.
   - Each task in the pipeline will run sequentially or in parallel, depending on the DAG configuration.

5. **Verify the Execution**:

   - Once the DAG has successfully completed, all tasks should show a **green** status indicating successful execution.

### Viewing the Results in MLflow

After the successful execution of the `etl_air_quality_postgres_training_pipeline` DAG, the results, including model metrics and artifacts, will be logged in MLflow under the experiment name `air_quality_experiment`.

1. **Access the MLflow Web UI**:

   - Use the MLflow URL obtained earlier to access the MLflow web UI.
   - Log in using the credentials retrieved from Kubernetes secrets.

2. **Navigate to the Experiment**:

   - In the MLflow UI, locate the experiment named `air_quality_experiment`.
   - Click on this experiment to view the runs associated with the `etl_air_quality_postgres_training_pipeline` DAG.

3. **Review the Results**:

   - You should be able to see the metrics, parameters, and artifacts logged during the pipeline execution.


## Accessing MLflow and Airflow Metrics Through Prometheus

Now that your MLflow and Airflow setups are integrated with Prometheus, you can monitor the metrics from both platforms using Prometheus.

### Steps to Access Prometheus

To access Prometheus from outside the Kubernetes cluster, follow these steps:

1. **Forward the Prometheus Port**:

   Run the following commands in your terminal to forward the Prometheus port to your local machine:

   ```bash
   echo "Prometheus URL: http://127.0.0.1:9090/"
   kubectl port-forward --namespace default svc/prometheus-release-kube-pr-prometheus 9090:9090
   ```

   This command will make Prometheus available on your local machine at `http://127.0.0.1:9090/`.

2. **Open Prometheus in Your Browser**:

   - Open a web browser and navigate to `http://127.0.0.1:9090/`.
   - You will be directed to the Prometheus web interface.

3. **Explore Metrics**:

   - In the Prometheus web UI, you can query and explore metrics collected from MLflow and Airflow.
   - Use the search bar at the top to run queries and visualize the data being monitored.

## Accessing Grafana and Setting Up Alerts

Grafana is a powerful monitoring and visualization tool that allows you to create custom dashboards and set up alerts for various metrics. Here's how you can access Grafana and set up an alert to monitor your Airflow instance.

### Steps to Access Grafana

1. **Get the Grafana Application URL**:

   Run the following commands in your terminal to forward the Grafana port to your local machine:

   ```bash
   echo "Browse to http://127.0.0.1:8080"
   kubectl port-forward svc/grafana-release 8080:3000 &
   ```

   This command will make Grafana available on your local machine at `http://127.0.0.1:8080`.

2. **Get the Admin Credentials**:

   Retrieve the admin credentials for Grafana by running the following commands:

   ```bash
   echo "User: admin"
   echo "Password: $(kubectl get secret grafana-release-admin --namespace default -o jsonpath="{.data.GF_SECURITY_ADMIN_PASSWORD}" | base64 -d)"
   ```

   Use these credentials to log in to the Grafana web interface.

### Setting Up Alerts in Grafana

Once you're logged into Grafana, you can set up alerts to monitor your systems. Here’s an example of how to set up an alert if Airflow has failed more than 2 times:

1. **Create a Dashboard**:

   - In Grafana, go to the "Create" button (plus icon) in the left-hand menu and select "Dashboard".
   - Add a new panel to this dashboard by selecting "Add new panel".

2. **Query the Airflow Metrics**:

   - In the new panel, write a Prometheus query to get the count of failed Airflow tasks.
   - Example query:
     ```prometheus
     sum(rate(airflow_task_fail_count[5m])) > 2
     ```

   This query checks if there have been more than 2 Airflow task failures in the last 5 minutes.

3. **Set Up the Alert**:

   - Below the query editor, click on the "Alert" tab.
   - Click on "Create Alert".
   - Set the condition to trigger the alert when the query returns a value greater than 2.
   - Configure the alert frequency and notification channels (e.g., email, Slack).

4. **Save the Dashboard**:

   - Give your dashboard and panel a name.
   - Save the dashboard so that the alert is active.

## Troubleshooting

If you encounter any issues, consider checking the logs for each service:

- For Airflow: `kubectl logs -l app.kubernetes.io/name=airflow`
- For Prometheus: `kubectl logs -l app.kubernetes.io/name=prometheus`
- For Grafana: `kubectl logs -l app.kubernetes.io/name=grafana`
- For MLflow: `kubectl logs -l app.kubernetes.io/name=mlflow`




                    
