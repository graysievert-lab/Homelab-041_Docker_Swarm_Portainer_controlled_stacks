# Repository to store Docker Swarm stacks controlled by Portainer

This repo contains all stacks deployed on the Docker Swarm cluster except for those described in [Homelab-040_Docker_Swarm](https://github.com/graysievert-lab/Homelab-040_Docker_Swarm)

Stacks require initial manual registration in Portainer. Subsequent re-deploys could be automated via the `GitOps updates` feature of Portainer business edition and GitHub Actions.

Conventions for GitHub Actions automation:

- Stack’s folder should be named with the `stack-` prefix. All other folders are ignored.
- File `webhook` should exist in the stack's folder. The file should contain one line with the URL of a Portainer webhook for this stack.
- GitHub Actions workflows in this repo require a self-hosted runner, with a network interface from which it can reach out to a Portainer instance at `portainer.swarm.lan`. Deployment of such a runner described in [Homelab-050_GitHub_Selfhosted_Runners_lasting](https://github.com/graysievert-lab/Homelab-050_GitHub_Selfhosted_Runners_lasting)

## Example

Taking stack `stack-100-test` as an example let's walk through the typical deployment process.

### Stack Features

Stack `stack-100-test` deploys a  `nginx` service from the latest available image.

The following directories and files are mounted to the container:

- directory `static` in stack's folder is mapped to container's `/usr/share/nginx/html`
- html file stored on the docker host is mapped to the container's `/usr/share/nginx/html/page.html`

NOTE: For this example to work it is mandatory to enable `Relative path volumes` during stack registration. This feature allows binding files and directories using [relative paths within the repository](https://docs.portainer.io/advanced/relative-paths).

Also, `nginx` service registers in Traefik, which will make it available at `https://100-test.swarm.lan/`.

### Stack deployment

To deploy this stack:

- Go to `Stacks` in Portainer dashboard and `+ Add Stack`
- Name: `100-test`
- Build method: `Repository`
- Repository URL: `https://github.com/graysievert-lab/Homelab-041_Docker_Swarm_Portainer_controlled_stacks`
- Compose path: `/stack-100-test/compose.yaml`
- GitOps updates: `ON`
- Mechanism: `Webhook`  (copy URL, we need it later)
- Force redeployment: `ON`
- Enable relative path volumes: `ON`
- Network filesystem path: `/mnt/stacks`

Deploy the stack and visit `https://100-test.swarm.lan/`.

Now create the file `webhook` in the stack's folder and paste there the webhook URL. Commit changes.

Push to the `master` branch triggers a workflow, and as a result, a self-hosted runner should make a POST request to the webhook URL.

Check stack details in Portainer to verify the `Updated` timestamp.
