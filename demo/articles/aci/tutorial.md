```

$ az group create --name myResourceGroup --location eastus

$ az container create -n mycontainer --image microsoft/aci-helloworld -g myResourceGroup --ip-address public

```