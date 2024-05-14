# Como usar FunctionCalling

## 🚀 Introducción
Este proyecto utiliza la API de OpenAI para realizar consultas sobre el clima en diversas ubicaciones, aprovechando también la API de WeatherAPI para obtener datos meteorológicos precisos.

## 🛠️ Instalación
Antes de ejecutar el código, asegúrate de instalar las dependencias necesarias con npm:

```bash
npm install openai dotenv, openai
```

## ⚙️ Configuración
Crea un archivo `.env` en la raíz de tu proyecto y añade tu clave de API de OpenAI y WeatherAPI de la siguiente manera:

```makefile
OPENAI_API_KEY=tu_clave_de_api_aqui
WEATHER_API_KEY=tu_clave_de_api_aqui
```

## 📝 Uso
El código permite realizar consultas interactivas a OpenAI para obtener información climática de cualquier ubicación. A continuación se describen las funciones principales:

### `getCurrentWeather(location)`
Obtiene el clima actual de la ubicación especificada. La función realiza una petición a WeatherAPI y devuelve un JSON con la información del clima.

**Parámetros:**
- `location`: String que representa la ubicación de la que se desea obtener el clima.

### `runConversation(question)`
Realiza una conversación con OpenAI enviando una pregunta específica sobre el clima en una ciudad. Esta función gestiona la interacción con OpenAI para obtener y procesar la respuesta adecuada.

**Parámetros:**
- `question`: String con la pregunta que se desea realizar, como por ejemplo "¿Cuál es el tiempo en la Capital de la Salsa?".

## 📘 Ejemplo de Ejecución
Para ejecutar una consulta sobre el clima, utiliza la siguiente función en tu código:

```javascript
runConversation("¿Cuál es el tiempo en la Capital de la Salsa?")
    .then(response => console.log({ response }))
    .catch(err => console.log(err));
```
Este código realizará una petición a la API de OpenAI y, posteriormente, procesará y responderá con el clima actual de la ubicación especificada.

## 📚 Más Información
Para más detalles sobre las APIs utilizadas:
- [CallingFunction](https://platform.openai.com/docs/guides/function-calling/)

## Explicacion

```javascript
//Importo librerias necesarias. Previamente debo correr npm i openai, dotenv

import OpenAIApi from "openai";
import dotenv from "dotenv";
//Permite usar variables almanacenados en un archivo .env
dotenv.config();


//Creo una instancia de OpenAI con la API Key almacenada en el archivo .env
const openai = new OpenAIApi(
    { apiKey: process.env.OPENAI_API_KEY, }
);

//Función que obtiene el clima actual de una ubicación determinada
//Esto devuelve un .json con la información del clima
async function getCurrentWeather(location) {
    const response = await fetch(
        `https://api.weatherapi.com/v1/current.json?key=${process.env.WEATHER_API_KEY}&q=${location}&aqi=no`
    );
    const weatherInfo = await response.json();

    return JSON.stringify(weatherInfo);
}

//Función que ejecuta la peticion a OpenAI para obtener una respuesta a una pregunta
//Se le pasa como parametro la pregunta, en nuestro caso cual es el clima en X Ciudad
async function runConversation(question) {
    //Metodo para crear una peticion a OpenAI
    //Se le indica el modelo
    //messages: es un array de objetos que contiene los mensajes de la conversación
    /*
    En la API de OpenAI, el campo role indica el papel del participante en la conversación. Hay tres roles principales:
    1. user: Representa al usuario final, es decir, la persona que está interactuando con el modelo. Este es el rol de quien está haciendo preguntas o dando instrucciones.
    2. assistant: Representa al modelo de IA, como ChatGPT, que proporciona respuestas o ayuda al usuario.
    3. system: Representa el contexto inicial o las instrucciones de configuración que se proporcionan al modelo antes de que comience la conversación. Este rol se usa para definir el comportamiento y el tono del asistente.

    *En el caso de role: 'user', significa que el mensaje fue enviado por el usuario.
    */
    let response = await openai.chat.completions.create({
        model: "gpt-3.5-turbo-0613",
        messages: [{ role: "user", content: question }],
        //functions: En una llamada API, puede describir funciones y hacer que el modelo elija de forma inteligente generar un objeto JSON que contenga argumentos para llamar a una o varias funciones. 
        //La API de finalización de chat no llama a la función; en cambio, el modelo genera JSON que puedes usar para llamar a la función en tu código.
        functions: [
            {
                //Nombre de la funcion... Obviamente debe estar definida
                name: "getCurrentWeather",
                //Descripcion espeficica de la funcionalidad
                description: "Obtener el clima actual en una ubicación determinada",
                //Parametros que recibe la funcion, tipo de dato y descripcion
                //La descripcion puede ser un ejemplo de la funcion
                //Puede ser mas de un parametro y tiene la opcion de ser requerido (tambien un arreglo)
                parameters: {
                    type: "object",
                    properties: {
                        location: {
                            type: "string",
                            description: "The city, eg: Madrid, Barcelona",
                        },
                    },
                    required: ["location"],
                },
            },
        ],
        //El modelo puede generar un objeto JSON que contiene argumentos para llamar a una o varias funciones.
        /*
         Significa que el modelo tiene la libertad de decidir si debe llamar a una función y cuál función debe llamar, en función del contexto de la conversación.
            1. "none": Indica que el modelo no debe llamar a ninguna función.
            2. "auto": Permite al modelo decidir automáticamente si debe llamar a una función y cuál debe llamar, basándose en el contenido del mensaje del usuario y en las funciones disponibles.
        */
        function_call: "auto",
    });
    
    //Esta respuesta es un Json que OpenAI devuelve con:
    //el name de la funcion que se ejecuto
    //los argumentos que se van a requerir.
    //El arma en este caso el argumento: Location por que en el parameters-properties de la funcion se definio asi su nombre
    /*
        {
            role: 'assistant',
            content: null,
            function_call: {
                name: 'getCurrentWeather',
                arguments: '{\n  "location": "Cali"\n}'
            }
        }
        
}
    */
    let message = response.choices[0].message;
    // console.log("Let message is:",message);

    if (message) {
        //Recorro la respuesta y le asigno el nombre de la funcion a functionName
        let functionName = message.function_call.name;
        
        //Recorro la respuesta y le asigno los argumentos a parameters
        const parameters = JSON.parse(message.function_call.arguments);

        //Ejecuto la funcion getCurrentWeather con el parametro location
        let functionResponse = await getCurrentWeather(parameters.location);

        //Realizo una nueva peticion a OpenAI con la respuesta de la funcion

        let secondResponse = await openai.chat.completions.create({
            model: "gpt-3.5-turbo-0613",
            messages: [
                {
                    role: "user",
                    content: `
                        Responde en espanol en este formato: 
                        El pronóstico del tiempo en {location} es {tiempo general}
                        Temperatura: {temperatura en celsius}
                        Sensación de mucho calor: Sí | No
                        Viento: Ninguno | Poco | Mucho | Pelgroso
                        Paraguas: Sí | No
                        Icono: {url del icono}
                        Hora: {hora de la consulta}
                        `,
                },
                message,
                //`role: "function": Indica que el mensaje es una respuesta de una función. Este rol se utiliza para diferenciar entre mensajes de usuario, asistente y el resultado de una función.
                { role: "function", name: functionName, content: functionResponse },
            ],
        });

        return secondResponse.choices[0].message;
    }
}

runConversation("¿Cuál es el tiempo en la Capital de la Salsa?")
    .then((response) => console.log({ response }))
    .catch((err) => console.log(err));

```


