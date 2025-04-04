# VSC Node deployment

This repository hosts the Docker Compose file necessary for deploying the VSC node.

### Setup

1. Install [Docker](https://docs.docker.com/get-docker/) and [Docker compose v2](https://docs.docker.com/compose/install/).

2. `git clone https://github.com/vsc-eco/vsc-deployment`
   Clone this repository as a regular user (not root/admin) to a desired location. It's crucial to ensure the Docker user has write permissions in the directory where you plan to initiate the Docker Compose file.

3. `docker compose run init`
   Initialize the configuration files

4. Edit the config file located at `./data/config/identityConfig.json` and be sure to add in your Hive username and active key. It is recommended that you use a dedicated Hive account like *myaccount.vsc* to run your node.

### Starting Up

1. `docker compose up -d`
   Start the Docker containers from within the `vsc-deployment` folder.
   This will add a GraphQL server on port 8080, a MongoDB instance on port 27021, and a libp2p connection on port 10720.

2. For real-time log observation, use `docker logs go-vsc-node -f`.

### Maintenance

The node is designed to self-update as necessary. However, on rare occasions, the deployment configuration may require manual updates not covered by automatic updates. Should such a situation arise, we will inform the community through our usual communication channels [discord](https://discord.gg/tm7YkW7A) and [twitter](https://twitter.com/vsc_eco).

You can disable automatic updates by setting the environment variable `AUTO_UPDATE` to _false_. However, we recommend to keep this feature enabled to ensure the node is always up-to-date. In our rapidly evolving ecosystem, it's crucial to keep the node updated for optimal network health.
