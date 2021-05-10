# boundary_demo
A simple demonstration on how to set up boundary on dev mode and also using terraform.

### Setting up boundary dev (testing purposes)

Prerequisites

* Docker is installed
* A route to download the Postgres Docker image image or a local image cached
* A Boundary binary in your PATH - Head to the official page to install [boundary](https://learn.hashicorp.com/tutorials/boundary/getting-started-install?in=boundary/getting-started)

```
boundary dev -login-name="dev-admin" -password="p@ssw0rd"
```

### On anoter tab type:

```
boundary authenticate password -auth-method-id ampw_1234567890 -login-name dev-admin -password "p@ssw0rd" -keyring-type=none
```

### You can now access the admin console using your browser of choice:

`http://localhost:9200/`


# SETTING UP BOUNDARY USING TERRAFORM ON AWS

### Install go 1.15 or later

```
wget https://golang.org/dl/go1.16.3.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.16.3.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
go version
```

### Use terraform version 0.13.0

```
tfenv install 0.13.0
tfenv use 0.13.0
```

**You can check tfenv installation instructions [here](https://github.com/tfutils/tfenv)**

### Download the binary and place it on the default location that the terraform expects it to be

```
BOUNDARY_BIN_DIR=~/projects/boundary/bin; curl https://releases.hashicorp.com/boundary/0.2.0/boundary_0.2.0_linux_amd64.zip --create-dirs -o $BOUNDARY_BIN_DIR/boundary && unzip -o $BOUNDARY_BIN_DIR/boundary -d $BOUNDARY_BIN_DIR 
```

or download directly [here](https://www.boundaryproject.io/downloads) extract and move to the `~/projects/boundary/bin` directory.

**note: this boundary binary will be sent to every controller/worker, we are extracting on ~/projects/boundary/bin because of [this](https://github.com/hashicorp/boundary-reference-architecture/blob/main/deployment/aws/vars.tf#L2)**

### Clone the boundary reference repository and execute the terraform plan.

```
git clone git@github.com:hashicorp/boundary-reference-architecture.git

cd boundary-reference-architecture/deployment/aws

terraform apply -target module.aws
```

### Log into the controllers/workers and check if the services are running.

```
ssh ubuntu@<controller-ip>
sudo systemctl status boundary-controller

ssh ubuntu@<worker-ip>
sudo systemctl status boundary-worker 
```

### Run the plan for the boundary module (without the target flag)

```
terraform apply 
```

***Check the terraform output and copy the auth-method-id that was created. We are going to use it to authenticate to the boundary***

### Authenticate to boundary using the CLI

```
BOUNDARY_ADDR='http://<YOUR-ELB-DNS-NAME>:9200' \
  boundary authenticate password \
  -login-name=jim \
  -password foofoofoo \
  -auth-method-id=ampw_SiNvfLXbjg \
  -keyring-type=none
```

***replace the `-auth-method-id` for the one you copied from the terraform output.***

***copy the token that is going to be generated and store it somewhere safe, we are going to need it to connect to our targets later***

### Access the web interface using your ELB dns name on port 9200

`http://<YOUR-ELB-DNS-NAME>:9200/`

### Let's log in to the web interface using our recently user created

![Image](images/login.png?raw=true)

### Let's create a new project called Example-Project

![Image](images/new_project.png?raw=true)

![Image](images/project_created.png?raw=true)

![Image](images/sidebar_project.png?raw=true)

### First, let's create a host catalog that contains a hostset 

![Image](images/create_hostset.png?raw=true)

### Let's imagine this hostset is a set of relational databases we want to give access to someone

![Image](images/create_hostset2.png?raw=true)

### Now, let's add some hosts to our hostset

![Image](images/add_hosts_to_hostset.png?raw=true)

![Image](images/add_hosts_to_hostset2.png?raw=true)

### Let's create a target that points to our hostset

![Image](images/create_target.png?raw=true)

**Considering we want to allow a person to connect via ssh to this host running postgres to perform some kind of report generation**

![Image](images/add_hostset_to_target.png?raw=true)

### Get the target ID 

![Image](images/get_id_tgt.png?raw=true)


### Connect to the target

```
BOUNDARY_ADDR='http://<YOUR-ELB-DNS-NAME>:9200' \
  boundary connect ssh --username ubuntu -target-id <target-id> -token <token-from-previous-steps>
```

### How to pass the usual flags that we are used to commands like boundary connect ssh? 

just use `--` at the end, like shown below:

```
BOUNDARY_ADDR='<YOUR-ELB-DNS-NAME>:9200' \
  boundary connect ssh --username ubuntu -target-id <target-id> -token <token-from-previous-steps> -- -i ~/.ssh/my-private-key
```
