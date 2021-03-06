# subscription-watch-operator

## About

Operator to obtain OCP usage data and upload it to subscription watch. The operator utilizes [ansible](https://www.ansible.com/) to collect usage data from an OCP cluster installation.

You must have access to an OpenShift v.4.3.0+ cluster..

## Development

This project was generated using Operator SDK. For a more in depth understanding of the structure of this repo, see the [user guide](https://github.com/operator-framework/operator-sdk/blob/master/doc/ansible/user-guide.md) that was used to generate it.

This project requires Python 3.6 or greater and Go 1.13 or greater if you plan on running the operator locally. To get started developing against `subscription-watch-operator` first clone a local copy of the git repository.

```
git clone https://github.com/chambridge/subscription-watch-operator.git
```

Developing inside a virtual environment is recommended. A Pipfile is provided. Pipenv is recommended for combining virtual environment (virtualenv) and dependency management (pip).

To install pipenv, use pip

```
pip3 install pipenv
```

Then project dependencies and a virtual environment can be created using

```
pipenv install --dev
```

**NOTE:** For Linux systems, use `pipenv --site-packages` or `mkvirtualenv --system-site-packages` to set up the virtual environment. Ansible requires access to libselinux-python, which should be installed system-wide on most distributions.

To activate the virtual environment run

```
pipenv shell
```

Next, install the Operator SDK CLI using the following [documentation](https://github.com/operator-framework/operator-sdk/blob/master/doc/user/install-operator-sdk.md).

Finally, install [jq](https://stedolan.github.io/jq/download/).

## Testing

We utilize [molecule](https://molecule.readthedocs.io/en/latest/) to test the ansible roles.

```
make test-local
```
## Setup for running the Operator

First, switch to the OpenShift project called `openshift-metering`. This is where we are going to deploy our Operator and its dependencies:

```
oc project openshift-metering
```
### Authentication setup

Decide if you are going to use `basic` authentication or `token` authentication to upload the Subscription Metric Reports to Ingress.

#### Token authentication

The default authentication method is token authentication. Inside of the cluster in the `openshift-config` namespace, there is a secret called `pull-secret` which has a `data` section that contains a `.dockerconfigjson`. In the `.dockerconfigjson` you need to grab the `auth` value associated with `cloud.openshift.com`. You can grab this token and configure the authentication secret by running the following command:

```
make setup-auth
```

#### Basic authentication

To use basic authentication, you must provide your username and password values for connecting to  [cloud.redhat.com](https://cloud.redhat.com/). To configure your authentication secret with your username and password, run the following:

```
make setup-auth username=YOUR_USERNAME password=YOUR_PASSWORD
```

Note: `YOUR_USERNAME` is your unencoded username & `YOUR_PASSWORD` is your unencoded password.

### Operator Configuration

The `subs-watch-operator` requires you to provide your cluster ID and reporting operator token name. Additionally, if you are using basic authentication, you must also specify that the authentication type is `basic`. To configure your operator, run the following:

```
make setup-operator clusterID=CLUSTER_ID report_token_name=REPORTING_OPERATOR_TOKEN_NAME authentication=basic
```

Note: If you are using `token` authentication, you can disregard the authentication parameter. There are two additional optional parameters, `validate_cert` and `ingress_url`, which you can learn more about by running `make help`.

### Creating the dependencies

OpenShift needs to know about the new custom resource definitions that the operator will be watching. Make sure that you are logged into a cluster and run the following command to deploy both the `SubscriptionWatch` and `SubscriptionWatchData` CRDs, the authentication secret, service account, role, and role binding to the cluster:

```
make deploy-dependencies
```

## Building & running the operator outside of a cluster

When running locally, we need to make sure that the path to the role in the `watches.yaml` points to an existing path on our local machine. Edit the `watches.yaml` to contain the absolute path to the setup and collect roles in the current repository:

```
# initial setup steps
- version: v1alpha1
  group: subscription-watch.openshift.io
  kind: SubscriptionWatch
  role: /ABSOLUTE_PATH_TO/subscription-watch-operator/roles/setup

# collect the reports
- version: v1alpha1
  group: subscription-watch-data.openshift.io
  kind: SubscriptionWatchData
  role: /ABSOLUTE_PATH_TO/subscription-watch-operator/roles/collect
  reconcilePeriod: 360m
```

Now, run the operator locally:

```
make run-locally
```

You will see some info level logs about the operator starting up. The operator works by watching for a known resource and then triggering a role based off of the presence of that resource.

## Building & running the Operator as a pod inside the cluster
Below are flows for the main development team and for external contributors.

### Development team options
If you have pushed your changes to a branch within the repository then an associated image branch will have been built in [quay.io/chambrid/subscription-watch-operator](https://quay.io/chambrid/subscription-watch-operator). You need only specify the branch you want to deploy the operator with using the following command:

```
make deploy-operator-dev-branch branch=$GIT_BRANCH
```

### Contributor options
To build the subs-watch-operator image and push it to a registry, run the following where `QUAY_USERNAME` is your quay username where the image will be pushed:

```
make build-operator-image username=$QUAY_USERNAME
```
Under the quay repository settings, make sure that you change the `Repository Visibility` to public.

OpenShift deployment manifests are generated in deploy/operator.yaml. The deployment image in this file needs to be modified from the placeholder REPLACE_IMAGE to the previous built image. To correctly SED replace the image and deploy the Operator, run the following where `QUAY_USERNAME` is the username under which the image has been pushed:

```
make deploy-operator-quay-user username=$QUAY_USERNAME
```

### Validating Deployment

Verify that the subs-watch-operator is up and running:

```
oc get deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
subs-watch-operator
```

In order to see the logs from the operator deployment you can run:

```
oc logs -f deployment/subs-watch-operator --container ansible
oc logs -f deployment/subs-watch-operator --container operator
```

Note: If you need to redeploy the operator, but do not need to sed replace the `operator.yaml` you can run:

```
make deploy-operator
```

## Kicking off the roles

The setup role is going to create the reports defined in `roles/setup/files` using the namespace defined inside of `roles/setup/defaults/main.yml`. The default is `openshift-metering`.

To start the setup and collect role, the associated custom resource in the `watches.yml` has to be present. To deploy both the `SubscriptionWatch` and `SubscriptionWatchData` custom resources, run the following:

```
make deploy-custom-resources
```


## Running Ansible locally for development

When developing and debugging roles locally, it can be quicker to run via Ansible than through the Operator.

At the top level directory, create a `playbook.yml` file:

```
---
- hosts: localhost
  roles:
    - setup
```

The above example points to the setup role but can be modified to point at any role. Use the following command to run the playbook:

```
ansible-playbook playbook.yml
```
This should show you the same output as if the role was being ran inside of the Operator. Once you are satisfied with the output of your role, test it by running the Operator locally.


## Cleaning up resources

After testing, you can cleanup the resources using the following:

```
make delete-operator
make delete-dependencies-and-resources
make delete-metering-report-resources
```

## Community Operator Release Process

To release a new version of the `subs-watch-operator`, you must first update the bundle with the release changes and do a pull request against the [community-operators repo](https://github.com/operator-framework/community-operators).

1. To update the bundle, view the operator-sdk olm-catalog [documentation](https://docs.openshift.com/container-platform/4.1/applications/operator_sdk/osdk-generating-csvs.html#osdk-how-csv-gen-works_osdk-generating-csvs) on generating and updating ClusterServiceVersions (CSVs).

2. After all changes to the operator have been merged into master, cut a release with the new version tag. This will kick start an quay image build with the new release version [here](https://quay.io/chambrid/subscription-watch-operator?tab=tags). Use this image to replace the previous image in the ClusterServiceVersion.

3. After generating the bundle, submit a pull request with the updated bundle to the `community-operators` repo. Before submitting your pull request make sure that you have read and completed the community [contributing guidelines](https://github.com/operator-framework/community-operators/blob/master/docs/contributing.md) and [checklist](https://github.com/operator-framework/community-operators/blob/master/docs/pull_request_template.md).