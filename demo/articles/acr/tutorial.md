
```
myacr=myContainerRegistry$RANDON


$ az acr create --resource-group myResourceGroup --name $myacr --sku Basic


$ az acr login –n myacr

$ docker pull myacr.azurecr.io/hello-world

$ docker push myacr.azurecr.io/hello-world 

$ docker rmi myacr.azurecr.io/hello-world

```