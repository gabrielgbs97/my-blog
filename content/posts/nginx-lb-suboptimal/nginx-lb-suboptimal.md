---
title: "¿Deberías desplegar NGINX como Proxy Inverso?"
date: 2024-27-08T14:30:00+02:00
kind: page
draft: false
#tags: ["linux", "containers"]
---

NGINX se ha convertido en el servidor HTTP más desplegado del mundo, conocido por su eficiencia y capacidad para manejar un gran número de conexiones concurrentes. Desarrollado por Igor Sysoev y lanzado en 2004, NGINX no solo actúa como un servidor web, sino también como un proxy inverso y un balanceador de carga. Su arquitectura basada en eventos y su bajo consumo de recursos lo han hecho popular en una variedad de entornos. Sin embargo, cuando se trata de despliegues específicos de balanceo de carga y proxy inverso, es importante considerar si NGINX es realmente la mejor opción.

Existen muchas publicaciones por la red que incluyen NGINX en varios despliegues de Balanceo de Carga y/o Proxy Inverso, o como una de las opciones fuertemente recomendadas. Sin acritud, creo que sus autores invierten más tiempo en mejorar su _copywriting_ que en plantear buenas arquitecturas.

Si eres arquitecto de soluciones se software libre, la inclusión de NGINX para cargas LB/RP creo que no es lo más adecuado.

> "Para un hombre con un martillo, todo parece un clavo"
> Abraham Maslow

Por ejemplo, si comparamos con HAProxy (Community) verás fácilmente que éste último tiene características superiores:

- Comprobación activa de salud de servidores (active health check). Solo por esta carencia ya descartaría NGINX en muchos escenarios de LB/RP.
- API datos/configuración (Dataplane API)
- Persistencia de sesiones HTTP mediante cookie (pre-existente o no)
- Tabla de persistencia de sesiones HTTP (para clientes http tipo máquina)
- Métricas para Prometheus/InfluxDB/Elastic
- Más opciones avanzadas de balanceo/proxy en https://docs.haproxy.org/3.0/configuration.html

Todas las características anteriores solo estan en NGINX Plus, un producto comercial de @F5 Inc derivado de NGINX.

Si queremos mezclar cargas de trabajo de servidor HTTP y LB/RP en una única solución existe el clásico Apache HTTPD. Tiene alguna de las carencias anteriores pero al menos cuenta con comprobación activa de servidores.

Otras alternativas para LB/RP más modernas pueden ser: Envoy, Traefik y Caddy. Pero siempre hay que contrastar nuestras necesidades con la matriz de características de cada solución.

En definitiva, cada herramienta tiene su propio foco. Aunque NGINX sea (casi) excelente como servicio HTTP y con un uso comprobado y extensivo, no significa que sea idóneo para cargas de LB/RP. También comentar que es una lástima que F5 Inc. incluya funcionalidades tan críticas detrás de una barrera de pago.
