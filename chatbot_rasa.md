# Desarrollo chatbot con Rasa y Python
## Instalación Entorno Virtual
Antes de comenzar es recomendable crear un entorno virtual, con el fin de evitar posibles conflictos de versiones.
Vamos al directorio en el que queramos crear el entorno virtual, por ejemplo "Documentos"
```
cd
cd Documents/
```
Instalamos python3.6
```
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt-get update
sudo apt-get install python3.6
```
Preparamos el entorno virtual
```
sudo apt-get install libpq-dev python-dev libxml2-dev libxslt1-dev libldap2-dev libsasl2-dev libffi-devsudo apt-get update
sudo apt-get -y upgrade
sudo apt-get install -y python3.6-venv
```
Nos situamos en la carpeta en cuestión, en este caso en "Documentos" y creamos un entorno virtual, por ejemplo "mi_entorno"
```
mkdir environments
cd environments
python3.6 -m venv mi_entorno
```
Para activar el entorno virtual escribiremos lo siguiente
```
source mi_entorno/bin/activate
```
Cuando queramos dejar de trabajar en él, simplemente escribiremos
```
deactivate
```
# Instalación RASA
Un chatbot básicamente está compuesto por dos componentes: 
* NLU (Natural Language Understanding): Es la parte que se encarga de comprender el mensaje del usuario.
* CORE: Decide qué contestar al usuario, en base al punto en el que se haya la conversación.
## Instalación RASA NLU
```
sudo apt-get install python3.6-dev
pip install twisted
pip install rasa_nlu
```
## Instalación Pipelines
El pileline define componentes que procesan un mensaje del usuario de manera secuencial con el fin de clasificar los mensajes de éste en intentos y de extraer las entidades (estos conceptos los trataremos en breve).
A continuación procedemos a instalar dos pipelines, spacy y tensorflow
```
pip install rasa_nlu[spacy]
python -m spacy download en_core_web_md
python -m spacy link en_core_web_md en
```
Vamos a instalar el pipeline de tensorflow
```
pip install rasa_nlu[tensorflow]
```
## Instalación RASA CORE
```
pip install rasa_core
```
# Construcción de nuestro chatbot
Antes de comenzar a desarrollar nuestro asistente, es crucial que queden claros los conceptos que voy a comentar a continuación
* Intent: Es lo que Rasa emplea para clasificar lo que el usuario quiere decir. Rasa NLU clasifica los mensajes del usuario en uno o varios intents de usuario.
* Entity: Trata de extraer la información valiosa que aporta el usuario durante la conversación.
* Action: Operaciones realizadas por el chatbot, ya sea pidiendo más detalles para obtener todas las entidades, integrando con algunas API, consultando bases de datos, etc.
* Story: Define la interacción entre el usuario y el chatbot en términos de intención y acción tomada por el bot.
## Estructura del proyecto
Procedemos a crear la estructura básica del proyecto
```
mkdir mi_chatbot
cd mi_chatbot
mkdir data
cd data
touch nlu.md
```
En el fichero nlu.md escribiremos lo siguiente
```
## intent:saludo  
	- hello
	- hi
## intent:estado_positivo
	- I am good
	- I am well 
	- I am fine
	- I'm good
	- I'm well 
	- I'm fine
## intent:estado_negativo
	- I am sad
	- I'm sad
## intent:gracias
	- thanks
	- thank you
## intent:despedida
 	- bye
 	- good bye
## intent: afirmacion
 	- yes
 	- of course
## intent: negacion
 	- no
  ```
