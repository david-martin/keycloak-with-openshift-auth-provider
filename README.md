# How to use your OpenShift credentials to sign into Keycloak

This article shows how to setup and configure a Keycloak instance to use OpenShift for authentication via Identity Brokering. This allows for Single Sign on between the OpenShift cluster and the Keycloak instance. The Keycloak instance will be running on the OpenShift cluster and leverage a ServiceAccount Oauth Client.

## Provisioning Keycloak to your OpenShift namespace

Use the below command to create the Keycloak resources in your OpenShift project.

```bash
oc process -f https://raw.githubusercontent.com/david-martin/keycloak-with-openshift-auth-provider/0.0.2/keycloak-with-openshift-auth-provider.yaml | oc create -f -
```

*IMPORTANT: This template is intended for demonstration purposes only.*

This will create the following resources:

* DeploymentConfig - Defines the Keycloak image to use and other Pod & container settings
* Service - Defines a Service in front of the Keycloak Pod
* Route - Exposes the Keycloak Service at an externally available hostname
* ServiceAccount - Defines a constrained form of Oauth Client in our namespace

For more info on the ServiceAccount Oauth Client, see https://docs.openshift.com/container-platform/3.7/architecture/additional_concepts/authentication.html#service-accounts-as-oauth-clients

After provisioning, the Keycloak service will be available at the exposed Route. Use the below command to get the route and sign in to the Administration Console.

```bash
oc get route keycloak --template "http://{{.spec.host}} "
```

You can sign in initially using the `admin` user and the generated password stored in the `KEYCLOAK_PASSWORD` environment variable.

```bash
oc env dc/keycloak --list | grep KEYCLOAK_PASSWORD
```

## Creating a new Realm

The `admin` user will be signed in to the `master` realm. This user has full control over the Keycloak instance. We can create a dedicated realm for our OpenShift project and allow OpenShift users to administer the realm. Only users who can access our OpenShift cluster will be able to sign in to Keycloak.

Create a realm, and name it after our OpenShift Project.

![create realm](create_realm_001.png?raw=true "Create Realm")

## Configuring Keycloak to use OpenShift for Identity Brokering

After creating the realm, the context should switch to the new realm. From the 'Identity Providers' menu, choose to 'Add provider...' and select 'OpenShift v3'. Fill in the below fields.

![openshift v3 provider](openshift_v3_provider_002.png?raw=true "Openshift v3 Provider")

### Client ID 

This field is the Oauth Client identifier in OpenShift. As we're using a ServiceAccount Oauth Client, the id will be in the below format:

```
system:serviceaccount:<project_id>:keycloak
```

For example, if our project had an id of `myproject`, the Client ID would be:

```
system:serviceaccount:myproject:keycloak
```

### Client Secret

The secret is stored as a token for the ServiceAccount in OpenShift. To retrieve the secret, execute the following:

```bash
oc sa get-token keycloak
```

### Base Url 

The Base Url is the OpenShift Master Url e.g. https://openshift.example.com.

*IMPORTANT: The OpenShift Master Url will need to have a trusted CA signed certificate for Keycloak to successfully call the oauth callback endpoint.*

### Default Scopes

These are the scopes to send to OpenShift when authorizing the user. As we're only interested in authentication, and not making modifications to OpenShift on behalf of users, we can just use the `user:info` scope.

After filling in all of the above fields, the Provider can be created.

## Giving your OpenShift user a Role in Keycloak

If you attempt to sign in to the realm now, any user who successfully signs in will *only* be able to manage their own account in Keycloak. To allow users to manage the realm, they'll need more permissions. There are 2 approaches to giving Users extra permissions:

* setting Default Roles/Groups for every user
* explicitly setting Roles per user

Explicit roles can be managed from the 'Users' menu after Users have signed in at least once.
However, we're going to set up Default Roles so Users have roles on first sign in.

To set a Default Role, choose the 'Default Roles' tab from the 'Roles' menu. Then choose the 'realm-management' Client Role and add the 'manage-realm' role to the 'Client Default Roles'. You may want to choose different or more restrictive roles depending on your requirements.

![realm roles](realm_roles_001.png?raw=true "Realm Roles")

## Trying it out

Try it out by navigating to the Realm Admin Console page in a new browser session or incognito window. The Url for the Realm Admin Console page can found in the 'Clients' menu as the Base URL for the 'security-admin-console' Client ID.

![realm client](realm_client_001.png?raw=true "Realm Client")

This will show a login screen with an 'Openshift v3' option. Choosing the 'Openshift v3' option should open an OpenShift login page.

![openshift v3 provider](openshift_v3_provider_001.png?raw=true "Openshift v3 Provider")

Login to OpenShift and you should eventually be redirected back to Keycloak and be able to manage the Realm. You may need to fill in some account details on first login.

## Where to go from here?

If you'd like to have more control over the permissions OpenShift users have in your Keycloak instance, you may want to remove the Default Roles. This would be particularly important if you don't know or trust all OpenShift users. In that case, you could remove the Default Roles, and only add specific Roles to users you trust. You could also create a Group with specific Roles to make it more manageable.