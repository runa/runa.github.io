---
layout: post
title:  "ADK Hackathon"
---

# El Hackathon de Agent Development Kit


Como escribí antes, todavía no quiero hacer una empresa de IA, por eso el [hackathon de ADK](https://googlecloudmultiagents.devpost.com/?utm_source=gamma&utm_medium=email&utm_campaign=FY25-Q2-NORTHAM-GOO34096-onlineevent-su-ADKHackathon&utm_content=innovators-newsletter&utm_term=-) me vino bien: vamos a jugar con esto, vamos a hacer algo que "parezca en serio" #adkhackathon

## Agent Development Kit
[ADK es una lib open source de google](https://google.github.io/adk-docs/) para 'hacer agentes'. Está bueno que muchos casos están contemplados entonces rápidamente se pueden hacer cosas:
* agentes que tienen [sub-agentes](https://google.github.io/adk-docs/agents/multi-agents/); que pueden ser llamados 'cuando hagan falta' [o secuencialmente o en paralelo.](https://google.github.io/adk-docs/agents/workflow-agents/)
* [herramientas de diversas](https://google.github.io/adk-docs/tools/) fuentes: podés definir herramientas con funciones en código ó importar las herramientas que ofrecen servers MCP ó usar herramientas de LangChain ó usar herramientas built in (`google_search`, la mejor de todas)
* [callbacks donde meter código](https://google.github.io/adk-docs/callbacks/) (para poder decir: "esta parte la hace la LLM, pero después, el resultado, lo vamos a filtrar a traves de este pedazo de código viejo y confiable")
* artifacts: archivos binarios que generan los agentes y que le van a interesar al usuario, etc.
* [deployment en GCP](https://google.github.io/adk-docs/deploy/): yo me siento cómodo en GCP, así que tener una manera sencillita de deployear esto me puso contento

## Qué hice
Como quería probar la parte 'desconocida', decidí atacar un problema que ya conocía, que sé que APIs hay disponibles, que conozco como usan, etc.

Hice un "real estate AI agent" que hace una tarea que los agentes inmobiliarios aprecian mucho: busca comparables.

Para tasar una propiedad es interesante encontrar propiedades 'similares' que esten actualmente publicadas o que hayan estado en el pasado cercano. 

En 'similares' está el secreto: muchas veces no hay algo _idéntico_ entonces hay que buscar cosas que sean parecidas; y ahí está el arte: si dos casas tienen la misma cantidad de dormitorios, baños y superficie similar, pero están a kilómetros de distancia; son comparables o no? Si dos casas están pegadas PERO son muy distintas, son comparables o no?

## Cómo lo hice
Hay una orden que es bastante secuencial y donde no hay mucho espacio para la 'creatividad': 
1. En base a una dirección, hay que georeferenciarla y normalizarla (usando el MCP de Google Maps)
2. Buscar datos de la propiedad en Google (usando una herramienta built in llamada `google_search`)
3. Buscar los comparables (usando unas herramientas definidas con código que llaman a un API de búsqueda de propiedades)
4. Generar un reporte.

Para esto, hay un coso muy piola llamado SequentialAgent que permite llamarlos en orden:
```python
root_agent = SequentialAgent(
  name="real_estate_agent_comparables",
  description="A real estate agent who can find comparables for a base property",
  sub_agents=[gmaps_agent, google_search_agent, api_search_agent, report_writer_agent],
)
``` 

El agente `gmaps_agent` importa las herramientas de un server [MCP de Google Maps](https://github.com/modelcontextprotocol/servers/tree/2025.4.24/src/google-maps) y le agrega algunas [instrucciones](https://google.github.io/adk-docs/agents/llm-agents/#guiding-the-agent-instructions-instruction) tipo `instruction='Georeference and normalize addresses',`

El `google_search_agent` es lo que me parece que le da un valor *enorme* a ADK: hace búsquedas en google y extrae datos de los documentos resultantes.  Lo uso especialmente para buscar datos importantes sobre las propiedades que en general no están tabulados. Es lo que hace un humano: ni bien le das una dirección, la googlea y mira "a ver que onda"
```python
from  google.adk.tools  import  google_search

root_agent = Agent(
name="basic_search_agent",
model="gemini-2.0-flash",
description="Agent to find property information using Google Search.",
instruction="""Search websites for information about the base property.
For the property sited EXACTLY in the address; ignore similar addresses.
For each property datum, please cite the source.
Find last sold date and price; property features (like pool, garden, shed; renovations done or needed, etc).
Other information that could be useful: robberies, murders, fires,etc.
Don't assume anything, only return facts""",
output_key="website_data",
tools=[google_search]
)
```

El `api_search_agent` está armado de funciones que van a pegarle al API; algunas cosas que encontré:
* El API que uso soporta OData Filters (yo no sabía que era esto, aparentemente es un standard para hacer queries a una DB, como un WHERE de SQL). Se me ocurrió que por ahí la LLM (Gemini en este caso) iba a saber construir el query, así que se lo dejé a disposición: decidió no usarlo. En la [documentación de Vertex recomiendan usar parámetros sencillos y concretos.](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/function-calling#write_clear_and_detailed_function_names_parameter_descriptions_and_instructions)
* Así que terminé creando muchos parámetros de `bathrooms_min` y `bathrooms_max`; que terminan escribiendo el query de OData que usa luego el API para filtrar.

## El Deploy
Para deployearlo, lo hice a Cloud Run; que en principio parecía muy práctico; pero me di cuenta que el Dockerfile que escribía no permitía incorporar dependencias del OS (ej, yo necesitaba Node para poder correr el MCP de Google Maps). [Hice un PR para esto](https://github.com/google/adk-python/pull/1524).


## Lo que (todavía) no usé
* Hay toda una parte de APIs en real-time; entiendo que para audio y video que dejé sin usar
* Hay un servicio de GCP llamado AgentEngine que se usa para deployear agentes (yo hice deploy en Google Cloud Run)

## Las preguntas que me quedan

Por qué cuando pensamos en "agentes" la UI que nos aparece es la de "un chat"? Es esta la mejor UI o es que todavía no se nos ocurrió algo mejor? 

No sería interesante tener "una AI que haga UIs" para que arme la mejor UI posible para:
* la interacción a desarrollar
* el tipo de usuario
* el dispositivo del usuario

Por ejemplo, cuando una AI te pregunta si querés hacer "A" o "B", yo me siento medio tonto escribiendo "B". Preferiría que ahí me muestre un botón; aunque es verdad que uno podría contestarle: "Quiero que hagas B pero que tomes esta idea de A que me parece útil" 