A continuación definimos el pipeline a través del cual se canalizarán estos datos y se hará la clasificación de intenciones y la extracción de entidades para el bot. Emplearemos "spacy_sklearn". Para ello, crearemos un archivo nlu_config.yml en el directorio del proyecto con el siguiente contenido:
```
language: "en"
pipeline: "spacy_sklearn"
```
Ahora vamos a proceder a entrenar el chatbot
```
python3 -m rasa_nlu.train -c nlu_config.yml --data data/nlu.md -o models --fixed_model_name nlu --project current --verbose
```
Se habrá guardado en el directorio ‘models/current/nlu’
De esta manera habremos logrado que nuestro bot sea capaz de entender lo que el usuario le dice (Rasa NLU). El siguiente paso será hacer que sea capaz de dar respuestas coherentes (Rasa CORE), con el fin de poder mantener el diálogo con el usuario.
## Administración del diálogo
Para el entrenamiento de los dialogos Rasa tiene cuatro componentes
### Domain (domain.yml)
Consta de cinco partes (intents, entities, slots, actions, y templates). Comentaré brevemente las dos que aún no había mencionado:
* Slot: Es la memoria de nuestro asistente virtual, es decir, actúan como un almacén de valores clave los cuales se pueden emplear para almacenar información que el usuario proporcionó (por ejemplo, su ciudad de origen), así como la información recopilada sobre el mundo exterior (por ejemplo, el resultado de una consulta a la base de datos).
* Template: Son los mensajes que el bot mandará al usuario.

Crearemos el archivo domain.yml en la carpeta raíz de nuestro proyecto
```
slots:
  category:
    type: text
intents:
- saludo
- estado_positivo
- festado_negativo
- estado_pregunta_positivo
- estado_pregunta_negativo
- gracias
- despedida
- afirmacion
- negacion
actions:
- action_restart
- utter_saludo
- utter_pregunta
- utter_ayudar
- utter_chiste
- utter_alegre
- utter_preguntar_ayuda_util
- utter_rpta_agradecimiento
- utter_despedida
- utter_default
templates:
  utter_saludo:
    - text: Hello
    - text: Hi
  utter_pregunta:
    - text: How are you?
    - text: How are you doing?
  utter_ayudar:
    - text: I hope this helps you. I'm going to tell you a joke 
  utter_chiste:
    - text: How did the computer programmer get out of prison? He used the escape key!
    - text: Some people, when confronted with a problem, think, 'I know, I'll use regex and then they have two problems
  utter_alegre:
    - text: I'm glad you're well
    - text: I'm glad you're good
    - text: I'm glad you're fine
  utter_preguntar_ayuda_util:
    - text: Did it help you?
  utter_rpta_agradecimiento:
    - text: You're welcome
  utter_despedida:
    - text: Bye!
    - text: Good bye!
    - text: See you!
  utter_default:
    - text: Sorry but I don't understand you
```
### Stories (stories.md)
Vamos a proceder a crear el fichero, dentro de la carpeta Data, con el siguiente contenido
```
$ cd mi_chatbot
$ cd data
$ touch stories.md
```
```
## retroceder
	- utter_default
## feliz 
* saludo
	- utter_saludo
	- utter_pregunta
* estado_positivo
	- utter_alegre
	- utter_despedida

## triste path 1
* saludo
	- utter_saludo
	- utter_pregunta
* estado_negativo
	- utter_ayudar
	- utter_chiste
	- utter_preguntar_ayuda_util
* negacion
	- utter_despedida

## triste path 2
* saludo
	- utter_saludo
	- utter_pregunta
* estado_negativo
	- utter_ayudar
	- utter_chiste
	- utter_preguntar_ayuda_util
* afirmacion
	- utter_alegre
	- utter_despedida

## despedida 1
* despedida
	- utter_despedida
```
### Policies (policy.yml)
Son las que deciden qué acción escoger en cada etapa de la conversación con el usuario. El archivo lo guardaremos en la raíz del proyecto
```
policies:
  - name: KerasPolicy
    epochs: 100
    max_history: 3
  - name: MemoizationPolicy
    max_history: 3
  - name: FallbackPolicy
    nlu_threshold: 0.1
    core_threshold: 0.2
    fallback_action_name: 'utter_default'
  - name: FormPolicy
  ```
  ## Entrenamiento del diálogo
  Una vez definidos los diferentes componentes, procederemos a entrenar el diálogo
  ```
  python3 -m rasa_core.train -d domain.yml -s data/stories.md -o models/dialogue -c policy.yml
  ```
  Después, hacemos correr el core
  ```
  python3 -m rasa_core.run -d models/dialogue -u models/current/nlu
  ```
  Una vez hecho esto ya podrás mantener conversaciones con el chatbot. Para finalizar la conversación, deberemos de escribir "/stop"
