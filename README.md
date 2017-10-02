# Solution to [REA Systems Engineer practical task](https://github.com/rea-cruitment/simple-sinatra-app)

## Deployment

The code provided has been divided in two main components. One **that
provisions an EC2 instance** and another one that **deploys the sinatra web
application on it**.

Provisioning code must be executed first, and then web application deployment.
This second step can be safely run as many time as required for the same EC2 instance.

### EC2 provisioning:
* **Prerequisites and assumptions**.
  * Software required:
    * [Ansible](https://www.ansible.com/) >=2.4
    * Python 3 (with both [boto3](https://github.com/boto/boto3) and
      [botocore](https://github.com/boto/botocore) libraries installed)
  * [AWS](https://aws.amazon.com/) public cloud resources:
    * An active AWS account.
    * AWS Web Management Console access (to delete the CloudFormation stack only)
    * IAM user with the following policy (with its access keys recorded in `aws/credentials/credentials_file`)
      ```
      {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Action": "ec2:*",
                    "Effect": "Allow",
                    "Resource": "*"
                },
                {
                    "Sid": "Stmt1506854584000",
                    "Effect": "Allow",
                    "Action": [
                        "cloudformation:*"
                    ],
                    "Resource": [
                        "*"
                    ]
                }
            ]
        }
      ```
    * Although it is trivial to modify this code to make it run in any region
      all tests have been run againg *eu-west-1*. 
    * An EC2 SSH keypair created in *eu-west-1* region named **sinatra-challenge-eu-west-1**.
* **Running the code** (from a Unix shell):
  * First load the AWS credentials into the shell environment.
    * `source export_variables.sh`
  * Execute the Ansible playbook:
    * `ansible-playbook launch-stack-playbook.yml`
    * This command will create minimal CloudFormation stack with one EC2 instance
      and a security group to later deploy the sinatra web application.
    * If everythig runs as expected, the command's output will be similar to this:
    ```
    TASK [Print out EC2 instance information.] *****************************
    ok: [localhost] => {
    "msg": {
        "AZ": "eu-west-1a",
        "InstanceId": "i-0ac90907b417f4483",
        "PublicDNS": "ec2-54-75-220-131.eu-west-1.compute.amazonaws.com",
        "PublicIP": "54.75.220.131"
    }
    ```
    * The `PublicDNS` of the newly launched instance will be required in the
      next step.
## Web app deployment.
This part will now configure the newly provisioned EC2 instance and deploy the
sinatra web application.
* **Prerequisites and assumptions**.
  * Same software as in the provisioning step.
  * Load EC2 keypair into the current shell ssh keychain.
    * e.g. `chmod 400 ~/Downloads/sinatra-challenge-eu-west-1.pem && ssh-add ~/Downloads/sinatra-challenge-eu-west-1.pem`
  * Update the `hosts` file with the `PublicIP` or the `PublicDNS` of the EC2 instance that was spun up
    in the previous step.
    * e.g.:
      ```
      [sinatra-challenge]
      ec2-54-75-220-131.eu-west-1.compute.amazonaws.com
      ```
* **Running the code** (from the same Unix shell in which the SSH key was
  loaded):
  * Execute the main Ansible playbook:
    * `ansible-playbook site.yml`
  * Once the execution of the playbook is complete the web app will be
    running and reachable via web browser at `http://ec2-54-75-220-131.eu-west-1.compute.amazonaws.com/`

# Design and implementation choices.
* For simplicity, it was decided to separate the AWS resource provisioning from
  the app deployment. This involves having to manually add the EC2 instance
  IP/FQDN to the `hosts` file wich is not ideal. There are ways of overcoming
  this, such as using [EC2 instance tagging together with Ansible to generate
  dynamic inventory files](http://docs.ansible.com/ansible/latest/intro_dynamic_inventory.html#example-aws-ec2-external-inventory-script), 
  but it was decided to leave that option out as it add more configuration
  complexity.
* AWS CloudFormation stacks a powerful and expressive way of spinning up any
  set (big or small) of AWS resources. The template used here was converted
  from JSON to YAML to make it easer to read to both creator and future
  maintainers.
* Ansible code has been broken down in different roles and tasks to make
  maintenance easier. It may look *overkill* for such a simple exercise, but it
  will pay off when the automation code needs to be extended or bugs need to be
  fixed.
* Instead of writing a new Ansible role to provision and configure the Apache web server, [the 
  one created an maintained by Jeff Geerling](https://galaxy.ansible.com/geerlingguy/apache/) is installed via
  **Ansible Galaxy**.
* Unfotunately, no tests have been included in this first iteration of the solution, due to the lack of time.

# Security choices.
* Unnecesary services (such as *NFS*) have been disabled in the EC2 instance.
* HTTPS (443/TCP) has been disabled in Apache via Ansible task since the exercise only asked for
  the app to be available via HTTP (80/TCP)
* A separate non-privileged system user (*sinatra-app*) is used to own the code
  checkout within the EC2 instance and to run the local sinatra web app on port 3000 TCP.
* Apache web server (running as a non-privileged user as well) has been deployed and configured to proxy the requests from port the
  public internet-facing port 80 to the internal one (3000 TCP)
* In the Apache web server, `ProxyRequests` is set to `Off` to avoid potential
  *open proxy* issues.
* Security groups' network access rules (0.0.0.0/0) are too open, mainly for
  SSH access. This was mostly a tradeoff for simplicity of the solution, since
  there are ways of, for example, getting the current public IP from which the
  CloudFormation code is being run and use it within the template.

# TODO and possible future improvements.
* Refactor the whole solution using an **immutable infrastructure** approach.
  I.e. *baking*, testing and deploying an AMI or Docker container with the app configured and deploy
  that via a CI pipeline.
* System and web app logs and metrics are lost once the CloudFormation stack is destroyed. That data should be saved for future analysis somewhere (S3) or pushed via API to services such as DataDog, CloudWatch or similar.
* Consider using other Ruby web servers, such as Phusion Passenger or Puma via
  a proper Apache or Nginx module.
