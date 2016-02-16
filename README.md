## High availability Drupal in AWS using Ansible
-----------------------------------------------

This is a demonstration deploying Drupal into Amazon Web Services using multiple Availability Zones for high availability.
The playbooks provision several different AWS resources and deploys the latest development Drupal 7.X version from Github.

I'm fairly new to Ansible so feedback and pull requests are welcome.

### Prerequisites

Tested with: Ansible 2.0.0.2 and boto 2.39.0

Create an IAM user with Admin permissions as described [here](http://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html).

Configure boto with the created IAM user as described [here](http://boto.readthedocs.org/en/latest/boto_config_tut.html).

Create a SSL certificate with AWS Certificate Manager as described [here](http://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request.html).
NOTE: Record the ARN from this certificate for use with `ssl_cert_id` below.

### Setup

Clone source and cd:

    git clone https://github.com/soccerties/Drupal-AWS-Ansible.git
    cd Drupal-AWS-Ansible

Uncomment and add desired credentials to group_vars/vault.yml:
```yml
---
vault_db_admin_password: <admin_db_pass>
vault_drupal_db_password: <drupal_db_pass>
vault_drupal_admin_password: <drupal_admin_user_pass>
```
Optional but recommended, encrypt vault.yml file:
```bash
ansible-vault encrypt group_vars/vault.yml
```

Modify group_vars/all as desired. `unique`, `ssl_cert_id`, and `drupal_hash_salt` need to be defined:
```yml
---
# unique all lowercase string appended to S3 bucket for uniqueness
# e.g. unique: 5av2ixmk2hf
unique:

...
# SSL cert ID created manually with AWS Certificate Manager
# e.g. ssl_cert_id: "arn:aws:acm:us-east-1:55555555555:certificate/55555555-5555-5555-5555-555555555555"
ssl_cert_id: 

...
# e.g. dupal_hash_salt: "K1pBc5AV2ixmk2hfz6X4WHlQq6rc9hRbsAp560MjRJrwlrbPw7bqIOlT-ji6ZRL645gomyXqSQ"
drupal_hash_salt: 
```

### Usage

NOTE: If you've encrypted your vault.yml file. Include ``--ask-vault-pass`` to all playbook commands. Or alternatively setup a password file as described [here](http://docs.ansible.com/ansible/playbooks_vault.html#running-a-playbook-with-vault).

Run the provision.yml playbook to setup your AWS resources:
```
ansible-playbook provision.yml
```

NOTE: The provision.yml playbook does several things including creating a Multi-AZ RDS instance. These instances take several minutes before they become available. So make sure the drupal-dev RDS instance is available before proceeding.

Run the deploy.yml playbook to setup and install Drupal:
```
ansible-playbook deploy.yml
```

After execution the playbook will display the hostname of the AWS Elastic Load Balancer.

Create a CNAME record on the domain you used when creating the SSL certificate with AWS Certificate Manager pointed to the DNS name of the ELB.

That's it. You should now have a high availability Drupal site. Visit your domain name and login with the admin credentials you defined in the variable files.

If you'd like to deploy Drupal again with a fresh database. You can pass in a variable to drop the current database during the deploy::
```bash
ansible-playbook deploy.yml --extra-vars "{'drop_database': True}"
```

# TODO
* Replace S3 with GlusterFS or similar solution to avoid performance and "eventual consistency" issues with S3 buckets.
* Add rolling deployment removing and adding instances to ELB as they're updated
* Use Capistrano like deployment with https://github.com/ansistrano/deploy
