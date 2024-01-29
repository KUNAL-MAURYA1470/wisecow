
# Objective
The primary objective of this project is to containerize and deploy the Wisecow application from the provided GitHub repository onto a Kubernetes environment. This deployment will be facilitated by ArgoCD, introducing a continuous delivery mechanism. Additionally, the deployment aims to implement secure TLS communication for enhanced security.

# Deployment Steps
- ##  Dockerization:
  The Dockerfile is used to build a Docker image for the Wisecow application.
  ![image](https://github.com/KUNAL-MAURYA1470/wisecow/assets/83691101/f48e6657-1972-4a77-8e68-838b991281a0)
  - ### Building the Docker Image
     To build the Docker image, execute the following command in the same directory as the Dockerfile:
      ``` bash
      docker build -t wisecow-image .
      ```
  - ### Running the Docker Container
     To run the Wisecow application in a Docker container, use the following command:
      ```bash
      docker run -p 8080:4499 wisecow-image
      ```
  - Access the application in the web browser at http://localhost:8080.

- ## Kubernetes Deployment:
  This repository contains Kubernetes YAML files for deploying the Wisecow application. The deployment is configured using a Kubernetes Deployment and Service.
  
  ![image](https://github.com/KUNAL-MAURYA1470/wisecow/assets/83691101/a5b42891-d176-47bf-8875-54fc9e1477b4)
  - Apply the Deployment configuration:
    ```bash
      kubectl apply -f deployment.yaml
    ```
  - The Wisecow application will be deployed, and you can access it externally using the specified NodePort (30080).

- ## Continuous Integration and Deployment (CI/CD):
  The GitHub Actions workflow automates the Continuous Integration (CI) and Continuous Deployment (CD) processes for the Wisecow application. The workflow is triggered on pushes to the main branch.
  ![image](https://github.com/KUNAL-MAURYA1470/wisecow/assets/83691101/0f03c7f8-3d16-4035-a15f-5844cff9c683)

 ### Jobs
  - ### 1. Docker Build and Push (docker)
    Steps:
    - Checkout: Check out the repository.
    - Set up QEMU: Set up QEMU for multi-platform builds.
    - Set up Docker Buildx: Set up Docker Buildx for efficient Docker builds.
    - Login to Docker Hub: Log in to Docker Hub using secrets for authentication.
    - Build and Push: Build the Docker image and push it to Docker Hub. Tags the image with the format kunalmaurya/wisecow:app-${{ github.run_number }}.
  - ### 2. Modify Git Repository (modifygit)
    Needs: docker job (waits for the completion of the Docker job).
    Steps:
    - Checkout: Check out the repository.
    - Modify Deployment: Change the deployment image tag in the deployment.yaml file to match the latest Docker image tag.
    - Commit and Push: Commit the changes and push them to the deployment repository.

    The above GitHub repo uses secrets for Docker Hub and Git.
    ![image](https://github.com/KUNAL-MAURYA1470/wisecow/assets/83691101/d32b9761-4942-4941-9ef9-10ac77ef9039)

The GitOps CI/CD pipeline is now set up to automate the build, push, and deployment processes. Whenever a commit is made to the main branch of Application repository, the pipeline will be triggered automatically. It performs the following actions:

- ### Builds and pushes the Docker image
   - The pipeline uses the Dockerfile in the repository to build a Docker image. It then pushes the image to a Docker registry, such as Docker Hub. This step ensures that the latest version of the application is available for deployment.
- ### Updates the version in the manifest repository
   - The pipeline updates the version of the newly built image in a separate Git repository that holds the deployment manifests. This ensures that the deployment manifests reflect the latest image version.
- ### Triggers ArgoCD deployment
  - The changes made to the deployment manifest repository automatically trigger ArgoCD, which is a GitOps tool for deploying applications to Kubernetes. ArgoCD uses the updated deployment manifests to deploy the application in the Kubernetes cluster.

### Installing ArgoCD in Kubernetes Cluster
``` bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

- After running the installation command, you can verify the deployment by checking the status of the ArgoCD pods:
``` bash
kubectl get pods -n argocd
```
- To access the ArgoCD dashboard, I will be using Port Forwarding to access the ArgoCD.
``` bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
- Access ArgoCD Dashboard from your local machine using the following link
```
http://127.0.0.1:8080
```

- To get the password you may execute the below command in your Kubernetes cluster. (username is admin)
``` bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
Next, we have to create an App in ArgoCD in which we basically define where is our application’s repository located and where to deploy it, and some other small configurations.

![image](https://github.com/KUNAL-MAURYA1470/wisecow/assets/83691101/ba6a7cb7-3c36-4a92-988c-9f1981459280)

![image](https://github.com/KUNAL-MAURYA1470/wisecow/assets/83691101/94427bcd-a967-46fc-b9c7-82ca83af2b74)

Now click on Create button. This will initiate the creation process of an application in ArgoCD. ArgoCD diligently takes charge and starts the synchronization process, aiming to seamlessly deploy the resources defined in the manifest file to our Kubernetes cluster.

![image](https://github.com/KUNAL-MAURYA1470/wisecow/assets/83691101/740c3daa-e257-438f-a695-bf1019d929ec)

# TLS Implementation
- ## Implement a self-signed TLS certificate for wisecow app
    - Create a self-signed certificate and private key:

  ``` bash
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=my-demo-app.local"
    ```
    - Then create a Kubernetes Secret containing the certificate and private key:
    ``` bash
    kubectl create secret tls my-demo-app-tls --key tls.key --cert tls.crt
    ```

Now, we need to install the Nginx Ingress Controller so that it can redirect incoming requests to your payment app to use HTTPS. Since you've exposed the app using nodePort, you need to install the Ingress using a custom value file that specifies the service type to NodePort.
  - Create a file called ingress-values.yaml with the following content:
    ![image](https://github.com/KUNAL-MAURYA1470/wisecow/assets/83691101/06599ab2-f76e-4708-804f-a58b29dccd24)

  - Then install the Nginx Ingress Controller using Helm and the custom values file:
    ``` bash
    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    helm repo update
    helm install ingress-nginx ingress-nginx/ingress-nginx -f ingress-values.yaml
    ```
  - Create an ingress.yaml file
    
    ![image](https://github.com/KUNAL-MAURYA1470/wisecow/assets/83691101/e09435ca-77cf-4293-8ec8-a88b4fff2f38)

  - And apply this manifest file by using the following command:
    ``` bash
    kubectl apply -f ingress.yaml
    ```
  - Update your /etc/hosts file with the following entry:
    ``` bash
    127.0.0.1 my-demo-app.local
    ```

- ## Implement a trusted TLS certificate
   - create a local CA and generate a trusted certificate for your local domain:
  ``` bash
    mkcert -install
    mkcert my-demo-app.local
  ``` 
   - Delete the previous Secret and create a new one for mkcert:
    ``` bash
    kubectl delete secret my-demo-app-tls
    kubectl create secret tls my-demo-app-tls --key 
    my-demo-app.local-key.pem --cert my-demo-app.local.pem
    ``` 
   - Finally, redeploy the Ingress resource by running the following:
    ``` bash
    kubectl delete ingress demo-payment-app-ingress
    kubectl apply -f ingress.yaml
    ```
## Verify the TLS for  Kubernetes application
- go to https://my-demo-app.local:32443
  ![image](https://github.com/KUNAL-MAURYA1470/wisecow/assets/83691101/57c82e00-acd9-4e0e-8c68-a5af994ea96a)
  
- Click on the lock icon in the URL bar to verify that the TLS certificate is trusted by your browser and that the connection is secure:
  
![image](https://github.com/KUNAL-MAURYA1470/wisecow/assets/83691101/62fbc0a1-9281-4e41-a474-523e2579f6b7)

![image](https://github.com/KUNAL-MAURYA1470/wisecow/assets/83691101/2f38f918-2f0b-49ce-b8cc-7b33a934396a)

