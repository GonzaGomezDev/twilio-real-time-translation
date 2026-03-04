#  Traducción de voz en tiempo real con Twilio y OpenAI Realtime
Esta aplicación demuestra cómo usar Twilio y la API Realtime de OpenAI para la
traducción bidireccional de voz entre una persona que llama y un agente de centro de contacto.

El asistente de IA intercepta el audio de voz de una de las partes, lo traduce y reproduce el audio en el idioma
preferido de la otra parte. El uso de la API Realtime de OpenAI ofrece una latencia significativamente menor,
lo que favorece una conversación de voz bidireccional natural.

Consulta [aquí](https://www.loom.com/share/71498319660943638e1ef2c9928bcd2a) una demo en video de la app de traducción en tiempo real en funcionamiento.

A continuación se muestra un diagrama de arquitectura de alto nivel de cómo funciona esta aplicación:
![Realtime Translation Diagram](/live-translation-readme-images/realtime-voice-translation-app.jpeg)

Esta aplicación utiliza los siguientes productos de Twilio en conjunto con la API Realtime de OpenAI, orquestados por esta aplicación de middleware:
- Voice
- Studio
- Flex
- Task Router

Se inician dos llamadas de Voice independientes, gestionadas por este servicio de middleware. Se le solicita a quien llama elegir su idioma preferido y luego la conversación
se encola para el siguiente agente disponible en Twilio Flex. Una vez conectada con el agente, este middleware intercepta el audio de ambas partes mediante
[Media Streams](https://www.twilio.com/docs/voice/media-streams) y lo reenvía a OpenAI Realtime para su traducción. El audio traducido
se reenvía después a la otra parte.

## Requisitos previos
Para ponerlo en marcha, necesitarás:
1. Una cuenta de Twilio Flex ([crear](https://console.twilio.com/user/unified-account/details))
2. Una cuenta de OpenAI ([registrarse](https://platform.openai.com/signup/)) y una [API Key](https://platform.openai.com/api-keys)
3. Un segundo número de teléfono de Twilio ([instrucciones](https://help.twilio.com/articles/223135247-How-to-Search-for-and-Buy-a-Twilio-Phone-Number-from-Console))
4. Node v20.10.0 o superior ([instalar](https://nodejs.org/en/download/package-manager))
5. Ngrok ([registrarse](https://dashboard.ngrok.com/signup) y [descargar](https://ngrok.com/download))

## Configuración local

Hay 3 pasos obligatorios para levantar la app localmente para desarrollo y pruebas:
1. Abrir un túnel de ngrok
2. Configurar la app de middleware
3. Configurar Twilio

### Abrir un túnel de ngrok
Al desarrollar y probar localmente, necesitarás abrir un túnel de ngrok que reenvíe solicitudes a tu servidor local de desarrollo.
Este túnel de ngrok se usa para los Media Streams de Twilio que envían audio de llamada hacia/desde esta aplicación.

Para iniciar un túnel de ngrok, abre una terminal y ejecuta:
```
ngrok http 5050
```
Una vez iniciado el túnel, copia la URL de `Forwarding`. Se verá algo así: `https://[your-ngrok-subdomain].ngrok.app`. La
necesitarás al configurar las variables de entorno del middleware en la siguiente sección.

Ten en cuenta que el comando `ngrok` anterior redirige a un servidor de desarrollo que corre en el puerto `5050`, que es el puerto predeterminado configurado en esta aplicación. Si
sobrescribes la variable de entorno `API_PORT` cubierta en la siguiente sección, deberás actualizar el comando `ngrok` en consecuencia.

Recuerda que cada vez que ejecutes `ngrok http`, se creará una URL nueva y tendrás que actualizarla en todos los lugares donde se referencia abajo.

### Configurar la app de middleware localmente
1) Clona este repositorio
2) Ejecuta `npm install` para instalar dependencias
3) Ejecuta `cp .env.sample .env` para crear tu archivo local de variables de entorno

Una vez creado, abre `.env` en tu editor de código. Debes configurar las siguientes variables de entorno para que la app funcione correctamente:
| Nombre de variable     | Descripción                                      | Valor de ejemplo          |
|-------------------|--------------------------------------------------|------------------------|
| `NGROK_DOMAIN` | La URL de reenvío de tu túnel de ngrok iniciado arriba | `[your-ngrok-subdomain].ngrok.app` |
| `TWILIO_ACCOUNT_SID` | Tu Account SID de Twilio, que puedes encontrar en la consola de Twilio. | `ACXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX` |
| `TWILIO_AUTH_TOKEN`  | Tu Auth Token de Twilio, que también se encuentra en la consola de Twilio.  | `your_auth_token_here`  |
| `TWILIO_CALLER_NUMBER`   | El número adicional de Twilio que compraste, **no** conectado a Flex. Se usa para el "tramo" de llamada del lado de quien llama. | `+18331234567` |
| `TWILIO_FLEX_NUMBER`   | El número de teléfono comprado automáticamente al aprovisionar tu cuenta Flex. Se usa para el "tramo" de llamada del lado del agente. | `+14151234567` |
| `TWILIO_FLEX_WORKFLOW_SID` | El Workflow SID de TaskRouter, aprovisionado automáticamente con tu cuenta Flex. Se usa para encolar llamadas entrantes con agentes de Flex. Para encontrarlo, en la consola de Twilio ve a TaskRouter > Workspaces > Flex Task Assignment > Workflows  |`WWXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX`|
| `OPENAI_API_KEY`              | Tu API Key de OpenAI             | `your_api_key_here`                 |

A continuación, variables de entorno opcionales con valores predeterminados que se pueden sobrescribir:
| Nombre de variable     | Descripción                                      | Valor predeterminado          |
|-------------------|--------------------------------------------------|------------------------|
| `FORWARD_AUDIO_BEFORE_TRANSLATION` | Pon `true` para habilitar el reenvío del audio hablado original entre participantes. Por ejemplo, si quien llama habla español, esto reproducirá el audio original en español para el agente antes de reproducir el audio traducido. Esta opción es útil en contextos de producción para minimizar silencios percibidos. No se recomienda en modo desarrollo cuando una sola persona está simulando simultáneamente a quien llama y al agente.     | `false`                 |
| `API_PORT`        | El puerto en el que se ejecuta tu servidor local.             | `5050`                 |

### Configuración de Twilio

#### Importar Studio Flow
Debes importar el Studio Flow incluido en el archivo [inbound_language_studio_flow.json](inbound_language_studio_flow.json) en tu cuenta de Twilio, y luego configurar el número de teléfono de Twilio orientado a quien llama para que use este Flow. Este Studio Flow manejará la llamada entrante inicial y presentará una IVR básica para que quien llama seleccione su idioma preferido para conversar con el agente.

En la consola de Twilio, ve a [Studio Flows](https://console.twilio.com/us1/develop/studio/flows?frameUrl=%2Fconsole%2Fstudio%2Fflows%3Fx-target-region%3Dus1) y haz clic en **Create New Flow**. Asigna un nombre al Flow, por ejemplo "Inbound Translation IVR", haz clic en Next, luego selecciona la opción **Import from JSON** y haz clic en Next.

Copia el contenido de [inbound_language_studio_flow.json](inbound_language_studio_flow.json) y pégalo en el cuadro de texto. Busca `[your-ngrok-subdomain]` y reemplázalo por el subdominio de tu túnel ngrok. Haz clic en **Next** para importar el Studio Flow y luego en **Publish**. 

El Studio Flow incluido reproducirá un mensaje pregrabado para quien llama solicitando seleccionar su idioma preferido:
1. Inglés
2. Español
3. Francés
4. Mandarín
5. Hindi

Puedes actualizar la lógica del Studio Flow para cambiar los idiomas que deseas soportar. Consulta [aquí](https://platform.openai.com/docs/guides/text-to-speech/supported-languages) para más información sobre los idiomas compatibles de OpenAI. 

#### Apuntar el número de quien llama al Studio Flow
Una vez importado y publicado tu Studio Flow, el siguiente paso es apuntar tu número entrante / del lado de quien llama (`TWILIO_CALLER_NUMBER`) a tu Studio Flow. En la consola de Twilio, ve a **Phone Numbers** > **Manage** > **Active Numbers** y haz clic en el número adicional que compraste (**no** el aprovisionado automáticamente por Flex).

En la configuración del número, cambia el primer desplegable **A call comes in** a **Studio Flow**, selecciona el nombre del Flow creado arriba y haz clic en **Save configuration**.
![Point Caller Phone Number to Studio Flow](/live-translation-readme-images/inbound-voice-number-webhook.png)

#### Apuntar el número del agente y el Workspace de TaskRouter al middleware
El último paso es apuntar el número del lado del agente (`TWILIO_FLEX_NUMBER`) y el Workspace "Flex Task Assignment" de TaskRouter a esta app de middleware. Esto es necesario para conectar la conversación con un agente de centro de contacto en Flex.

En la consola de Twilio, ve a **Phone Numbers** > **Manage** > **Active Numbers** y haz clic en el número de Flex aprovisionado automáticamente. En la configuración del número, cambia el primer desplegable **A call comes in** a **Webhook** y establece la URL como `https://[your-ngrok-subdomain].ngrok.app/outbound-call`, asegúrate de que **HTTP** esté en **HTTP POST**, y haz clic en **Save configuration**.
![Point Agent Phone Number to Middleware]/live-translation-readme-images(/flex-voice-number-webhook.png)

Asegúrate de reemplazar `[your-ngrok-subdomain]` por el subdominio asignado de tu túnel ngrok.

Luego, ve a **TaskRouter** > **Workspaces** > **Flex Task Assignment** > **Settings**, y configura **Event callback URL** como `https://[your-ngrok-subdomain].ngrok.app/reservation-accepted`, nuevamente reemplazando `[your-ngrok-subdomain]` por el subdominio asignado de tu túnel ngrok.

![Point TaskRouter Workspace to Middleware](/live-translation-readme-images/task-router-event-callback-url.png)

Finalmente, en **Select events**, marca la casilla **Reservation Accepted**.

![Select events > Reservation Accepted](/live-translation-readme-images/task-router-reservation-accepted.png)

### Ejecutar la app
Una vez instaladas las dependencias, configurado `.env` y Twilio correctamente, ejecuta el servidor de desarrollo con el siguiente comando:
```
npm run dev
```
### Probar la app
Con el servidor de desarrollo en ejecución, ya puedes comenzar a probar la app de traducción. Si quieres probarla por tu cuenta, simulando tanto al agente como a quien llama, recomendamos configurar `FORWARD_AUDIO_BEFORE_TRANSLATION` en `false` para no escuchar audio duplicado.

Para responder la llamada como agente, debes iniciar sesión en Flex Agent Desktop. La forma más sencilla es ir a [Flex Overview](https://console.twilio.com/us1/develop/flex/overview) y hacer clic en **Log in with Console**. Una vez cargado Agent Desktop, asegúrate de que tu estado de agente esté en **Available** usando el desplegable en la esquina superior derecha de la ventana. Esto garantiza que las tareas encoladas se enruten hacia ti.

Con tu teléfono móvil, marca `TWILIO_CALLER_NUMBER` y realiza una llamada (No marques `TWILIO_FLEX_NUMBER`). Deberías escuchar una indicación para seleccionar tu idioma deseado y luego conectarte con Flex. En Flex Agent Desktop, una vez seleccionado el idioma, deberías ver la llamada asignada a ti. Usa Flex para responder la llamada.

Una vez conectados, ya deberías poder hablar en un extremo de la llamada y escuchar el audio traducido por OpenAI en el otro extremo (y viceversa). Por defecto, el idioma del agente está configurado en inglés. La API Realtime traducirá audio desde el idioma elegido por quien llama hacia inglés, y el habla en inglés del agente hacia el idioma elegido por quien llama.

## Configuración de la API Realtime de OpenAI
### Actualizar instrucciones del modelo
Puedes actualizar las instrucciones usadas para prompting de la API Realtime de OpenAI en [`src/prompts.ts`](/src/prompts.ts). Ten en cuenta que hay dos conexiones separadas a la API Realtime: una para quien llama y otra para el agente. Esto permite mayor precisión y flexibilidad en el comportamiento del traductor para ambos lados de la llamada. Observa que `[CALLER_LANGUAGE]` se inserta dinámicamente en el prompt según la selección de idioma de quien llama durante la IVR inicial de Studio. El comportamiento predeterminado asume que el agente habla inglés.

Para cambiar el prompt de quien llama, actualiza `AI_PROMPT_CALLER`. Para el agente, actualiza `AI_PROMPT_AGENT`. A continuación se muestran las instrucciones predeterminadas usadas para traducción:

**Caller**
```
export const AI_PROMPT_CALLER = `
You are a translation machine. Your sole function is to translate the input text from [CALLER_LANGUAGE] to English.
Do not add, omit, or alter any information.
Do not provide explanations, opinions, or any additional text beyond the direct translation.
You are not aware of any other facts, knowledge, or context beyond translation between [CALLER_LANGUAGE] and English.
Wait until the speaker is done speaking before translating, and translate the entire input text from their turn.
Example interaction:
User: ¿Cuantos días hay en la semana?
Assistant: How many days of the week are there?
User: Tengo dos hermanos y una hermana en mi familia.
Assistant: I have two brothers and one sister in my family.
`;
```
**Agent**
```
export const AI_PROMPT_AGENT = `
You are a translation machine. Your sole function is to translate the input text from English to [CALLER_LANGUAGE].
Do not add, omit, or alter any information.
Do not provide explanations, opinions, or any additional text beyond the direct translation.
You are not aware of any other facts, knowledge, or context beyond translation between English and [CALLER_LANGUAGE].
Wait until the speaker is done speaking before translating, and translate the entire input text from their turn.
Example interaction:
User: How many days of the week are there?
Assistant: ¿Cuantos días hay en la semana?
User: I have two brothers and one sister in my family.
Assistant: Tengo dos hermanos y una hermana en mi familia.
`;
```