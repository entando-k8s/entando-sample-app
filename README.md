# Entando Sample App 
A sample application that can be used to produce a Docker image for deployment to Kubernetes using the new Entando
Kubernetes Operator

## Develop using the jetty-maven-plugin
If you need to develop fast against this project, you can run the application using the jetty-maven-plugin.
You will need though to use the correct profiles to run it.

Here the command to use 
```
mvn clean package jetty:run -Pjetty-local -Pderby
```

If you want to use keycloak as external authorization service, add the keycloak profile and update the proper variables (you can find them in the `properties` tag in the pom)

```
mvn clean package jetty:run -Pjetty-local -Pderby -Pkeycloak
```

## Using docker
You can use the fabric8 plugin  to build the docker image for this project. 
You can choose between different profiles both for the servlet container (jetty, tomcat, wildfly, eap) 
and whether or not to use the embedded Derby  dbms.

Here is the command to build the image (in this case I'm choosing to build the image using wildfly)
```
mvn clean package -Pwildfly docker:build
```

Here is the command to build the image with a pre-initialized embedded database
```
mvn clean package -Pwildfly -Pderby-init docker:build
```

# Deploying to Kubernetes

## Prerequisites

* You have installed either the Openshift client `oc` or the Kubernetes client `kubectl` locally
* You have access to an Openshift or Kubernetes cluster, with a sandbox namespace/project you have admin access to. In 
  subsequent instructions we will refer to this namespace as '[your-sandbox-namespace]'
* The Entando K8S Custom Resource Definitions have been registered on your Kubernetes Cluster. For more information, you
  can consult these [instructions](https://github.com/entando-k8s/entando-k8s-custom-model/blob/master/src/main/resources/crd/README.md).   
* You have installed Java and Maven
* You have a Git client
* You have installed Helm, ideally the Helm 2 client without Tiller
* You have read/write access to a Docker registry and an organization hosted on this Docker registry. The easiest will 
  be to just use Docker Hub, i.e. docker.io, as the registry and your username on Docker Hub. We will assume such 
  a configuration in subsequent instructions (docker.io/[your-dockerhub-username]) but you
  are free to use any alternative Docker registry and organization. Ensure you have logged into this registry e.g.
 
 `docker login -u [your-dockerhub-username] -p [your-dockerhub-password] docker.io` 

## Steps to deploy

1. From your Git client, clone this repository. Using the Git CLI, you could do this:    
    `git clone https://github.com/entando-k8s/entando-sample-app.git`
2. Change the project name to your preferred project name. 
    1.  From your favourite IDE, do a global search/replace for the 
string `entando-sample-app` and replace it with your new app name. We will refer to this name in subsequent instructions 
as '[your-app-name]'. The files updated should be:
        * pom.xml: the artifactId and hence the finalName of the project will be set to your new project name 
        * entando-app.yaml: this is the yaml that the Entando Kubernetes Operator will pick up and deploy your app.
        * skaffold.yaml: if you choose to use Jenkins X or Skaffold to build your Docker images, this file could be useful. 
        * various files in the ./charts directory, primarily serving as a name for the Helm chart this project produces.
    2. Rename the `./charts/entando-sample-app` directory to `./charts/[your-app-name]` 
3. Deploy the Entando Kubernetes operator to K8S:
   1. Navigate to the ./charts/preview directory
   2. Modify the values.yaml file to work in your Kubernetes cluster
      * Specify a routing suffix for you K8S or Openshift cluster using the ENTANDO_DEFAULT_ROUTING_SUFFIX environment variable
      * If you are using Openshift, set the `supportOpenshift` property to `true`
   3. From the commandline, in the `./charts/` directory, use Helm to first download the Entando Operator chart and then generate a yaml file to deploy the Entando operator:    
     `helm dependency update preview/`     
     `helm template --name=[your-app-name]  --namespace=[your-sandbox-namespace] preview/ > operator-deployment.yaml`
   4. Inspect the resulting yaml file (`operator-deployment.yaml`) to form an understanding of what will happen on your cluster.
   5. Create the operator deployment
       * In vanilla Kubernetes:            
     `kubectl create -f operator-deployment.yaml -n [your-sandbox-namespace]`
       * In Openshift:       
     `oc create -f operator-deployment.yaml -n [your-sandbox-namespace]`
   6. Wait for all the infrastructure to be deployed until you can access the endpoint URLS for all the Ingresses in your namespace. You can also
      run this command to follow the status updates against the various pods:    
      `watch kubectl get pods -n [your-sandbox-namespace]`      
4. Build the Docker image for your app. You have one of two choices here, depending your skill-set and available technology
   * Using Maven only
       1. In the `pom.xml` file, modify the Fabric8 Docker Maven Plugin image `<name>` element to reflect you Docker registry and organization, e.g.:       
         `<name>docker.io/[your-dockerhub-username]/${project.artifactId}</name>`
       2. Use the maven command to build your image on the base image of choice selecting its profile:       
         `mvn clean package -Pwildfly -Pderby docker:build`
   * Using the `docker` command:    
       1. On the command line, from the project base directory, run the docker build command. Ensure that the tag you 
          assign to the resulting image (using the `-t` flag) reflects your Docker registry and organization, e.g.:      
         `docker build . -t docker.io/[your-dockerhub-username]/[your-app-name]:latest `
5. Push the resulting image to your docker registry:     
    `docker push docker.io/[your-dockerhub-username]/[your-app-name]:latest`   
6. Deploy Entando App itself using Helm.
    1. Navigate to the `./charts/[your-app-name]` directory
    2. Modify the `values.yaml` file to reflect:    
        * your selection of DBMS in `app.dbms`
        * and the number of replicas of the App you require in `app.replicaCount`
        * the Docker image name of your (e.g. docker.io/[your-dockerhub-username]/[your-app-name]) App in `app.repository`
        * that tag of your Docker image (e.g. 1.0.0-SNAPSHOT) in `app.tag`
    3. From the commandline, in the `./charts/` directory, use Helm to generate a yaml file to deploy the Entando App:    
        `helm template --name=[your-app-name]  --namespace=[your-sandbox-namespace] [your-app-name]/ > entando-app.yaml`
    4. Inspect the resulting yaml file (`entando-app.yaml`) to inspect the resulting EntandoApp custom resource
    5. Deploy the app
        * In vanilla Kubernetes:    
           `kubectl create -f entando-app.yaml -n [your-sandbox-namespace]`
        * In Openshift:    
           `oc create -f entando-app.yaml -n [your-sandbox-namespace]`
    6. Wait for all the resulting HTTP endpoint URLS for all the Ingresses in your namespace to be available.      
   
# Creating a pipeline for this project.

At the time of the writing of this document, several different approaches are available to establish a build pipeline
for Entando Apps. Generally we would suggest a cloud native approach. Some options would be:
* Openshift's Jenkins extension, a hybrid approach using a traditional monolithic Jenkins server fully integrated with Openshift.
* Tekton Pipelines, the Kubernetes project's own, cloud native internal pipeline engine
* Argo CI/CD, fully cloud native, similar to Tekton Pipelines, but not related in any way 
* Jenkins X, serverless, based entirely on Tekton Pipelines, developed by CloudBees themselves. 

Internally, Entando uses Jenkins X. We will therefore provide some instructions on how to estabalish a pipeline for
 an Entando app using Jenkins X. However, in no way is this intended to restrict our customers to a single product. However,
 it is likely that in future, Entando will provide integration into and visibility of one or more of the Tekton Pipelines based
 products
## Steps:        

1. Navigate to the`./chars/preview/ directory
2. Uncomment the local filesystem dependency to ../[your-app-name]/
3. Import into Jenkins X using:    
`jx import . `

## Resulting pipeline
The resulting pipeline will automatically create a preview environment for pull requests
You may want to optimize this environment to use a shared Keycloak and Entando Cluster infrastructure
instance.

## Deploying to other environments (e.g. staging, production)

In the end, only an EntandoApp custom resource gets deployed to these environments. The current template
in `./charts/[your-app-name]/templates/entando-app.yaml` parameterizes the replica count and 
automatically deployed DBMS image used for your Entando App. You can provide values for
these parameters in the `values.yaml` file for the environment in question. 
