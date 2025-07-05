# Scraper Serverless de Datos Inmobiliarios en AWS

Este proyecto implementa un sistema de web scraping automatizado, escalable y eficiente en costos para extraer datos de propiedades inmobiliarias del sitio web `infocasas.com.pe`. La arquitectura está completamente basada en servicios serverless de AWS, lo que garantiza un bajo mantenimiento y un modelo de pago por uso.

## 🎯 Objetivo del Proyecto

El objetivo principal es recopilar, enriquecer y almacenar información detallada de propiedades en venta (precio, ubicación, características, coordenadas geográficas, etc.) de forma periódica y automática. Los datos recopilados se almacenan en una base de datos NoSQL (Amazon DynamoDB), dejándolos listos para ser consumidos por otros servicios, como análisis de datos, APIs o paneles de visualización.

## 🏛️ Arquitectura de la Solución

La solución utiliza un patrón de diseño **"Fan-Out"** para desacoplar las tareas y permitir un procesamiento paralelo y resiliente. Esto evita los timeouts de las funciones Lambda y distribuye la carga de trabajo de manera eficiente.


### Flujo de Trabajo:

1.  **Disparo Programado**: Un disparador de **Amazon EventBridge** se activa según una programación definida (p. ej., una vez al día).
2.  **Orquestación**: El disparador invoca una función **AWS Lambda ("Orchestrator")**. Esta función es responsable de:
    *   Navegar por las páginas de listado del sitio web.
    *   Extraer los datos básicos de cada propiedad.
    *   Enviar cada propiedad como un mensaje individual a una cola de **Amazon SQS**.
3.  **Cola de Tareas**: La cola de **Amazon SQS** actúa como un búfer, almacenando de forma fiable todas las "tareas" (propiedades a procesar) pendientes.
4.  **Procesamiento en Paralelo**: La cola SQS activa una segunda función **AWS Lambda ("Worker")** por cada lote de mensajes. Gracias a la escalabilidad de Lambda, múltiples workers pueden ejecutarse en paralelo. Cada worker:
    *   Recibe un mensaje con los datos de una propiedad.
    *   Realiza una segunda petición al sitio web para visitar la página de detalle y extraer información adicional (en este caso, las coordenadas geográficas).
    *   Limpia y formatea el registro completo para que sea compatible con la base de datos.
5.  **Almacenamiento**: El worker guarda el registro final y enriquecido como un ítem en una tabla de **Amazon DynamoDB**.

### Ventajas de esta Arquitectura:
*   **Escalabilidad**: El sistema puede manejar desde unas pocas hasta miles de propiedades sin cambios en la arquitectura.
*   **Eficiencia en Costos**: Gracias a la capa gratuita de AWS y al modelo de pago por uso, el costo de operación es prácticamente nulo para volúmenes bajos y muy bajo para volúmenes altos.
*   **Resiliencia**: Si el scraping de una página de detalle falla, solo afecta a un mensaje en la cola, que puede ser reintentado automáticamente sin detener todo el proceso.
*   **Bajo Mantenimiento**: No hay servidores que gestionar, parchear o monitorizar.

## 🛠️ Componentes del Código

El repositorio está organizado en dos partes principales, correspondientes a cada función Lambda:

### 1. `lambda_orchestrator.py`
*   **Responsabilidad**: Scrapear las páginas de listado.
*   **Librerías Clave**: `httpx` para las peticiones HTTP, `BeautifulSoup4` para el parseo de HTML y `boto3` para enviar mensajes a SQS.

### 2. `lambda_worker.py`
*   **Responsabilidad**: Recibir tareas de SQS, scrapear páginas de detalle y guardar en DynamoDB.
*   **Librerías Clave**: `httpx`, `BeautifulSoup4`, y `boto3` para interactuar con DynamoDB. Incluye lógica para limpiar los datos y manejar tipos numéricos (`Decimal`) para la compatibilidad con DynamoDB.

## 🚀 Cómo Desplegar

Para desplegar esta solución en tu propia cuenta de AWS, sigue estos pasos:

1.  **Prerrequisitos**:
    *   Una cuenta de AWS.
    *   Python 3.9+ y `pip` instalados localmente.
    *   Tener configurada la [AWS CLI](https://aws.amazon.com/cli/) (opcional, pero recomendado).

2.  **Crear Recursos de AWS**:
    *   **DynamoDB**: Crea una tabla con una clave de partición de tipo String llamada `id`.
    *   **SQS**: Crea una cola estándar. Copia su URL.
    *   **IAM Role**: Crea un rol para Lambda con permisos para escribir en CloudWatch Logs (`AWSLambdaBasicExecutionRole`), acceso completo a SQS (`AmazonSQSFullAccess`) y a DynamoDB (`AmazonDynamoDBFullAccess`).
      *(Nota: En entornos restrictivos como AWS Academy, puede que necesites usar un rol pre-configurado como `LabRole`)*.

3.  **Empaquetar las Dependencias**:
    *   Para cada función (`orchestrator` y `worker`), crea una carpeta e instala las dependencias dentro de ella:
      ```bash
      # Ejemplo para el orquestador
      mkdir orchestrator_pkg
      cd orchestrator_pkg
      pip install httpx beautifulsoup4 boto3 -t .
      cp ../lambda_orchestrator.py ./lambda_function.py
      ```
    *   Comprime el contenido de cada carpeta en un archivo `.zip`, asegurándote de que `lambda_function.py` quede en la raíz del zip.

4.  **Desplegar las Funciones Lambda**:
    *   Crea dos funciones Lambda en la consola de AWS.
    *   Sube el `.zip` correspondiente a cada una.
    *   Asigna el rol de IAM creado.
    *   Configura las **variables de entorno**:
        *   Para `scraper-orchestrator`: `SQS_QUEUE_URL` con la URL de tu cola.
        *   Para `scraper-worker`: `DYNAMODB_TABLE_NAME` con el nombre de tu tabla.
    *   Ajusta los **tiempos de espera (timeout)** (ej. 5 min para el orquestador, 1 min para el worker).

5.  **Configurar los Disparadores (Triggers)**:
    *   Para `scraper-orchestrator`, añade un disparador de **EventBridge** con una regla de programación.
    *   Para `scraper-worker`, añade un disparador de **SQS** apuntando a tu cola.

¡Y listo! Ya puedes lanzar una prueba manual desde la Lambda del orquestador para verificar que todo el flujo funciona correctamente.
