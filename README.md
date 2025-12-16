ola, ya tengo este pedo, el código de las apps está en ./apps, están subidas a dockerhub en mi cuenta: https://hub.docker.com/u/m0r4a

Todos los manifest y deployments y ese pedo están en ./k8s/, en teoría lo único que necesitas hacer es escribir `./deploy.sh` y se deployea todo el pedo y para testear solo ocupas hacer `./test.sh`. Para eliminarlo so haz `./deploy.sh rm`

> ![WARNING]
> NO LOS EJECUTES FUERA DE LA CARPETA DE k8s, no va a pasar nada pero no ajusté los paths a ese caso así que no lo hagas así xd, la documentación está medio culera pero tu eres un pilín, estoy seguro que le entiendes.

