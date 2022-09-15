
![Users and RBAC authorizations in Kubernetes](/static/ce2279a25f86b612f1bcda5b90467b2d/07465/rbac.png "Users and RBAC authorizations in Kubernetes")

Users and RBAC authorizations in Kubernetes
===========================================

![Robert Walid SOARES](https://avatars.githubusercontent.com/robertwalid?v=4&s=160)

By [Robert Walid SOARES](/en/author/robert/)

Aug 7, 2019


Categories

[Containers Orchestration](/en/skills/containers/)

[Data Governance](/en/skills/governance/)

Tags

[Cyber Security](/en/skills/cyber-security/)

[RBAC](/en/tag/rbac/)

[Authentication](/en/tag/authentication/)

[Authorization](/en/tag/authorization/)

[Kubernetes](/en/tag/kubernetes/)

[SSL/TLS](/en/tag/ssl-tls/)



Never miss our publications, subscribe to the Adaltas' newsletter about Open Source, big data and distributed systems. We maintain a low frequency of one email every two months.



Having your [Kubernetes](https://kubernetes.io/) cluster up and running is just the start of your journey and you now need to operate. To secure its access, user identities must be declared along with authentication and authorization properly managed.

This article focus on how to create users with [X.509](https://fr.wikipedia.org/wiki/X.509) client certificates and how to manage authorizations with the basic [Kubernetes Role-based access control (RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) API Objects. We will also talk about some open-source projects which simplify cluster administration: [rakkess](https://github.com/corneliusweig/rakkess), [kubectl-who-can](https://github.com/aquasecurity/kubectl-who-can), [rbac-lookup](https://github.com/reactiveops/rbac-lookup), and [RBAC Manager](https://github.com/reactiveops/rbac-manager).

[](#prerequisites-and-assumptions)**Prerequisites and assumptions**
-------------------------------------------------------------------

We need to make the following assumptions:

*   You have a basic level of understanding of [how Kubernetes works](https://kubernetes.io/docs/concepts/).
*   You have a [Kubernetes](https://kubernetes.io/) cluster or [Minikube](https://kubernetes.io/docs/setup/minikube/) running
*   You have the [Kubectl command line](https://kubernetes.io/docs/tasks/tools/install-kubectl/) installed
*   [OpenSSL](https://wiki.openssl.org/index.php/) installed

If you don’t have at your disposal a Kuberntes cluster, you can follow Arthur’s article [Installing Kubernetes on CentOS 7](/en/2019/01/29/installing-kubernetes-centos-7/) which uses [Vagrant](https://www.vagrantup.com/).

We have a 4 nodes [Kubernetes](https://kubernetes.io/) cluster, one master and three workers. The master will also be used as our edge node to interact with the cluster.

### [](#rbac-api-objects)[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) API Objects

[Role-based access control (RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) is a method of regulating access to computers and network resources based on the roles of individual users within an enterprise. We can use Role-based access control on all the [Kubernetes resources](https://kubernetes.io/fr/docs/concepts/) that allow CRUD (Create, Read, Update, Delete). Examples of resources:

*   [Namespaces](https://kubernetes.io/docs/user-guide/namespaces/)
*   [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/)
*   [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
*   [PersistentVolumes](https://kubernetes.io/docs/concepts/storage/volumes/)
*   [ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)

Examples of possible operations over these resources are:

*   create
*   get
*   delete
*   list
*   update

To manage [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) in [Kubernetes](https://kubernetes.io/), we need to declare:

*   [Role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) and [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)  
    They are just a set of rules that represent a set of permissions. A Role can only be used to grant access to resources within [namespaces](https://kubernetes.io/docs/user-guide/namespaces/). A ClusterRole can be used to grant the same permissions as a Role but they can also be used to grant access to cluster-scoped resources, non-resource endpoints.
*   [Subjects](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)  
    A subject is the entity that will make operations in the cluster. They can be [user accounts](https://kubernetes.io/docs/reference/access-authn-authz/authentication/), [services accounts](https://kubernetes.io/docs/reference/access-authn-authz/authentication/) or even a group.
*   [RoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) and [ClusterRoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)  
    As the name implies, it’s just the binding between a subject and a Role or a ClusterRole.

The default Roles defined in [Kubernetes](https://kubernetes.io/) are:

*   view: read-only access, excludes [secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
*   edit: above + ability to edit most [resources](https://kubernetes.io/docs/concepts/), excludes [roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) and [role bindings](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
*   admin: above + ability to manage [roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) and [role bindings](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) at a [namespace](https://kubernetes.io/docs/user-guide/namespaces/) level
*   cluster-admin: everything

We can, of course, create specific [Roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) and [ClusterRoles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/), but we recommend you to use the default as long as you can. It can quickly become difficult to manage all of this.

[](#use-case)Use Case:
----------------------

We will create two [namespaces](https://kubernetes.io/docs/user-guide/namespaces/) “my-project-dev” and “my-project-prod” and two users “jean” and “sarah” with different roles to those [namespaces](https://kubernetes.io/docs/user-guide/namespaces/):

*   my-project-dev:
    *   jean: Edit
*   my-project-prod:
    *   jean: View
    *   sarah: Edit

[](#users-creation-and-authentication-with-x509-client-certificates)Users creation and authentication with [X.509](https://fr.wikipedia.org/wiki/X.509) client certificates
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------

We mainly have two types of users: [service accounts](https://kubernetes.io/docs/reference/access-authn-authz/authentication/) managed by [Kubernetes](https://kubernetes.io/) and [normal users](https://kubernetes.io/docs/reference/access-authn-authz/authentication/). We will focus on [normal users](https://kubernetes.io/docs/reference/access-authn-authz/authentication/). Here is how the official documentation describes a normal user:

> Normal users are assumed to be managed by an outside, independent service. An admin distributing private keys, a user store like Keystone or Google Accounts, even a file with a list of usernames and passwords. In this regard, Kubernetes does not have objects which represent normal user accounts. Normal users cannot be added to a cluster through an API call.

They are multiple ways of [managing normal users](https://kubernetes.io/docs/reference/access-authn-authz/authentication/):

*   Basic Authentication:
    *   Pass a configuration with content like the following to API Server
        *   password,username,uid,group
*   X.509 client certificate
    *   Create a user’s private key and a certificate signing request
    *   Get it certified by a CA (Kubernetes CA) to have the user’s certificate
*   Bearer Tokens (JSON Web Tokens)
    *   OpenID Connect
        *   On top of OAuth 2.0
    *   Webhooks

For the purpose of this article we will use [X.509](https://fr.wikipedia.org/wiki/X.509) client certificates with [OpenSSL](https://wiki.openssl.org/index.php/) for their simplicity. There are different steps for users creation. We will go step by step. You have to perform the actions as a user with cluster-admin credentials. These are the steps for user creation (here for “jean”):

*   Create a user on the master machine then go into its home directory to perform the remaining steps.
    
        useradd jean && cd /home/jean
    
*   Create a private key:
    
        openssl genrsa -out jean.key 2048
    
*   Create a certificate signing request (CSR). CN is the username and O the group. We can set permissions by group, which can simplify management if we have, for example, multiple users with the same authorizations.
    
        # Without Group
        openssl req -new -key jean.key \
          -out jean.csr \
          -subj "/CN=jean"
        
        # With a Group where $group is the group name
        openssl req -new -key jean.key \
          -out jean.csr \
          -subj "/CN=jean/O=$group"
        
        #If the user has multiple groups
        openssl req -new -key jean.key \
          -out jean.csr \
          -subj "/CN=jean/O=$group1/O=$group2/O=$group3"
    
*   Sign the CSR with the Kubernetes CA. We have to use the CA cert and key which are normally in `/etc/kubernetes/pki/`. Our certificate will be valid for 500 days.
    
        openssl x509 -req -in jean.csr \
          -CA /etc/kubernetes/pki/ca.crt \
          -CAkey /etc/kubernetes/pki/ca.key \
          -CAcreateserial \
          -out jean.crt -days 500
    
*   Create a “.certs” directory where we are going to store the user public and private key.
    
        mkdir .certs && mv jean.crt jean.key .certs
    
*   Create the user inside Kubernetes.
    
        kubectl config set-credentials jean \
          --client-certificate=/home/jean/.certs/jean.crt \
          --client-key=/home/jean/.certs/jean.key
    
*   Create a context for the user.
    
        kubectl config set-context jean-context \
          --cluster=kubernetes --user=jean
    
*   Edit the user config file. The config file has the information needed for the authentication to the cluster. You can use the cluster admin config which is normally in `/etc/kubernetes`. The “certificate-authority-data” and “server” variables have to be as in the cluster admin config.
    
        apiVersion: v1
        clusters:
        - cluster:
           certificate-authority-data: {Parse content here}
           server: {Parse content here}
          name: kubernetes
        contexts:
        - context:
           cluster: kubernetes
           user: jean
          name: jean-context
        current-context: jean-context
        kind: Config
        preferences: {}
        users:
        - name: jean
          user:
           client-certificate: /home/jean/.certs/jean.cert
           client-key: /home/jean/.certs/jean.key
    
    Then we need to copy the config above in the `.kube` directory.
    
        mkdir .kube && vi .kube/config
    
*   Now we need to grant all the created files and directories to the user:
    
        chown -R jean: /home/jean/
    

Now we have a user “jean” created. We will do the same for user “sarah”. There are many steps to perform and it can be very time consuming to do if we have multiple users to create. This is why I edit bash scripts which automate the process. You can find them on [my Github repository](https://github.com/robertwalid/K8s_User_Creation_X509_Certs).

Now we have our users, we can create the two [namespaces](https://kubernetes.io/docs/user-guide/namespaces/):

    kubectl create namespace my-project-dev
    kubectl create namespace my-project-prod

As we have not defined any authorization to the users, they should get forbidden access to all cluster resources.

*   User: Jean

    kubectl get nodes
    Error from server (Forbidden): nodes is forbidden: User "jean" cannot list resource "nodes" in API group "" at the cluster scope
    kubectl get pods -n default
    Error from server (Forbidden): pods is forbidden: User "jean" cannot list resource "pods" in API group "" in the namespace "default"
    kubectl get pods -n my-project-prod
    Error from server (Forbidden): pods is forbidden: User "jean" cannot list resource "pods" in API group "" in the namespace "my-project-prod"
    kubectl get pods -n my-project-dev
    Error from server (Forbidden): pods is forbidden: User "jean" cannot list resource "pods" in API group "" in the namespace "my-project-dev"

*   User: Sarah

    kubectl get nodes
    Error from server (Forbidden): nodes is forbidden: User "sarah" cannot list resource "nodes" in API group "" at the cluster scope
    kubectl get pods -n default
    Error from server (Forbidden): pods is forbidden: User "sarah" cannot list resource "pods" in API group "" in the namespace "default"
    kubectl get pods -n my-project-prod
    Error from server (Forbidden): pods is forbidden: User "sarah" cannot list resource "pods" in API group "" in the namespace "my-project-prod"
    kubectl get pods -n my-project-dev
    Error from server (Forbidden): pods is forbidden: User "sarah" cannot list resource "pods" in API group "" in the namespace "my-project-dev"

[](#create-role-and-clusterrole)Create [Role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) and [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

We will use the default [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) available. However we will show you how to create specific [Role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)/[ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/). A [Role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)/[ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) are just a list of verbs (actions) permitted on specific resources and [namespaces](https://kubernetes.io/docs/user-guide/namespaces/). Here is an example of a YAML file:

    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: Role
    metadata:
      name: list-deployments
      namespace: my-project-dev
    rules:
      - apiGroups: [ apps ]
        resources: [ deployments ]
        verbs: [ get, list ]
    ---------------------------------
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRole
    metadata:
      name: list-deployments
    rules:
      - apiGroups: [ apps ]
        resources: [ deployments ]
        verbs: [ get, list ]

To create them:

    kubectl create -f /path/to/your/yaml/file

[](#bind-role-or-clusterrole-to-users)Bind [Role](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) or [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) to Users
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

We are now going to bind default [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) (Edit and View) to our users as below:

*   jean:
    *   Edit on [namespace](https://kubernetes.io/docs/user-guide/namespaces/) “my-project-dev”
    *   View on [namespace](https://kubernetes.io/docs/user-guide/namespaces/) “my-project-prod”
*   sarah:
    *   Edit on [namespace](https://kubernetes.io/docs/user-guide/namespaces/) “my-project-prod”

We need to create [RoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) by [namespaces](https://kubernetes.io/docs/user-guide/namespaces/) and not by user. It means that for our user “jean” we need to create two [RoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) for his authorizations. Example of [RoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) YAML file for Jean:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: jean
      namespace: my-project-dev
    subjects:
    - kind: User
      name: jean
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: ClusterRole
      name: edit
      apiGroup: rbac.authorization.k8s.io
    ---------------------------------
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: jean
      namespace: my-project-prod
    subjects:
    - kind: User
      name: jean
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      kind: ClusterRole
      name: view
      apiGroup: rbac.authorization.k8s.io

We assign to “jean” the view Role on “my-project-prod” and the edit Role on “my-project-dev”. We will do the same for “sarah” authorizations. To create them:

    kubectl apply -f /path/to/your/yaml/file

We have used `kubectl apply` here instead of `kubectl create`. The difference between “apply” and “create” is that “create” will create the object if it doesn’t exist and do nothing else. But if we “apply” a yaml file it means that it will create the object if it doesn’t exist and update it if needed.

Let’s check if our users have the right permissions.

*   User: sarah (Edit on “my-project-prod”)
    *   my-project-prod:
        *   Can list [pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/)(1)
        *   Can Create [deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)(2)
    *   my-project-dev:
        *   Can not list [pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/)(4)
        *   Can not create [deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)(5)

    kubectl get pods -n my-project-prod 
    No resources found. 
    
    kubectl run nginx --image=nginx --replicas=1 -n my-project-prod 
    deployment.apps/nginx created 
    
    [[email protected] ~]$kubectl get pods -n my-project-prod 
    NAME                    READY  STATUS   RESTARTS  AGE 
    nginx-7db9fccd9b-t14qw  1/1    Running  0         4s 
    
    kubectl get pods -n my-project-dev
    Error from server (Forbidden): pods is forbidden: User "sarah" cannot list resource "pods" in API group "" in the namespace "my-project-dev"
    
    kubectl run nginx --image=nginx --replicas=1 -n my-project-dev 
    Error from server (Forbidden): deployments.apps is forbidden: User "sarah" cannot create resource "deployments" in API group "apps" in the namespace "my-project-dev"

*   User: jean (View on “my-project-prod” & Edit on “my-project-dev”)
    *   my-project-prod:
        *   Can list [pods](https://kubernetes.io/docs/concepts/workloads/pods/)(1)
        *   Can list [deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)(2)
        *   Can not delete [deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)(3)
    *   my-project-dev:
        *   Can list [pods](https://kubernetes.io/docs/concepts/workloads/pods/)(4)
        *   Can create [deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)(5)
        *   Can list [deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)(6)
        *   Can delete [deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)(7)

    kubectl get pods -n my-project-prod 
    NAME                    READY  STATUS   RESTARTS  AGE 
    nginx-7db9fccd9b-t14qw  1/1    Running  0         101s 
    
    [[email protected] —]$ kubectl get deploy -n my-project-prod 
    NAME   READY  UP-TO-DATE  AVAILABLE  AGE 
    nginx  1/1    1           1          110s 
    
    kubectl delete deploy/nginx -n my-project-prod 
    Error from server (Forbidden): deployments.extensions "nginx" is forbidden: User "jean" cannot delete resource "deployments" in API group "extensions" in the namespace "my-project-prod" 
    
    kubectl get pods -n my-project-dev
    No resources found. 
    
    kubectl run nginx --image=nginx --replicas=1 -n my-project-dev 
    deployment.apps/nginx created 
    
    kubectl get deploy -n my-project-dev 
    NAME   READY  UP-TO-DATE  AVAILABLE  AGE 
    nginx  0/1    1           0          13s 
    
    kubectl delete deploy/nginx -n my-project-dev 
    deployment.extensions "nginx" deleted 
    
    kubectl get deploy -n my-project-dev 
    No resources found.

[](#manage-users-and-theirs-authorizations)Manage users and theirs authorizations
---------------------------------------------------------------------------------

Now that we have set different roles and authorizations to our users, how can we manage all of this? How can we know if a user has the right access? How to know who can actually perform a specific action? How to have an overview of all the users access? These are the questions we need to answer to ensure cluster security. In [Kubernetes](https://kubernetes.io/) we have the command `kubectl auth can-i` that allows us to know if a user can perform a specific action.

    # kubectl auth can-i $action $resource --as $subject
    
    kubectl auth can-i list pods 
    kubectl auth can-i list pods --as jean

This first command allows a user to know if he can perform an action. The second command allows an administrator to impersonate a user to know if the targeted user can perform an action. The impersonation can only be used by a user with cluster-admin credentials. Apart from that, we can’t do much more. This is why we will introduce you some open-source projects that allows us to extend the functionalities covered by the `kubectl auth can-i` command. Before introducing them, we will install some of their dependencies such as [Krew](https://github.com/kubernetes-sigs/krew) and [GO](https://en.wikipedia.org/wiki/Go_(programming_language)).

### [](#installation-of-go)Installation of [GO](https://en.wikipedia.org/wiki/Go_(programming_language))

[Go](https://en.wikipedia.org/wiki/Go_(programming_language)) is an open source programming language that makes it easy to build simple, reliable, and efficient software. Inspired by [C](https://en.wikipedia.org/wiki/C_(programming_language)) and [Pascal](https://en.wikipedia.org/wiki/Pascal_(programming_language)), this language was developed by [Google](https://en.wikipedia.org/wiki/Google) from an initial concept of [Robert Griesemer](https://de.wikipedia.org/wiki/Robert_Griesemer), [Rob Pike](https://en.wikipedia.org/wiki/Rob_Pike) and [Ken Thompson](https://en.wikipedia.org/wiki/Ken_Thompson).

    wget https://dl.google.com/go/go1.12.5.linux-amd64.tar.gz
    sudo tar -C /usr/local -xzf go1.12.5.linux-amd64.tar.gz
    export PATH=$PATH:/usr/local/go/bin

### [](#installation-of-krew)Installation of [KREW](https://github.com/kubernetes-sigs/krew)

[Krew](https://github.com/kubernetes-sigs/krew) is a tool that makes it easy to use [kubectl plugins](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/). [Krew](https://github.com/kubernetes-sigs/krew) helps you discover plugins, install and manage them on your machine. It is similar to tools like apt, dnf or [brew](https://brew.sh/). [Krew](https://github.com/kubernetes-sigs/krew) is only compatible with [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) v1.12 and above.

    set -x; cd "$(mktemp -d)" &&
    curl -fsSLO "https://storage.googleapis.com/krew/v0.2.1/krew.{tar.gz,yaml}" &&
    tar zxvf krew.tar.gz &&
    ./krew-"$(uname | tr '[:upper:]' '[:lower:]')_amd64" install \
        --manifest=krew.yaml --archive=krew.tar.gz
    
    export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

### [](#rakkess)[rakkess](https://github.com/corneliusweig/rakkess)

This project helps us to know all the authorizations that have been granted to a user. It helps to answer to the question: what can do “jean”? Firstly, let’s install [Rakkess](https://github.com/corneliusweig/rakkess):

    kubectl krew install access_matrix

You can find the documentation on [the project Github repository](https://github.com/corneliusweig/rakkess). Here is an example:

    kubectl access-matrix -n my-project-dev --as jean
    NAME                                            LIST  CREATE  UPDATE  DELETE
    bindings                                               ✖               
    configmaps                                      ✔     ✔       ✔       ✔
    controllerrevisions.apps                        ✔     ✖       ✖       ✖
    cronjobs.batch                                  ✔     ✔       ✔       ✔
    daemonsets.apps                                 ✔     ✔       ✔       ✔
    daemonsets.extensions                           ✔     ✔       ✔       ✔
    deployments.apps                                ✔     ✔       ✔       ✔
    deployments.extensions                          ✔     ✔       ✔       ✔
    endpoints                                       ✔     ✔       ✔       ✔
    events                                          ✔     ✖       ✖       ✖
    events.events.k8s.io                            ✖     ✖       ✖       ✖
    horizontalpodautoscalers.autoscaling            ✔     ✔       ✔       ✔
    ingresses.extensions                            ✔     ✔       ✔       ✔
    ingresses.networking.k8s.io                     ✔     ✔       ✔       ✔
    jobs.batch                                      ✔     ✔       ✔       ✔
    leases.coordination.k8s.io                      ✖     ✖       ✖       ✖
    limitranges                                     ✔     ✖       ✖       ✖
    localsubjectaccessreviews.authorization.k8s.io         ✖               
    networkpolicies.extensions                      ✔     ✔       ✔       ✔
    networkpolicies.networking.k8s.io               ✔     ✔       ✔       ✔
    persistentvolumeclaims                          ✔     ✔       ✔       ✔
    poddisruptionbudgets.policy                     ✔     ✔       ✔       ✔
    pods                                            ✔     ✔       ✔       ✔
    podtemplates                                    ✖     ✖       ✖       ✖
    replicasets.apps                                ✔     ✔       ✔       ✔
    replicasets.extensions                          ✔     ✔       ✔       ✔
    replicationcontrollers                          ✔     ✔       ✔       ✔
    resourcequotas                                  ✔     ✖       ✖       ✖
    rolebindings.rbac.authorization.k8s.io          ✖     ✖       ✖       ✖
    roles.rbac.authorization.k8s.io                 ✖     ✖       ✖       ✖
    secrets                                         ✔     ✔       ✔       ✔
    serviceaccounts                                 ✔     ✔       ✔       ✔
    services                                        ✔     ✔       ✔       ✔
    statefulsets.apps                               ✔     ✔       ✔       ✔

### [](#kubect-who-can)[kubect-who-can](https://github.com/aquasecurity/kubectl-who-can)

This project allows us to know who are the users who can perform a specific action. It helps to answer the question: who can do this action? Installation:

    go get -v github.com/aquasecurity/kubectl-who-can

You can find the documentation on [the project Github repository](https://github.com/aquasecurity/kubectl-who-can). Here is an example:

    kubectl-who-can list pods -n default
    No subjects found with permissions to list pods assigned through RoleBindings
    
    CLUSTERROLEBINDING                                    SUBJECT                             TYPE            SA-NAMESPACE
    cluster-admin                                         system:masters                      Group           
    rbac-manager                                          rbac-manager                        ServiceAccount  rbac-manager
    system:controller:attachdetach-controller             attachdetach-controller             ServiceAccount  kube-system
    system:controller:clusterrole-aggregation-controller  clusterrole-aggregation-controller  ServiceAccount  kube-system
    system:controller:cronjob-controller                  cronjob-controller                  ServiceAccount  kube-system
    system:controller:daemon-set-controller               daemon-set-controller               ServiceAccount  kube-system
    system:controller:deployment-controller               deployment-controller               ServiceAccount  kube-system
    system:controller:endpoint-controller                 endpoint-controller                 ServiceAccount  kube-system
    system:controller:generic-garbage-collector           generic-garbage-collector           ServiceAccount  kube-system
    system:controller:horizontal-pod-autoscaler           horizontal-pod-autoscaler           ServiceAccount  kube-system
    system:controller:job-controller                      job-controller                      ServiceAccount  kube-system
    system:controller:namespace-controller                namespace-controller                ServiceAccount  kube-system
    system:controller:node-controller                     node-controller                     ServiceAccount  kube-system
    system:controller:persistent-volume-binder            persistent-volume-binder            ServiceAccount  kube-system
    system:controller:pod-garbage-collector               pod-garbage-collector               ServiceAccount  kube-system
    system:controller:pvc-protection-controller           pvc-protection-controller           ServiceAccount  kube-system
    system:controller:replicaset-controller               replicaset-controller               ServiceAccount  kube-system
    system:controller:replication-controller              replication-controller              ServiceAccount  kube-system
    system:controller:resourcequota-controller            resourcequota-controller            ServiceAccount  kube-system
    system:controller:statefulset-controller              statefulset-controller              ServiceAccount  kube-system
    system:coredns                                        coredns                             ServiceAccount  kube-system
    system:kube-controller-manager                        system:kube-controller-manager      User            
    system:kube-scheduler                                 system:kube-scheduler               User            

### [](#rbac-lookup)[rbac-lookup](https://github.com/reactiveops/rbac-lookup)

This project permits us to have a RBAC overview. It helps to answer the questions: Which Role has “jean”? “sarah”? All the users? all the group? To install the project:

    kubectl krew install rbac-lookup

The official documentation is available on the [project Github repository](https://github.com/reactiveops/rbac-lookup). An example:

    kubectl-rbac_lookup jean
    SUBJECT   SCOPE       ROLE
    jean      my-project-dev    ClusterRole/edit
    jean      my-project-prod   ClusterRole/view
    
    kubectl-rbac_lookup sarah
    SUBJECT   SCOPE       ROLE
    sarah     my-project-prod   ClusterRole/edit
    
    kubectl-rbac_lookup --kind user
    SUBJECT                           SCOPE          ROLE
    jean                              my-project-dev       ClusterRole/edit
    jean                              my-project-prod      ClusterRole/view
    sarah                             my-project-prod      ClusterRole/edit
    system:anonymous                  kube-public    Role/kubeadm:bootstrap-signer-clusterinfo
    system:kube-controller-manager    kube-system    Role/extension-apiserver-authentication-reader
    system:kube-controller-manager    kube-system    Role/system::leader-locking-kube-controller-manager
    system:kube-controller-manager    cluster-wide   ClusterRole/system:kube-controller-manager
    system:kube-proxy                 cluster-wide   ClusterRole/system:node-proxier
    system:kube-scheduler             kube-system    Role/extension-apiserver-authentication-reader
    system:kube-scheduler             kube-system    Role/system::leader-locking-kube-scheduler
    system:kube-scheduler             cluster-wide   ClusterRole/system:kube-scheduler
    system:kube-scheduler             cluster-wide   ClusterRole/system:volume-scheduler
    
    kubectl-rbac_lookup --kind group
    SUBJECT                                            SCOPE          ROLE
    system:authenticated                               cluster-wide   ClusterRole/system:basic-user
    system:authenticated                               cluster-wide   ClusterRole/system:discovery
    system:authenticated                               cluster-wide   ClusterRole/system:public-info-viewer
    system:bootstrappers:kubeadm:default-node-token    cluster-wide   ClusterRole/system:node-bootstrapper
    system:bootstrappers:kubeadm:default-node-token    cluster-wide   ClusterRole/system:certificates.k8s.io:certificatesigningrequests:nodeclient
    system:bootstrappers:kubeadm:default-node-token    kube-system    Role/kube-proxy
    system:bootstrappers:kubeadm:default-node-token    kube-system    Role/kubeadm:kubelet-config-1.14
    system:bootstrappers:kubeadm:default-node-token    kube-system    Role/kubeadm:nodes-kubeadm-config
    system:masters                                     cluster-wide   ClusterRole/cluster-admin
    system:nodes                                       kube-system    Role/kubeadm:kubelet-config-1.14
    system:nodes                                       kube-system    Role/kubeadm:nodes-kubeadm-config
    system:nodes                                       cluster-wide   ClusterRole/system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
    system:unauthenticated                             cluster-wide   ClusterRole/system:public-info-viewer

### [](#rbac-manager)[RBAC Manager](https://github.com/reactiveops/rbac-manager)

This project allows us to have a manager for [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/), as its name suggests. It simplifies many manipulations. The most important one is [RoleBindings](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) creation. Indeed we saw that if we needed to create different Roles for a user we needed to create different [RoleBindings](https://kubernetes.io/docs/reference/access-authn-authz/rbac/). [RBAC Manager](https://github.com/reactiveops/rbac-manager) helps us by allowing us to create just one [RoleBinding](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) with all the authorizations inside. To install it, you can download the YAML file from [the Github repository](https://github.com/reactiveops/rbac-manager):

    kubectl apply -f /path/to/rbac/manager/yaml/file

The official documentation is available on the [project Github repository](https://github.com/reactiveops/rbac-manager). An example:

    apiVersion: rbacmanager.reactiveops.io/v1beta1
    kind: RBACDefinition
    metadata:
      name: jose
    rbacBindings:
      - name: jose
        subjects:
          - kind: User
            name: jose
        roleBindings:
          - namespace: my-project-prod
            clusterRole: edit
          - namespace: my-project-dev
            clusterRole: edit

[](#conclusion)Conclusion
-------------------------

We have created users inside [Kubernetes](https://kubernetes.io/) cluster using [X.509](https://fr.wikipedia.org/wiki/X.509) client certificate with OpenSSL and granting them authorizations. You can use the script available on [my Github repository](https://github.com/robertwalid/K8s_User_Creation_X509_Certs) in order to create users easily. As for the cluster administration, you can use the open-source projects that have been introduced in this article. To sum up those projects:

*   [kubectl auth can-i](https://kubernetes.io/docs/reference/access-authn-authz/authorization/): know if a user can perform a specific action
*   [rakkess](https://github.com/corneliusweig/rakkess): know all the actions that a user can perform
*   [kubectl-who-can](https://github.com/aquasecurity/kubectl-who-can): know who are the users that can perform a specific action
*   [rbac-lookup](https://github.com/reactiveops/rbac-lookup): get a [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) overview
*   [RBAC Manager](https://github.com/reactiveops/rbac-lookup): get simpler configuration that groups bindings together, easy to automate [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) changes, and label selectors.

It can be very time consuming to handle all the steps about user creation. Especially if we have multiple users to create at once and others to create frequently. It could be easier if an enterprise LDAP is connected to [Kubernetes](https://kubernetes.io/) cluster. There are open-source projects that provide a direct [LDAP](https://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol) authentication webhook for Kubernetes: [Kismatic](https://github.com/apprenda-kismatic/kubernetes-ldap) and [ObjectifLibre](https://github.com/ObjectifLibre/k8s-ldap). Another solution is to configure an OpenId server with your enterprise LDAP for its backend.
