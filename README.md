# Virtual Cluster UI Extension
Rancher extension used to provision clusters with [k3k](https://github.com/rancher/k3k/)

Requires Rancher 2.11.x

## Installing released versions
1. Open the Rancher Manager UI and navigate to the 'Extensions' page
2. Use the three-dot menu in the upper right and select 'Manage Repositories'
3. Click 'Create' to add the repository
4. Configure the repository:
    * **name**: virtual-clusters-ui 
    * **target**: http(s) URL to an index generated by Helm 
    * **index URL**: https://rancher.github.io/virtual-clusters-ui
5. Click 'Create' 
6. Wait for the virtual-clusters-ui repository status to be Active
7. Go back to the 'Extensions' page and select the 'Available' tab
8. Find the virtual clusters card and click 'Install'
9. Select a version (or use the latest by default) and click 'Install'
10. Once the extension has finished installing, click the 'Reload' button that will appear at the top of the page.

To get started, navigate to the Cluster Management page, click 'Create' and select k3k.


## Running for Development
This is what you probably want to get started.
```bash
# Install dependencies
yarn install

# For development, serve with hot reload at https://localhost:8005
# using the endpoint for your Rancher API
API=https://your-rancher yarn dev
# or put the variable into a .env file
# Goto https://localhost:8005
```

## Updating @shell package
This is about updating the @shell package which is the base from rancher/dashboard
```bash
# Update
yarn create @rancher/update
```

## Building the extension for production
Bump the app version on `package.json` file, then run:
```bash
# Build for production
./scripts/publish -g 
# add flag -f if you need to overwrite an existing version


# If you need to test the built extension
yarn serve-pkgs
```
