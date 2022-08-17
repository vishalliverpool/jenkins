# jenkins


Step 1 — Installing Jenkins on Kubernetes
Kubernetes has a declarative API and you can convey the desired state using either a YAML or JSON file. For this tutorial, you will use a YAML file to deploy Jenkins. Make sure you have the kubectl command configured for the cluster.

First, use kubectl to create the Jenkins namespace:

kubectl create namespace jenkins
Next, create the YAML file that will deploy Jenkins.

Step 2
Create and open a new file called jenkins.yaml using nano or your preferred editor:

vim jenkins.yaml
Now add the following code to define the Jenkins image, its port, and several more configurations:

jenkins.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
        ports:
          - name: http-port
            containerPort: 8080
          - name: jnlp-port
            containerPort: 50000
        volumeMounts:
          - name: jenkins-vol
            mountPath: /var/jenkins_vol
      volumes:
        - name: jenkins-vol
          emptyDir: {}
This YAML file creates a deployment using the Jenkins LTS image and also opens port 8080 and 50000. You use these ports to access Jenkins and accept connections from Jenkins workers respectively.

Now create this deployment in the jenkins namespace:

kubectl create -f jenkins.yaml --namespace jenkins
Give the cluster a few minutes to pull the Jenkins image and get the Jenkins pod running.

Use kubectl to verify the pod’s state:

kubectl get pods -n jenkins
You will receive an output like this:

NAME                       READY   STATUS    RESTARTS   AGE
jenkins-6fb994cfc5-twnvn   1/1     Running   0          95s
Note that the pod name will be different in your environment.

Once the pod is running, you need to expose it using a Service. You will use the NodePort Service type for this tutorial. Also, you will create a ClusterIP type service for workers to connect to Jenkins.

Create and open a new file called jenkins-service.yaml:

Step 3
nano jenkins-service.yaml
Add the following code to define the NodePort Service:

jenkins-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30000
  selector:
    app: jenkins

---

apiVersion: v1
kind: Service
metadata:
  name: jenkins-jnlp
spec:
  type: ClusterIP
  ports:
    - port: 50000
      targetPort: 50000
  selector:
    app: jenkins
In the above YAML file, you define your NodePort Service and then expose port 8080 of the Jenkins pod to port 30000.

Now create the Service in the same namespace:

kubectl create -f jenkins-service.yaml --namespace jenkins
Check that the Service is running:

kubectl get services --namespace jenkins
You will receive an output like this:

Output
NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
jenkins   NodePort   your_cluster_ip   <none>        8080:30000/TCP   15d
With NodePort and Jenkins operational, you are ready to access the Jenkins UI and begin exploring it.

Step 2 — Accessing the Jenkins UI
In this step, you will access and explore the Jenkins UI. Your NodePort service is accessible on port 30000 across the cluster nodes. You need to retrieve a node IP to access the Jenkins UI.

Use kubectl to retrieve your node IPs:

kubectl get nodes -o wide
kubectl will produce an output with your external IPs:

Output
NAME        STATUS   ROLES    AGE   VERSION    INTERNAL-IP        EXTERNAL-IP        OS-IMAGE                       KERNEL-VERSION          CONTAINER-RUNTIME
your_node   Ready    <none>   16d   v1.18.8   your_internal_ip   your_external_ip   Debian GNU/Linux 10 (buster)   4.19.0-10-cloud-amd64   docker://18.9.9
your_node   Ready    <none>   16d   v1.18.8   your_internal_ip   your_external_ip   Debian GNU/Linux 10 (buster)   4.19.0-10-cloud-amd64   docker://18.9.9
your_node   Ready    <none>   16d   v1.18.8   your_internal_ip   your_external_ip   Debian GNU/Linux 10 (buster)   4.19.0-10-cloud-amd64   docker://18.9.9
Copy one of the your_external_ip values.

Now open a web browser and navigate to http://your_external_ip:30000.

A page will appear asking for an administrator password and instructions on retrieving this password from the Jenkins Pod logs.

Let’s use kubectl to pull the password from those logs.

First, return to your terminal and retrieve your Pod name:

kubectl get pods -n jenkins
You will receive an output like this:

NAME                       READY   STATUS    RESTARTS   AGE
jenkins-6fb994cfc5-twnvn   1/1     Running   0          9m54s
Next, check the Pod’s logs for the admin password. Replace the highlighted section with your pod name:

kubectl logs jenkins-6fb994cfc5-twnvn -n jenkins
You might need to scroll up or down to find the password:

Running from: /usr/share/jenkins/jenkins.war
webroot: EnvVars.masterEnvVars.get("JENKINS_HOME")
. . .

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

your_jenkins_password

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword
. . .
Copy your_jenkins_password. Now return to your browser and paste it into the Jenkins UI.

Once you enter the password, Jenkins will prompt you to install plugins. Because you are not doing anything unusual, select Install suggested plugins.

After installation, Jenkins will load a new page and ask you to create an admin user. Fill out the fields, or skip this step by pressing the skip and continue as admin link. This will leave your username as admin and your password as your_jenkins_password.

Another screen will appear asking about instance configuration. Click the Not now link and continue.

After this, Jenkins will create a summary of your choices and print Jenkins is ready! Click on start using Jenkins and the Jenkins home page will appear.

jenkins wizard
