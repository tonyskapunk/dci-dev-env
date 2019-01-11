# Distributed CI environment for development

This repository is used to store the configuration of docker
compose in order to provide an environment for development.

## Getting started

For running dci in docker compose follow those steps:

 * clone this repository
 * bootstrap dci-dev-env, run `./utils/bootstrap.sh`
 * install docker-compose and git-review if you want to contribute,
   you can install those requirements by simply typing:
   `pip install -U -r requirements.txt`
 * launch the environment `docker-compose -f dci.yml up -d`

Now the environment is up and running, you can attach containers in order to
run parts of the applications:

    docker exec -it <container-name> bash

e.g.

    docker exec -it dci-dev-env_api_1 bash

## Containers

Here is the list of containers for running the application:

 * **dci_db**: contains the postgresql database, it is started by default and
   serve the database on localhost port 5432.
 * **dci_api**: contains the api of the application, it must be started manually
   The API is served on localhost port 5000.
 * **dci_ui**: contains the web app of dci. The web application is served on localhost port 8000.
 * **dci_doc**: helper for building the documentation of the project.
 * **dci_client**: contains the python-dciclient.
 * **dci_ansible**: contains the dci-ansible code.
 * **dci_keycloak**: keycloak server for SSO.
   Please use `localhost` and not `127.0.0.1` because `keycloak` valid domain is set to `localhost`.
 * **dci_swift**: swift server as storage backend for the api.


### api container

You can initialize or reinitialize the database by running db_provisioning script:

    docker exec -it dci-dev-env_api_1 ./bin/dci-dbprovisioning

### client container

This container allows one to run the python-dciclient within it.

This container is special in several ways compares to the others:

 * It runs systemd
 * It runs an sshd daemon (root/root)

To initialize this container you need to perform some operations:

 * Install the dciclient library, as well as the agents and feeders:

    cd /opt/python-dciclient && pip install -e .
    cd /opt/python-dciclient/agents && pip install -e .
    cd /opt/python-dciclient/feeders && pip install -e .


 * Create a local.sh file with the following credentials and source it:

```shell
export DCI_LOGIN=admin
export DCI_PASSWORD=admin
export DCI_CS_URL=http://$API_CONTAINER_IP:5000
```

Note: The $API_CONTAINER_IP can be obtained by running

    docker inspect --format '{{ .NetworkSettings.IPAddress }}' <container-id>

### keycloak container

This container allows to use SSO based authentication.

* It's provisioned by default by the following:

    - The user admin: username=admin, password=admin
    - A realm 'dci-test'.
    - A client 'dci-cs'.
    - A lambda user within the dci-test realm: username=dci, password=dci

* The client dci-cs is configured to allows OIDC Implicit flow and Direct Access protocols.

    - To use the implicit flow the browser should go to the following link:
      http://localhost:8180/auth/realms/dci-test/protocol/openid-connect/auth?\
      client_id=dci-cs&\
      response_type=id_token&\
      scope=openid&\
      nonce=ab40edba-9187-11e7-a921-c85b7636c33f&\
      redirect_uri=http://localhost:5000/rh-partner

      The 'nonce' value must be randomly generated. This link will redirect the browser to
      Keycloak SSO login page. Once the user is authenticated then Keycloak will redirect
      to the URL indicated by the 'redirect_uri' parameter in the following way:
      http://localhost:5000/rh-partner#\
      id_token=eyJhs ... EPscxhRMuEdA&\
      not-before-policy=0

      The id_token parameter correspond to the Json Web Token (jwt) generated by Keycloak
      and will be used to authenticated the client on the server api.

    - To use the Direct Access protocol, for instance from the CLI:

      $ curl -d "client_id=dci-cs"\
      -d "username=dci" -d\
      "password=dci"\
      -d "grant_type=password"\
      http://localhost:8180/auth/realms/dci-test/protocol/openid-connect/token

      This will get a JWT and will be used to authenticated the client on the server api.

### ansible container

You can use the Ansible container to run tox:

    docker exec -it dci-dev-env_ansible_1 tox

and the functional tests:

    docker exec -it dci-dev-env_ansible_1 bash -c 'cd tests; ./run_tests.sh'

### swift container

If you want to enable the swift storage for the api:
    docker-compose -f dci.yml -f dci-swift.yml build
    docker-compose -f dci.yml -f dci-swift.yml up -d

Swift is exposed on the non-standard port 5001. If you want to interact with it, you can use
your local `swift` client.

    source dci-swift/openrc_dci.sh
    swift upload fstab /etc/fstab
    swift list

### doc container

This container generates dci documentation.
If you want to generate the dci documentation run the container:

    docker-compose -f dci.yml -f dci-extra.yml run doc
